# Go Package Design

Rules for naming, shaping, and wiring Go packages so dependencies stay clean and APIs stay small.

## The one-sentence purpose test

- If you cannot describe a package's purpose in one sentence, split it. (Zen of Go — "Each package fulfils a single purpose")
- "A good Go package starts with its name, which is a succinct description of its purpose." (Zen of Go)
- The package name IS the first line of its documentation. If the name lies, the docs lie. (Effective Go — Package Names)

## Package name rules

- Short, lowercase, single word. No under_scores, no camelCase. (Effective Go — Package Names; CodeReviewComments — Package Names)
- Singular, not plural: `net/http`, not `https`; `bytes`, not `byteutils`. (CodeReviewComments)
- Avoid generic dumping-ground names: `util`, `common`, `misc`, `helpers`, `shared`, `base`, `lib`. These signal missing design. (Google Style Guide — Best Practices; CodeReviewComments)
- Name describes what it *provides*, not what it *contains*: `ring` (not `ringbuffer`), `suffixarray` (not `suffixarraymanager`). (Effective Go)
- Don't stutter at call sites: `bytes.Buffer`, not `bytes.BytesBuffer`; `http.Server`, not `http.HTTPServer`. (Effective Go; CodeReviewComments — Mixed Caps)
- Rename callers' imports only for clarity, never because the name is bad — fix the package name. (CodeReviewComments)
- Never use import dot (`import . "pkg"`) outside tests. It breaks readability and tooling. (CodeReviewComments — Import Dot)

## API surface discipline — export less

- Every exported identifier is a contract. Prefer unexported until a caller demands access. (Google Best Practices — API surface)
- "Design the architecture, name the components, document the details." (Go Proverbs)
- Reduce call-site repetition by naming: `time.Now()` not `time.GetCurrentTime()`; `user.ID()` not `user.GetUserID()`. (Google Best Practices — Function naming)
- Getters drop `Get`: `obj.Owner()`, not `obj.GetOwner()`. Setters keep `Set`. (Effective Go)
- Prefer a few wide structs with clear fields over many narrow accessors. Fields ARE the API for data types. (Effective Go — Composition)
- Return concrete types; accept interfaces. The caller decides the abstraction. (Go Proverbs — "accept interfaces, return structs")

## Zero value useful

- "Make the zero value useful." (Go Proverbs; Zen of Go)
- A struct should be usable immediately after `var x T` with no `New` call when possible.

```go
var mu sync.Mutex     // ready to Lock
var buf bytes.Buffer  // ready to Write
var list List         // ready to PushFront (container/list)
```

- If construction is unavoidable, name it `New` (returns `*T`) or `NewT` if the package exports multiple types. (Effective Go)
- A `New` that does nothing but `return &T{}` is a smell — drop it and let the zero value work.

## Package-level state — avoid

- Package-level mutable state is global state. Global state is a test hazard and a concurrency hazard. (Zen of Go — "Avoid package-level state"; Google Best Practices — Global state)
- Acceptable package-level values: typed constants, sentinel errors (`var ErrNotFound = errors.New(...)`), registered codecs at `init` time when unavoidable.
- Not acceptable: loggers, config, DB handles, caches, clients. Inject via constructor or function parameter.

```go
// Bad
var defaultClient = http.DefaultClient
func Fetch(url string) (*Response, error) { ... }

// Good
type Fetcher struct{ client *http.Client }
func New(c *http.Client) *Fetcher { return &Fetcher{client: c} }
```

- Singletons disguised as `GetInstance()` or `Default()` are still globals. Prefer explicit passing.

## `init()` — use sparingly

- `init` runs before `main` and cannot fail gracefully. Any `init` that can panic is a deployment risk. (Effective Go — Initialization)
- Legitimate uses: registering drivers (`database/sql`), registering codecs (`image/png`), computing tables from constants.
- Illegitimate: reading config, opening files, dialing network, starting goroutines.
- One `init` per file maximum; prefer zero.

## Package comments — `doc.go`

- Every package needs a package comment immediately above `package foo`. (CodeReviewComments — Package Comments; Effective Go)
- For multi-file packages, put it in a dedicated `doc.go`:

```go
// Package ring implements operations on circular lists.
//
// The zero value for Ring is a ring of size zero.
package ring
```

- Start with "Package foo ...". Describe purpose, not implementation. (Effective Go — Package Comments)
- Skip the comment only for `main` packages of trivial commands.

## Layered package dependencies

- Dependencies form a DAG. Cycles are a compile error and a design smell — the compiler is telling you two packages are one package. (Effective Go — Composition)
- Layer downward: `cmd/` → `internal/service/` → `internal/store/` → `pkg/types/`. Never the reverse.
- `internal/` discipline: anything under `internal/` is importable only by code rooted at the parent of `internal/`. Use it aggressively. Make packages `internal/` by default; promote to `pkg/` only when an external consumer appears. (Go toolchain — internal packages)
- `pkg/` is a publication contract. Breaking changes there break downstream. `internal/` can churn freely.

## Accept interfaces, expose structs

- Define interfaces where they are *consumed*, not where types are defined. (Go Proverbs — "The bigger the interface, the weaker the abstraction")
- Small interfaces compose: `io.Reader`, `io.Writer`, `io.Closer`. One-method interfaces named `-er` are the canonical form. (Effective Go)

```go
// In the consumer package:
type userStore interface {
    Get(ctx context.Context, id string) (*User, error)
}

func NewHandler(s userStore) *Handler { ... }
```

- Return concrete structs (`*Server`, `*Client`) so callers get full method sets and future additions don't break them. (Go Proverbs)
- Exception: factory functions that legitimately return one of several implementations.

## Copy-over-dep threshold

- "A little copying is better than a little dependency." (Go Proverbs)
- If you need one 20-line function from a 5000-line library, copy it (with attribution). Every import is a transitive-dep surface, a version-pin decision, and a supply-chain risk.
- Vendor a whole library when you use ≥ ~30% of its surface or depend on its semantic stability; copy a snippet otherwise.
- Standard library is the exception — always depend, never copy from `std`.

## The clarity hierarchy (drives every package decision)

Google's Go style guide ranks values in strict order: **Clarity > Simplicity > Concision > Maintainability**. (Google Style Guide — Guide)

- **Clarity** wins when two designs are equally simple: a 10-line function with a named helper beats a 10-line function with a clever inline closure.
- **Simplicity** beats concision: `for i := 0; i < len(xs); i++` is fine; don't reach for `range` tricks that obscure index math.
- **Concision** beats maintainability only when the concise form is also clear: prefer `xs[:0]` to `xs = make([]T, 0, cap(xs))` for reset.
- **Maintainability** is the tiebreaker, not the goal. Code optimized purely for future change tends to be abstract and unclear today.

When these values conflict in package design: split for clarity, even if it means more packages. Merge for simplicity only when the merged package still passes the one-sentence test.

## Quick checklist

- [ ] Package name is one lowercase word that describes what it provides.
- [ ] Package passes the one-sentence purpose test.
- [ ] `doc.go` or leading file has a `// Package foo ...` comment.
- [ ] No `util`, `common`, `helpers`, `misc`, `shared`.
- [ ] Zero values work, or `New` does real work.
- [ ] No package-level mutable state (sentinel errors and typed constants OK).
- [ ] No `init()` that can fail or do I/O.
- [ ] Interfaces defined at consumer, not producer.
- [ ] Exported surface is the minimum that callers actually need.
- [ ] No import cycles; `internal/` used by default.
