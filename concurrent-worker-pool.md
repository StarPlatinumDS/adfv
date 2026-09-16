# Concurrent Worker Pool in Go

## Goal

Build a small worker pool that processes multiple jobs concurrently while limiting how many jobs may run at the same time.

For this exercise:

- there are 10 jobs,
- there are 3 workers,
- each worker processes one job at a time,
- jobs are distributed through a channel,
- `main()` waits until all workers have finished.

This was mainly an exercise in understanding:

- goroutines,
- channels,
- blocking,
- worker lifecycle,
- `sync.WaitGroup`,
- buffered vs unbuffered channels,
- scheduler behavior,
- bounded concurrency.

---

## 1. What is a worker?

A worker is not a separate process.

In this implementation, a worker is simply a goroutine that repeatedly receives jobs from a channel and processes them.

Conceptually:

```text
worker:
    wait for job
    process job
    wait for next job
    ...
    stop when no more jobs can arrive
```

A worker can therefore process many jobs during its lifetime.

Example:

```text
worker 0 -> job 1 -> job 5 -> job 8
worker 1 -> job 2 -> job 3 -> job 7
worker 2 -> job 0 -> job 4 -> job 6 -> job 9
```

The exact distribution is not predetermined.

---

## 2. Representing a job

The job contains the data required to perform the work:

```go
type Job struct {
    ID          int
    JobDuration time.Duration
}
```

The duration is part of the job rather than the worker because different jobs may require different amounts of time.

---

## 3. The worker function

The worker receives jobs through a receive-only channel:

```go
func worker(id int, jobs <-chan Job, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        fmt.Printf("Worker %d received job %d\n", id, job.ID)

        time.Sleep(job.JobDuration)

        fmt.Printf("Worker %d finished job %d\n", id, job.ID)
    }
}
```

### Receive-only channel

This parameter:

```go
jobs <-chan Job
```

means that the function may receive values from the channel but may not send values into it.

Compare:

```go
<-chan Job   // receive only
chan<- Job   // send only
chan Job     // send and receive
```

This makes the intended ownership of the channel clearer.

---

## 4. Receiving jobs with `range`

This loop:

```go
for job := range jobs {
    ...
}
```

keeps receiving jobs until the channel has been closed and all previously sent values have been consumed.

Conceptually:

```text
wait for job
receive job
process it

wait for next job
receive it
process it

...

channel closed and empty
exit loop
```

If the channel is temporarily empty but still open, the worker blocks and waits.

It does not constantly poll the channel.

---

## 5. Blocking

Receiving from an empty open channel blocks:

```go
job := <-jobs
```

This means the goroutine waits until another goroutine sends a value.

Similarly, sending on an unbuffered channel blocks until another goroutine is ready to receive:

```go
jobs <- job
```

This is one of the main synchronization mechanisms used by channels.

---

## 6. Buffered vs unbuffered channels

Initially, jobs were sent through a buffered channel:

```go
jobsChannel := make(chan Job, 10)
```

This allows up to 10 jobs to wait inside the channel without being received immediately.

Conceptually:

```text
[job0][job1][job2]...[job9]
```

Because all 10 jobs fit in the buffer, `main()` can send them before a worker starts receiving.

---

### Unbuffered channel

The final implementation uses:

```go
jobsChannel := make(chan Job)
```

An unbuffered channel has no storage for waiting values.

A send requires a receiver to be ready.

Conceptually:

```text
sender -------------------- receiver
         job handoff
```

This code would deadlock:

```go
jobs := make(chan Job)

jobs <- Job{ID: 1}

go worker(0, jobs, wg)
```

because `main()` blocks while trying to send the job and therefore never reaches the line that starts the worker.

The workers must exist first:

```go
for i := 0; i < 3; i++ {
    go worker(i, jobs, wg)
}

jobs <- Job{ID: 1}
```

Now a worker can receive the value.

---

## 7. First `WaitGroup` mistake

The first implementation counted jobs:

```go
for i := 0; i < 10; i++ {
    wg.Add(1)
    jobsChannel <- Job{...}
}
```

but never called:

