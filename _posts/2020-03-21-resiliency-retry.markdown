---
layout: post
title:  "Resiliency Patterns [Part 1]: Use Retry Judiciously"
date:   2020-03-21 22:21:23 +0700
categories: [resiliency, advanced]
---

*Failure is simply the opportunity to begin again, only this time more wisely.* 
    **-Henry Ford**

# Be aware of the business requirements

**- Is this functionality critical for your business?**
If fulfiling a request is not critical for your business or some type of failover functionality such as hitting a cache or service degradation(fail-over) is enough, you may completely ignore the retrying and opt for those alternatives or event let it crash! As a rule of thumb for non critical applications it is better to failfast rather than risking the throughput and response time.
You also have to ask yourself is it likely to succeed a failed action when repeated?
Retry should not be used as a solution to scalability issues

**- Consider your latency budget:**
Retries usually are better suited for usecases that does not have any constraint on response time, for example, offline batch processing can be a good context for retries, in fact using retries properly in such systems can save you a lot of time because failing in any step of your batch jobs can cause the whole pipeline to halt and may require a manual fix.
So it is important to consider your latency and throughput constraints and make proper calculations on how much time you can 'waste' on retrying. As a mater of fact, latency is not the only issue, retrying can impose extra load on your system and you should make tradeoffs between higher success rate and the amount of resources used for retrying. 
Another concept you have to be aware of is `cascading retries`. what happens if other downstream or upstream services use retry too? You have to consider that in your timeout and backoff strategies. 
These are some of the reasons execution context is important.


# Be aware of the operational characteristics
**- Is this operation idempotent?**
A function is `idempotent` if it can be applied multiple times without changing the result of the initial application.
Retries are mostly applicable to idempotent operations. (CRDT)

# Be aware of the nature of the failure

**- Is this a long-term or a short-term failure?**
Retries are effective if your failure is transient and there is a high chance of success in your next retries. The most related transient failures are momentary network partitioning and temporal service unavailability. In these two cases retry can help you recover from a short-term failure.
Retries can also be effective for extremely unusual failures (like network packet corruption).

## What should I be cautious about?
- use timeouts (too short timeout causes premature failure)
- If you continually overwhelm a service with retry requests, it will take the service longer to recover
- you should log the details of failure and your retry execution
- for transactions be careful with consistency and the tradeoff between retry and rollback costs
- be specific about the situations that you want to retry (white/black list mechanism)
- it is a good practice to retry behind a circuit breaker for better failure management
- retry storm : self-inflicted denial-of-service attack
- retry budget: the ratio between regular requests and retries
- retry with async can cause problems in ordering of messages in message queues

## Case Study 1 - AWS Lambda Functions
Using lambda functions, many issues can result retires and duplicated requests, to prepare for these occurrences your function must always be `idempotent`.

When you invoke a lambda function (either directly or indirectly) two types of errors can occur:

**1. Invocation Errors:** when the invocation request is rejected before your function receives it.

**2. Function Errors:** when your function's code or runtime returns an error.

Depending on the type of error (`Invocation Error` | `Function Error`), the type of invocation (`Sync` | `Async` | `Event Source Mappings`), and the client or service that invokes the function, the retry behavior and the strategy for managing errors varies.
For instance, AWS CLI by default retrys on client timeouts, throttling errors (429), and 5XX errors.

For indirect function calls the situation becomes more complex:

**1. Async Invocation:** Lambda retries function errors twice by default, but fortunately you can configure the retry behavior:
The following example configures a function with a maximum event age of 1 hour and no retries.
{% highlight bash %}
$ aws lambda put-function-event-invoke-config --function-name myfunction \
--maximum-event-age-in-seconds 3600 --maximum-retry-attempts 0
{% endhighlight %}

**2. Event Source Mapping:** In this case the entire batch will be retried until the error is resolved or the items expire (may take up to 6hrs).

**3. AWS Services:** if your functions are synchronously invoked by other services, the service decides whether to retry. And in asynchronous invocations it is the same as 1.

To make the situation worse, due to queues being eventually consistent, you may receive an event multiple times, so you should always insure that your functions can handle duplicates.

## Case Study 2 - Cassandra RetryPolicy
Cassandra is definitely my favorite when it comes to implementing retry policies! The reason is that you have to specify (either explicitly or implicitly) the idempotency of any operation you perform. The byproduct is that you can define your retry policies once based on the idempotency of an action without knowing about the action itself.
You can create statements yourself or use a QueryBuilder to build one for you:

`Statements` start out as non-idempotent by default (which you can change but I dont recommend), but you can override the flag on each statement.

{% highlight java %}
Statement s = new SimpleStatement("SELECT * FROM users WHERE id = 1");
s.setIdempotent(true);
{% endhighlight %}

`QueryBuilder` DSL also tries to infer the idempotency of your statement in a conservative way but you can set idempotency on that too.

{% highlight java %}
BuiltStatement s = update("mytable").with(set("v", fcall("anIdempotentFunc"))).where(eq("k", 1));
s.setIdempotent(true);
{% endhighlight %}

[`RetryPolicy`](https://docs.datastax.com/en/drivers/java/3.6/com/datastax/driver/core/policies/RetryPolicy.html) is a pluggable component that is configured when initializing the cluster. If you don't explicitly configure it, you get a `DefaultRetryPolicy` which is conservative in that it will never retry with a different consistency level (`service degradation`) than the one of the initial operation.

{% highlight java %}
Cluster cluster = Cluster.builder()
        ...
        .withRetryPolicy(new MyCustomPolicy())
        .build();
RetryPolicy policy = cluster.getConfiguration().getPolicies().getRetryPolicy();
{% endhighlight %}

The policy's methods cover different types of errors including `OnUnavailable`, `onReadTimeout`, `onWriteTimeout`, and `onRequestError` which you should implement considering the idempotency of the statement given as input parameter and returning a [`RetryDecision`](https://docs.datastax.com/en/drivers/java/3.6/com/datastax/driver/core/policies/RetryPolicy.RetryDecision.html) with three possible decisions which are RETHROW, RETRY and IGNORE.

There are a few cases where retrying is always the right thing to do. These are hard-coded in the driver itself. For example, if any error occures before writing to network it is always safe to retry since the request is not sent yet; or when the driver executes a prepared statement that the coordinator doesn't know about it, and needs to re-prepare it on the fly. On the other hand some errors have no chance of being solved by retry like `QueryValidationException` or `TruncateException`.

**Reference:**

* [Resilience4j Library](https://resilience4j.readme.io/docs/retry)
* [Microsoft Retry Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/retry)