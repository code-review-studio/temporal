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

These rules are derived from `.github/copilot-instructions.md` (section "3. Testify Suite Correctness and Reliability"), `AGENTS.md` (section "Testing"), and `docs/development/testing.md` in this repository. See the source for original wording and examples.

## What to flag

### Testify assertions in goroutines (🔴 Must fix)

- Any call to `s.NoError`, `s.Equal`, `require.NoError`, `assert.NoError`, or any other testify assertion **inside a `go func()` body**. If the goroutine outlives the test, the assertion panics the binary with `panic: Fail in goroutine after TestXxx has completed`. Move the assertion to the test goroutine or use a buffered error channel.
- Any suite assertion (`s.NoError`, `s.Equal`, etc.) called from a goroutine — same failure mode.
- Any `s.T()` reference inside a subtest — use the subtest's own `t` parameter. Inside `EventuallyWithT`, use that block's `t`, not `s.T()`.

### Hangs and missing context cancellation (🔴 Must fix)

- Any `<-ch` that isn't inside a `select` with a `ctx.Done()` fallback. If the sender never sends, the receive hangs indefinitely.
- `EventuallyWithT` (or similar) waiting on a condition driven by a background goroutine where the goroutine's timeout is shorter than the `EventuallyWithT` deadline — the condition will never be satisfied and the wait will hang until its own deadline.

### Time-based ordering (🔴 Must fix)

- Any use of `time.Sleep` or `time.Since(start) > threshold` to enforce ordering between operations. Use channels, `sync.WaitGroup`, or `EventuallyWithT` instead. `time.Sleep` is forbidden by the project linter — prefer `require.Eventually`.

### Single-shot helper goroutines (🔴 Must fix)

- Patterns like `go s.someHelper(ctx, ...)` where the goroutine runs exactly once and the test then immediately waits for something that helper was supposed to cause. If the operation can fail transiently (network, tight deadline, busy CI), the single attempt may fail silently and the wait will never succeed. Either loop the goroutine until `ctx.Done()`, or check that the operation succeeded before proceeding.
- A goroutine launched to maintain a precondition for later assertions (e.g., keeping pollers active so a deployment version gets registered) that runs once instead of looping until context cancellation. A single attempt that times out exits silently, leaving downstream `Eventually`/propagation waits to hang.

### Cross-test process safety (🔴 Must fix)

- Writes to package-level or global variables inside tests. Parallel tests share the same process; thread values through function parameters instead.

### Error type assertions and silent discards (🟡 Should fix)

- Single-value type assertions on errors (`err.(*T)`) — these panic instead of failing the test when the type doesn't match. Use `errors.As` with a guarded return, or `require.ErrorAs(t, err, &specificErr)` for specific error type checks.
- Silent error discards on preconditions: `_, _ = f()` where `f()` failing invalidates the rest of the test. Surface the error or loop until it succeeds.

### Assertion style (🟡 Should fix)

- Use of `assert.X` / `protoassert.X` where `require.X` / `protorequire.X` would be appropriate — `assert` lets a failed test continue, producing confusing cascading errors. Prefer `require` unless there is a clear reason to continue.
- For error assertions in testify suites, use `s.Require().NoError(err)` instead of `s.NoError(err)`.
- For float comparisons in tests, use `InDelta` or `InEpsilon` instead of `Equal`.

### Eventually documentation (🟢 Nit)

- `Eventually` / `EventuallyWithT` calls without a comment explaining why eventual consistency is needed. Add a brief comment so future readers know the wait is intentional.

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (golangci-lint config, including `testifylint`).
- Ignore generated code, vendored dependencies, and lockfiles.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line. After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
