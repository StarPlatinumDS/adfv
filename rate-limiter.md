# Go Rate Limiter — Fixed Window and Concurrency

## Goal

Build a small per-client rate limiter to understand how to represent request limits over time, then deliberately introduce concurrent calls to learn why shared state needs synchronization.

This is an educational, in-memory component, not an HTTP server or a production-ready distributed rate limiter.

## Requirements

- Identify a client by a string supplied by the caller (for example, `"client-a"`). Client identification itself belongs to the HTTP/API layer, not to the limiter.
- Permit at most **5 requests per 10 seconds per client** in our experiment.
- Reject excess requests immediately rather than enqueueing them.
- Give each client an independent counter and window.
- Allow requests again once that client's window expires.

The intended interface is `Allow(clientID string) bool`: `true` means allow the request; `false` means reject it. An HTTP handler could translate a rejection into HTTP **429 Too Many Requests**, but HTTP integration was not part of this exercise.

## 1. From an idea to a state model

Our initial idea was a client identifier, a timer, and a request counter. Instead of running a timer in a background goroutine, the limiter checks the current time **when a request arrives**. It needs only two pieces of mutable state per client:

```go
type ClientState struct {
    Count       int
    WindowStart time.Time
}

type RateLimiter struct {
    Clients map[string]ClientState
    Limit   int
    Window  time.Duration

    mu sync.Mutex
}
```

`Clients` maps a client ID to its current request count and the start of its active window. `Limit` and `Window` are configuration, not flags indicating whether an individual client is blocked. The mutex was added later, when we tested concurrent calls.

Because `map[string]ClientState` stores **struct values**, retrieving a client returns a copy. After updating `Count` or `WindowStart`, the method must write the modified value back: `r.Clients[clientID] = client`. This reinforced a lesson from the earlier LRU-cache exercise.

## 2. Fixed-window algorithm

We implemented a *per-client fixed window anchored at the client's first request*, rather than globally aligned windows at fixed wall-clock times.

The four cases in `Allow` are:

1. **New client:** create state with `Count = 1`, `WindowStart = now`; allow.
2. **Expired window:** reset `Count = 1`, set `WindowStart = now`, save the new state; allow. The current request is already the first request in the new window.
3. **Window still active and count at the limit:** reject without changing the counter.
4. **Window still active and count below the limit:** increment the count, save the state; allow.

Check expiration **before** checking the limit; otherwise a client that has exhausted an old window may remain blocked. Compare elapsed time with the configured `r.Window`, not a hardcoded ten seconds. `time.Now()` provides a timestamp, and `time.Since(client.WindowStart)` gives the elapsed duration. Comparing two timestamps for exact equality would not answer whether the window has expired.

An illustrative version of the resulting method, incorporating the synchronization added below:

```go
func (r *RateLimiter) Allow(clientID string) bool {
    r.mu.Lock()
    defer r.mu.Unlock()

    now := time.Now()
    client, exists := r.Clients[clientID]
    if !exists || now.Sub(client.WindowStart) >= r.Window {
        r.Clients[clientID] = ClientState{
            Count:       1,
            WindowStart: now,
        }
        return true
    }

    if client.Count >= r.Limit {
        return false
    }

    client.Count++
    r.Clients[clientID] = client
    return true
}
```

This example assumes `Clients` was initialized and `Limit` and `Window` are positive; the exercise did not add configuration validation or a constructor. The method is shown for reference, not as a verbatim copy of the final code shared in the chat.

## 3. Sequential behavior verified

With a limit of 5 and a 10-second window, repeated sequential calls for `client-a` returned five `true` values followed by two `false` values. `client-b` had its own allowance. After `client-a`'s window expired, a new request was allowed and began a new window.

This established that the basic request-counting and time-window logic worked **without concurrency**.

## 4. Deliberately introduce concurrent access

We then launched ten goroutines calling `limiter.Allow("client-a")` on the **same** limiter instance and used a `sync.WaitGroup` to wait for them. All ten requests shared one client's counter, with a configured limit of five.

Before adding synchronization:

- A normal run produced a `concurrent map writes` runtime error.
- Runs with `go run -race .` reported unsafe concurrent access; one observed run printed `true` for all ten calls, and some runs crashed.

Two separate issues were exposed:

**Unsafe map access:** an ordinary Go map is not safe for unsynchronized concurrent reads and writes.

**Lost updates / incorrect check-then-update:** two goroutines can both read `Count = 3`, independently increment their copies to 4, and both save 4. Two requests were allowed, but the stored count increased by only one. Even a map implementation with thread-safe *individual operations* would not automatically make the entire decision-and-update sequence atomic.

## 5. Protect the complete operation with `sync.Mutex`

We added `mu sync.Mutex` to `RateLimiter`, acquired it at the beginning of `Allow`, and released it with `defer r.mu.Unlock()`.

```go
r.mu.Lock()
defer r.mu.Unlock()
```

Locking only individual map reads or writes is insufficient: the limiter must make the **read → check → update** sequence behave as one protected operation. Holding one mutex throughout `Allow` lets the next goroutine read the state only after the previous request's decision and state update finish. `defer` also covers every return path.

With the mutex, we ran `go run -race .` six times. Each run printed **exactly five `true` and five `false` results**, in varying orders, with **no race warnings in the shared output**. Goroutine scheduling determines which calls acquire the mutex first, not whether the limit is enforced. A clean race-detector run is useful evidence for the exercised paths, not proof that all possible bugs are absent.

## 6. Design trade-off: one lock for all clients

Our single mutex protects the entire clients map and all clients' counters. This is simple and correct for this small exercise, but an `Allow("client-a")` call blocks an `Allow("client-b")` call while either holds the same lock, even though their counters are independent.

We discussed per-client locking or other approaches as possible future optimizations, **but did not implement or benchmark them**. More complicated synchronization is not automatically better; the protected operation is short, and performance should be measured before redesigning it.

We also discussed why swapping the map for `sync.Map` would not, by itself, make the multi-step rate-limit decision atomic.

## 7. Limitation of this fixed-window design

A client can send five requests just before its window expires and five more immediately after expiration. All ten may be allowed in a short burst, because only the count and start of the **current** window are stored; individual request timestamps are not tracked.

We reasoned through this boundary case and identified it as a property of the chosen algorithm, **not a concurrency bug**. Sliding-window and token-bucket approaches were mentioned as alternatives, but **were not implemented or compared experimentally**. The Go-maintained `golang.org/x/time/rate` package was also mentioned; it is outside the standard library and was not used here.

## Takeaways

- State and policy are different: `Limit`/`Window` configure the limiter; `Count`/`WindowStart` describe an individual client's activity.
- Time windows can be evaluated lazily on each request; a timer goroutine is unnecessary for this design.
- Map lookups of struct values return copies that need to be written back after mutation.
- Correct sequential behavior does not imply correct concurrent behavior.
- Protect the entire invariant-preserving operation, not just isolated container accesses.
- A mutex guarantees consistent state transitions, **not** an execution order among goroutines.
- A race detector and behavioral assertions answer different questions; both are worth using.
- A fixed window limits requests **per window**, but does not strictly limit every rolling ten-second interval.

## Status / Possible follow-ups

**Completed:** basic fixed-window limiter, sequential behavior checks, deliberate race reproduction, synchronization with one mutex, and repeated concurrent runs under the race detector.

**Not done:** automated unit/concurrency tests, input validation, removing inactive clients from the map, benchmarking alternative lock designs, implementing a sliding-window/token-bucket limiter, or HTTP integration. The in-memory map will retain client entries unless a cleanup policy is added. These are future extensions, not prerequisites for marking this learning exercise complete.
