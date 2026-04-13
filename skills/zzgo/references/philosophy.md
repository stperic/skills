# Go Design Philosophy

The tiebreaker reference for design disagreements: values, proverbs, zen.

## The Four Core Values (Google hierarchy)

Go style ranks values in strict priority order: **Clarity > Simplicity > Concision > Maintainability**. (Google Style Guide — Guide)

1. **Clarity** — Code must communicate intent to a reader who has not seen it before. "Programs must be written for people to read, and only incidentally for machines to execute." Clarity is the reader's experience; everything else is secondary.
2. **Simplicity** — Prefer the straightforward mechanism. No clever tricks, no unnecessary layers. Simple code is easier to verify correct.
3. **Concision** — Shorter is better when it doesn't cost clarity or simplicity. A one-liner that takes ten seconds to understand loses to a three-liner that takes one.
4. **Maintainability** — Code should be easy to change later. But "maintainable" is not a synonym for "flexible" — premature abstraction costs clarity today for flexibility that may never be needed.

**Tiebreaker rules when values conflict:**

- Clarity vs simplicity → clarity wins. A slightly more complex structure that names its parts beats a terse blob.
- Simplicity vs concision → simplicity wins. `for i := 0; i < n; i++` beats clever index arithmetic.
- Concision vs maintainability → concision wins at the point of use; maintainability is paid forward, concision is paid now.
- Maintainability vs clarity → clarity wins. Abstractions introduced "for future flexibility" that obscure today's code are net-negative.
- When all four seem to conflict, re-read the code as a stranger. The design that makes a stranger nod is the right one.

Effective Go frames this succinctly: "The convention in Go is that package names are lowercase, simple, and short" — the language's whole aesthetic is "the obvious thing, clearly named." (Effective Go — Introduction)

---

## The Go Proverbs (with commentary)

From Rob Pike's go-proverbs.github.io. The list is not ordinal — cite by text, not by number. Grouped here by concern.

### Design

**"Don't communicate by sharing memory, share memory by communicating."**
Channels express ownership transfer. If two goroutines touch the same bytes, one must own it at a time — a channel pass makes ownership explicit. Mutexes are for guarding small, internal state (a counter, a map lookup), not for orchestrating workflow.

**"Concurrency is not parallelism."**
Concurrency is about structure — independent activities that may interleave. Parallelism is about execution — things actually running simultaneously. Goroutines give you concurrency; the scheduler plus cores give you parallelism when possible. Design for concurrency; parallelism falls out.

**"Channels orchestrate; mutexes serialize."**
Channels coordinate sequences of events across goroutines (pipelines, fan-out). Mutexes protect a single data structure from concurrent mutation. Pick the one that matches your intent, not what you learned first.

**"The bigger the interface, the weaker the abstraction."**
A one-method interface like `io.Reader` is implemented by dozens of unrelated types; that's what makes it useful. Each method added narrows the set of types that can satisfy it. The proverb gives a direction, not a numeric threshold — resist the urge to impose one.

**"Make the zero value useful."**
`var mu sync.Mutex` is ready. `var buf bytes.Buffer` is ready. Every `New` you avoid is a caller's line saved and an initialization ordering bug avoided.

**"interface{} says nothing."**
`any` is the absence of a type contract. Use it at boundaries (encoding/json, fmt) where genuine type erasure is the point. Elsewhere it's a signal that a type or generic is missing.

**"Gofmt's style is no one's favorite, yet gofmt is everyone's favorite."**
Formatting debates are negative-sum. One style, enforced by a tool, removes an entire class of review friction.

**"Design the architecture, name the components, document the details."**
Start with the shape — which packages exist, how they depend. Then the nouns — types and functions. Details (algorithm choice, exact field layout) come last and are easiest to change.

**"Documentation is for users."**
Write godoc for the caller, not for the maintainer. "Does X" beats "uses Y-algorithm internally" in the exported doc. Save the internals for a `// implementation note:` comment inside the function.

### Errors

**"Errors are values."**
`error` is an ordinary interface. You can store them, compare them (`errors.Is`), unwrap them (`errors.As`), build them with data. Don't reach for exception-style control flow — handle errors like any other value.

**"Don't just check errors, handle them gracefully."**
`if err != nil { return err }` is the default, not the goal. Add context (`fmt.Errorf("reading config: %w", err)`), decide whether to retry, fall back, or surface. A bare propagation with no context is a debugging tax.

### Concurrency and safety

