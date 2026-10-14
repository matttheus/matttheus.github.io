---
layout: post
title: "Concurrency vs parallelism"
description: "Concurrency is dealing with many things at once, parallelism is doing many things at once. I built the same endpoints in Express and FastAPI and load-tested them to see the difference in numbers."
excerpt: "Concurrency is dealing with many things at once, parallelism is doing many things at once. I built the same endpoints in Express and FastAPI and load-tested them to see the difference in numbers."
tags: [concurrency, nodejs, python, performance]
---

## TL;DR
{:.no_toc}

Concurrency is *dealing* with many things at once: one worker switches
between tasks whenever a task is waiting. Parallelism is *doing* many things at
once: several CPU cores run code at the same instant. Waiting (I/O) only needs
concurrency. Heavy computation needs parallelism. To see this in numbers
instead of definitions, I built the same endpoints in Express and FastAPI, one
that waits for a second and one that computes Fibonacci for about a second,
and load-tested them with k6 in containers capped at 4 CPUs. The I/O endpoints
serve 8 users at 8 req/s on a single thread. The CPU endpoints on the event
loop drop to about 1.1 req/s in both runtimes, Python threads don't help
because of the GIL, and worker threads in Node or processes in Python take
both to about 3.3 req/s.

## Contents
{:.no_toc}

* TOC
{:toc}

