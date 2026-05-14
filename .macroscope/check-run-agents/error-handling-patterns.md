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

Review changed Go code for project-specific error-handling patterns. Flag panics in library code, wrong logger severity, custom error types where standard ones exist, deep-buried validation, and dropped errors.

## What to flag

### No Panic In Library Code (🔴 Must fix)

- Any new `panic(...)` call in non-`cmd`, non-`main` library code. Don't panic in library code — return errors and let caller decide.

**Cite as:** No Panic In Library Code
**Source:** [`.github/copilot-instructions.md` § 5. Proper Error Handling](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#5-proper-error-handling)
> "Don't panic in library code - return errors and let caller decide"

### logger.Fatal Only For Core Invariant Violations (🔴 Must fix)

- Use of `logger.Fatal` for anything other than core invariant violations. `logger.Fatal` crashes the process; reserve it for invariants that cannot continue.

**Cite as:** logger.Fatal Only For Core Invariant Violations
**Source:** [`AGENTS.md` § Error Handling](https://github.com/temporalio/temporal/blob/main/AGENTS.md#error-handling)
> "Use `logger.Fatal` for core invariant violations"

### Use logger.DPanic For Important Non-Crash Issues (🟡 Should fix)

- Use of plain `panic` or `logger.Error` for important-but-not-fatal issues where `logger.DPanic` is the project convention. `logger.DPanic` panics in dev/test but logs in production, surfacing the issue without crashing.

**Cite as:** Use logger.DPanic For Important Non-Crash Issues
**Source:** [`AGENTS.md` § Error Handling](https://github.com/temporalio/temporal/blob/main/AGENTS.md#error-handling)
> "Use `logger.DPanic` for issues that are important but should not crash production"

### Use Standard Error Types (🟡 Should fix)

- New custom error types in handlers/services where a standard gRPC code would do. Use standard error types (`InvalidArgument`, `NotFound`, `FailedPrecondition`) over custom error types.

**Cite as:** Use Standard Error Types
**Source:** [`.github/copilot-instructions.md` § 5. Proper Error Handling](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#5-proper-error-handling)
> "Use standard error types (`InvalidArgument`, `NotFound`, `FailedPrecondition`) over custom error types"

### Mark Non-Retryable Errors (🟡 Should fix)

- Errors returned from task processing where the task shouldn't retry but the error isn't marked non-retryable. Mark errors as non-retryable when the task shouldn't retry in the queue.

**Cite as:** Mark Non-Retryable Errors
**Source:** [`.github/copilot-instructions.md` § 5. Proper Error Handling](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#5-proper-error-handling)
> "Mark errors as non-retryable when task shouldn't retry in queue"

### Use errors.AsType, Not errors.As (🟡 Should fix)

- Use of `errors.As(err, &target)` where the project utility `errors.AsType` is preferred.

**Cite as:** Use errors.AsType, Not errors.As
**Source:** [`.github/copilot-instructions.md` § 5. Proper Error Handling](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#5-proper-error-handling)
> "Use `errors.AsType` instead of `errors.As`"

### Validate Early In Handlers (🟡 Should fix)

- Argument validation performed inside business-logic helpers rather than at handler entry points. Validate early in handlers, not deep in business logic.

**Cite as:** Validate Early In Handlers
**Source:** [`.github/copilot-instructions.md` § 5. Proper Error Handling](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#5-proper-error-handling)
> "Validate early in handlers, not deep in business logic"

### Wrap Errors With Context (🟡 Should fix)

- `fmt.Errorf("%w", err)` or bare `return err` in a path where there is something interesting or informative to add. Wrap errors with context when it helps, e.g. `fmt.Errorf("multi-operation part 2: %w", err)`. Conversely, don't wrap with no added information.

**Cite as:** Wrap Errors With Context
**Source:** [`.github/copilot-instructions.md` § 5. Proper Error Handling](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#5-proper-error-handling)
> "Wrap errors with context when there's something interesting or informative to add, e.g. `fmt.Errorf(\"multi-operation part 2: %w\", err)`"

### Errors Must Not Be Ignored (🟡 Should fix)

- Returns from functions that produce an error which are silently dropped (`_ = f()`). Errors MUST be handled, not ignored.

**Cite as:** Errors Must Not Be Ignored
**Source:** [`AGENTS.md` § Best Practices](https://github.com/temporalio/temporal/blob/main/AGENTS.md#best-practices)
> "errors MUST be handled, not ignored"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs.
- Do not flag issues already caught by the repository's static linters (golangci-lint config, including `errcheck`).
- Ignore generated code, vendored dependencies, and lockfiles.
- Test files are out of scope for this agent.
- Internal-concept names leaking into user-facing errors are handled by the Proto & API Design agent.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line.

Every inline comment must end with a collapsible citation block pointing back to the original rule in this repository's own docs. Identify which `###` rule above the finding violates, then build the footer from that rule's `Cite as` / `Source` / quote — verbatim, no paraphrasing. Insert a blank line between the comment body and the `<details>` block.

```
<details><summary><em>Violates</em>: {Cite as value}</summary>

**Source:** [{path} § {section}]({anchored_url})

> {verbatim quote}

</details>
```

After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
