# Concurrency

Idioms for goroutines, channels, and synchronization in idiomatic Go.

## Mental model

- Concurrency is not parallelism — concurrency structures a program as independent components; parallelism executes them simultaneously. Go is a concurrent language, not inherently parallel. (Effective Go — Parallelization; Go Proverbs)
- Don't communicate by sharing memory; share memory by communicating. Channels orchestrate, mutexes serialize. Use channels for ownership transfer and signalling; use mutexes to protect short critical sections around a shared value. (Effective Go — Share by communicating; Go Proverbs)
- Reference counts or simple shared counters are usually best done with a mutex around the value, not a channel. Don't over-apply the slogan. (Effective Go — Share by communicating)

## When to use which primitive

- Channel: ownership transfer, pipeline stages, fan-in/fan-out, signalling completion, rate-limiting via buffered semaphore. (Effective Go — Channels)
- `sync.Mutex` / `sync.RWMutex`: protect a single shared struct's fields for short sections. Zero-value mutex is valid — never use `new(sync.Mutex)`. (Uber — Zero-value Mutexes are Valid)
- `sync.RWMutex` only helps if reads are long and writes are rare; otherwise its overhead is worse than plain `Mutex`. Benchmark before choosing it. (Code Review Concurrency — Sc.2)
  ```go
  // Bad: RWMutex to guard one int
  type Box struct { mu sync.RWMutex; x int }
  // Good: plain Mutex
  type Box struct { mu sync.Mutex; x int }
  ```
- `sync.Once`: lazy one-shot initialization that must happen exactly once across goroutines.
- `sync/atomic`: lock-free counters and flags on primitives; easy to misuse — consider `atomic.Bool`, `atomic.Int64` (Go 1.19+) for type safety. (Uber — Use go.uber.org/atomic)
  ```go
  // Bad: mixed atomic writes and plain reads race
  atomic.SwapInt32(&f.running, 1)
  _ = f.running == 1 // RACE
  // Good
  var running atomic.Bool
  running.Store(true)
  _ = running.Load()
  ```
- `sync.Map`: only for the two documented cases (caches, disjoint key sets per goroutine). For read-mostly but not disjoint, a plain map + `RWMutex` is usually clearer.

## Goroutine lifecycle

