---
layout: post
title:  "Use retry judiciously"
date:   2020-03-21 22:21:23 +0700
categories: [resilience]
---

*Failure is simply the opportunity to begin again, only this time more wisely.* 
    **-Henry Ford**

## When should I use retry?
- use retry with care, in a sparse manner
- retry policy usually depends on business requirements and nature of the failure
- idempotent actions are potentially appropriate for retry
- transient failures ( momentary network loss or temporal service unavailability)
- it is likely to succeed a failed action when repeated
- for extremely unusual failures (like network packet corruption)
- retry should not be used as a solution to scalability issues

## What should I be causious about?
- If you continually overwhelm a service with retry requests, it will take the service longer to recover
- you should log the details of failure and your retry execution
- for transactions be careful with consistency and the tradeoff between retry and rollback costs
- be cautious about the latency and throughput constraints of your application when implementing a retry policy
- retry only when the full context is well understood. better make lower level tasks fail fast (cascading retries)
- be specific about the situations that you want to retry (white/black list mechanism)
- as a rule of thumb for non critical applications it is better to failfast rather than risking the throughput and response time
- batch processing is a better place to use retry compared to real-time services
- you can adjust the delay time based on the nature of the failure (long/short term)
- it is a good practice to retry behind a circuit breaker for better failure management
- instead of retrying the same action you can either switch to an alternative action or be satisfied with a degraded functionality (reduce consistency level)
- retry storm : self-inflicted denial-of-service attack
- retry budget: the ratio between regular requests and retries
- retry policy tradeoff is between improving success rate and adding extra load to system


**Reference:**

* [resilience4j](https://resilience4j.readme.io/docs/retry)
* [Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/retry)