```go
wg.Done()
```

The `WaitGroup` counter therefore stayed at `10`.

Later:

```go
wg.Wait()
```

waited forever.

All workers had already stopped, so there was nobody left who could change the counter.

Go detected this state and produced:

```text
fatal error: all goroutines are asleep - deadlock!
```

The important lesson:

> `Wait()` only returns after the `WaitGroup` counter reaches zero.

Every `Add(1)` must eventually correspond to a `Done()`.

---

## 8. What should the `WaitGroup` count?

Two approaches are possible.

### Count jobs

```text
10 jobs
WaitGroup = 10

job finishes -> Done()
job finishes -> Done()
...
counter reaches 0
```

This is valid.

---

### Count workers

For this worker pool, it is cleaner to count worker goroutines instead:

```text
3 workers
WaitGroup = 3

worker exits -> Done()
worker exits -> Done()
worker exits -> Done()

counter reaches 0
```

The final implementation uses this approach.

```go
for i := 0; i < 3; i++ {
    wg.Add(1)
    go worker(i, jobsChannel, wg)
}
```

and each worker contains:

```go
defer wg.Done()
```

The responsibilities are then separated clearly:

```text
close(jobsChannel)
    -> no more jobs will arrive

wg.Wait()
    -> wait until all workers have exited
```

---

## 9. `WaitGroup.Add` placement

Both of these are correct:

```go
for i := 0; i < 3; i++ {
    wg.Add(1)
    go worker(i, jobs, wg)
}
```

and:

```go
wg.Add(3)

for i := 0; i < 3; i++ {
    go worker(i, jobs, wg)
}
```

The important rule is:

> Increment the `WaitGroup` before the goroutine can call `Done()`.

For a fixed number of workers, this is especially clear:

```go
workerCount := 3

wg.Add(workerCount)

for i := 0; i < workerCount; i++ {
    go worker(i, jobs, wg)
}
```

Avoid doing the `Add` inside the newly launched goroutine because `main()` could potentially reach `Wait()` before that goroutine increments the counter.

---

## 10. Closing the channel

After all jobs have been sent:

```go
close(jobsChannel)
```

Closing does not mean:

> remove all values immediately

It means:

> no more values will ever be sent.

Workers may still receive jobs that were already available.

Once the channel is closed and no jobs remain, this loop ends:

```go
for job := range jobs {
    ...
}
```

The worker function then returns, and:

```go
defer wg.Done()
```

decrements the worker counter.

---

## 11. Final implementation

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID          int
    JobDuration time.Duration
}

func worker(id int, jobs <-chan Job, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        fmt.Printf("Worker %d received job %d\n", id, job.ID)

        time.Sleep(job.JobDuration)

        fmt.Printf("Worker %d finished job %d\n", id, job.ID)
    }
}

func main() {
    jobsChannel := make(chan Job)
    wg := &sync.WaitGroup{}

    for i := 0; i < 3; i++ {
        wg.Add(1)
        go worker(i, jobsChannel, wg)
    }

    for i := 0; i < 10; i++ {
        jobsChannel <- Job{
            ID:          i,
            JobDuration: 2 * time.Second,
        }
    }

    close(jobsChannel)

    wg.Wait()
}
```

---

## 12. Why only three jobs run concurrently

There are exactly three worker goroutines:

```text
worker 0
worker 1
worker 2
```

Each worker processes one job at a time.

Therefore:

```text
maximum jobs executing simultaneously = 3
```

Even though there are ten jobs, the concurrency is bounded by the number of workers.

This is the main purpose of the worker pool.

---

## 13. How jobs are distributed

The workers all receive from the same channel:

```go
for job := range jobs
```

A single value sent through the channel is received by one worker.

It is not broadcast to all workers.

Conceptually:

```text
             jobs channel
                  |
        ---------------------
        |         |         |
     worker 0  worker 1  worker 2
