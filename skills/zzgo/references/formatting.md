# Formatting

Go formatting and layout rules from Effective Go, Code Review Comments, Google Style, and Go proverbs.

## gofmt is law

- "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite." (Go Proverbs)
- All committed code runs through `gofmt` (or `goimports`). No exceptions, no local overrides. (Code Review Comments — Gofmt; Effective Go — Formatting)
- `goimports` = `gofmt` + automatic import management. Most projects require it.

## Imports

Imports are grouped, groups separated by a single blank line:

1. Standard library
2. Third-party
3. Local / same-module

```go
import (
    "context"
    "fmt"
    "net/http"

    "github.com/google/go-cmp/cmp"
    "go.uber.org/zap"

    "example.com/myproj/internal/server"
    "example.com/myproj/pkg/config"
)
```
(Google — Imports grouping; Code Review Comments — Imports)

- `goimports` produces this grouping automatically in most setups; two-group stdlib+rest is also acceptable.
- No aliasing unless necessary (see naming.md).

## Import dot — don't

- `import . "pkg"` pollutes the namespace and is banned in normal code. (Code Review Comments — Import Dot)
- Acceptable only in tests, and only when it materially helps readability (e.g., a DSL-like test package).

## Line length

- No hard limit in Go. (Code Review Comments — Line Length)
- In practice, wrap at 100–120 columns when natural break points exist. Don't wrap just to hit a column.
- Prefer splitting long expressions at logical boundaries (function arg, chained method) over arbitrary mid-expression wraps.

## Doc comments

- Every exported identifier has a doc comment. (Effective Go — Commentary; Code Review Comments — Doc Comments)
- The comment begins with the identifier name and is a complete sentence.

```go
// Parse reads r and returns the parsed Config, or an error if r is malformed.
func Parse(r io.Reader) (*Config, error) { ... }

// MaxRetries is the default number of retries for transient failures.
const MaxRetries = 3
```

- Package doc comment sits immediately above the `package` clause, or in a dedicated `doc.go` for longer docs. (Google — Doc comments)

```go
// Package config loads and validates zzrouter configuration.
package config
```

## Comment conventions

- Full sentences, ending with a period. (Code Review Comments — Comment Sentences)
- `//` for almost all comments; `/* */` reserved for package docs in `doc.go` or generated code.
- `TODO` format: `// TODO(username): description`. (Google — Comment conventions)

```go
// TODO(eric): replace with upstream's retry policy once v2 lands.
```

- Don't state the obvious. Comments explain *why*, code shows *what*.

## Declaring empty slices

Prefer the nil slice; it needs no allocation and behaves identically for `len`, `range`, and `append`.

```go
// Good
var names []string

// Bad
names := []string{}
```
(Code Review Comments — Declaring Empty Slices)

- Exception: when JSON-marshalling to `[]` instead of `null` matters, use `[]string{}` explicitly.

## Control flow — indent error flow

Happy path stays left-aligned; errors and edge cases are the indented branches. (Code Review Comments — Indent Error Flow; Google — Indent flow)

```go
// Good
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()
// ... use f ...

// Bad
f, err := os.Open(name)
if err == nil {
    defer f.Close()
    // ... use f (indented, buried) ...
    return nil
}
return err
```

- Prefer early `return` / `continue` / `break` over a deep `else` branch.

## In-band errors

- Don't overload return values with sentinel "error" values (`-1`, `""`). Return a second `ok bool` or `error` instead. (Google — In-band errors)

```go
// Good
func Lookup(key string) (Value, bool)

// Bad
func Lookup(key string) Value // returns zero Value on miss, unrecoverable
```

## `init` — use sparingly

- `init` runs once at package load, in file order. (Effective Go — Init)
- Legitimate uses: register a driver, compute a complex constant, validate config at startup.
- Avoid `init` for anything that does I/O, reads env vars, or has ordering dependencies between packages.

## Semicolons

- Go's lexer inserts a semicolon at the end of a non-blank line when the last token is one of a specific set (identifier, literal, `break`/`continue`/`fallthrough`/`return`, `++`/`--`, a closing bracket or brace). Never type them manually except inside `for` clauses and multi-statement lines. (Effective Go — Semicolons; Go spec — Semicolons)
- Opening `{` must sit on the same line as the preceding clause — otherwise the lexer inserts a semicolon after the clause and the `{` becomes a standalone block.

```go
// Good
if x > 0 {
    ...
}

// Compile error — auto-semicolon after `if x > 0`
if x > 0
{
    ...
}
```

## Functions and methods

### Pointer vs value receivers

- Consistency wins: all methods on a type should use the same receiver kind unless there's a specific reason. (Code Review Comments — receiver types; Effective Go — Methods)
- Use pointer receivers when:
  - The method mutates the receiver.
  - The struct is large (copying is expensive).
  - The type contains a `sync.Mutex` or other non-copyable field.
  - Consistency with other methods on the type.
- Value receivers are fine for small, immutable value types (`time.Time`, small structs).

```go
// Good — mutation
func (b *Buffer) Write(p []byte) (int, error) { ... }

// Good — small value type
func (p Point) Add(q Point) Point { return Point{p.X + q.X, p.Y + q.Y} }
```

### Declaration order

- Exported identifiers first where possible; group related helpers near their primary type. (Effective Go — Commentary)

## Crypto randomness

- Use `crypto/rand` for anything security-sensitive (tokens, keys, session IDs). (Code Review Comments — Crypto Rand)
- `math/rand` is a PRNG suitable only for non-security uses (jitter, shuffling test data).

```go
// Good
import "crypto/rand"
b := make([]byte, 32)
if _, err := rand.Read(b); err != nil { ... }

// Bad — predictable
import "math/rand"
b := make([]byte, 32)
rand.Read(b)
```

## Contexts

- `context.Context` is always the first parameter: `func Do(ctx context.Context, ...) error`. (Code Review Comments — Contexts)
- Never store a context in a struct; pass it explicitly per call.
- Don't pass `nil` context — use `context.TODO()` if you don't have one yet.

```go
// Good
func Fetch(ctx context.Context, url string) ([]byte, error)

// Bad
type Client struct{ ctx context.Context }
func (c *Client) Fetch(url string) ([]byte, error)
```

## Unused imports and vars

- Compile errors, not warnings. Delete them or use `_ "pkg"` for side-effect imports. (Google — Unused imports/vars)
- `goimports` removes unused imports automatically on save in most editors.

## Declaration style

- Prefer `var x T` when you need the zero value; `x := expr` when you have an initializer.
- Group related package-level `var`/`const` declarations in a single block:

```go
const (
    defaultPort    = 9090
    defaultTimeout = 5 * time.Second
)

var (
    ErrNotFound = errors.New("not found")
    ErrTimeout  = errors.New("timeout")
)
```

## Why these rules

- One enforced format = zero bikeshedding, trivial diff review, tooling (LSP, linters) has a single target.
- Indent-error-flow keeps the main story on the left margin; reviewers skim the happy path without chasing braces.
- Doc-comment-starts-with-identifier is what `go doc` renders — the convention *is* the tooling contract.
- `crypto/rand`, `context`-first, no-in-band-errors: each is a rule that prevents a specific class of real bug.
- `goimports` import grouping makes merge conflicts localized to real additions, not formatting noise.
