# RateGate

RateGate is a distributed API rate-limiting gateway designed to validate
API keys, enforce per-client request plans, route allowed traffic to
healthy backend services, and expose operational metrics through an
analytics dashboard.

This README documents the high-level design shown in
`rategate_hld.drawio.png`.

## 1. Overview

A client sends an HTTP request along with an API key. The request passes
through a gateway instance, where RateGate:

1.  Validates the API key.
2.  Retrieves the client's rate-limit policy.
3.  Atomically checks and consumes a token from the client's token
    bucket.
4.  Routes allowed requests to a healthy backend.
5.  Applies circuit-breaker and retry rules when a backend is unhealthy.
6.  Returns the backend response with rate-limit information.
7.  Publishes metrics asynchronously so analytics work does not slow
    down the request path.

Requests with invalid credentials or exhausted rate limits are rejected
before reaching a backend.

## 2. Architecture at a glance

The design contains four main areas:

  -----------------------------------------------------------------------
  Area                    Responsibility          Owner shown in design
  ----------------------- ----------------------- -----------------------
  Auth and Config Service API-key validation and  Member 3
                          client-policy lookup    

  Rate Limiter Service    Distributed             Member 1
                          token-bucket            
                          enforcement             

  Router and Circuit      Backend selection,      Member 2
  Breaker                 health-aware routing,   
                          retries and failover    

  Analytics, Alerts and   Metrics processing,     Member 3
  Dashboard               alert rules and live    
                          operational view        
  -----------------------------------------------------------------------

The gateway cluster contains stateless instances behind a load balancer.
Shared configuration and token state are stored outside the gateway
instances so that any instance can process a client's request
consistently.

### Main infrastructure components

-   **Clients:** Web, mobile and partner applications that send requests
    with an API key.
-   **Load balancer:** Distributes traffic across gateway instances,
    performs traffic/health checks and removes unhealthy gateway
    instances.
-   **Gateway instances:** Run the request path. The design allows more
    stateless instances to be added as traffic grows.
-   **Config DB:** Stores API keys and client policies, managed by
    Admin/Ops.
-   **Redis (primary + replica):** Stores shared token-bucket counters
    used by all gateway instances.
-   **Backend services:** Multiple services, each with multiple server
    instances.
-   **Message queue:** Receives request events asynchronously.
-   **Metrics store:** Persists metrics over time for dashboards and
    alert evaluation.
-   **Admin/Ops:** Manages keys and limits and views the dashboard.

## 3. Request lifecycle

### Step 1 --- Client request and gateway selection

A client sends a request and API key. The load balancer selects a
gateway instance. Gateway instances are stateless, so a request can be
handled by any healthy instance.

### Step 2 --- API-key validation

The Auth and Config Service:

1.  Reads the API key/token from the request header.
2.  Checks the key locally first, where applicable.
3.  Checks the Config DB when required.
4.  Loads the client's policy, including rate, burst size and allowed
    routes.
5.  Stops the request if the key is invalid, blocked or unauthorized.

Invalid or blocked keys return **401 or 403**. The dashed red path in
the diagram represents this rejection response returning to the client.

### Step 3 --- Client policy is passed forward

For a valid key, the gateway passes the client ID and policy to the Rate
Limiter Service. The policy determines the applicable request limit and
burst capacity.

### Step 4 --- Token-bucket rate limiting

The Rate Limiter Service builds a counter name using the client ID and
route. It then runs **one atomic Redis script** to check and update the
token bucket.

The token-bucket operation:

1.  Reads the current token count and last-refill time.
2.  Refills tokens according to the configured refill rate and elapsed
    time.
3.  Does not allow the token count to exceed the configured burst
    capacity.
4.  Checks whether at least one token is available.
5.  Consumes one token atomically when the request is allowed.

The atomic operation is important because multiple gateway instances may
receive requests for the same client concurrently. Without an atomic
update, concurrent requests could all observe the same available token
and exceed the limit.

-   **Token available:** The request is allowed to continue to routing.
-   **No token available:** The gateway returns **429 Too Many
    Requests** and includes a retry-after indication.

