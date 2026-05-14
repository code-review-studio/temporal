---
title: Testify Suite & Goroutine Safety
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "**/*_test.go"
exclude:
  - "vendor/**"
  - "**/*.pb.go"
  - "**/*_generated.go"
conclusion: neutral
---

Review changed test code for goroutine-safety bugs and testify-suite correctness issues. Flag patterns that cause test panics, hangs, silent failures, or cross-test data races.

## What to flag

### Testify Assertions In Goroutines (🔴 Must fix)

- Any call to `s.NoError`, `s.Equal`, `require.NoError`, `assert.NoError`, or any other testify assertion **inside a `go func()` body**. If the goroutine outlives the test, the assertion panics the binary with `panic: Fail in goroutine after TestXxx has completed`. Move the assertion to the test goroutine or use a buffered error channel.

**Cite as:** Testify Assertions In Goroutines
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Never call testify assertions (`s.NoError`, `s.Equal`, `require.NoError`, even `assert.NoError`) inside a `go func()` — if the goroutine outlives the test, the assertion panics the binary with `panic: Fail in goroutine after TestXxx has completed`. Move assertions to the test goroutine or use a buffered error channel."

### Subtest Uses `t`, Not `s.T()` (🟡 Should fix)

- `s.T()` references inside subtests or `EventuallyWithT` blocks. Use the subtest's own `t` parameter (or `EventuallyWithT`'s `t`) so failures are scoped to the right test.

**Cite as:** Subtest Uses `t`, Not `s.T()`
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Never use `s.T()` in subtests - use the subtest's `t` parameter"

### Channel Receives Need ctx.Done() (🔴 Must fix)

- Any `<-ch` that isn't inside a `select` with a `ctx.Done()` fallback. If the sender never sends, the receive hangs indefinitely. Always provide a context cancellation fallback.

**Cite as:** Channel Receives Need ctx.Done()
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Any `<-ch` that isn't inside a `select` with `ctx.Done()` will hang indefinitely if the sender never sends. Always provide a context cancellation fallback."

### EventuallyWithT Timeout Must Outlast Background Op (🔴 Must fix)

- `EventuallyWithT` (or similar) waiting on a condition driven by a background goroutine where the goroutine's timeout is shorter than the `EventuallyWithT` deadline. If the background op times out first, the condition will never be satisfied and the wait hangs until its own deadline.

**Cite as:** EventuallyWithT Timeout Must Outlast Background Op
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "When using `EventuallyWithT` (or similar) to wait for a condition driven by a background goroutine, ensure the goroutine's timeout is longer than the `EventuallyWithT` deadline — if the background op times out first, the condition will never be satisfied and the wait will hang until its own deadline."

### No time.Sleep For Ordering (🔴 Must fix)

- Any use of `time.Sleep` or `time.Since(start) > threshold` to enforce ordering between operations. Use channels, `sync.WaitGroup`, or `EventuallyWithT` instead. (`time.Sleep` is forbidden by the project linter — prefer `require.Eventually`.)

**Cite as:** No time.Sleep For Ordering
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Never use `time.Sleep` or `time.Since(start) > threshold` to enforce ordering — use channels, `sync.WaitGroup`, or `EventuallyWithT` instead."

### Loop Precondition Goroutines (🔴 Must fix)

- `go s.someHelper(ctx, ...)` patterns where the goroutine runs exactly once and the test then waits for something the helper was supposed to cause. If the operation can fail transiently (network, tight deadline, busy CI), the single attempt may fail silently and the wait will never succeed. Either loop the goroutine until `ctx.Done()`, or check that the operation succeeded before proceeding.

**Cite as:** Loop Precondition Goroutines
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Be suspicious of `go s.someHelper(ctx, ...)` calls where the goroutine runs exactly once and the test then immediately waits for something that helper was supposed to cause. If the operation can fail transiently (network, tight deadline, busy CI), the single attempt may fail silently and the wait will never succeed. Either loop the goroutine until `ctx.Done()`, or check that the operation succeeded before proceeding."

### No Package-Level Writes In Tests (🔴 Must fix)

- Writes to package-level or global variables inside tests. Parallel tests share the same process; thread values through function parameters instead.

**Cite as:** No Package-Level Writes In Tests
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Never write to package-level or global variables in tests — parallel tests share the same process; thread values through function parameters instead."

### No Single-Value Error Type Assertions (🟡 Should fix)

- Single-value type assertions on errors (`err.(*T)`) — these panic instead of failing the test when the type doesn't match. Use `errors.As` with a guarded return, or `require.ErrorAs(t, err, &specificErr)` for specific error type checks.

**Cite as:** No Single-Value Error Type Assertions
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Do not use single-value type assertions on errors (`err.(*T)`); this panics instead of failing the test when the type doesn't match. Use `errors.As` with a guarded return."

### Don't Silently Discard Errors (🟡 Should fix)

- Silent error discards on preconditions: `_, _ = f()` where `f()` failing invalidates the rest of the test. Surface the error or loop until it succeeds.

**Cite as:** Don't Silently Discard Errors
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Do not silently discard errors from precondition operations with `_, _ = f()` — if `f()` failing invalidates the rest of the test, surface the error or loop until it succeeds."

### Prefer require Over assert (🟡 Should fix)

- Use of `assert.X` / `protoassert.X` / `s.NoError` / `s.Equal` where the failure should halt the test. `assert` lets a failed test continue, producing confusing cascading errors. Prefer `require` (or `s.Require()`); use plain `assert` only when there's a clear reason to continue.

**Cite as:** Prefer require Over assert
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Prefer `require` over `assert` - it's rarely useful to continue a test after a failed assertion"

### Document Why Eventually (🟢 Nit)

- `Eventually` / `EventuallyWithT` calls without a comment explaining why eventual consistency is needed. Add a brief comment so future readers know the wait is intentional.

**Cite as:** Document Why Eventually
**Source:** [`.github/copilot-instructions.md` § 3. Testify Suite Correctness and Reliability](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#3-testify-suite-correctness-and-reliability)
> "Add comments explaining why `Eventually` is needed (e.g., eventual consistency)"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (golangci-lint config, including `testifylint`).
- Ignore generated code, vendored dependencies, and lockfiles.

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
