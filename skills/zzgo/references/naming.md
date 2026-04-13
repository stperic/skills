# Naming

Go naming conventions distilled from Effective Go, Code Review Comments, Google Style, and Uber Style.

## Package names

- Short, lowercase, single word. No under_scores or camelCase. (Effective Go — Package names)
- Singular, not plural: `package http`, not `httputil` unless it's genuinely utility. (Uber — Package Names)
- Avoid meaningless names like `util`, `common`, `base`, `helpers`. (Uber — Package Names)
- The package name is part of every exported identifier — choose so call sites read well. (Google — best-practices)

Avoid stutter: name functions so `package.Call` reads cleanly, not `package.PackageCall`.

```go
// Good
chubby.Client          // chubby.Client
http.Server            // http.Server

// Bad
chubby.ChubbyClient    // stutters
http.HTTPServer        // stutters
```
(Google — best-practices: avoiding call-site repetition)

## MixedCaps, not snake_case

- Multiword names use `MixedCaps` or `mixedCaps`, never underscores. (Effective Go — MixedCaps; Code Review Comments — Mixed Caps)
- Exported: `MixedCaps`. Unexported: `mixedCaps`.

```go
// Good
var maxRetries int
func ParseRequest() {}

// Bad
var max_retries int
func parse_request() {}
```

## Initialisms

Keep initialisms uniformly cased. `URL`, not `Url`. `ID`, not `Id`. `HTTP`, not `Http`. (Code Review Comments — Initialisms; Google — Naming)

```go
// Good
var userID string
func ServeHTTP() {}
type URLParser struct{}

// Bad
var userId string
func ServeHttp() {}
type UrlParser struct{}
```

Exception: when the initialism begins an unexported identifier, lowercase the whole thing: `urlParser`, `httpClient`.

## Receiver names

- 1–2 characters, a short abbreviation of the type. (Code Review Comments — Receiver Names)
- Be consistent: same receiver name on every method of a type across the whole package.
- Don't use generic names like `me`, `this`, `self`.

```go
// Good
func (c *Client) Do(...) {}
func (c *Client) Close() {}

// Bad
func (client *Client) Do(...) {}
func (self *Client) Close() {}
```
(Code Review Comments — Receiver Names; Google — Naming)

## Variable names

- Shorter names for shorter scopes; longer names for longer scopes or exported API. (Code Review Comments — Variable Names)
- Loop indices: `i`, `j`. Readers/writers: `r`, `w`. Common: `err`, `ok`, `ctx`.
- Avoid embedding type in the name; Go's type system already tracks it.

```go
// Good
for i, v := range items { ... }
ctx := context.Background()

// Bad
for index, value := range items { ... }
ctxContext := context.Background()
```

## Getters

- No `Get` prefix on getters. `Owner()`, not `GetOwner()`. (Effective Go — Getters; Code Review Comments)
- Setters do use `Set`: `SetOwner(x)`.
- OK to keep `Get` when the underlying concept genuinely uses it (HTTP GET, protobuf `Get*` generated code).

```go
// Good
func (u *User) Name() string { return u.name }
func (u *User) SetName(n string) { u.name = n }

// Bad
func (u *User) GetName() string { return u.name }
```

## Interface names

- Single-method interface: method name + `-er` suffix. `Reader`, `Writer`, `Formatter`, `Stringer`. (Effective Go — Interface names; Code Review Comments)
- Don't add `I` prefix or `Interface` suffix.

```go
// Good
type Reader interface { Read(p []byte) (n int, err error) }

// Bad
type IReader interface { ... }
type ReaderInterface interface { ... }
```

Multi-method interfaces: descriptive noun (`FileSystem`, `ResponseWriter`).

## Constants

- Use `MixedCaps`, not `ALL_CAPS`. (Google — Naming: constants)
- Don't encode the type in the name.

```go
// Good
const MaxRetries = 3
const defaultTimeout = 5 * time.Second

// Bad
const MAX_RETRIES = 3
const DEFAULT_TIMEOUT_DURATION = 5 * time.Second
```

## Error naming

- Sentinel error variables: `ErrFoo`. (Code Review Comments; Uber — Error Naming)
- Custom error types: `FooError`.
- Error messages: lowercase, no trailing punctuation. (Code Review Comments — Error Strings)

```go
// Good
var ErrNotFound = errors.New("not found")

type ValidationError struct{ Field string }
func (e *ValidationError) Error() string {
    return fmt.Sprintf("invalid field: %s", e.Field)
}

// Bad
var NotFoundErr = errors.New("Not Found.")
type ErrValidation struct{ ... }
```
(Uber — Error Naming)

## Named result parameters

- Use only when they meaningfully document the return, or when needed for `defer`. (Code Review Comments — Named Result Parameters)
- Don't name them just to silence linters or save a `var` declaration.

```go
// Good — name clarifies semantics
func Split(path string) (dir, file string)

// Good — needed for deferred mutation
func do() (err error) {
    defer func() { err = wrap(err) }()
    ...
}

// Bad — adds noise
func Count(s string) (count int) {
    count = len(s)
    return
}
```

## Function names

- Verb or verb phrase for actions: `Parse`, `Compute`, `WriteTo`. (Uber — Function Names)
- Noun for constructors: `NewClient`, `NewBuffer`.
- Don't repeat the package name in the function: `http.Get`, not `http.HTTPGet`. (Google — best-practices)

## Import aliasing

- Avoid aliasing unless required to disambiguate. (Uber — Import Aliasing)
- Required cases: two packages with the same basename; package name mismatches path.

```go
// Acceptable — disambiguation
import (
    netlib "example.com/net"
    "net"
)

// Bad — cosmetic
import fmt2 "fmt"
```

- Never dot-import outside of tests. (Code Review Comments — Import Dot)

## Why these rules

- Consistency across the ecosystem: anyone reading stdlib can read your code.
- Initialism casing and MixedCaps make `grep`/tooling output scan cleanly.
- Receiver consistency makes method sets easy to read at a glance.
- No-`Get` getters make field-like access uniform with direct field reads.
- `-er` interfaces signal "this is behavior, not a noun" — drives small interface design.
