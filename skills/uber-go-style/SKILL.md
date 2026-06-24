---
name: uber-go-style
description: Agent-ready Uber Go style guide. Use when writing, reviewing, or modifying Go code to apply practical best practices for APIs, interfaces, errors, concurrency, data ownership, initialization, globals, naming, imports, tests, performance, linting, and idiomatic Go style. Use for implementation, refactoring, bug fixing, and code review when an agent needs concrete good/bad guidance without loading the full upstream guide.
---

# Uber Go Style

## Operating Mode

Use this skill as an agent-ready Go coding standard based on the Uber Go Style Guide. It is intentionally not a copy of `style.md`: it condenses the guide into decisions an agent can apply while coding or reviewing, with concrete bad/good rewrite patterns for the traps that most often cause bugs or noisy reviews.

Before changing code:

1. Inspect `go.mod`, package layout, nearby files, tests, and lint configuration.
2. Prefer consistent local convention when it is deliberate and does not hide bugs.
3. Use this skill to decide what good looks like, what bad looks like, and how to rewrite.
4. Read `references/src/*.md` only when exact upstream examples or nuance are needed.
5. After edits, run `gofmt` or `go fmt`; use `goimports` if imports changed and it exists.
6. Run focused tests first; run broader tests when shared behavior, APIs, or concurrency changed.

## Priority Order

When guidance conflicts, prefer this order:

1. Correct behavior and clear failure modes.
2. Stable, unsurprising package APIs.
3. Safe ownership of slices, maps, goroutines, locks, and globals.
4. Explicit error handling with useful context.
5. Readable, idiomatic Go that follows local conventions.
6. Performance improvements with clear evidence or obvious low cost.
7. Cosmetic style cleanup only when it helps the current change.

Do not create review churn. Fix code near the task. Leave unrelated large rewrites alone unless the user asks for a cleanup.
## Fast Review Checklist

Use this checklist when reviewing or touching Go code:

- Interfaces are values, not `*interface`.
- Important interface contracts are asserted with `var _ Interface = (*Type)(nil)`.
- Errors are handled once, wrapped with useful context, and not both logged and returned.
- Goroutines have cancellation, ownership, and a wait or shutdown path.
- Slices and maps are copied when crossing ownership boundaries.
- `sync.Mutex` and `sync.RWMutex` are named fields, not embedded and not allocated with `new`.
- Public structs do not embed implementation details accidentally.
- `init` avoids I/O, hidden ordering, goroutines, and mutable global setup.
- Libraries return errors instead of calling `os.Exit`, `log.Fatal`, or `panic`.
- Imports are grouped and names do not shadow built-ins.
- Tests cover changed behavior and edge cases, preferably table-driven when cases repeat.
- Hot paths avoid obvious allocation waste such as repeated conversions and missing capacity.

## Interfaces and APIs
Good Go APIs are small, explicit, and easy for callers to evolve with.

Use interfaces at the consumer boundary. Define an interface where it is used unless it is a stable public contract. Avoid exporting abstractions before there are real callers.

Never use a pointer to an interface. An interface value already contains type information and a data pointer. If the dynamic value needs mutation, the concrete value stored in the interface can be a pointer.

Bad:

```go
func Decode(r *io.Reader) error
```

Good:

```go
func Decode(r io.Reader) error
```

Verify interface compliance when a type must satisfy an API contract:

```go
var _ http.Handler = (*Handler)(nil)
```

Use the zero value on the right-hand side: `(*T)(nil)` for pointer receivers, `T{}` for value receivers, `nil` for slices and maps.

Choose receivers deliberately:

- Use pointer receivers when the method mutates the receiver, avoids copying large structs, or needs a consistent method set.
- Use value receivers for small immutable values where copying is expected.
- Avoid mixing pointer and value receivers for the same type unless there is a clear reason.

Avoid embedding types in public structs unless exposing the embedded type is truly part of the public API. Public embedding leaks methods and fields and makes future changes harder.

Bad:

```go
type Client struct {
    *http.Client
}
```

Good:

```go
type Client struct {
    httpClient *http.Client
}
```

Use functional options for constructors with several optional settings, especially when the option set may grow. Keep required values as explicit constructor parameters.

## Errors and Failure Behavior
Errors should tell the caller what failed and preserve matching when matching is useful.

Use:

- `errors.New` for static errors.
- `fmt.Errorf` for dynamic errors.
- `%w` when callers should be able to match or unwrap the cause.
- `%v` when the lower-level error should be opaque.
- `ErrFoo` for exported sentinel errors.
- `errFoo` for unexported package errors.
- `FooError` for custom error types.

