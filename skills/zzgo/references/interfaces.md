# Interfaces

Designing and using Go interfaces — small, consumer-defined, implicitly satisfied.

## Core principles

- The bigger the interface, the weaker the abstraction. Small interfaces compose; giant ones entangle. `io.Reader` and `io.Writer` are the archetypes. (Go Proverbs; Effective Go — Interfaces)
- `interface{}` (or `any`) says nothing. Use it only at true boundaries — serialization, reflection, plugin dispatch. Prefer concrete types or narrow interfaces. (Go Proverbs)
- Accept interfaces, return concrete types. Callers get flexibility on input; callers of the returning function aren't locked into an abstraction they don't need. (Google Best Practices — Interfaces; Uber; Effective Go)
  ```go
  // Good
  func NewStore(log io.Writer) *Store { ... }   // accept interface
                                                 // return concrete *Store
  ```

## Keep interfaces small

- One or two methods is usually ideal. Interfaces with many methods are hard to implement, hard to mock, and rarely fit new types. (Google Best Practices — Designing effective interfaces)
- Single-method interfaces: name them with the `-er` suffix of the method (`Reader`, `Closer`, `Stringer`).
  ```go
  type Reader interface {
      Read(p []byte) (n int, err error)
  }
  ```
- Compose small interfaces rather than defining fat ones:
  ```go
  type ReadWriter interface {
      Reader
      Writer
  }
  ```
- Multi-method interfaces must document each method's contract individually; single-method interfaces can document the type. (Google Best Practices)

## Consumer defines the interface

- Interfaces belong in the package that *uses* values of the interface, not the package that implements them. The implementing package returns concrete types so new methods can be added without refactoring the interface. (Code Review Comments — Interfaces; Google Best Practices — Interface ownership)
  ```go
  // package payment (consumer)
  type Charger interface {
      Charge(*Card, Money) error
  }

  // package creditcard (producer)
  type Service struct { ... }
  func (s *Service) Charge(c *Card, m Money) error { ... } // returns concrete *Service
  ```
