---
title: Error Handling Patterns
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "**/*.go"
exclude:
  - "**/*_test.go"
  - "vendor/**"
  - "**/*.pb.go"
  - "**/*_generated.go"
conclusion: neutral
---

Review changed Go code for project-specific error-handling patterns. Flag custom error types where standard ones would do, panics in library code, validation placed too deep, and incorrect logger severity.

These rules are derived from `.github/copilot-instructions.md` (section "5. Proper Error Handling") and `AGENTS.md` (section "Error Handling") in this repository. See the source for original wording and examples.

## What to flag

### Panics in library code (🔴 Must fix)

- Any new `panic(...)` call in non-`cmd`, non-`main` library code. Don't panic in library code — return errors and let caller decide.

### Wrong logger severity (🔴 Must fix)

- Use of `logger.Fatal` for anything other than core invariant violations (it crashes the process).
- Use of plain panic / crash paths where `logger.DPanic` is the project convention. Use `logger.DPanic` for issues that are important but should not crash production.

### Custom error types where standard ones exist (🟡 Should fix)

- New custom error types in handlers/services where a standard gRPC code would do. Use standard error types (`InvalidArgument`, `NotFound`, `FailedPrecondition`) over custom error types.
- Internal-concept names leaking into user-facing errors (e.g., `LowCardinalityKeyword`). Internal type names should not appear in messages returned to clients.

### Missing non-retryable marking (🟡 Should fix)

- Errors returned from task processing where the task shouldn't retry but the error isn't marked non-retryable. Mark errors as non-retryable when task shouldn't retry in queue.

### errors.As vs errors.AsType (🟡 Should fix)

- Use of `errors.As(err, &target)` where the project utility `errors.AsType` is preferred. Use `errors.AsType` instead of `errors.As`.

### Validation placed deep in business logic (🟡 Should fix)

- Argument validation performed inside business-logic helpers rather than at handler entry points. Validate early in handlers, not deep in business logic.

### Wrap with context, not without (🟡 Should fix)

- `fmt.Errorf("%w", err)` or bare `return err` in a path where there is something interesting or informative to add. Wrap errors with context when it helps, e.g. `fmt.Errorf("multi-operation part 2: %w", err)`. Conversely, don't wrap with no added information.

### Errors must not be ignored (🟡 Should fix)

- Returns from functions that produce an error which are silently dropped (`_ = f()`). Errors MUST be handled, not ignored.

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs.
- Do not flag issues already caught by the repository's static linters (golangci-lint config, including `errcheck`).
- Ignore generated code, vendored dependencies, and lockfiles.
- Test files are out of scope for this agent.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line. After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
