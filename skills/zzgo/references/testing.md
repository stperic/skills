# Testing

Go test conventions from the Go wiki TestComments, Google Style, Uber Style, and Go proverbs.

## Table-driven tests

Canonical Go pattern: one test function, one slice of cases. (TestComments — Table-Driven Tests; Uber — Test Tables)

```go
func TestParse(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {name: "empty", input: "", want: 0, wantErr: true},
        {name: "single", input: "1", want: 1},
        {name: "multi",  input: "1,2,3", want: 6},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Parse(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

Conventions:
- Field names: `name`, `input`/`in`, `want`, `wantErr`. (TestComments)
- Always use `t.Run(tt.name, ...)` so failures point at a specific case. (Uber — Subtests)
- Prefer a slice of structs over a map; slice order is deterministic. (TestComments; Uber)

## Subtest naming

- Names should be short, unique, and valid identifiers (no spaces — Go rewrites them to `_`). (TestComments — Subtest names)
- Useful for running one case: `go test -run TestParse/multi`.

## `t.Fatal` vs `t.Error`

- `t.Fatal` / `t.Fatalf`: stop the current test/subtest immediately. Use when subsequent code would panic or produce misleading output. (TestComments — Fatal vs Error)
- `t.Error` / `t.Errorf`: record failure, keep going. Use to report multiple independent assertions in one run.

```go
got, err := Do()
if err != nil {
    t.Fatalf("Do() error = %v", err) // can't dereference got below
}
if got.Name != "x" {
    t.Errorf("Name = %q, want %q", got.Name, "x") // keep checking other fields
}
if got.Size != 10 {
    t.Errorf("Size = %d, want 10", got.Size)
}
```

- `t.Fatal` is unsafe in goroutines other than the test goroutine — use `t.Error` + `return`. (TestComments — Goroutines)

## Test helpers

- Call `t.Helper()` at the top of any helper that calls `t.Error`/`t.Fatal`. Reported failures then point at the caller. (TestComments — Helper; Google — Test helpers)

```go
func mustParse(t *testing.T, s string) int {
    t.Helper()
    v, err := Parse(s)
    if err != nil {
        t.Fatalf("Parse(%q): %v", s, err)
    }
    return v
}
```

- A helper that doesn't touch `*testing.T` doesn't need `t.Helper()`.

## Struct / deep comparison

- Prefer `github.com/google/go-cmp/cmp` with `cmp.Diff` over `reflect.DeepEqual`. (Google — cmp vs reflect.DeepEqual)
- `cmp.Diff` produces human-readable diffs; `reflect.DeepEqual` only returns bool.

```go
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("Result mismatch (-want +got):\n%s", diff)
}
```

- Use `cmpopts.IgnoreFields`, `cmpopts.EquateApproxTime` for controlled comparisons.
- `reflect.DeepEqual` still fine for simple cases, but never for types with unexported fields you don't control.

## Assertion libraries

- Standard Go style: plain `if`/`t.Errorf`. No assertion libraries in stdlib or Google style. (TestComments)
- Projects may adopt `stretchr/testify` (`require` for fatal, `assert` for non-fatal) — this is a project-level convention, not a language one.
- If a project uses testify, be consistent: don't mix handwritten asserts and testify in the same file.

## Parallel tests

- `t.Parallel()` at the top of each test that is safe to run concurrently. (Uber — Parallel tests)
- Subtests in a `t.Run` loop must capture the loop variable before Go 1.22:

```go
// Pre-Go-1.22
for _, tt := range tests {
    tt := tt // capture
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        ...
    })
}

// Go 1.22+ — per-iteration scope, no capture needed
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        ...
    })
}
```
(TestComments — Parallel)

- Parallel subtests only start after the parent returns — `t.Cleanup` runs after all children.

## Golden file patterns

- Store expected output in `testdata/` (Go tooling ignores this directory). (TestComments — Testdata)
- Provide an `-update` flag to regenerate golden files:

```go
var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
    got := Render(input)
    golden := filepath.Join("testdata", "render.golden")
    if *update {
        os.WriteFile(golden, got, 0644)
    }
    want, err := os.ReadFile(golden)
    if err != nil { t.Fatal(err) }
    if !bytes.Equal(got, want) {
        t.Errorf("mismatch; run `go test -update` to regenerate")
    }
}
```

## Testing private APIs

- Same package: `foo_test.go` with `package foo`. Access to unexported identifiers. (TestComments — Test Package)
- Black-box: `foo_test.go` with `package foo_test`. Only exported API.
- Many projects mix: `foo_test.go` (internal) and `foo_external_test.go` (`package foo_test`).
- An `internal_test.go` naming convention exists in some projects for the internal-package variant; not universal.

## Goroutines in tests

- Never let a goroutine outlive the test. Failures in leaked goroutines get attributed to the wrong test or crash the binary. (TestComments — Goroutines)
- Use `sync.WaitGroup`, context cancellation, or `t.Cleanup` to join.
- Goroutines must not call `t.Fatal`; use `t.Error` + return, and signal the main test to fail.

```go
func TestAsync(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    t.Cleanup(cancel)
    done := make(chan error, 1)
    go func() { done <- Work(ctx) }()
    select {
    case err := <-done:
        if err != nil { t.Errorf("Work: %v", err) }
    case <-time.After(time.Second):
        t.Fatal("timeout")
    }
}
```

## Fuzz tests

- `func FuzzXxx(f *testing.F)`; seed corpus with `f.Add(...)`; body in `f.Fuzz(func(t *testing.T, ...) {...})`. (TestComments — Fuzz)
- Run: `go test -fuzz=FuzzXxx`. Failing inputs saved to `testdata/fuzz/FuzzXxx/`.

```go
func FuzzParse(f *testing.F) {
    f.Add("1,2,3")
    f.Fuzz(func(t *testing.T, s string) {
        _, _ = Parse(s) // assert no panic
    })
}
```

## Example functions

- `func ExampleFoo()` compiled, run, and rendered in `godoc`. (TestComments — Examples)
- `// Output:` comment is checked at runtime.

```go
func ExampleJoin() {
    fmt.Println(strings.Join([]string{"a", "b"}, "-"))
    // Output: a-b
}
```

- Use `// Unordered output:` when order is nondeterministic.
- Examples double as tests *and* documentation — favor them for package entry points.

## Proverbs in play

- "Tests lock in behavior." (Go Proverbs) — a test that passes for the wrong reason is worse than no test.
- Prefer tests that fail loudly when the API shape changes over ones that silently paper over it.

## Why these rules

- Table-driven + subtests: one place to add a case, one test ID per case in failure output.
- `t.Helper` + `t.Fatal` discipline: failure lines point at the real problem.
- `cmp.Diff`: a diff beats `false`.
- `testdata/`: Go tooling and `go vet` ignore it by convention.
- No-goroutine-outlives-test rule: prevents flakes and cross-test contamination.