The diagram shows token counters shared through Redis primary and
replica infrastructure. The exact Redis replication and failover
configuration is an implementation detail to define during deployment.

### Step 5 --- Backend routing

After rate-limit approval, the Router and Circuit Breaker service:

1.  Matches the request path to a backend service.
2.  Selects a server using round-robin among healthy servers.
3.  Checks the circuit-breaker state.
4.  Applies a timeout to the backend request.
5.  Retries once on another server when the configured retry conditions
    allow it.
6.  Returns the backend response to the client, including rate-limit
    headers.

### Step 6 --- Circuit-breaker behavior

The circuit breaker protects the gateway from repeatedly sending traffic
to failing backend services.

The design identifies three states:

-   **Closed:** Requests can be sent to the backend.
-   **Open:** Requests are not sent to the failing backend.
-   **Half-Open:** The service can be tested to determine whether it has
    recovered.

Each success or failure from a backend server contributes to the
circuit-breaker health decision, as indicated in the diagram. The
precise failure threshold, open duration and half-open probe policy
should be configured as implementation parameters.

If a backend is unavailable and no eligible retry/fallback can complete
the request, the gateway returns **503 Service Unavailable**.

### Step 7 --- Asynchronous metrics and events

After the response is handled, the Metrics and Event Emitter publishes
request events to the message queue. This is an asynchronous,
non-blocking path.

The Analytics Service consumes those events and tracks:

-   Request counts
-   429 responses
-   Errors
-   Latency

The Analytics Service saves metrics to the Metrics Store. Alert rules
evaluate conditions such as many 429 responses or an open circuit
breaker. Admin/Ops can view the Live Dashboard, which reads the stored
metrics.

Keeping analytics off the synchronous request path helps prevent
dashboard processing or alert evaluation from increasing request
latency.

## 4. Functional requirements

The diagram lists the following functional requirements:

  -----------------------------------------------------------------------
  ID                      Requirement             Owner
  ----------------------- ----------------------- -----------------------
  FR1                     Issue and validate API  Member 3
                          keys                    

  FR2                     Plan per client: Free   Member 3
                          10/min and Pro 100/min  

  FR3                     Token-bucket            Member 1
                          throttling, 429 with    
                          Retry-After             

  FR4                     X-RateLimit-Remaining   Member 1
                          header on success       

  FR5                     Round-robin routing to  Member 2
                          healthy backends        

  FR6                     Regular health checks;  Member 2
                          skip unhealthy backends 

  FR7                     Circuit breaker, 503    Member 2
                          when no backend is left 

  FR8                     Live dashboard: allowed Member 3
                          and rejected per        
                          client, backend health  
  -----------------------------------------------------------------------

## 5. Non-functional requirements

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  NFR1 --- Latency                    Less than 10 ms added at p95;
                                      approximately 4 ms is estimated in
                                      the design.

  NFR2 --- Availability               99.9%, with two gateway instances
                                      and a Redis replica.

  NFR3 --- Scalability                Add capacity by adding instances;
                                      the gateway is stateless.

  NFR4 --- Concurrency accuracy       Use an atomic Lua script so no
                                      request slips past the rate limit
                                      due to concurrent updates.

  NFR5 --- Fault tolerance            Fail-open limiter behavior, one
                                      retry and 503 fallback are listed
                                      in the design.

  NFR6 --- Observability              Allowed, rejected and
                                      backend-health information is
                                      visible live.
  -----------------------------------------------------------------------

**Target load shown in the design:** 100 requests/second average and 500
requests/second peak.

> **Design clarification:** "Fail-open limiter" is stated in the
> diagram, while the token-bucket flow also describes rejecting requests
> when no token is available. The exact behavior when Redis itself is
> unavailable should be explicitly decided and documented before
> implementation. A fail-open policy generally allows traffic when the
> limiter cannot be consulted; a fail-closed policy rejects it.

## 6. API behavior and response outcomes

