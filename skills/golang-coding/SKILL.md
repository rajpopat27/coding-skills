---
name: golang-coding
description: High-level Go coding best-practice skill based on the Uber Go Style Guide. Use when writing, refactoring, debugging, or reviewing Go code; designing Go packages, APIs, interfaces, errors, concurrency, tests, data structures, initialization, globals, performance-sensitive paths, naming, imports, linting, or style conventions.
---

# Golang Coding

## Purpose

Use this skill as the working Go style guide. It distills the Uber Go Style Guide into practical defaults an AI agent can apply while implementing or reviewing Go code. The goal is not cosmetic formatting alone; the goal is readable, testable, maintainable Go that behaves correctly under real production pressure.

Start with the repository in front of you. Inspect `go.mod`, package layout, nearby code, tests, naming, error patterns, and linting configuration before making changes. Prefer established local conventions when they are deliberate and consistent. Use the guidance below as the default when the local codebase is unclear or inconsistent.

## Core Priorities

Apply these priorities in order:

1. Correctness and clear behavior.
2. API stability and low surprise for callers.
3. Explicit error handling and resource cleanup.
4. Safe concurrency and bounded lifetimes.
5. Simple data ownership across package boundaries.
6. Readability, consistency, and idiomatic Go style.
7. Performance where it matters and where the code path justifies it.

Do not turn style guidance into churn. Make the smallest change that improves the code for the task at hand.

## Working Loop

1. Understand the package role, public API, and tests before editing.
2. Search first with `rg`; read only the files needed to understand the change.
3. Implement with small functions, clear names, explicit errors, and simple control flow.
4. Run `gofmt` or `go fmt`; use `goimports` if imports changed and it is available.
5. Run focused tests first. Use `go test ./...` when the change crosses packages or touches shared behavior.
6. Mention checks run and any remaining risk in the final response.

## Package and API Design

Keep package APIs small, boring, and predictable. Put interfaces near the code that consumes them unless the interface is part of a deliberate public contract. Avoid exporting names before there is a real caller need.

Pass interfaces as values, not pointers. A pointer to an interface is almost never useful because the interface value already carries type and data information. If mutation is required, store a pointer concrete value inside the interface.

Verify important interface contracts at compile time when breaking that contract would surprise callers:

```go
var _ http.Handler = (*Handler)(nil)
```

Choose receiver types deliberately. Use pointer receivers when methods mutate the receiver, avoid copying large structs, or need consistent method sets. Use value receivers for small immutable values when copying is expected.

Avoid embedding types in public structs unless exposing the embedded type's methods and fields is truly part of the API. Public embedding can leak implementation details and make future changes harder.

Use functional options when constructors need optional configuration and the option set is likely to grow. Keep options simple and validate invalid combinations at construction time.

## Errors and Failure Behavior

Return errors instead of panicking for expected failure modes. Reserve `panic` for programmer mistakes, impossible states, or initialization failures where continuing is invalid.

Add useful context when returning errors across boundaries. Preserve the original error with `%w` when callers may need `errors.Is` or `errors.As`.

Handle each error once. Do not log an error and also return it unless the log adds operational value and duplicate reporting is intentional. Prefer returning enough context so the caller can decide how to handle or report the failure.

Use sentinel errors, custom error types, or wrapped errors based on caller needs:

- Use simple errors for local failures that callers do not inspect.
- Use sentinel errors when callers need stable identity with `errors.Is`.
- Use custom error types when callers need structured data with `errors.As`.

Name exported error values with an `Err` prefix, and name error types with an `Error` suffix. Always check the `ok` result from type assertions unless a panic is intentionally desired.

Only call `os.Exit` or `log.Fatal` from `main` or a tightly controlled process boundary. Libraries should return errors.

## Concurrency

Every goroutine should have a clear lifetime, shutdown path, and owner. Avoid fire-and-forget goroutines. If a function starts a goroutine, it should usually also provide cancellation, waiting, or ownership transfer.

Prefer unbuffered channels or channels with size one unless there is a measured or well-explained reason for a larger buffer. Larger buffers can hide backpressure and make ordering bugs harder to see.

Use `sync.Mutex` and related zero-value sync types directly; they are valid zero values. Do not allocate them unnecessarily. Use typed atomics such as `go.uber.org/atomic` where that dependency is already accepted by the project or local conventions support it.

