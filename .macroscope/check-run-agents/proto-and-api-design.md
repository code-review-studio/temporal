---
title: Proto & API Design
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "**/*.proto"
  - "proto/**"
  - "api/**"
  - "service/frontend/**"
  - "service/history/**"
exclude:
  - "**/*_test.go"
  - "vendor/**"
  - "**/*.pb.go"
  - "**/*_generated.go"
conclusion: neutral
---

Review proto definitions and RPC handler signatures for design issues. Flag undocumented fields, naming-convention violations, leaked internal concepts in user-facing errors, growing signatures, and weak typing for closed value spaces.

## What to flag

### Document All Proto Fields (🔴 Must fix)

- New proto fields added without ANY preceding doc comment (`// ...`). Flag *only* fields with zero accompanying comment — "vague but present" is not a violation of this rule. Comment quality is out of scope for this agent.

**Cite as:** Document All Proto Fields
**Source:** [`.github/copilot-instructions.md` § 7. API and Proto Design](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#7-api-and-proto-design)
> "Document all proto fields with comments"

### snake_case Proto Field Names (🔴 Must fix)

- New proto fields using `camelCase` rather than `snake_case`. Use proper field names: `request_id` not `requestId`, `schedule_time` not `scheduledTime`.

**Cite as:** snake_case Proto Field Names
**Source:** [`.github/copilot-instructions.md` § 7. API and Proto Design](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#7-api-and-proto-design)
> "Use proper field names: `request_id` not `requestId`, `schedule_time` not `scheduledTime`"

### No Internal Concepts In User-Facing Errors (🔴 Must fix)

- User-facing error messages (returned via gRPC to clients) that name internal concepts, internal type names, or internal flag identifiers. Internal type names should not appear in messages returned to clients.

**Cite as:** No Internal Concepts In User-Facing Errors
**Source:** [`.github/copilot-instructions.md` § 7. API and Proto Design](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#7-api-and-proto-design)
> "Don't expose internal concepts in user-facing errors: \"LowCardinalityKeyword is not a user facing concept\""

### Prefer Enums Over int/string (🟡 Should fix)

- New fields typed as `int32`, `int64`, or `string` whose value space is closed (a known set of states, kinds, or modes). Prefer enums over int/string for well-known values.

**Cite as:** Prefer Enums Over int/string
**Source:** [`.github/copilot-instructions.md` § 7. API and Proto Design](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#7-api-and-proto-design)
> "Prefer enums over int/string for well-known values"

### Accept Attributes Structs, Don't Grow Signatures (🟡 Should fix)

- RPC handlers, builders, or event constructors gaining additional scalar parameters where an attributes/options struct is the established pattern. Accept event attributes structs instead of growing function signatures.

**Cite as:** Accept Attributes Structs, Don't Grow Signatures
**Source:** [`.github/copilot-instructions.md` § 7. API and Proto Design](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#7-api-and-proto-design)
> "Accept event attributes structs instead of growing function signatures"

## What to ignore

- Do not duplicate the built-in Correctness check.
- Do not flag issues already caught by buf lint or other proto linters configured in the repo.
- Ignore generated `*.pb.go` files, vendored dependencies, and lockfiles.
- Existing field naming inconsistencies — only flag fields added or renamed in this PR.

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
