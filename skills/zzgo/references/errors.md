# Errors

Error construction, wrapping, matching, and control flow in idiomatic Go.

## Core principles

- Errors are values. Build, inspect, and combine them with ordinary code — don't treat them as magic. (Go Proverbs; Effective Go — Errors)
- Don't just check errors, handle them gracefully. Decide at each call site: propagate (with context), recover (degraded behavior), or log-and-ignore (only when safe). (Go Proverbs)
- Don't panic for control flow. Return an `error`; let the caller decide. Panic is reserved for truly unrecoverable program states. (Go Proverbs; Code Review Comments — Don't Panic)

## The `error` type

- Built-in interface; anything with `Error() string` is an error. (Effective Go — Errors)
  ```go
  type error interface {
      Error() string
  }
  ```
- Never use the single-return form `_` to discard errors — either handle, return, or document the reason. (Code Review Comments — Handle Errors)
- Prefer an error return over an in-band sentinel like `-1` or `""`. Multiple returns are a core Go feature; use them. (Code Review Comments — In-Band Errors)
  ```go
  // Bad
  func Lookup(k string) string          // "" for missing
  // Good
  func Lookup(k string) (string, bool)  // ok=false for missing
  ```

## Choosing an error representation

`error` is a value — compose it the way the caller needs to use it. Two questions drive the choice: does the caller need to match on it, and does the message need to carry dynamic data? (Uber — Error Types)

- No match, static message → `errors.New("...")`.
- No match, dynamic message → `fmt.Errorf("...%s...", x)`.
- Callers match on identity (static category) → exported `var ErrFoo = errors.New(...)`.
- Callers match and need programmatic access to fields (path, code, index) → a struct type implementing `Error()`.

Don't elevate this into a rigid matrix — novel cases appear regularly (wrapping a third-party error, a domain error that sometimes carries fields and sometimes doesn't). Pick the form that makes the call site read naturally.

### Sentinel errors

- Exported `var ErrFoo = errors.New("foo")` for known categories callers branch on. Name with `Err` prefix (exported) or `err` (unexported). Avoid suffixing with `Error` — that's for types. (Uber — Error Naming; Google Best Practices — Sentinel Error Values)
  ```go
  var (
      ErrNotFound  = errors.New("not found")
      ErrConflict  = errors.New("conflict")
      errInternal  = errors.New("internal") // unexported, still usable via errors.Is within the package
  )
  ```

### Typed errors

- Use when callers need programmatic access to extra fields (path, code, index). Suffix with `Error`. (Uber — Error Naming; Google Best Practices — Error structure)
  ```go
  type NotFoundError struct {
      Resource string
      ID       string
  }
  func (e *NotFoundError) Error() string {
      return fmt.Sprintf("%s %q not found", e.Resource, e.ID)
  }
  ```
- Implement `Unwrap() error` when your typed error wraps a cause.

## Wrapping: `%w` vs `%v`

- `fmt.Errorf` with `%w` preserves the underlying error for `errors.Is` / `errors.As`. Use when the caller is meant to unwrap. (Uber — Error Wrapping; Google Best Practices — %w vs %v)
- `fmt.Errorf` with `%v` interpolates the error as text — the original is lost for matching. Use at system/network boundaries where you want to hide internal error structure. (Uber — Error Wrapping; Google Best Practices)
- Default to `%w` inside a package; switch to `%v` when crossing out to external clients. Be aware `%w` makes the wrapped error part of your API contract — document it.
  ```go
  // Good: internal, preserves chain
  return fmt.Errorf("load config %q: %w", path, err)
  // Good: external boundary, opaque
  return status.Errorf(codes.Internal, "fortune db unavailable: %v", err)
  ```
- Place `%w` at the end of the format string so the printed chain reads newest-to-oldest. Exception: when wrapping a sentinel that categorizes the failure, put it at the start. (Google Best Practices — Placement of %w)
  ```go
  // Good (trailing)
  return fmt.Errorf("load user %q: %w", id, err)
  // Good (leading sentinel)
  return fmt.Errorf("%w: invalid character in header", ErrParse)
  ```

## Adding context

- Add information the caller doesn't already have: what operation, which key, which stage. (Google Best Practices — Adding information)
- Don't duplicate information the underlying error already carries (e.g. `os.PathError` already has the path).
  ```go
  // Bad: duplicates "settings.txt"
  return fmt.Errorf("could not open settings.txt: %v", err)
  // Good
  return fmt.Errorf("launch codes unavailable: %v", err)
  // Result: "launch codes unavailable: open settings.txt: no such file or directory"
  ```
- Avoid boilerplate prefixes like `"failed to"` — they stack uselessly as the error propagates. (Uber — Error Wrapping)
  ```go
  // Bad
  return fmt.Errorf("failed to create new store: %w", err)
  // Good
  return fmt.Errorf("new store: %w", err)
  ```

