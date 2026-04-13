# Idiomatic Go Patterns

Concurrency and API patterns that fit Go's grain. Skip GoF ports that fight the language. For basic goroutine / channel / mutex / context primitives, see `concurrency.md`.

## Functional Options

**When:** A constructor has >2 optional parameters, or options will grow over time. (Uber Style Guide — Functional Options; go-patterns)

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

type Option func(*Server)

func WithPort(p int) Option        { return func(s *Server) { s.port = p } }
func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

func New(host string, opts ...Option) *Server {
    s := &Server{host: host, port: 8080, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// srv := New("localhost", WithPort(9090), WithTimeout(time.Minute))
```

**Tradeoff:** Adds a type and a file of `With...` funcs. Use a config struct instead if the set is small and stable.

## Pipeline

**When:** A series of stages each transform a stream. Each stage is a goroutine connected by channels. (Effective Go — Channels; go-patterns — Pipeline)

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums { out <- n }
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in { out <- n * n }
    }()
    return out
}

// for v := range sq(sq(gen(1, 2, 3))) { fmt.Println(v) }
```

**Tradeoff:** Cancellation requires passing `ctx` or a `done` channel into every stage — otherwise goroutines leak when the consumer stops reading.

## Fan-Out

**When:** One producer, many identical workers consuming the same channel to parallelize CPU or I/O. (go-patterns — Fan-Out)

```go
in := gen(jobs...)
var wg sync.WaitGroup
for i := 0; i < runtime.NumCPU(); i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for job := range in { process(job) }
    }()
}
wg.Wait()
```

**Tradeoff:** Work distribution is only fair if jobs are roughly equal cost; otherwise some workers starve.

## Fan-In

**When:** Many producers, one consumer. Merge N channels into one. (Effective Go; go-patterns — Fan-In)

```go
func merge(cs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    wg.Add(len(cs))
    for _, c := range cs {
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c { out <- v }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

**Tradeoff:** Output ordering is nondeterministic. Closing discipline (WaitGroup + goroutine that closes `out`) is easy to get wrong.

## Worker Pool

**When:** Bounded parallelism for a known-size job set. Jobs flow in on one channel, results flow out on another. (go-patterns — Worker Pool)

```go
type Job struct{ ID int }
type Result struct{ ID int; Err error }

func pool(ctx context.Context, jobs <-chan Job, n int) <-chan Result {
    results := make(chan Result)
    var wg sync.WaitGroup
    wg.Add(n)
    for i := 0; i < n; i++ {
        go func() {
            defer wg.Done()
            for j := range jobs {
                select {
                case <-ctx.Done(): return
                case results <- Result{ID: j.ID, Err: do(j)}:
                }
            }
        }()
    }
    go func() { wg.Wait(); close(results) }()
    return results
}
```

**Tradeoff:** Prefer this to unbounded `go` for any loop that processes N items from an external source — unbounded goroutines under load is a classic OOM.

## Semaphore (bounded concurrency)

**When:** You have many independent tasks but must cap in-flight count (e.g., HTTP client connections, DB queries). (Effective Go — A leaky buffer; go-patterns — Semaphore)

```go
sem := make(chan struct{}, 8) // limit 8 concurrent
var wg sync.WaitGroup

for _, item := range items {
    sem <- struct{}{}           // acquire
    wg.Add(1)
    go func(it Item) {
        defer wg.Done()
        defer func() { <-sem }() // release
        handle(it)
    }(item)
}
wg.Wait() // wait for all launched goroutines
```

**Tradeoff:** Simpler than a worker pool when you don't need a result channel. Doesn't propagate errors — pair with `errgroup.WithContext` for cancellation + first-error. Note: "drain the semaphore to wait" is a common mistake — it deadlocks if fewer than `cap(sem)` goroutines were launched. Use a `WaitGroup` for the join.

## Circuit Breaker

**When:** Protect a caller from a failing downstream — fail fast instead of piling up requests. (go-patterns — Circuit Breaker; `sony/gobreaker` is the canonical impl)

```go
// Sketch — use sony/gobreaker in production.
type Breaker struct {
    mu           sync.Mutex
    failures     int
    threshold    int
    openUntil    time.Time
    cooldown     time.Duration
}

func (b *Breaker) Do(fn func() error) error {
    b.mu.Lock()
    if time.Now().Before(b.openUntil) { b.mu.Unlock(); return ErrOpen }
    b.mu.Unlock()

    err := fn()

    b.mu.Lock(); defer b.mu.Unlock()
    if err != nil {
        b.failures++
        if b.failures >= b.threshold {
            b.openUntil = time.Now().Add(b.cooldown)
            b.failures = 0
        }
        return err
    }
    b.failures = 0
    return nil
}
```

**Tradeoff:** Three states (closed/open/half-open) in real impls; rolling windows beat simple counters. Don't roll your own for production — use `gobreaker`.

## Observer — channel-based pub/sub

**When:** One event source, many subscribers. Use channels, not callback lists. (go-patterns — Observer)

```go
type Broker[T any] struct {
    mu   sync.RWMutex
    subs map[chan T]struct{}
}

func (b *Broker[T]) Subscribe() chan T {
    ch := make(chan T, 8)
    b.mu.Lock(); b.subs[ch] = struct{}{}; b.mu.Unlock()
    return ch
}

func (b *Broker[T]) Unsubscribe(ch chan T) {
    b.mu.Lock(); delete(b.subs, ch); close(ch); b.mu.Unlock()
}

func (b *Broker[T]) Publish(v T) {
    b.mu.RLock(); defer b.mu.RUnlock()
    for ch := range b.subs {
        select {
        case ch <- v:
        default: // drop for slow subscriber; alternative: block
        }
    }
}
```

**Tradeoff:** Decide slow-subscriber policy up front: drop, block, or disconnect. Silent blocking on a full channel is the #1 bug in this pattern.

## Strategy — function types, not interfaces

**When:** You need pluggable behavior with a single operation. Use a function type; reserve interfaces for multi-method contracts. (go-patterns — Strategy; Effective Go)

```go
type PriceFn func(qty int) decimal.Decimal

var (
    Flat     PriceFn = func(q int) decimal.Decimal { return decimal.NewFromInt(int64(q) * 10) }
    Discount PriceFn = func(q int) decimal.Decimal {
        p := int64(q) * 10
        if q > 100 { p = p * 9 / 10 }
        return decimal.NewFromInt(p)
    }
)

type Cart struct { price PriceFn }
func (c *Cart) Total(qty int) decimal.Decimal { return c.price(qty) }
```

**Tradeoff:** Function types lose method-receiver state. If the strategy needs state, use an interface with one method and a struct implementation — but still prefer a closure when possible.

## Anti-patterns — skip these

- **Singleton**: Go already gives you package-level `var` and `sync.Once`. But package-level mutable state is bad (see packages.md). If you think you want a singleton, pass the value explicitly.
- **Abstract Factory**: Just a constructor. `New` is enough.
- **Inheritance via embedding gymnastics**: Embed for composition, not to fake inheritance. The embedded type's methods are promoted; that's the entire mechanism.
- **Generic "Manager" / "Handler" / "Processor" types**: Name the actual behavior (`Dispatcher`, `Renderer`, `Encoder`).

## Cancellation — applies to every pattern above

- Every long-lived goroutine takes `context.Context` as its first parameter.
- `select` on `<-ctx.Done()` in any blocking operation.
- By convention, the sender closes the channel, not the receiver — this is a coordination rule, not a language rule. The spec only forbids closing a receive-only channel or a nil channel, and panics on sending to a closed channel. "Sender closes" exists so that receivers can `range` safely and detect completion; invert it and you get panic bugs.
- A goroutine without a defined exit path is a leak.