Bad:

```go
if err != nil {
    return fmt.Errorf("failed to create new store: %w", err)
}
```

Good:

```go
if err != nil {
    return fmt.Errorf("new store: %w", err)
}
```

Handle errors once. Either wrap and return, or log and degrade gracefully. Do not log and return the same error unless duplicate reporting is explicitly desired.

Bad:

```go
if err != nil {
    log.Printf("read config: %v", err)
    return err
}
```

Good:

```go
if err != nil {
    return fmt.Errorf("read config: %w", err)
}
```

Always use comma-ok type assertions unless a panic is truly the desired behavior:

```go
v, ok := x.(string)
if !ok {
    return fmt.Errorf("expected string, got %T", x)
}
```

Do not panic in production control flow. Return errors. In tests, use `t.Fatal` or `require.NoError` instead of `panic`. `template.Must` and similar initialization helpers are acceptable when failure means the program cannot start.

Only `main` should call `os.Exit` or `log.Fatal`. Prefer:

```go
func main() {
    if err := run(); err != nil {
        log.Fatal(err)
    }
}

func run() error {
    // testable work
    return nil
}
```

## Concurrency

Every goroutine needs an owner and a way to stop. If code starts background work, the API should expose cancellation, `Close`, `Shutdown`, a returned channel, or a wait path.

Bad:

```go
go func() {
    for item := range ch {
        process(item)
    }
}()
```

Good:

```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            return
        case item, ok := <-ch:
            if !ok {
                return
            }
            process(item)
        }
    }
}()
```

Prefer unbuffered channels or size one. Larger buffers need a specific reason, such as known bounded work or documented throughput behavior.

Use zero-value `sync.Mutex` and `sync.RWMutex`. Do not allocate locks with `new`. Do not embed locks in structs because embedding exposes lock methods as part of the struct API.

Bad:

```go
type Cache struct {
    sync.Mutex
    items map[string]Item
}
```

Good:

```go
type Cache struct {
    mu    sync.Mutex
    items map[string]Item
}
```

Prefer typed atomics such as `go.uber.org/atomic` when the project already accepts that dependency. Otherwise follow local atomic conventions.

Do not start goroutines from `init`. Startup, shutdown, and error handling should be explicit.

## Data Ownership

Slices and maps share backing data. Copy them when ownership crosses package or component boundaries.

Copy inputs you store:

```go
func NewConfig(hosts []string) Config {
    return Config{hosts: append([]string(nil), hosts...)}
}
```

Copy internal state you return:

```go
func (c *Config) Hosts() []string {
    return append([]string(nil), c.hosts...)
}
```

Use nil slices as the default empty slice unless encoding semantics require `[]`. Check emptiness with `len(s) == 0`.

Preallocate when size is known:

```go
out := make([]User, 0, len(ids))
```

Initialize structs with field names, especially across package boundaries.

Bad:

```go
addr := net.TCPAddr{"127.0.0.1", 8080, ""}
```

Good:

```go
addr := net.TCPAddr{
    IP:   net.ParseIP("127.0.0.1"),
    Port: 8080,
}
```

Use `var s T` for a zero-value struct. Use `&T{Field: value}` instead of `new(T)` when setting fields.

Start enum-like constants at one when zero should mean unset or invalid:

```go
const (
    StatusUnknown Status = iota
    StatusReady
    StatusFailed
)
```

## Initialization and Globals

Avoid `init`. It hides order, errors, global state mutation, and test setup. If it is unavoidable, keep it deterministic, fast, and free of I/O and goroutines.

Avoid mutable package globals. Prefer constructor injection:

Bad:

```go
var db *sql.DB
```

Good:

```go
type Store struct {
    db *sql.DB
}
```

Immutable package values are fine. Mutable shared state needs explicit ownership and synchronization.

## Naming, Imports, and File Shape

Package names should be lowercase, short, and meaningful. Avoid `common`, `util`, `shared`, `base`, and `lib` unless the codebase already has a very clear convention.

Use `MixedCaps`, not underscores, for normal Go names. Test names may use underscores to group scenarios.

Avoid shadowing predeclared identifiers:

Bad:

```go
var error string
func copy(dst, src []byte) {}
```

Good:

```go
var errorMessage string
func copyBytes(dst, src []byte) {}
```

Use import aliases only when the package name differs from the last path element, there is a conflict, or a convention requires it.

Group imports:

```go
import (
    "context"
    "fmt"

    "go.uber.org/zap"
)
```