## Error string style

- Lowercase. No trailing punctuation. No newline. Errors are fragments composed into larger messages. (Code Review Comments — Error Strings)
  ```go
  // Bad
  errors.New("Something bad.")
  // Good
  errors.New("something bad")
  ```
- When feasible, prefix with the package or operation origin: `"image: unknown format"`. (Effective Go — Errors)

## Matching: `errors.Is` and `errors.As`

- `errors.Is(err, target)` for sentinel value comparisons, walks the wrap chain.
- `errors.As(err, &target)` for typed errors, walks the chain and binds the matched value.
  ```go
  if errors.Is(err, os.ErrNotExist) {
      // degrade gracefully
  }

  var pErr *os.PathError
  if errors.As(err, &pErr) {
      log.Printf("op=%s path=%s", pErr.Op, pErr.Path)
  }
  ```
- Do not inspect error strings with regex or `strings.Contains` for branching — that couples you to the exact text. (Google Best Practices — Error structure)

## Handle errors once

At each call site pick exactly one of: log-and-ignore, log-and-degrade, match-and-handle, or wrap-and-return. Don't log AND return — the upstream frame will log again, flooding logs. (Uber — Handle Errors Once)

```go
// Bad: log + return
if err != nil {
    log.Printf("get user %q: %v", id, err)
    return err
}

// Good: wrap + return
if err != nil {
    return fmt.Errorf("get user %q: %w", id, err)
}

// Good: match + degrade
tz, err := getUserTZ(id)
if err != nil {
    if errors.Is(err, ErrUserNotFound) {
        tz = time.UTC
    } else {
        return fmt.Errorf("get user %q: %w", id, err)
    }
}

// Good: ignorable side-effect
if err := emitMetrics(); err != nil {
    log.Printf("metrics: %v", err) // non-fatal
}
```

## Indent error flow (early return)

Keep the happy path at minimal indentation; handle errors first and return. (Code Review Comments — Indent Error Flow)

```go
// Bad
if err != nil {
    // handle
} else {
    // do the work
}

// Good
if err != nil {
    return fmt.Errorf("...: %w", err)
}
// do the work
```

When an `if` with an initializer forces an `else`, hoist the declaration:
```go
// Bad
if x, err := f(); err != nil {
    return err
} else {
    use(x)
}

// Good
x, err := f()
if err != nil {
    return err
}
use(x)
```

## Don't panic

- Production code should not panic for expected failure modes. A panic crashes the goroutine (and usually the process), causing cascading failures. (Code Review Comments — Don't Panic; Uber — Don't Panic)
- Legitimate panic sites:
  - Program invariants (`panic("unreachable")`, `panic("impossible")`).
  - Package initialization when the program truly cannot start: `template.Must`, a mandatory env var missing. (Effective Go — Panic)
  - Re-panicking inside a narrow package-internal parse/compile flow where `recover` in the public entry point converts back to `error`. Do not expose panic to callers. (Effective Go — Recover)
- In tests, prefer `t.Fatal` / `t.FailNow` over `panic`. (Uber — Don't Panic)
- `run()` pattern: call `os.Exit` / `log.Fatal` only in `main`; every other function returns `error` so `defer` runs and code is testable. (Uber — Exit in Main)
  ```go
  func main() {
      if err := run(); err != nil {
          fmt.Fprintln(os.Stderr, err)
          os.Exit(1)
      }
  }
  func run() error { /* ... */ }
  ```

## Recover

- `recover()` only works inside a deferred function and only regains control in the current goroutine. Uncaught panics in spawned goroutines crash the program. (Effective Go — Recover)
  ```go
  func safelyDo(work *Work) {
      defer func() {
          if r := recover(); r != nil {
              log.Println("work failed:", r)
          }
      }()
      do(work)
  }
  ```
- Use it at server-worker boundaries to keep one bad request from taking down a pool, but don't let recovered panics masquerade as normal completions — log, increment a metric, and surface the failure.

## Wrap at boundaries

- Wrap when crossing a meaningful layer: handler → service, service → repository, repo → driver. Each wrap should add information, not repeat it.
- At public-package boundaries, decide: do you expose the wrapped chain (`%w` + documented sentinels) or opaque it (`%v`, canonical status codes)? This is part of the package's API contract. (Google Best Practices — %w when you document and test)

## errgroup for grouped operations

For orchestrating related concurrent operations where the first error should cancel the rest, `golang.org/x/sync/errgroup` is idiomatic. (Google Best Practices — Error handling)

```go
g, ctx := errgroup.WithContext(ctx)
for _, u := range urls {
    u := u
    g.Go(func() error { return fetch(ctx, u) })
}
if err := g.Wait(); err != nil {
    return fmt.Errorf("fetch batch: %w", err)
}
```