**"Don't panic."**
Panics are for truly unrecoverable states (corrupt invariants, impossible code paths). Library code should return errors; only `main` and top-level handlers should recover.

**"A little copying is better than a little dependency."**
Every import is a version pin, a supply-chain surface, and someone else's breaking-change risk. Copy a small function rather than import a large library for it. Standard library excepted.

**"Clear is better than clever."**
If a reviewer has to think about what a line does, it's too clever. Boring code ships; clever code is rewritten.

**"Reflection is never clear."**
`reflect` defeats the type system. Use it only in the three legitimate places: serialization libraries, ORMs, and test frameworks. Everywhere else, a type switch or a generic parameter is clearer.

**"Cgo is not Go."**
The moment you `import "C"`, you lose Go's build speed, cross-compile simplicity, race detector, and runtime guarantees at the boundary. Worth it for GPU, graphics, and platform APIs. Not worth it for JSON.

**"Syscall must always be guarded with build tags."**
Platform-specific syscall code needs `//go:build linux` (or similar) tags. Otherwise Windows builds break mysteriously.

**"With the unsafe package, there are no guarantees."**
`unsafe.Pointer` is the escape hatch. Use it, document why, and accept that the language no longer helps you. One `unsafe` misuse can corrupt memory silently.

---

## The Zen of Go (Dave Cheney, 2020)

Ten points. Each matters.

**1. Each package fulfils a single purpose.**
If you can't describe the package in one sentence, split it. "util" fails this test. "net/http" passes.

**2. Handle errors explicitly.**
`if err != nil` is not boilerplate — it's the handling decision surfaced at every call site. That visibility is the point.

**3. Return early rather than nesting deeply.**
Guard clauses at the top, happy path at the bottom. Nested `if err == nil { ... }` blocks force the reader to track state through indentation levels.

```go
// Good
if err != nil { return err }
if x == nil   { return ErrMissing }
// happy path, unindented
```

**4. Leave concurrency to the caller.**
Library functions should be synchronous. Let the caller decide whether to wrap in a goroutine. Forcing concurrency inside an API denies the caller cancellation, rate-limit, and error-propagation choices.

**5. Before you launch a goroutine, know when it will stop.**
Every `go` statement is a promise that there's a defined exit. Without one, it's a leak. `ctx.Done()`, a closed input channel, or a bounded loop — pick one and commit.

**6. Avoid package-level state.**
Globals are invisible inputs to every function in the package. They break tests, break parallelism, and hide dependencies. Inject what you need.

**7. Simplicity matters.**
"Simple" is not "easy." Simple means few moving parts, clear boundaries, predictable behavior. Easy means familiar. Prefer simple over easy.

**8. Write tests to lock in the behaviour of your package's API.**
Tests are the executable specification of what callers can rely on. Test through the exported surface; don't couple tests to internals that can change.

**9. If you think it's slow, first prove it with a benchmark.**
Intuition about performance is wrong more often than right in Go, where allocations, escape analysis, and inlining drive cost. `testing.B` and `pprof` before refactoring.

**10. Moderation is a virtue.**
Moderation in interfaces (small), in abstractions (few), in dependencies (minimal), in generics (only when they earn their weight), in channels vs mutexes (either, not both for the same state).

### "A good Go package…"

> A good Go package starts with its name, which is a succinct description of its purpose. Its public API exposes only what is necessary, and is carefully considered. It has a well-defined scope. Its internal details are not leaked to the consumer. It makes doing the right thing easy and the wrong thing hard. It has minimal dependencies. It says what it means, and means what it says.

This paragraph is the compass for every package-level design question.

---

## Tiebreakers in practice

When two engineers disagree on a Go design choice, walk the ladder in this order:

1. **Does one choice violate a proverb?** If one side is `interface{}`-typed, has unbounded goroutines, or shares memory across goroutines without a channel or mutex, the proverb decides. Cite the proverb by its text, not by a number.
2. **Does one choice violate the Zen?** Nested error handling, package-level state, synchronous-API-forced-async — the Zen point decides.
3. **Which is clearer to a stranger reading the diff?** Clarity wins over simplicity, concision, and maintainability.
4. **Which is simpler today?** Fewer moving parts wins.
5. **Which is more concise without losing the above?** Shorter wins.
6. **Which is easier to change in six months?** Maintainability is the last tiebreaker, not the first.

If steps 1–6 still tie, the choice doesn't matter — pick one, move on, and don't relitigate.