- Do not define interfaces on the implementor side "for mocking". Design the API so tests can use the public API of the real implementation (or an explicit public fake). (Code Review Comments — Interfaces; Google Best Practices — Don't define back doors for tests)
- Exceptions where the **producer** legitimately defines an interface:
  - The interface is the product (`io.Writer`, `hash.Hash`, generated protobuf service interfaces). (Google Best Practices)
  - Breaking an import cycle (treat as a smell — usually indicates package layering needs fixing).
  - Hiding complexity behind a rate-limited or wrapped handle (e.g. returning `io.Reader` instead of `*ThrottledReader` to prevent callers from tampering with internals). (Google Best Practices)

## When NOT to define an interface

- Don't create an interface before a real second implementation or a real test-double need exists. Premature interfaces force readers to understand three things (interface, impl, fake) when one would do. (Google Best Practices — Avoid unnecessary interfaces)
- Don't wrap generated RPC client/server code in a hand-written interface for abstraction's sake — use the generated interface or the concrete client directly.
- Don't export an interface when only internal code uses it — exporting commits you to that API. (Google Best Practices — Interface ownership)

## Interface satisfaction is implicit

- No `implements` keyword. A type satisfies an interface by having the right method set. This enables retrofitting interfaces onto existing types without editing them.
- For an exported type whose API contract includes satisfying a specific interface, add a compile-time assertion so a broken method set fails at build time, not at runtime or at a caller site. (Uber — Verify Interface Compliance)
  ```go
  type Handler struct { /* ... */ }

  var _ http.Handler = (*Handler)(nil)  // fails to compile if Handler stops satisfying http.Handler

  func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) { /* ... */ }
  ```
- Use the zero value of the asserted type on the right: `(*T)(nil)` for pointer types, `T{}` for value-receiver struct types.
  ```go
  var _ http.Handler = LogHandler{}    // value receivers
  var _ io.Reader    = (*MyReader)(nil) // pointer receivers
  ```

## Pointer vs value receivers affecting satisfaction

- Method set rules:
  - A value `T` has the methods declared with receiver `T`.
  - A pointer `*T` has the methods declared with receiver `T` **or** `*T`.
- Consequence: if any method on `T` uses a pointer receiver, only `*T` satisfies interfaces that include that method; value `T` does not. (Uber — Receivers and Interfaces; Effective Go — Pointers vs. Values)
  ```go
  type F interface{ f() }

  type S1 struct{}
  func (S1)  f() {}         // value receiver
  type S2 struct{}
  func (*S2) f() {}         // pointer receiver

  var i F
  i = S1{}      // ok: value satisfies F
  i = &S1{}     // ok
  i = &S2{}     // ok
  // i = S2{}   // compile error: S2 has no value-receiver f()
  ```
- Values stored in a `map[K]T` are not addressable — you cannot call pointer-receiver methods on them. Use `map[K]*T` when you need that.
- Rule of thumb: if any method mutates the receiver or the type contains a `sync.Mutex`, use pointer receivers consistently across all methods.

## Never use a pointer to an interface

- An interface value is already a two-word (type, data) handle. `*MyInterface` almost always reflects confusion. Pass the interface by value; if the implementation needs to mutate, it uses a pointer receiver internally. (Uber — Pointers to Interfaces)
  ```go
  // Bad
  func register(h *http.Handler) { ... }
  // Good
  func register(h http.Handler) { ... }
  ```

## Embedding for composition

- Go has no inheritance; embedding gives you delegation plus promoted methods. (Effective Go — Embedding)
  ```go
  type ReadCloser interface {
      Reader
      Closer
  }
  ```
- Struct embedding promotes methods of the inner type onto the outer:
  ```go
  type Logger struct { *log.Logger }
  // Logger now has Print, Printf, etc. via promotion
  ```
- Avoid embedding types in **exported** structs. The embedded type and its methods become part of your public API — removing, replacing, or upgrading it is a breaking change. Prefer a named field + hand-written delegation methods. (Uber — Avoid Embedding Types in Public Structs)
  ```go
  // Bad
  type ConcreteList struct { *AbstractList }

  // Good
  type ConcreteList struct { list *AbstractList }
  func (l *ConcreteList) Add(e Entity)    { l.list.Add(e) }
  func (l *ConcreteList) Remove(e Entity) { l.list.Remove(e) }
  ```

## Type assertions and type switches

- Always use the "comma, ok" form; the single-value form panics on a type mismatch. (Uber — Handle Type Assertion Failures)
  ```go
  // Bad
  s := i.(string)  // panics if i isn't a string

  // Good
  s, ok := i.(string)
  if !ok { /* handle */ }
  ```
- Type switch for dispatch on several concrete types behind an interface:
  ```go
  switch v := x.(type) {
  case *os.PathError:
      return fmt.Errorf("path: %w", v)
  case net.Error:
      return fmt.Errorf("net: %w", v)
  default:
      return v
  }
  ```
- Prefer `errors.As` over manual assertion when unwrapping errors — it walks the chain. (Uber — Error Matching; see errors.md)

## interface{} / any — when it's ok

- Boundaries where the type genuinely isn't known: JSON unmarshal targets, reflection, container libraries predating generics. (Go Proverbs — interface{} says nothing)
- Go 1.18+: prefer type parameters (generics) over `any` for collections and algorithms that were forced to erase types pre-generics.
  ```go
  // Pre-generics
  func First(xs []interface{}) interface{} { return xs[0] }
  // Modern
  func First[T any](xs []T) T { return xs[0] }
  ```

## Avoid

- Distinguishing between a nil slice and a non-nil empty slice at an interface boundary — prefer `var s []T` everywhere. (Code Review Comments — Declaring Empty Slices)
- Returning `interface{}`/`any` when a concrete type or narrow interface would do — it forces callers to assert.
- Interfaces designed to cover every method the implementation has (dual of "the bigger the interface, the weaker the abstraction"). Carve out only what the consumer calls.
