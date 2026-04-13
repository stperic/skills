---
name: zzgo
description: Senior Go (Golang) expertise for writing, reviewing, and designing idiomatic Go. Covers concurrency (goroutines, channels, mutexes, race patterns), interfaces and embedding, error handling (%w wrapping, sentinel vs typed), naming and formatting (MixedCaps, gofmt, imports, doc comments), table-driven testing, package/API design, functional options, and the Go Proverbs / Zen of Go design philosophy. Use when writing Go code, reviewing a Go diff, designing a Go package or API, choosing concurrency primitives, structuring errors, or answering "is this idiomatic Go?". Draws on Effective Go, Go Code Review Comments, Test Comments, Code Review Concurrency, Google Go Style Guide, Uber Go Style Guide, and Dave Cheney's Zen of Go.
---

# zzgo — Senior Go Engineering & Architecture

Pure-Go guidance. Covers both day-to-day idioms (how/what) and system design (why/when) in one skill. Project-specific conventions live in the project's CLAUDE.md, not here.

## When to use

Invoke whenever you are:

- Writing, reviewing, or refactoring Go code
- Designing a package, interface, or API boundary
- Choosing between concurrency primitives (channels vs mutex, buffered vs unbuffered, context vs done-channel)
- Structuring errors (wrap with `%w`, sentinel vs typed, when to `panic`)
- Deciding what belongs in a table test, when to use subtests, when `t.Helper()` matters
- Resolving a review comment about naming, receivers, doc comments, or line length
- Answering "is this idiomatic?" or "would a senior Go engineer write it this way?"

## How to navigate this skill

Each topic below is a self-contained reference file. Open only the files relevant to the current task — do not load them all.

| Topic | File | When to open |
|---|---|---|
| Goroutines, channels, mutexes, context, race patterns (primitives) | `references/concurrency.md` | Any concurrent-code task; composition patterns live in `patterns.md` |
| Interface design, embedding, "accept interfaces return structs", small interfaces, type assertions | `references/interfaces.md` | API boundaries, mocks, polymorphism |
| `error` as value, `%w` wrapping, sentinel errors, typed errors, `errors.Is`/`As`, when to panic | `references/errors.md` | Error plumbing, logging, recovery |
| MixedCaps, receivers, package names, call-site de-repetition, import aliasing | `references/naming.md` | Naming choices, review comments |
| Table-driven tests, subtests, `t.Helper()`, `fatal` vs `error`, no-assertion-libs, golden files | `references/testing.md` | Writing or reviewing `_test.go` |
| Package layout, API surface, zero-value-useful, package-level state, export discipline | `references/packages.md` | New package, public API, refactor |
| `gofmt`, import groups, line length, doc comments, control flow, `init`, methods | `references/formatting.md` | Style questions, doc comments |
| Functional options, pipeline, fan-in/out, semaphore, circuit breaker, observer, strategy | `references/patterns.md` | Choosing a pattern, API ergonomics |
| Zen of Go, clarity > simplicity > concision hierarchy, design values | `references/philosophy.md` | Design tradeoff calls, PR disagreements |

If unsure which file to open, start with the topic that matches the **concrete artifact** you're touching (e.g., editing a `_test.go` → `testing.md`). Open `philosophy.md` only when a design call needs a tiebreaker.

## The Go Proverbs (Rob Pike)

From https://go-proverbs.github.io/. The canonical list is not ordinal — don't cite these by number. See `references/philosophy.md` for commentary.

- Don't communicate by sharing memory, share memory by communicating.
- Concurrency is not parallelism.
- Channels orchestrate; mutexes serialize.
- The bigger the interface, the weaker the abstraction.
- Make the zero value useful.
- `interface{}` says nothing.
- Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.
- A little copying is better than a little dependency.
- Syscall must always be guarded with build tags.
- Cgo must always be guarded with build tags.
- Cgo is not Go.
- With the unsafe package there are no guarantees.
- Clear is better than clever.
- Reflection is never clear.
- Errors are values.
- Don't just check errors, handle them gracefully.
- Design the architecture, name the components, document the details.
- Documentation is for users.
- Don't panic.

## Zen of Go (Dave Cheney) — one-liner summary

- Each package fulfils a single purpose.
- Handle errors explicitly.
- Return early rather than nesting deeply.
- Leave concurrency to the caller.
- Before you launch a goroutine, know when it will stop.
- Avoid package-level state.
- Simplicity matters.
- Write tests to lock in the behaviour of your package's API.
- If you think it's slow, first prove it with a benchmark.
- Moderation is a virtue.

Full commentary and source links: `references/philosophy.md`.

## Operating rules for this skill

1. **Topic-first, not role-first.** Pick the reference file by *what you're touching*, not by whether you're "architecting" or "engineering". Both lenses appear inside each file.
2. **Don't inline a reference into SKILL.md.** If a topic file is relevant, read it; don't ask the user to paste content.
3. **Don't fabricate style rules or thresholds.** If a rule isn't in Effective Go, Code Review Comments, Test Comments, Code Review Concurrency, the Google style docs, the Uber guide, the Proverbs, or Zen of Go, flag it as opinion, not canon. Do not attach numbers (method counts, file sizes, abstraction thresholds) to proverbs that never stated them.
4. **Project CLAUDE.md overrides this skill.** When the project's CLAUDE.md contradicts a general Go idiom (e.g., assertion library choice, test helper patterns), CLAUDE.md wins.
5. **Code > prose.** When explaining an idiom, show a minimal Go snippet; don't paraphrase.

## Canonical sources

- Effective Go — https://go.dev/doc/effective_go
- Go Code Review Comments — https://go.dev/wiki/CodeReviewComments
- Go Test Comments — https://go.dev/wiki/TestComments
- Go Code Review: Concurrency — https://go.dev/wiki/CodeReviewConcurrency
- Go Proverbs — https://go-proverbs.github.io/
- Google Go Style Guide — https://google.github.io/styleguide/go/guide.html
- Google Go Style Decisions — https://google.github.io/styleguide/go/decisions.html
- Google Go Best Practices — https://google.github.io/styleguide/go/best-practices.html
- Uber Go Style Guide — https://github.com/uber-go/guide/blob/master/style.md
- Go Patterns (tmrts) — https://github.com/tmrts/go-patterns
- Dave Cheney — The Zen of Go — https://dave.cheney.net/2020/02/23/the-zen-of-go
