# Raw TCP Echo Server in Go

## Goal

The purpose of this exercise was to go one layer below the HTTP frameworks and APIs I already use and understand what a server is actually doing at the TCP level.

The target was intentionally small:

- listen on `127.0.0.1:9000`
- accept TCP connections
- read raw bytes from a client
- send those bytes back
- keep a connection alive for multiple messages
- accept multiple clients
- handle multiple clients concurrently
- briefly explore graceful shutdown with OS signals

The main learning goal was not to build a production server.

It was to understand what abstractions such as `net/http`, Gin, and other web frameworks normally hide.

---

# 1. Listening on a TCP port

The first step was:

```go
listener, err := net.Listen("tcp", "127.0.0.1:9000")
```

Conceptually, this asks the operating system:

> Associate incoming TCP connections for `127.0.0.1:9000` with this program.

Important distinction:

```text
IP address + port
    = network endpoint/address

listener/socket
    = OS-backed networking object used by the program
```

Because the address is:

```text
127.0.0.1
```

the server is only reachable from the local machine through the loopback interface.

---

# 2. Accepting a connection

The next operation was:

```go
conn, err := listener.Accept()
```

`Accept()` does not process a request.

It waits for a client connection and returns an object representing that specific connection.

Conceptually:

```text
listener
   |
   | Accept()
   v
connection to client
```

`Accept()` is a blocking operation.

If no client connects, the goroutine waits there.

This was tested using both a browser and `nc`.

Example output:

```text
Waiting for connections...
Received connection!
127.0.0.1:53302
```

The address printed by:

```go
conn.RemoteAddr()
```

contained:

```text
127.0.0.1:<temporary client port>
```

The client-side port was chosen temporarily by the OS.

---

# 3. TCP is not HTTP

Opening:

```text
http://127.0.0.1:9000
```

in a browser successfully created a TCP connection, but the browser still reported an error.

That happened because the server had only implemented TCP.

A browser expects an HTTP exchange after the TCP connection is established.

Conceptually:

```text
TCP
    establishes the connection
    transports bytes

HTTP
    defines the meaning/format of those bytes
```

A browser might send something like:

```text
GET / HTTP/1.1
Host: 127.0.0.1:9000
...
```

but the raw TCP server did not understand or respond using HTTP.

This was a useful demonstration that HTTP is layered on top of TCP.

---

# 4. Reading bytes

A receive buffer was created with:

```go
buf := make([]byte, 1024)
```

Then data was read using:

```go
n, err := conn.Read(buf)
```

Important result:

```go
n
```

tells us how many bytes in the buffer actually belong to this read.

Therefore the valid received data is:

```go
buf[:n]
```

not the entire buffer.

When text was sent through `nc`, printing:

```go
fmt.Println(buf[:n])
```

showed numeric byte values.

Example:

```text
[104 101 108 108 111 10]
```

which corresponds to:

```text
hello\n
```

The final byte `10` was the newline created by pressing Enter.

---

# 5. TCP is a byte stream

An important conceptual point:

> TCP does not preserve application-level message boundaries.

If the sender performs:

```text
write("hello")
write("world")
```

the receiver is not guaranteed to read:

```text
"hello"
"world"
```

as two separate reads.

It may receive:

```text
"hel"
"lowor"
"ld"
```

or:

```text
"helloworld"
```

or another grouping.

TCP provides an ordered stream of bytes.

Application protocols must define their own framing when message boundaries matter.

For this echo-server exercise, framing was unnecessary because the server simply echoed whatever bytes each `Read()` returned.

---

# 6. Echoing data

After reading:

```go
n, err := conn.Read(buf)
```

the same bytes were written back:

```go
_, err = conn.Write(buf[:n])
```

The first working echo behaved like:

```text
client -> "yo yo yo wassup"
server -> "yo yo yo wassup"
```

At this stage the server only handled one read and then exited.

---

# 7. Keeping one connection alive

The read/write operations were placed inside a loop:

```go
for {
    n, err := conn.Read(buf)

    if err == io.EOF {
        break
    }

    if err != nil {
        break
    }

    _, err = conn.Write(buf[:n])
    if err != nil {
        break
    }
}
```

`io.EOF` means the peer cleanly reached the end of the stream / disconnected.

The connection lifecycle became:

```text
Accept connection
    |
    v
Read
Write
Read
Write
Read
Write
    |
    v
io.EOF
    |
    v
connection finished
```

This allowed a single `nc` client to send many messages before disconnecting.

---

# 8. `break` vs `return`

One mistake during the exercise was using:

```go
return
```

when a client disconnected.

Inside `main()`:

```go
return
```

means:

> Exit `main()` completely.

That shuts down the server.

What was needed inside the connection-processing loop was:

```go
break
```

which means:

> Leave only the current loop.

This distinction became important once the server needed to keep accepting future clients.

---

# 9. Accepting multiple clients sequentially

The listener was kept alive while `Accept()` was placed in an outer loop:

```text
Listen once

for {
    Accept client

    for {
        Read
        Write
    }

    close client
}
```

Conceptually:

```text
listener lifecycle
================================================>

client A
        ========>

client B
                  ========>

client C
                            ========>
```

The listener exists for the whole server lifetime.

Each accepted connection has a shorter, independent lifetime.

---

# 10. `defer` and lifetimes

Initially it was tempting to write:

```go
defer conn.Close()
```

inside the long-running `Accept()` loop in `main()`.

That would be a poor fit because `defer` runs when the surrounding function returns, not when the loop iteration ends.

So many deferred connection closes could accumulate until `main()` eventually exited.

For sequential connections, the connection could instead be explicitly closed after handling.

Later, once connection handling moved into its own function:

```go
func handleConnection(conn net.Conn)
```

this became correct and natural:

```go
defer conn.Close()
```

because each handler invocation has its own function lifetime.

---

# 11. Why sequential handling was insufficient

With sequential handling, the server behaved like:

```text
Accept client A
    |
    v
handle client A completely
    |
    v
client A disconnects
    |
    v
Accept client B
```

If client A connected and stopped sending data, the server was blocked in:

```go
conn.Read(buf)
```

and never returned to:

```go
listener.Accept()
```

Client B could establish a TCP connection at the OS level and wait in the listener's pending connection queue, but the Go program was not yet handling it.

This was observed directly when a second `nc` session connected but did not get an immediate echo until the first client disconnected.

---

# 12. Concurrent connection handling

The solution reused the goroutine model from the previous worker-pool exercise.

The listener remains responsible for accepting connections:

```go
for {
    conn, err := listener.Accept()

    ...

    go handleConnection(conn)
}
```

Each client receives its own goroutine:

```text
                 listener
                    |
              main goroutine
                    |
        -------------------------
        |           |           |
      client A    client B    client C
        |           |           |
        v           v           v
    goroutine A goroutine B goroutine C
```

Each connection handler independently performs:

```text
Read
Write
Read
Write
...
```

The final test used three simultaneous `nc` clients.

All three could send data and receive immediate echoes without waiting for the others to disconnect.

This confirmed that:

> `Accept()` and each client connection can progress independently when handlers run in separate goroutines.

---

# 13. Final concurrent echo-server structure

The educational implementation reached approximately this form:

```go
package main

import (
    "fmt"
    "io"
    "net"
)

func main() {
    listener, err := net.Listen("tcp", "127.0.0.1:9000")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer listener.Close()

    fmt.Println("Waiting for connections...")

    for {
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println(err)
            return
        }

        fmt.Println("Received connection:", conn.RemoteAddr())

        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()

    buf := make([]byte, 1024)

    for {
        n, err := conn.Read(buf)

        if err == io.EOF {
            fmt.Println("Connection closed")
            break
        }

        if err != nil {
            fmt.Println(err)
            break
        }

        _, err = conn.Write(buf[:n])
        if err != nil {
            fmt.Println(err)
            break
        }
    }

    fmt.Println("connection handler finished")
}
```

This is intentionally educational rather than production-ready.

---

# 14. Blocking points

Two important blocking operations were observed directly.

## `Accept()`

```go
listener.Accept()
```

blocks while the server waits for a new client.

## `Read()`

```go
conn.Read(buf)
```

blocks while a connected client has not sent any data.

This connected naturally to previous concurrency work.

A channel receive:

```go
<-jobs
```

and a socket read:

```go
conn.Read(buf)
```

can both suspend a goroutine while it waits.

They are not implemented using the same mechanism, though.

Channels are Go synchronization primitives.

Socket I/O ultimately uses OS networking facilities together with the Go runtime's network polling and goroutine scheduling machinery.

---

# 15. Graceful shutdown experiment

The TCP echo-server assignment itself was already complete before this point, but graceful shutdown was explored as a useful extension.

The first signal experiment used:

```go
shutdown := make(chan os.Signal, 1)

signal.Notify(
    shutdown,
    os.Interrupt,
    syscall.SIGTERM,
)
```

Then:

```go
sig := <-shutdown
```

blocked until the OS delivered a matching signal.

Pressing `Ctrl+C` produced:

```text
Received shutdown signal interrupt
Shutting down...
```

Important distinction:

```go
sig := <-shutdown
```

receives and stores the value.

```go
<-shutdown
```

also performs the receive and blocks, but discards the value.

---

# 16. `Ctrl+C` and SIGINT

Before registering signal handling:

```text
Ctrl+C
    |
    v
OS sends SIGINT
    |
    v
process terminates by default
```