```

Whichever worker becomes ready to receive work can receive the next job.

---

## 14. Worker ordering is not guaranteed

Launching goroutines like this:

```go
for i := 0; i < 3; i++ {
    go worker(i, jobs, wg)
}
```

does not mean they execute in this order:

```text
worker 0
worker 1
worker 2
```

The loop creates the goroutines in that order, but the Go scheduler decides when each goroutine actually executes.

Observed output included:

```text
Worker 2 received job 0
Worker 0 received job 1
Worker 1 received job 2
```

Repeatedly seeing worker 2 first does not create a guarantee that the last-created goroutine always runs first.

The correct mental model is:

> goroutine creation order is not execution order.

Programs must not rely on a specific scheduling order unless they explicitly synchronize it.

---

## 15. Which worker gets the next job?

After the first jobs have been assigned, the next available worker generally receives the next job.

Example:

```text
worker 0 -> 5 second job
worker 1 -> 1 second job
worker 2 -> 3 second job
```

Worker 1 will likely become available first:

```text
worker 1 finishes
        ↓
waits on jobs channel
        ↓
receives next available job
```

This means worker pools naturally distribute work based largely on worker availability rather than assigning an equal number of jobs to every worker.

The exact execution and print ordering still depends on scheduling.

---

## 16. Role of the Go scheduler

Creating a goroutine makes it runnable:

```go
go worker(...)
```

It does not mean:

> execute this function immediately.

The runtime scheduler decides when runnable goroutines get CPU time.

Therefore concurrency introduces nondeterminism in execution order.

The program should depend on synchronization guarantees such as:

- channel communication,
- channel closing,
- `WaitGroup`,
- locks when necessary,

rather than observed goroutine ordering.

---

## 17. Important concepts learned

### Goroutine

A lightweight concurrent execution unit managed by the Go runtime.

```go
go worker(...)
```

starts the function as another goroutine.

---

### Channel

A mechanism for passing values between goroutines.

```go
jobs := make(chan Job)
```

Workers receive from the same channel, so each job is handed to one worker.

---

### Blocking

Channel operations can suspend a goroutine until communication can happen.

This lets workers wait efficiently instead of polling repeatedly.

---

### Channel close

Communicates:

```text
no more values will ever be sent
```

and allows workers ranging over the channel to stop naturally.

---

### WaitGroup

Allows one goroutine to wait for a known number of concurrent operations to finish.

In the final design it tracks worker lifetimes.

---

### Bounded concurrency

Instead of starting one goroutine per job:

```go
for _, job := range jobs {
    go process(job)
}
```

the worker pool starts a fixed number of long-lived goroutines.

This limits how much work can execute concurrently.

---

## 18. Main design

The final system can be summarized as:

```text
                    main
                     |
              create 3 workers
                     |
                     v
             +----------------+
jobs ------> | jobs channel   |
             +----------------+
               |      |      |
               v      v      v
             worker worker worker
               0      1      2

main:
    send jobs
    close channel
    wait for workers

workers:
    receive jobs
    process jobs
    exit after channel is closed and drained
```

---

## 19. Main lessons

The most important lessons from this exercise were:

1. A worker can simply be a long-lived goroutine that repeatedly consumes jobs.
2. Channels can distribute work safely between several goroutines.
3. An unbuffered channel performs a direct handoff between sender and receiver.
4. Empty and closed channels are different states.
5. Closing a channel communicates that no future values will arrive.
6. `main()` does not automatically wait for other goroutines.
7. `WaitGroup` provides explicit lifecycle synchronization.
8. `WaitGroup.Add` must happen before the corresponding goroutine can call `Done`.
9. Goroutine creation order does not determine execution order.
10. The Go scheduler should generally be treated as nondeterministic from application code.
11. A fixed number of workers gives bounded concurrency.
12. The next job naturally tends to go to whichever worker becomes available first.

---

## Possible future extensions

These are intentionally not part of the first worker-pool implementation:

- different job durations,
- job errors,
- result channels,
- cancellation with `context.Context`,
- worker shutdown on cancellation,
- buffered queues,
- backpressure,
- race conditions,
- shared mutable state,
- mutexes,
- worker count configuration,
- benchmarks,
- retries,
- panic recovery.

The basic worker pool should remain simple until those concepts are introduced deliberately.
