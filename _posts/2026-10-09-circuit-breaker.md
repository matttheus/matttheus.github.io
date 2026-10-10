---
layout: post
title: "Circuit breaker"
description: "Resilient applications, fault tolerance and the circuit breaker pattern: what Martin Fowler and Netflix say about it, and what I built on top of that."
excerpt: "Resilient applications, fault tolerance and the circuit breaker pattern: what Martin Fowler and Netflix say about it, and what I built on top of that."
tags: [distributed-systems, resilience, nodejs]
---

## TL;DR
{:.no_toc}

A circuit breaker sits between your app and a dependency. After too many
failures it opens and fails fast instead of making everyone wait for a
timeout, and after a while it lets a test call through to check if the
dependency is back (closed → open → half-open). Martin Fowler explains the
state machine, and Netflix adds the rest: tripping on an error rate in a
time window, a bulkhead per dependency, timeouts and fallbacks. To really
exercise it, I built a project with NestJS and BullMQ that goes beyond the
articles, with a semaphore sized by Little's law, a background health check
for the half-open state, ignoring 4xx errors, and state shared through Redis.

## Contents
{:.no_toc}

* TOC
{:toc}

Every application depends on things it doesn't control: a database, a payment
API, another team's service. Sooner or later, one of them is going to fail. A
resilient application is one that keeps working when that happens, even if
in a degraded way, and recovers by itself when the dependency comes back. The
formal name for this property is fault tolerance: the system keeps doing
its job even when some of its parts fail.

The idea is older than software. In 1956, John von Neumann published a paper
on how to build reliable machines out of unreliable components. In 1967,
Algirdas Avižienis coined the term "fault-tolerant computing" while working on
computers for long space missions at NASA's Jet Propulsion Laboratory. In the
80s, Ericsson created Erlang to build telephone switches that were never
supposed to go down.

With distributed systems, the problem moved into the application. Every
network call is a point of failure, and a set of patterns appeared to deal with
it: timeouts, retries, bulkheads, fallbacks and the one this post is about, the
circuit breaker. Michael Nygard popularized it in his book *Release It!*
(2007), and Netflix took it to production at scale with Hystrix.

I wanted to understand this pattern beyond the definition, so I read the two
classic references on the subject:

