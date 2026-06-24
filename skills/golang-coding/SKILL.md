---
name: golang-coding
description: Go coding implementation and review guidance based on the Uber Go Style Guide. Use when writing, refactoring, debugging, or reviewing Go code; designing Go APIs, interfaces, errors, concurrency, tests, performance-sensitive paths, package structure, naming, imports, initialization, globals, structs, slices, maps, time handling, panic/exit behavior, linting, or style consistency.
---

# Golang Coding

## Operating Model

Use this skill to produce idiomatic, maintainable Go code while keeping the full style guide behind progressive disclosure.

Start from the repository in front of you: inspect `go.mod`, package layout, nearby code, existing tests, and local conventions before applying style guidance. Prefer existing project patterns when they intentionally differ from the reference docs.

Use the split reference files under `references/src/` as needed. Do not read every reference file by default, and do not try to reconstruct the monolithic upstream `style.md`. This `SKILL.md` is the entrypoint; `references/src/*.md` are the detailed source docs to load only when relevant.

## Workflow

1. Classify the task: implementation, refactor, review, bug fix, test work, API design, concurrency, performance, or style cleanup.
2. Inspect the local Go code first with `rg`, `go test`, `go list`, `go env`, or focused file reads as appropriate.
3. Choose the smallest relevant reference set from the routing guide below. When unsure, read `references/src/SUMMARY.md` or search `references/src/` with `rg`.
4. Apply guidance in priority order: correctness, clarity, API stability, concurrency safety, error behavior, testability, performance, then formatting/style.
5. Run `gofmt` or `go fmt`; use `goimports` when imports change and it is available. Run focused tests first, then broader tests when the change risk warrants it.
6. In the final response, mention any tests or checks run and call out unresolved tradeoffs or references that affected the decision.

## Reference Routing

Use these files only when the task touches the corresponding topic.

### Orientation

- Overall map: `references/src/SUMMARY.md`
- Guide background and external Go resources: `references/src/intro.md`
- Linting expectations: `references/src/lint.md`

### Interfaces and APIs

- Pointer-to-interface questions: `references/src/interface-pointer.md`
- Compile-time interface assertions: `references/src/interface-compliance.md`
- Receiver choice and interface behavior: `references/src/interface-receiver.md`
- Public embedding and exported API surface: `references/src/embed-public.md`
- Functional options: `references/src/functional-option.md`
- Package names: `references/src/package-name.md`
- Function names: `references/src/function-name.md`
- Function ordering: `references/src/function-order.md`

### Errors, Panics, and Process Exit

- Error type selection: `references/src/error-type.md`
- Error wrapping: `references/src/error-wrap.md`
- Error naming: `references/src/error-name.md`
- Single-point error handling: `references/src/error-once.md`
- Type assertion handling: `references/src/type-assert.md`
- Panic avoidance: `references/src/panic.md`
- Calling exit from `main`: `references/src/exit-main.md`
- Exiting once: `references/src/exit-once.md`

### Concurrency and Synchronization

- Mutex zero values: `references/src/mutex-zero-value.md`
- Atomic values: `references/src/atomic.md`
- Channel buffering: `references/src/channel-size.md`
- Goroutine lifecycle: `references/src/goroutine-forget.md`
- Waiting for goroutines: `references/src/goroutine-exit.md`
- Avoiding goroutines in `init`: `references/src/goroutine-init.md`

### Data Structures and Values

- Copying slices and maps at boundaries: `references/src/container-copy.md`
- Preallocating container capacity: `references/src/container-capacity.md`
- Nil slices: `references/src/slice-nil.md`
- Map initialization: `references/src/map-init.md`
- Struct embedding: `references/src/struct-embed.md`
- Struct tags: `references/src/struct-tag.md`
- Struct field names: `references/src/struct-field-key.md`
- Zero-value struct fields: `references/src/struct-field-zero.md`
- Zero-value struct initialization: `references/src/struct-zero.md`
- Struct references: `references/src/struct-pointer.md`
- Enum starts: `references/src/enum-start.md`

### Initialization, Globals, and Scope

- Avoiding `init`: `references/src/init.md`
- Mutable globals: `references/src/global-mut.md`
- Top-level declarations: `references/src/global-decl.md`
- Unexported global names: `references/src/global-name.md`
- Local variable declarations: `references/src/var-decl.md`
- Variable scope reduction: `references/src/var-scope.md`
- Declaration grouping: `references/src/decl-group.md`

### Imports, Formatting, and Naming

- Import group ordering: `references/src/import-group.md`
- Import aliases: `references/src/import-alias.md`
- Built-in name shadowing: `references/src/builtin-name.md`
- Line length: `references/src/line-length.md`
- Consistency: `references/src/consistency.md`
- Reducing nesting: `references/src/nest-less.md`
- Unnecessary `else`: `references/src/else-unnecessary.md`
- Naked parameters: `references/src/param-naked.md`
- Raw string literals: `references/src/string-escape.md`
- Constant format strings: `references/src/printf-const.md`
- Printf-style function names: `references/src/printf-name.md`

### Performance

- Performance overview: `references/src/performance.md`
- Prefer `strconv` over `fmt`: `references/src/strconv.md`
- Avoid repeated string-byte conversions: `references/src/string-byte-slice.md`
- Prefer specified capacity: `references/src/container-capacity.md`

### Time and Tests

- Time handling: `references/src/time.md`
- Table-driven tests: `references/src/test-table.md`

## Review Heuristics

For code review, prioritize findings that can cause bugs, data races, API breakage, resource leaks, unclear error behavior, test gaps, or measurable performance regressions. Style-only comments should be concise and grounded in a loaded reference file or strong local convention.

For implementation, keep changes small and idiomatic. Avoid adding abstractions until there is real duplication or API pressure. Prefer explicit error handling, bounded goroutine lifetimes, context-aware APIs where already used, and table-driven tests for meaningful behavioral branches.

## Validation

After editing Go code, prefer:

```bash
go test ./...
go vet ./...
```

Narrow the command when the repository is large or the task is localized. If a check cannot run because dependencies, services, or generated files are missing, report that clearly instead of treating the code as fully verified.