Never start goroutines from `init`. Initialize explicit components from constructors or `main` so startup order, errors, and shutdown are visible.

## Data Ownership

Copy slices and maps at package boundaries when retaining them or returning internal state. Slices and maps are references to shared backing data; without copies, callers can mutate data they should not own.

Preallocate slices and maps when the expected size is known or easy to estimate. Do not add capacity noise where it obscures simple code, but use it for loops and hot paths that build collections.

Prefer `nil` slices as the empty slice value unless JSON or API semantics require an empty array. This keeps zero values useful and avoids unnecessary allocation.

Initialize structs with field names when values are not self-evident or when the type comes from another package. Omit zero-value fields when they do not communicate anything useful. Use `var value T` for a zero-value struct.

Use struct tags only when they are part of marshaling, validation, storage, or framework behavior. Keep tags accurate and review them as part of API changes.

Start custom enum-like constants at one when zero should mean unset or invalid. Let the zero value remain useful as "not configured" when that makes the API safer.

## Initialization and Globals

Avoid `init` unless there is no cleaner explicit initialization path. `init` hides order, hides errors, and complicates tests.

Avoid mutable package globals. Prefer dependency injection through constructors, function parameters, or explicit package-level values that are immutable. If shared mutable state is unavoidable, protect it and make ownership obvious.

Group top-level declarations by role. Keep public API, constructors, methods, helpers, and test fixtures easy to scan.

## Naming and Style

Let `gofmt` handle formatting. Do not hand-align code in ways that fight the tool.

Use short names for narrow scopes and descriptive names for exported APIs, package-level values, and longer scopes. Avoid stutter such as `user.UserService` when `user.Service` is clear.

Avoid shadowing built-in names like `error`, `string`, `len`, `cap`, `new`, and `copy`. Avoid naked parameters in function signatures when adjacent values have the same type; use small option structs or meaningful types when needed.

Keep imports grouped as standard library, third-party packages, then local packages. Use aliases only when they improve clarity, avoid a collision, or follow a common convention.

Reduce nesting with early returns. Avoid unnecessary `else` after a branch that returns, breaks, continues, or panics. Prefer straightforward control flow over clever compactness.

Use raw string literals for strings with heavy escaping. Keep format strings constant where possible, and name printf-style functions so linters can recognize them.

Consistency beats isolated preference. When a package already has a clear local style, match it unless it creates a correctness or maintainability problem.

## Performance Defaults

Prefer clear code until a path is plausibly hot or the cost is obvious. When performance matters, reduce avoidable allocations first: preallocate containers, avoid repeated string-to-byte conversions, and prefer `strconv` over `fmt` for basic numeric string conversion.

Do not make performance changes without preserving readability unless measurement, profiling, or code-path evidence supports the tradeoff.

## Tests

Use table-driven tests when the behavior has multiple cases, edge conditions, or error paths. Give each case a meaningful name. Keep setup close to the case when it clarifies intent.

Test observable behavior rather than implementation details. Include failure cases, boundary values, and concurrency cancellation paths when relevant. Match the repository's existing assertion style and helper patterns.

## Review Checklist

When reviewing Go code, look first for issues that can change behavior:

- Incorrect error wrapping, lost error context, or duplicated handling.
- Goroutines without cancellation, waiting, or ownership.
- Shared slices, maps, or mutable globals crossing boundaries unsafely.
- Public APIs that expose implementation details or are hard to evolve.
- Panics, exits, or initialization side effects outside process boundaries.
- Missing tests for changed behavior, edge cases, or failure paths.
- Hot-path allocation or conversion patterns that are easy to avoid.

Style-only feedback should be brief, grounded in a concrete rule, and worth the author's attention.

## Optional Deep References

The detailed Uber guide source files live in `references/src/`. Do not load them by default. Load a specific file only when you need exact examples, edge-case wording, or a rule that is not clear from this skill.

Useful lookup commands:

```bash
sed -n '1,120p' references/src/SUMMARY.md
rg -n "error|goroutine|interface|slice|map|init|test|strconv" references/src
```

Treat the reference files as supporting material for nuanced decisions, not as the primary workflow.

## Validation

After editing Go code, prefer:

```bash
go test ./...
go vet ./...
```

Narrow checks for small changes and expand them for shared packages, public APIs, concurrency, or behavior changes. If a check cannot run because dependencies, services, generated files, or toolchain pieces are missing, report that clearly.