Order files so readers can scan them: public types and constructors first, methods grouped by receiver, helpers later. Put `NewType` near `Type`.

Use early returns to reduce nesting. Remove unnecessary `else` after `return`, `break`, `continue`, or `panic`.

Bad:

```go
if err != nil {
    return err
} else {
    return nil
}
```

Good:

```go
if err != nil {
    return err
}
return nil
```

Avoid naked boolean parameters in public or unclear APIs. Use named option structs, custom types, or comments for local obvious calls.

## Time, Tags, and Strings

Use `time.Time` for instants and `time.Duration` for periods. If a duration cannot use `time.Duration`, include the unit in the name, such as `timeoutMillis`.

Use RFC 3339 for timestamp strings unless another protocol requires something else.

Use field tags for marshaled structs:

```go
type Product struct {
    PriceCents int64 `json:"price_cents"`
}
```

Use raw string literals when escaping makes normal strings hard to read.

Declare format strings as constants when it helps `go vet` inspect printf-style usage. Name printf-style helpers with an `f` suffix, such as `Wrapf`.

## Performance Defaults

Prefer simple code until the path is plausibly hot or the waste is obvious. Good low-risk defaults:

- Use `strconv` for primitive-to-string conversion instead of `fmt`.
- Avoid repeated `string` to `[]byte` conversion for fixed values.
- Preallocate slices and maps when size is known.
- Avoid unnecessary allocation from pointer-heavy APIs when values are small and immutable.

Bad:

```go
for _, n := range nums {
    s = append(s, fmt.Sprint(n))
}
```

Good:

```go
for _, n := range nums {
    s = append(s, strconv.Itoa(n))
}
```

Do not trade readability for performance unless there is a benchmark, profile, or clear hot path.

## Tests

Use table-driven tests when cases share setup or assertions. Use subtests for readable failure output.

Good pattern:

```go
func TestParse(t *testing.T) {
    tests := []struct {
        name    string
        give    string
        want    Value
        wantErr bool
    }{
        {name: "empty", give: "", wantErr: true},
        {name: "valid", give: "42", want: Value{N: 42}},
    }

    for _, tt := range tests {
        tt := tt
        t.Run(tt.name, func(t *testing.T) {
            got, err := Parse(tt.give)
            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error")
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if got != tt.want {
                t.Fatalf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

Avoid complex branching inside table tests. Split tests when each case needs a different scenario or setup. For parallel subtests, capture loop variables.

Test observable behavior rather than private implementation details. Cover error paths, boundary values, cancellation, ownership copies, and public API guarantees. Use the repository's existing assertion library if it already has one; otherwise standard `testing` is enough.

## Code Review Behavior

When reviewing, lead with correctness and maintainability findings. A good finding says what can break, where, why, and how to fix it.

Prioritize:

- Data races or goroutine leaks.
- Lost, double-logged, or unmatchable errors.
- Caller mutation through slices or maps.
- Public API choices that are hard to evolve.
- Hidden startup behavior from `init` or globals.
- Missing tests for behavior that changed.
- Performance issues in loops or hot paths with clear fixes.

Do not flood the review with style nits. Mention style only when it affects readability, consistency, linting, or future maintenance.

## Deep Reference Lookup

The detailed source docs from the Uber guide live in `references/src/`. Load only the files needed for the current issue.

Use this routing:

- Interfaces and receivers: `interface-pointer.md`, `interface-compliance.md`, `interface-receiver.md`
- Errors: `error-type.md`, `error-wrap.md`, `error-name.md`, `error-once.md`
- Concurrency: `mutex-zero-value.md`, `channel-size.md`, `goroutine-forget.md`, `goroutine-exit.md`, `atomic.md`
- Data ownership: `container-copy.md`, `container-capacity.md`, `slice-nil.md`, `map-init.md`
- Initialization and globals: `init.md`, `global-mut.md`, `global-decl.md`, `global-name.md`
- Style and naming: `import-group.md`, `import-alias.md`, `builtin-name.md`, `nest-less.md`, `var-scope.md`
- Structs: `struct-field-key.md`, `struct-field-zero.md`, `struct-zero.md`, `struct-pointer.md`, `struct-tag.md`, `struct-embed.md`
- Tests and patterns: `test-table.md`, `functional-option.md`
- Performance: `performance.md`, `strconv.md`, `string-byte-slice.md`
- Full map: `SUMMARY.md`

Prefer this skill body for normal work. Open reference files only when you need exact upstream wording, a larger Bad/Good example, or a rule not captured here.