Concurrency and parallelism are two words that get used as if they were
synonyms, and they're not. The confusion is old enough that Rob Pike gave a
whole talk about it in 2012,
[Concurrency is not parallelism](https://go.dev/blog/waza-talk). His one-line
version: concurrency is about *dealing* with lots of things at once,
parallelism is about *doing* lots of things at once.

I knew the definitions, but I wanted to see the difference in numbers: the
same work, the same number of users, and how the latency and the throughput
change depending on whether the server is concurrent, parallel, both or
neither. So I built a small project with two servers, Express and FastAPI, and
a load test:
[concurrency-vs-parallelism](https://github.com/matttheus/concurrency-vs-parallelism).

In this post I'll go through the two definitions, what I built, what the
numbers showed, and the things I only understood after looking at them.

## The two definitions

![Concurrency: one cook switching between four boiling pots. Parallelism: four cooks chopping at four boards, one per CPU core.]({{ "/assets/img/posts/concurrency-vs-parallelism/concurrency-vs-parallelism.svg" | relative_url }})

- Concurrency: one cook, four pots. The cook can't stir all of them at the
  same instant, but a pot that is boiling doesn't need the cook. So the cook
  moves to the next pot whenever the current one is waiting. One worker,
  many tasks in progress.
- Parallelism: four cooks, four boards, chopping at the same time. The work
  really happens at the same instant, because there are four pairs of hands.
  In a computer, the hands are CPU cores.

The rule that falls out of this: waiting only needs concurrency. A request
that waits on a database, the network or a timer holds no CPU while it waits,
so a single thread can have hundreds of them in flight. Computation needs
parallelism. A request that spends a second computing holds a core for a
second, and no amount of task switching will make two of them finish faster
on one core.

## What I built

Two servers, as small as I could make them:

- Express, running on Node.js.
- FastAPI, running on Uvicorn.

Both have an `/io` endpoint that waits for one second, and a `/cpu` endpoint
that computes Fibonacci recursively for about one second (`fib(40)` in Node,
`fib(35)` in Python, since Python is slower at this). Then each one has the
variations that its runtime offers to escape the single thread:

| Endpoint | Node | Python |
| --- | --- | --- |
| `/io` | `await sleep(1000)` | `await asyncio.sleep(1)` in `async def` |
| `/cpu` | `fib()` on the main thread | `fib()` inside `async def` |
| `/cpu-thread` | | `fib()` inside a plain `def`, which FastAPI runs on a threadpool |
| `/cpu-parallel` | `fib()` in a `Worker` thread | `fib()` in a `ProcessPoolExecutor` |

Every response says who handled it: the process id and the thread id. That's
how you confirm that `/cpu-parallel` really ran on different threads or
processes, instead of trusting the latency alone.

The Node server is this:

```js
app.get('/io', async (req, res) => {
  await sleep(1000);
  res.json(whoAmI());
});

app.get('/cpu', (req, res) => {
  fib(N);
  res.json(whoAmI());
});

app.get('/cpu-parallel', (req, res, next) => {
  new Worker(require.resolve('./fib'), { workerData: N })
    .once('message', (result) => res.json(result))
    .once('error', next);
});
```

And the Python one:

```python
@app.get("/io")
async def io():
    await asyncio.sleep(1)
    return who_am_i()

@app.get("/cpu")
async def cpu():
    return cpu_work()

@app.get("/cpu-thread")
def cpu_thread():
    return cpu_work()

@app.get("/cpu-parallel")
async def cpu_parallel():
    return await asyncio.get_running_loop().run_in_executor(pool, cpu_work)
```

The only difference between `/cpu` and `/cpu-thread` in Python is the word
`async`. Keep that in mind for later.

## The quick check: 4 requests at the same time

Before the load test, the project has a shell script that fires N requests at
the same instant and prints each latency and the total. With 4 requests, each
one worth about 1 second of work, this is what you get:

![Timelines of 4 simultaneous requests: /io all finish at 1s, /cpu finish at 1, 2, 3 and 4s, /cpu-thread take turns and all finish near 4s, /cpu-parallel all finish at 1s]({{ "/assets/img/posts/concurrency-vs-parallelism/timelines.svg" | relative_url }})

| Endpoint | Latencies (s) | Total | Why |
| --- | --- | --- | --- |
| node `/io` | 1, 1, 1, 1 | ~1 s | concurrency: the one JS thread is free while it waits |
| node `/cpu` | 1, 2, 3, 4 | ~4 s | the one JS thread is busy, so requests queue |
| node `/cpu-parallel` | 1, 1, 1, 1 | ~1 s | parallelism: 4 threads on 4 cores |
| py `/io` | 1, 1, 1, 1 | ~1 s | concurrency: the event loop interleaves the awaits |
| py `/cpu` | 1, 2, 3, 4 | ~4 s | blocks the event loop, just like Node |
| py `/cpu-thread` | 4, 4, 4, 4 | ~4 s | 4 threads, but the GIL lets only one run Python at a time |
| py `/cpu-parallel` | 1, 1, 1, 1 | ~1 s | parallelism: 4 processes, each with its own GIL |

The shape of the latencies tells you what happened inside:

- 1, 2, 3, 4, a staircase: the requests waited in line behind a single busy
  thread. Each one had to wait for all the ones before it.
- 4, 4, 4, 4: the requests took turns on one core. They ran concurrently,
  since all four were in progress at the same time, but not in parallel, so
  everyone finished at the end.
- 1, 1, 1, 1: the requests really overlapped.

The second shape is the interesting one. It's what concurrency without
parallelism looks like for CPU work: you pay the cost of switching and get
nothing back. Everyone waits the full 4 seconds.

## What happens inside each server

Both servers have a single event loop, and the question on every route is the
same: does the work keep that loop busy, or does it get handed off?

![Inside Express: on /io the event loop sets a timer and moves on; on /cpu it is stuck computing fib(40) while every request waits; on /cpu-parallel it hands fib to worker threads on other cores and stays free.]({{ "/assets/img/posts/concurrency-vs-parallelism/node-inside.svg" | relative_url }})

In Node, `/io` sets a timer and the loop moves on to the next request. On
`/cpu`, the loop is stuck inside `fib()`, and nothing else happens in the
whole process until it returns: not the other `/cpu` requests, and not `/io`
either. On `/cpu-parallel`, the loop hands `fib()` to a worker thread and goes
back to accepting requests.

![Inside FastAPI: async def /io awaits and moves on; async def /cpu blocks the event loop; def /cpu-thread runs on threads that share one GIL so only one computes at a time; /cpu-parallel runs fib in 4 processes, each with its own GIL.]({{ "/assets/img/posts/concurrency-vs-parallelism/python-inside.svg" | relative_url }})

In FastAPI, `async def` routes run *on* the event loop, so `/cpu` blocks it
exactly like Node. Plain `def` routes run on a threadpool, so `/cpu-thread`
has 8 threads working on 8 requests. But the GIL, Python's global interpreter
lock, lets only one thread execute Python code at a time, so the 8 threads
share what is effectively one core. `/cpu-parallel` uses processes instead,
and each process has its own interpreter with its own GIL.

## Load test with k6

4 requests is enough to see the shapes, but I wanted sustained numbers. So I
set up a [k6](https://grafana.com/docs/k6/) load test that runs with a single
`docker compose up`:

- Both servers run in containers capped at 4 CPUs, so "parallel" means the
  same thing on every machine.
- 7 scenarios, one per endpoint, run one at a time so they never compete for
  CPU.
- Each scenario is a closed loop: 8 virtual users each send a request, wait
  for the response, and immediately send the next one, for 30 seconds. So
  there are always 8 requests in flight.

![k6 closed loop: 8 virtual users each keep one request in flight. If the server handles all 8 at once, 8 req/s at 1 s; if it handles one at a time, 1 req/s at about 8 s.]({{ "/assets/img/posts/concurrency-vs-parallelism/closed-loop.svg" | relative_url }})

Because every request is about 1 second of work, the ideal results are easy
to predict before running anything:

| If the server... | Throughput | Latency |
| --- | --- | --- |
| works on all 8 requests at once | ≈ 8 req/s | ≈ 1 s |
| works on one request at a time | ≈ 1 req/s | ≈ 8 s (7 in line ahead of you, plus your own) |
| runs them in parallel on 4 CPUs | ≤ 4 req/s | ≈ 2 s |

Notice that throughput × latency ≈ 8 on every row, and 8 is the number of
users. That's Little's law, the same one I used to size the semaphore in the
[circuit breaker post]({% post_url 2026-10-09-circuit-breaker %}). In a closed
loop a slow server doesn't drop requests, each user just waits longer.

### Results

Measured with 8 users, 30 seconds, 4 CPUs, on a Ryzen 5 5600G. Latency here
is the full time from sending the request to getting the response, including
the time spent queued behind other requests.

| Scenario | Requests | Req/s | Avg | p95 | Max | What it shows |
| --- | --- | --- | --- | --- | --- | --- |
| `node-io` | 240 | 7.99 | 1.00 s | 1.00 s | 1.01 s | concurrency |
| `node-cpu` | 43 | 1.19 | 6.18 s | 13.28 s | 24.36 s | one thread, blocked |
| `node-cpu-parallel` | 105 | 3.34 | 2.38 s | 3.06 s | 3.31 s | parallelism (worker threads) |
| `python-io` | 240 | 7.98 | 1.00 s | 1.00 s | 1.00 s | concurrency |
| `python-cpu` | 41 | 1.11 | 6.59 s | 7.35 s | 7.39 s | one thread, blocked |
| `python-cpu-thread` | 38 | 1.10 | 6.97 s | 9.83 s | 12.02 s | threads, but the GIL serializes them |
| `python-cpu-parallel` | 104 | 3.35 | 2.35 s | 3.00 s | 3.74 s | parallelism (processes) |

## What the numbers taught me

### For waiting, both runtimes are the same

`node-io` and `python-io` are identical: 8 req/s at exactly 1 second, on a
single thread, with flat lines in the dashboard. When the work is waiting,
concurrency alone is enough, and it doesn't matter which event loop you
picked.

The same is true in the bad direction: `node-cpu` and `python-cpu` are both
stuck at about 1.1 req/s. An `async def` route with CPU work inside blocks
FastAPI's event loop exactly like `fib()` blocks Node's. The word `async`
doesn't make anything run in the background. It only lets the function give
the loop back when it hits an `await`, and `fib()` never does.

### Blocking the loop freezes the whole server, not just that route

While the `node-cpu` scenario was running, a `/io` request wasn't answering
in 1 second anymore. Under `python-cpu`, a single `/io` call took 14 seconds
instead of 1. Every route in the process shares the same loop, so a CPU-heavy
endpoint that gets some traffic degrades the cheap endpoints that get most of
it. This is the real cost of the blocked event loop: not that `/cpu` is slow,
but that everything else becomes slow with it.

### The average hides the tail

`node-cpu` and `python-cpu` have almost the same average (6.2 s and 6.6 s),
close to the predicted 7 seconds. But look at the max: 7.4 s in Python and
24 s in Node.

In the Python run, requests were served roughly in arrival order, so everyone
waited about the same. In the Node run, queued requests weren't taken in
arrival order, and some requests were unlucky again and again: in the
dashboard, one latency line climbs in steps, 5, 9, 13, 18, 24 seconds, as the
same request keeps getting skipped. The average doesn't show any of this. The
p95 and the max do, which is why I stopped looking only at averages.

### Python threads are concurrency, not parallelism

`python-cpu-thread` runs 8 threads on 8 requests and gets 1.10 req/s, the
same as `python-cpu` with one thread. The threads add zero CPU power to
pure-Python code, because the GIL lets only one of them execute at a time.
Latency is actually a bit worse and less even (max 12 s) because of the
constant switching between threads.

But the threads do buy something, and it's not in the throughput column.
Under the `python-cpu-thread` load, a `/io` call took 1.4 seconds instead of
the 14 seconds it took under `python-cpu`. The event loop stays free, so the
rest of the application keeps working. That's what makes a plain `def` route
the right tool for *blocking I/O*, like `requests` or a synchronous database
driver: it keeps a blocking call off the loop. It's not the right tool for
CPU work.

### Parallelism works, up to the number of cores

`node-cpu-parallel` and `python-cpu-parallel` both land at about 3.3 req/s,
about 3× the blocked version, with a tight tail (max 3.3 s and 3.7 s). Worker
threads in Node and processes in Python hit the same ceiling, because the
ceiling is the hardware: 4 CPUs.

Why 3.3 and not 4? Because 8 users means 8 threads or processes competing for
4 CPUs, so they take turns, and oversubscribing has a cost. I reran the
parallel scenarios with 4 users, one per CPU:

| Scenario | 8 users | 4 users |
| --- | --- | --- |
| `node-cpu-parallel` | 3.34 req/s at 2.38 s | 4.44 req/s at 0.90 s |
| `python-cpu-parallel` | 3.35 req/s at 2.35 s | 4.25 req/s at 0.94 s |

Fewer users, more throughput. More threads than cores made it slower, not
faster. The lesson for a pool of workers is to size it by the number of cores,
not by the number of requests.

### Processes with no code changes

The project's `/cpu-parallel` endpoints change the code to hand the work off.
There's a simpler way that doesn't touch the code: run more processes.

```bash
uvicorn main:app --port 8000 --workers 4
```

With 4 Uvicorn workers, the blocking `/cpu` route serves 4 simultaneous
requests in about 1 second, because each worker has its own event loop and
its own GIL. The Node equivalent is `node:cluster` or `pm2 -i 4`. This is the
usual way to get parallelism from a single-threaded runtime in production:
one process per core, and a load balancer in front.

## Which tool for which work

| The work is... | Node | Python (FastAPI) |
| --- | --- | --- |
| waiting on I/O with an async client | `async`/`await` on the main thread | `async def` with `await` |
| waiting on I/O with a blocking client | avoid it, or use a worker | plain `def` (threadpool) |
| CPU heavy | `worker_threads`, or a pool like `piscina` | `ProcessPoolExecutor` |
| CPU heavy, and you'd rather not change code | `node:cluster`, `pm2 -i N` | `uvicorn --workers N` |

One thing that will change this table: free-threaded Python builds (`3.13t`
and later, from [PEP 703](https://peps.python.org/pep-0703/)) have no GIL. On
those, `/cpu-thread` becomes parallel too, and threads start making sense for
CPU work in Python.

## What I took from this

- Concurrency and parallelism solve different problems. I/O needs the first,
  CPU needs the second, and no amount of one replaces the other. The 4, 4, 4,
  4 latencies of `/cpu-thread` are what you get when you apply concurrency to
  a parallelism problem.
- `async` is not "run in the background". It only helps if the function
  actually awaits something. CPU work inside an async function blocks the
  loop, in Node and in Python alike.
- A blocked event loop is a whole-server problem. The cheap endpoints suffer
  as much as the expensive one.
- Look at p95 and max, not only the average. Two servers with the same
  average gave one user a 7-second wait and another a 24-second wait.
- Size worker pools by cores, not by load. More threads than cores made the
  parallel endpoints slower.
- Little's law shows up everywhere. Throughput × latency ≈ users held on
  every row of the results, and it made the ideal numbers predictable before
  running the test.

Thanks for reading, see ya!