- Every goroutine must have a predictable exit: either it finishes bounded work or it receives a stop signal. Callers must be able to wait for it. A live goroutine is a GC root — anything it references stays alive, and the goroutine itself is never collected until it returns. A goroutine blocked forever on a channel that will never be sent on is a leak, regardless of whether any caller still holds a reference to it. (Uber — Don't fire-and-forget goroutines; Code Review Comments — Goroutine Lifetimes)
- Do not spawn goroutines in `init()`; startup work must be deterministic and synchronous. (Uber — No goroutines in init)
- Structure: a `stop` channel to signal exit, a `done` channel (or `sync.WaitGroup`) to observe exit.
  ```go
  // Bad: fire-and-forget
  go func() {
      for { flush(); time.Sleep(delay) }
  }()

  // Good: explicit lifetime
  stop := make(chan struct{})
  done := make(chan struct{})
  go func() {
      defer close(done)
      t := time.NewTicker(delay)
      defer t.Stop()
      for {
          select {
          case <-stop:
              return
          case <-t.C:
              flush()
          }
      }
  }()
  // shutdown:
  close(stop)
  <-done
  ```
- `time.Ticker` in a loop must be `defer t.Stop()`'d or the underlying goroutine leaks. (Code Review Concurrency — Tm.1)
- Loop variable capture: on Go < 1.22, `for _, x := range xs { go func(){ use(x) }() }` shares `x`. Pass it as an argument or re-declare inside the loop. Go 1.22 changed the language spec so that each iteration gets its own variable — this is a spec change, not a compiler fix; behaviour depends on the module's `go` directive. (Code Review Concurrency — RC; Go 1.22 spec change)

## Context propagation

- `context.Context` is the first argument of any function that may block, do I/O, or spawn. Pass it all the way through; never store it in a struct. (Code Review Comments — Contexts)
  ```go
  func F(ctx context.Context, ...) error { ... }
  ```
- Don't invent custom `Context` types or alternate interfaces — use `context.Context` directly.
- Use `context.Background()` only at program entry points (`main`, tests, top-level RPC handlers); everywhere else, accept a ctx parameter.
- Goroutines you spawn in a handler should derive from the request's `ctx` and exit when it is cancelled.

## Channel sizing

- Default to unbuffered. An unbuffered channel synchronizes sender and receiver — sends block until a receiver is ready. (Effective Go — Channels)
- Any buffer capacity is a design decision that must be justified: why won't it fill, what happens when it does? Buffered channels can hide deadlocks and mask backpressure problems. Unbuffered or size-1 are the common answers but are not style rules — pick the size the problem calls for, and document why. (Uber — Channel Size; Code Review Concurrency — Sc.1)
  ```go
  // Bad: magic buffer
  c := make(chan int, 64)
  // Good
  c := make(chan int)      // unbuffered
  c := make(chan int, 1)   // non-blocking send from one producer
  ```
- Capacity-N buffered channel is a natural semaphore:
  ```go
  sem := make(chan struct{}, maxInflight)
  sem <- struct{}{}   // acquire
  defer func() { <-sem }()  // release
  ```

## select patterns

- Cancellation + work:
  ```go
  select {
  case <-ctx.Done():
      return ctx.Err()
  case v := <-work:
      handle(v)
  }
  ```
- Non-blocking send / receive with `default`:
  ```go
  select {
  case results <- v:
  default:
      // drop or handle full buffer
  }
  ```
- Leaky-buffer free list uses `default` to allocate or drop rather than block. (Effective Go — A leaky buffer)
  ```go
  var freeList = make(chan *Buffer, 100)
  // get:
  var b *Buffer
  select { case b = <-freeList: default: b = new(Buffer) }
  // put:
  select { case freeList <- b: default: } // drop when full
  ```

## Composition patterns — see `patterns.md`

Fan-out, fan-in, pipeline, worker pool, bounded parallelism (`errgroup` + semaphore) are composition patterns built on the primitives above. Open `patterns.md` for snippets and tradeoffs. This file stays focused on the primitives (goroutines, channels, mutexes, context, race avoidance).

## Race conditions to avoid

- HTTP handlers are invoked concurrently — any state they touch must be thread-safe. (Code Review Concurrency — RC.1)
- Any write to a shared primitive must be synchronised, even `bool` or `int`. Per the Go memory model, a read that races with a write has undefined behaviour *by specification* — it is not a question of whether your hardware happens to tear word-sized writes. Use `sync/atomic`, a mutex, or a channel. (Go memory model; Code Review Concurrency — RC.2)
- A thread-safe getter must not return a pointer to protected internal state — return a copy:
  ```go
  // Bad
  func (c *Counters) Get(k Key) *Counter { c.mu.Lock(); defer c.mu.Unlock(); return c.vals[k] }
  // Good
  func (c *Counters) Get(k Key) Counter  { c.mu.Lock(); defer c.mu.Unlock(); return c.vals[k] }
  ```
  (Code Review Concurrency — RC.3)
- Slices and maps share backing storage — copy them at trust boundaries when the caller might retain a reference. (Uber — Copy Slices and Maps at Boundaries)
- Check-then-act on `sync.Map` is racy; use `LoadOrStore` or `LoadAndDelete`. (Code Review Concurrency — RC.4)
  ```go
  // Bad
  if _, ok := m.Load(k); !ok { m.Store(k, v) } // two goroutines can both Store
  // Good
  m.LoadOrStore(k, v)
  ```
- Always run tests with `-race` in CI. The race detector is the primary validation. (Code Review Concurrency — Testing)

## Mutex placement

- Don't embed `sync.Mutex` in an exported struct — it leaks `Lock`/`Unlock` into the public API. Use a named unexported field `mu`. (Uber — Zero-value Mutexes are Valid)
  ```go
  // Bad
  type SMap struct { sync.Mutex; data map[string]string }
  // Good
  type SMap struct { mu sync.Mutex; data map[string]string }
  ```
- Pair `Lock` with `defer Unlock()` for readability and exception safety.

## Time

- Compare `time.Time` with `t.Equal(u)`, not `==`, because `==` also compares `Location` and monotonic reading. (Code Review Concurrency — Tm.2)
- `time.Time` carries a monotonic reading in addition to the wall clock. `t.Before(u)` and `t.Sub(u)` prefer the monotonic reading, which is *immune* to NTP jumps — that's usually what you want for in-process duration math. Strip the monotonic reading with `t.Round(0)` only before *sending, persisting, or serializing* the value, because the monotonic reading is meaningless outside the originating process. Stripping in-process reintroduces vulnerability to wall-clock jumps. (Code Review Concurrency — Tm.3, Tm.4)

## Avoid

- Panics in goroutines — uncaught panics crash the process. Wrap long-lived worker goroutines with `defer recover()` only at the outermost layer of the goroutine. (Effective Go — Recover)
- Goroutines with no known exit (fire-and-forget), even for "it's just a log send". (Uber — Don't fire-and-forget goroutines)
- Mutable global state as a coordination mechanism — inject via struct fields so tests can swap it. (Uber — Avoid Mutable Globals)