The high-level design specifies these outcomes:

  -----------------------------------------------------------------------
  Condition                           Expected outcome
  ----------------------------------- -----------------------------------
  Invalid API key                     `401 Unauthorized`

  Blocked or unauthorized key         `403 Forbidden`

  Rate limit exhausted                `429 Too Many Requests`, with
                                      `Retry-After`

  Request allowed by limiter          Continue to backend routing

  No usable backend / circuit-breaker `503 Service Unavailable`
  fallback                            

  Successful request                  Return backend response with
                                      rate-limit headers, including
                                      `X-RateLimit-Remaining`
  -----------------------------------------------------------------------

The diagram does not define exact response-body schemas, header formats,
or the full set of status-code mapping rules. Those should be specified
in the API contract.

## 7. Data and state ownership

  ------------------------------------------------------------------------
  Data                     Storage / component     Purpose
  ------------------------ ----------------------- -----------------------
  API keys                 Config DB               Validate client
                                                   credentials

  Client policies          Config DB, with cached  Store plan limits,
                           policy reads            burst size and allowed
                                                   routes

  Token-bucket counters    Redis                   Share rate-limit state
                                                   across gateway
                                                   instances

  Request and health       Message queue           Buffer events away from
  events                                           the request path

  Aggregated/time-series   Metrics Store           Power dashboard and
  metrics                                          alert evaluation
  ------------------------------------------------------------------------

The gateway instances should not rely on local in-memory counters for
authoritative rate limiting. Shared Redis state is what allows requests
handled by different instances to observe a common token bucket.

## 8. Scalability, availability and resilience

-   **Horizontal gateway scaling:** Add stateless gateway instances
    behind the load balancer.
-   **Shared rate-limit state:** Keep token counters in Redis so all
    gateway instances enforce the same client limit.
-   **Redis redundancy:** The diagram includes a primary and replica.
-   **Backend health awareness:** Avoid routing requests to unhealthy
    servers.
-   **Circuit breaking:** Stop repeatedly calling a backend that is
    failing.
-   **Bounded retry:** Retry once on another server, as shown in the
    request flow.
-   **Asynchronous observability:** Publish events to a queue and
    process them outside the request path.

Operational details still to define include health-check intervals,
circuit-breaker thresholds, retry eligibility, Redis failover behavior,
queue delivery guarantees and retention policies.

## 9. Ownership / work split

Based on the member labels in the diagram:

-   **Member 1 --- Rate Limiter Service:** Token-bucket logic, atomic
    Redis update, 429 response and rate-limit headers.
-   **Member 2 --- Router and Circuit Breaker:** Path matching,
    round-robin server selection, health checks, circuit-breaker states,
    timeout, retry and 503 fallback.
-   **Member 3 --- Auth/Config and Analytics:** API-key validation,
    client policy and plan configuration, plus analytics, alerts and
    dashboard.

## 10. Suggested end-to-end validation scenarios

These scenarios follow directly from the architecture and requirements
and can be used as a starting point for testing:

1.  A valid client key is accepted and its policy is loaded.
2.  An invalid key is rejected with 401.
3.  A blocked key is rejected with 403.
4.  A client within its token allowance reaches the backend.
5.  A client that exhausts its tokens receives 429 and `Retry-After`.
6.  A successful response includes `X-RateLimit-Remaining`.
7.  Concurrent requests across multiple gateway instances do not exceed
    the shared token-bucket limit.
8.  Round-robin routing distributes requests among healthy backend
    servers.
9.  An unhealthy server is skipped.
10. A backend failure triggers the configured retry on another server.
11. An open circuit prevents calls to the affected backend.
12. When no backend is available, the gateway returns 503.
13. Request and error metrics appear in the dashboard through the
    asynchronous event pipeline.
14. Alert rules react to high 429 counts and an open circuit breaker.

## 11. Diagram

The source high-level design diagram is:

`rategate_hld.drawio.png`

It illustrates the service boundaries, request flow, shared stores,
response paths and asynchronous analytics pipeline.

------------------------------------------------------------------------

## Notes

This README is based on the supplied high-level design diagram. It
describes the architecture and requirements shown there; it does not
assume a particular programming language, framework, deployment
platform, database schema, endpoint contract or infrastructure-as-code
setup. Those details should be added once they are finalized.
