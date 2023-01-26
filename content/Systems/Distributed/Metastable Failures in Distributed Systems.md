[Paper](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf)
![[metastable-failures-1.png]]

# Abstract & Intro

- Metastable failures: A blackswan event, outliers, no foreshocks, severe impact, much easier to explain in hindsight than to predict.
- Paradoxically, the root cause of the failures is often the result of an effort to improve the efficiency or reliability of the system.
- Precipitated by uncontrolled source of load where a trigger causes the system to enter a bad state that persists even when the trigger is removed.
- Goodput becomes unusably low
- Sustaining effect → often involving work amplification or decreased overall efficiency
- Differs from DOS, Limplock, Livelock as those recover when trigger is resolved
- Transition into the vulnerable state largely goes undetected, and one can continue to exist within this state for long periods of time without ever reaching meta instability.
- In other instances, the vulnerable state is chosen because it is significantly more efficient than the healthy state.
- Errors in the unstable state compound. Where the trigger is often blamed, the true cause of severe issues is the sustaining effect.
- Metastable failures have an outsized impact on hyperscale distributed systems, as the contagious nature allows them to spread across the entire system
- It is increasingly difficult to test these types of failures, as the trigger can often only incite the avalanche at sufficient scale and distribution

# Common Sources

Sustaining effect is almost always associated with exhaustion of some resource

### Request Retries

- Results in work amplification, as increasing failure rates cause clients to retry, put additional load on the system, further exacerbating the resource overload
- This is often the cause when performance degrades super-linearly beyond the trigger point, or beyond a cliff point (often some arbitrary timeout set by the caller)

### Look-aside Cache

- Similar to request retries, invalidation or failure of a cache could result in a flood of queries to a database. If the cache is populated by the client, timeouts from the query could result in the cache never being repopulated.

### Slow Error Handling

- If handling the error case takes significantly more processing than the happy path, this will only increase the pressure on the system as the failure rates rise.

### Link Imbalance/HotSpots

- Arises when short term favoring of one resource brings about a failure state that leads to further preference to the failed resources

# Approaches to Handling Metastability

- Address the sustaining effects, not the trigger
    - Many different triggers can result in the same instability, and are often impossible to predict or prevent.
- Change the Policy during the Overload
    - Try to keep goodput high (to prevent the cascading of failures)
    - Change Policies
        - Routing logic
        - Disable or delay failovers and retries
        - Switch to non deterministic queueing
        - Reduce internal queue sixes
        - Enforce priorities
        - Shed load by rejecting a fraction of requests immediately
        - Use the circuit breaker patter to block all requests
        - Prioritize new queries over retries
- Fast Error Paths
    - Throttle error processing when error counts/rates increase beyond some safe bound
- Autoscaling
    - While not a panacea, autoscaling to leaf some buffer reduces the vulnerability to most triggers
- Minimize Feedback loops & Work Amplification
    - Avoid letting errors manifest more errors, and avoid extra work in the atypical case