- [Martin Fowler's article on circuit breakers](https://martinfowler.com/bliki/CircuitBreaker.html) (2014)
- [Fault Tolerance in a High Volume, Distributed System](https://netflixtechblog.com/fault-tolerance-in-a-high-volume-distributed-system-91ab4faae74a),
  from the Netflix Tech Blog (2012)

Then I built a project to really exercise it. In this post I'll go through
the main ideas of both articles, and then show what I did beyond them.

## The problem

When your service calls another service over the network, that call can fail
or, even worse, hang. Now imagine hundreds of requests per second calling a
service that hangs. Each one waits for a timeout, holding resources (threads,
connections, memory) the whole time. Soon your service runs out of resources
too, and the failure spreads to everyone that depends on you. This is called a
cascading failure.

A failure in one small dependency becomes an outage of your whole system.

## What Martin Fowler says

Fowler uses the analogy of the electrical circuit breaker in your house: when
something goes wrong, it cuts the circuit instead of letting the damage spread.
In software, you wrap the remote call in a circuit breaker object that watches
for failures. It has three states:

![The three states of a circuit breaker: CLOSED, OPEN and HALF-OPEN, and the transitions between them]({{ "/assets/img/posts/circuit-breaker/circuit-breaker-states.svg" | relative_url }})

- Closed: everything is normal, calls go through. Failures are counted.
- Open: after too many failures, the breaker trips. Calls fail
  immediately, without even trying the remote service.
- Half-open: after a timeout, the breaker lets a trial call through. If it
  works, the circuit closes again. If it fails, it opens again.

Other points from the article that I found important:

- Not every error should trip the breaker. Timeouts and connection failures
  are a sign the service is in trouble. Other errors are just normal failures
  that your regular logic should handle.
- Monitoring. "Any change in breaker state should be logged and breakers
  should reveal details of their state for deeper monitoring." Operators
  should also be able to trip or reset a breaker manually.
- You need a plan for when the circuit is open: queue the request for
  later, show cached data, or degrade the feature.
- It also works for asynchronous calls, for example by watching a queue
  pile up instead of individual failures.

## What Netflix says

The Netflix article is from 2012 and shows the problem at a scale most of us
will never see. The Netflix API received more than 1 billion calls per day,
which fanned out to several billion calls to dozens of other services, with
peaks of over 100,000 dependency requests per second.

The math in the article is what made it click for me. If you depend on 30
services and each one has 99.99% uptime, you get 0.9999³⁰ ≈ 99.7% uptime. That's
more than 2 hours of downtime per month, even though every single
dependency is "excellent". That's why the article's section title says: *fault tolerance
is a requirement, not a feature*.

Their solution, the `DependencyCommand` (which later became the open source
library Hystrix), wraps every call to a dependency and combines four
techniques:

1. Network timeouts and retries.
2. A separate thread pool for each dependency. If one dependency gets slow,
   it can only block the threads of its own pool (for example, 10 threads),
   not the threads that serve users. This is the bulkhead pattern, named
   after the walls that divide a ship's hull so one leak doesn't sink the
   whole ship.
3. Semaphores for calls that don't touch the network, where a thread would
   cost too much.
4. Circuit breakers that trip on an error rate, "such as 50% error rate
   in a 10 second period", and "reject all requests until health checks
   succeed".

And when something fails, every request goes through a fallback before
returning to the user. In order of how well they preserve the experience:

- Cache: return stale data instead of nothing.
- Eventual consistency: put writes in a queue to be done when the
  dependency comes back.
- Stubbed data: return default values.
- Fail silent: return an empty response that the UI can ignore.

## What I built

To exercise this concept, I built a project with NestJS and BullMQ:
[circuit-breaker](https://github.com/matttheus/circuit-breaker). It has two
dependencies simulated by mock endpoints, where I can change the error rate
and the latency at runtime:

- `payments`: fast but unstable.
- `notifications`: stable but slow (1.5 seconds per call).

The ideas from the articles map like this:

| Concept | Where it comes from | In my project |
| --- | --- | --- |
| Closed / open / half-open | Fowler | the same state machine |
| Trip on error rate in a time window | Netflix | 50% of errors, at least 10 calls, in a 10-second sliding window |
| Bulkhead per dependency | Netflix | one queue per dependency in the async path, a semaphore per dependency in the sync path |
| Fail fast | both | `503` with a `Retry-After` header, in less than 1 ms |
| Queue writes for later | both | the async path: jobs wait in the queue while the circuit is open |
| Not every error counts | Fowler | `4xx` responses are ignored by the circuit |
| Log and expose the state | Fowler | every transition is logged, and there's an endpoint to see and reset each circuit |

## What I did beyond the articles

This is the part where I learned the most, because each of these came from a
question or a bug I found along the way.

### Threads don't make sense in Node.js, but the problem still exists

Netflix uses a thread pool per dependency because Java (at least back then)
blocked one thread per request. Node.js has a single thread and non-blocking
I/O, so a slow call doesn't block anything. But it still holds a socket and
memory, and with no limit these pile up until the process dies.

So the right tool in Node is the semaphore: a limit of concurrent calls per
dependency. Instead of picking a random number, I size it with
Little's law: `slots = peak requests per second × p99 latency + headroom`.
For `payments` that's 100 req/s × 0.1 s + 25% = 13 slots.

### The semaphore goes outside the circuit breaker

Hystrix counts a full thread pool as a failure. I tested it and found a
problem: during a traffic spike to a *healthy* dependency, the rejections
opened the circuit, and for 10 seconds every request failed. My own
traffic took down a dependency that was working fine.

So I moved the semaphore outside the breaker: being full is not a failure. If
the dependency is really slow, the calls hit the timeout, and timeouts *do*
count. In the load test, a spike to 300 req/s rejected only the extra
requests (in 0.37 ms), and the circuit never opened.

### The half-open test shouldn't be a real user

In the classic version, the first call in half-open is the test. In a
synchronous API, that call is a real user's request, and if the dependency is
still down, that user gets the error.

My solution: while half-open, users get a fast `503`, and a background
health check decides whether the circuit closes. Later I noticed this is
exactly what the Netflix article says: "reject all requests until health
checks succeed".

### Ignoring 4xx is not the same as counting it as success

My first version counted `4xx` responses as successes, since the dependency
did answer. The load test showed the bug: with 80% of requests being invalid,
the error rate never reached 50%, and the circuit never opened, even with the
dependency completely down. A client sending bad requests could hide an
outage.

The fix: `4xx` responses are ignored. They don't count as failures or as
successes.

### Async and sync are different worlds

To exercise different scenarios, I built two setups in the project, both
calling the same dependencies:

- Async: the API puts a job in a queue (one queue per dependency) and
  answers right away. Later, a BullMQ worker picks up the job and calls the
  dependency through the circuit breaker. Nobody is waiting for the result, so
  this fits work that doesn't need an immediate answer, like sending a
  notification.
- Sync: the dependency is called inside the HTTP request, and the client
  waits for the answer. The call goes through the semaphore and then the
  circuit breaker, and the response has to go out right away, with the result
  or with an error.

The same circuit breaker behaves very differently in each one.

In the async setup, an open circuit doesn't need to fail anything: the job just
waits. I postpone it until the circuit can be tested again, without consuming
a retry attempt. But postponing alone isn't enough. It needs two more things:

- Jitter: without a random delay, all postponed jobs wake up at the same
  instant, only one becomes the test call, and the rest get rejected together.
- Maximum age: if the dependency is down for hours, a job fails after a
  configurable time (10 minutes in my project) instead of waiting forever.
  Whether a job is still worth running hours later is a business decision: a
  monthly subscription charge can wait, but charging for an order the customer
  already gave up on probably can't. The point is to make that decision
  explicit, instead of letting the queue decide by accident.

In the sync setup there's no queue, so there's nowhere to keep the work for
later: it's fail fast with a `Retry-After` header, no retries inside the
request (they only multiply the load on a dependency that is already
struggling).

### Where the state lives: memory or Redis

With several instances, each one keeps its own circuit in memory, so each one
needs to see the failures by itself. I made the storage configurable: in
memory, or in Redis with a Lua script, so that reading the state and
changing it is one atomic operation shared by all workers.

For synchronous requests I use memory: two extra Redis round trips on every
request is a cost, and if Redis goes down, the whole API shouldn't go down with
it. But this doesn't work in ephemeral environments like AWS Lambda: each
instance handles one request at a time and starts from zero, so the circuit
never sees enough failures to open. There you need external storage.

### Load tests with k6

I wrote six load test scenarios with [k6](https://k6.io) and ran them against
the app in a Docker container limited to 1 CPU and 512 MB, to simulate a small
production environment. Some results:

| Scenario | Result |
| --- | --- |
| Healthy dependency | ~2.4 ms of overhead per call |
| Dependency goes down | the circuit opened in 5 s, and failures dropped from 102 ms to less than 1 ms |
| Dependency gets slow (3 s) | nobody waited more than 2 s (the timeout) |
| Traffic spike to 300 req/s | only the extra requests were rejected, circuit stayed closed |
| `payments` down the whole time | `notifications` kept working normally |

The charts below are read the same way, each bar is one second of the test,
split by the response the user got. The strip on top shows the state of the
circuit, and the gray area is when I broke the dependency on purpose.

#### When the dependency goes down

![Responses per second during a 20-second outage of the dependency. Before the outage all responses are 200. For the first 5 seconds of the outage they are 502 errors that take 100 ms. Then the circuit opens and every response is a 503 that takes about 1 ms. The circuit closes 8 seconds after the dependency recovers.]({{ "/assets/img/posts/circuit-breaker/load-test-outage.svg" | relative_url }})

Read it from left to right:

- Before 15 s, everything is blue: requests go to the dependency and come
  back in ~100 ms.
- From 15 s to 20 s, the bars turn orange. The dependency is down, but we
  keep calling it and each failure still takes 100 ms. The circuit takes
  5 seconds to open because the 10-second window still holds the successes
  from before the outage: it needs as many failures as successes to reach 50%.
  A smaller window would react faster, but it would also trip on small hiccups.
- From 20 s on, the bars turn green: the circuit is open. The user still
  gets an error, but look at the bottom chart: it comes back in less than
  1 ms instead of 100 ms, and the dependency stops receiving calls, which gives
  it room to recover.
- At 35 s the dependency is back, but the circuit only closes at 43 s,
  after the background health checks pass. Those 8 seconds are the price
  of the 10-second open timeout: we wait a little longer to make sure we don't
  hit a dependency that's still unstable.

#### When the dependency gets slow

![Responses per second while the dependency takes 3 seconds to answer. Most requests get an immediate 503 because the 13 slots are busy, and every 2 seconds a group of 504 timeouts appears. The slowest response in each second never goes above 2 seconds. The circuit opens 11 seconds after the slowdown starts.]({{ "/assets/img/posts/circuit-breaker/load-test-slow-dependency.svg" | relative_url }})

Here the dependency doesn't fail, it just takes 3 seconds to answer, which is
worse, because every call holds a slot for a long time:

- The first 13 requests take the 13 slots of the semaphore and get stuck. All
  the others get an immediate 503 (green), because there's no free slot.
- Every 2 seconds the stuck requests hit the timeout and return 504
  (orange). That's the zigzag in the bottom chart: the slowest response
  never goes above 2 seconds, even with a dependency that takes 3.
- The circuit takes 11 seconds to open, longer than in the outage, because
  only the timeouts count as failures. The semaphore rejections don't.

There's a lesson here that the chart doesn't show: with 13 slots, the
dependency had 26 calls at the same time. A timeout makes *us* give up on
the request, but it doesn't cancel the work on the other side. The dependency
kept processing calls nobody was waiting for anymore.

#### When the traffic spikes

![Responses per second while traffic ramps from 20 to 300 requests per second to a healthy dependency. Served requests stop growing at about 130 per second, the capacity of the semaphore, and everything above that line is rejected with a 503 in under 1 ms. The circuit stays closed the whole time.]({{ "/assets/img/posts/circuit-breaker/load-test-spike.svg" | relative_url }})

This time the dependency is healthy, and it's *our* traffic that goes from 20
to 300 requests per second:

- The blue part stops growing at ~130 requests per second. That's the
  capacity of the semaphore: 13 slots, each handling a 100 ms call, is about
  130 calls per second.
- Everything above that line is green: rejected in 0.37 ms, while the
  requests that got a slot kept their normal 102 ms.
- The strip on top stays CLOSED the whole time. This is the chart that
  justifies putting the semaphore outside the circuit breaker: if the
  rejections counted as failures, this spike would have opened the circuit and
  every request would have failed for 10 seconds.

## Don't build your own (unless it's to learn)

I built my own circuit breaker to understand it, but in a real project I'd use
a library first. Hystrix itself has been in maintenance mode since 2018. In
Java, the usual choice today is [resilience4j](https://resilience4j.readme.io).
In Node.js, these are the two I'd look at:

- [opossum](https://github.com/nodeshift/opossum): the most popular circuit
  breaker for Node.js, maintained by the Red Hat team. It has timeouts,
  fallbacks, an error-rate window, the half-open state, events for monitoring,
  and a `capacity` option that works as a semaphore.
- [cockatiel](https://github.com/connor4312/cockatiel): the closest thing to
  Hystrix and resilience4j in Node.js. Besides the circuit breaker, it has
  bulkhead, timeout, retry and fallback policies that you combine with
  `wrap()`. It's written in TypeScript and supports `AbortSignal`.

## What I took from this

- Solving a problem intuitively is a good sign, but knowing its name opens the
  door to years of other people's experience. In a couple of articles I found
  edge cases I would have taken months to hit on my own.
- The basic pattern is simple. The hard part is the details: which errors
  count, where the state lives, who makes the test call, what happens during a
  spike.
- Measure. Half of the lessons above came from load tests.

Thanks for reading, see ya!