After using `signal.Notify`:

```text
Ctrl+C
    |
    v
OS sends SIGINT
    |
    v
Go delivers signal to shutdown channel
    |
    v
program decides how to respond
```

This made graceful shutdown an explicit part of application control flow.

---

# 17. Listener shutdown

One way to wake a server blocked inside:

```go
listener.Accept()
```

is to close the listener from another goroutine.

Conceptually:

```text
main goroutine
    |
    +---- Accept()  <- blocked


shutdown goroutine
    |
    +---- <-shutdown
              |
            Ctrl+C
              |
              v
       listener.Close()
```

Closing the listener causes the blocked `Accept()` call to return with an error.

At this point the exercise had moved mostly into shutdown/error-handling details, so it was intentionally stopped rather than turning the echo-server exercise into a production server implementation.

---

# 18. Important mental models established

## Listener vs connection

```text
listener
    = accepts new client connections

connection
    = communicates with one specific client
```

These have different lifetimes and responsibilities.

---

## TCP vs HTTP

```text
TCP
    = ordered byte stream + connection

HTTP
    = application protocol carried over that stream
```

---

## TCP does not provide messages

TCP transports bytes.

Applications define message boundaries themselves if they need them.

---

## Blocking does not mean the process is frozen

A goroutine can block on:

```go
Accept()
```

or:

```go
Read()
```

while other goroutines continue running.

---

## Concurrency belongs naturally at the connection level

The sequential design exposed the problem first:

```text
one client blocks all others
```

Then the goroutine solution had a clear reason to exist:

```text
one connection
    =
one handler goroutine
```

for this simple educational server.

---

# 19. Mistakes that were useful

The exercise exposed several useful mistakes rather than hiding them.

### Putting `net.Listen` inside a `for` initializer

This confused listener lifetime with connection-processing lifetime.

Correct model:

```text
Listen once
Accept repeatedly
```

---

### Using the wrong loop condition

A loop such as:

```go
for listener, err := net.Listen(...); err != nil; {
```

would only execute while listening failed.

Successful setup gives:

```text
err == nil
```

so the loop would never run.

---

### Using `return` instead of `break`

`return` ended the entire server.

`break` was needed to leave only the current client-processing loop.

---

### Deferring connection closes in the wrong scope

`defer` is tied to function lifetime, not loop lifetime.

Moving connection handling into its own function made:

```go
defer conn.Close()
```

appropriate.

---

### Expecting multiple clients to work before introducing concurrency

The sequential version made it obvious why concurrency was needed instead of introducing goroutines just because server examples usually contain them.

---

### Forgetting that `main()` does not wait for goroutines

During the signal experiment:

```go
go loopTenSeconds()
```

did not keep the process alive.

Once `main()` returned, the whole process exited.

A blocking receive:

```go
<-shutdown
```

kept `main()` alive until a signal arrived.

This reinforced the lifecycle lesson from the worker-pool exercise.

---

# 20. What this exercise demonstrated

By the end of the exercise I had practical experience with:

- `net.Listen`
- `Listener.Accept`
- `net.Conn`
- `Conn.Read`
- `Conn.Write`
- byte buffers
- `io.EOF`
- connection lifetimes
- blocking network operations
- listener vs connection responsibilities
- sequential connection handling
- concurrent connection handling
- one-goroutine-per-connection architecture
- raw TCP vs HTTP
- TCP byte-stream semantics
- basic OS signal handling
- `signal.Notify`
- `SIGINT` / `SIGTERM`
- shutting down a listener to wake `Accept()`

---

# 21. Things deliberately left for later

This server is not production-ready, and that is intentional.

Topics left for future exercises include:

- partial write handling
- protocol framing
- read/write deadlines
- idle connection timeouts
- maximum connection limits
- backpressure
- goroutine limits
- graceful waiting for active handlers
- structured error handling
- distinguishing intentional listener closure from real failures
- context cancellation
- TLS
- malformed protocol input
- HTTP parsing
- production observability

These should be introduced when they solve a concrete problem rather than being added only for completeness.

---

# Main takeaway

The most useful progression was:

```text
listen
    ↓
accept one connection
    ↓
read bytes
    ↓
write bytes back
    ↓
keep one connection alive
    ↓
accept multiple clients sequentially
    ↓
observe blocking problem
    ↓
move each client into its own goroutine
    ↓
handle multiple clients concurrently
    ↓
briefly explore OS-driven shutdown
```

The exercise filled gaps between a high-level understanding of networking and the actual mechanics used by a Go server.

The key mental model is now:

> A TCP server owns a long-lived listener, accepts independent client connections from it, and reads/writes an ordered byte stream through each connection.

Frameworks such as `net/http` and Gin build substantially richer abstractions on top of these underlying operations.
