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

Review changed test code for redundancy and over-specification. Flag unnecessary setup, assertions for things that can be assumed, unmotivated randomness, unnecessary exports, suite helpers that aren't subtest-safe, and accidental abstractions.

## What to flag

### Don't Assert What You Can Assume (🟡 Should fix)

- Assertions that re-verify postconditions of code that was just executed elsewhere (e.g., asserting `TerminateWorkflowExecution` worked inside a test of something else). You can assume standard mutable-state accessors and execution APIs work — test only what is under test.

**Cite as:** Don't Assert What You Can Assume
**Source:** [`.github/copilot-instructions.md` § 1. Remove Redundant Code (Highest Priority)](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#1-remove-redundant-code-highest-priority)
> "Don't add assertions for things you can assume work (e.g., \"You are not testing TerminateWorkflowExecution here, you can assume it works\")"

### No Unnecessary Test Activities (🟡 Should fix)

- Tests that exercise unrelated functionality as setup. Don't add unnecessary activities/complexity in tests — test only what you need.

**Cite as:** No Unnecessary Test Activities
**Source:** [`.github/copilot-instructions.md` § 1. Remove Redundant Code (Highest Priority)](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#1-remove-redundant-code-highest-priority)
> "Don't add unnecessary activities/complexity in tests - test only what you need"

### Question Test Randomness (🟡 Should fix)

- Randomized inputs (`rand.X`, fuzzed values) introduced without a clear hypothesis being tested. Question randomness in tests — test explicitly what you want.

**Cite as:** Question Test Randomness
**Source:** [`.github/copilot-instructions.md` § 1. Remove Redundant Code (Highest Priority)](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#1-remove-redundant-code-highest-priority)
> "Question randomness in tests - test explicitly what you want"

### Remove Redundant Nil Checks (🟡 Should fix)

- A nil check on a value immediately after the value was assigned a non-nil literal or returned from a function that already errored on failure. Remove redundant nil checks after you just set a value.

**Cite as:** Remove Redundant Nil Checks
**Source:** [`.github/copilot-instructions.md` § 1. Remove Redundant Code (Highest Priority)](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#1-remove-redundant-code-highest-priority)
> "Remove redundant nil checks after you just set a value"

### Don't Export What Isn't Used Externally (🟡 Should fix)

- Test helpers, fixture types, or constants exported (capitalized) when they're only used within the same test package. Do not export anything that doesn't need to be exported.

**Cite as:** Don't Export What Isn't Used Externally
**Source:** [`.github/copilot-instructions.md` § 1. Remove Redundant Code (Highest Priority)](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#1-remove-redundant-code-highest-priority)
> "Do not export anything that doesn't need to be exported"

### Subtest-Safe Suite Helpers (🟡 Should fix)

- New methods on a test suite type that capture suite-scoped state (channels, counters, mocks tied to `s.T()`) where the helper is then called from subtests. Don't create testsuite-level helpers that can't be safely used in subtests.

**Cite as:** Subtest-Safe Suite Helpers
**Source:** [`.github/copilot-instructions.md` § 4. Inline Code / Avoid Abstractions](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#4-inline-code--avoid-abstractions)
> "Don't create testsuite-level helpers that can't be safely used in subtests"

### No Constants For Single-Use Strings (🟢 Nit)

- New constants introduced for strings used exactly once. Repeat strings instead of adding constants for single use.

**Cite as:** No Constants For Single-Use Strings
**Source:** [`.github/copilot-instructions.md` § 4. Inline Code / Avoid Abstractions](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#4-inline-code--avoid-abstractions)
> "Repeat strings instead of adding constants for single use"

### Avoid Wrapper Types (🟢 Nit)

- New wrapper types or generic structs that wrap a single field or pass-through method. Avoid unnecessary wrapper types and generic structs.

**Cite as:** Avoid Wrapper Types
**Source:** [`.github/copilot-instructions.md` § 4. Inline Code / Avoid Abstractions](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#4-inline-code--avoid-abstractions)
> "Avoid unnecessary wrapper types and generic structs"

### No Dependencies For A Few Lines (🟢 Nit)

- New third-party dependencies pulled in for a small amount of logic. Just write 5 lines of code instead of adding more dependency bloat.

**Cite as:** No Dependencies For A Few Lines
**Source:** [`.github/copilot-instructions.md` § 4. Inline Code / Avoid Abstractions](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#4-inline-code--avoid-abstractions)
> "Don't add dependencies for 5 lines of code - \"just write 5 lines of code instead of adding more dependency bloat\""

## What to ignore

- Do not duplicate the built-in Correctness check.
- Do not flag issues already caught by the repository's static linters.
- Ignore generated code, vendored dependencies, and lockfiles.
- Don't flag pre-existing redundancy untouched by this PR — only what the diff adds or modifies.
- This agent overlaps with Testify Suite & Goroutine Safety; defer to that agent for goroutine, channel, and assertion-correctness issues.

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
