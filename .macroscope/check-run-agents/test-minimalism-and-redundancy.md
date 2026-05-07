---
title: Test Minimalism & Redundancy
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

Review changed test code for redundancy and over-specification. Flag unnecessary setup, assertions for things that can be assumed, unmotivated randomness, and unnecessarily exported test helpers.

These rules are derived from `.github/copilot-instructions.md` (sections "1. Remove Redundant Code (Highest Priority)" and "4. Inline Code / Avoid Abstractions") in this repository. See the source for original wording and examples.

## What to flag

### Unnecessary setup activities (🟡 Should fix)

- Tests that exercise unrelated functionality as setup (e.g., calling `TerminateWorkflowExecution` and asserting on it inside a test of something else). Don't add unnecessary activities/complexity in tests — test only what you need.
- Assertions that re-verify postconditions of code that was just executed elsewhere — "you can assume `TerminateWorkflowExecution` works". Don't add assertions for things you can assume work.

### Redundant nil checks (🟡 Should fix)

- A nil check on a value immediately after the value was assigned a non-nil literal or returned from a function that already errored on failure. Remove redundant nil checks after you just set a value.

### Unmotivated randomness (🟡 Should fix)

- Randomized inputs (`rand.X`, fuzzed values) introduced without a clear hypothesis being tested. Question randomness in tests — test explicitly what you want.

### Unnecessary exports (🟡 Should fix)

- Test helpers, fixture types, or constants exported (capitalized) when they're only used within the same test package. Do not export anything that doesn't need to be exported.

### Suite-level helpers unsafe in subtests (🟡 Should fix)

- New methods on a test suite type that capture suite-scoped state (channels, counters, mocks tied to `s.T()`) where the helper is then called from subtests. Don't create testsuite-level helpers that can't be safely used in subtests.

### Unnecessary abstractions (🟢 Nit)

- New constants introduced for strings used exactly once. Repeat strings instead of adding constants for single use.
- New wrapper types or generic structs that wrap a single field or pass-through method. Avoid unnecessary wrapper types and generic structs.
- New third-party dependencies pulled in for a few lines of logic — "just write 5 lines of code instead of adding more dependency bloat."

## What to ignore

- Do not duplicate the built-in Correctness check.
- Do not flag issues already caught by the repository's static linters.
- Ignore generated code, vendored dependencies, and lockfiles.
- Don't flag pre-existing redundancy untouched by this PR — only what the diff adds or modifies.
- This agent overlaps with Testify Suite & Goroutine Safety; defer to that agent for goroutine, channel, and assertion-correctness issues.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line. After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
