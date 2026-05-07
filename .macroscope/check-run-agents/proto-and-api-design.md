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

Review proto definitions and RPC handler signatures for design issues. Flag undocumented fields, naming convention violations, leaked internal concepts in user-facing errors, and signatures that should accept attribute structs.

These rules are derived from `.github/copilot-instructions.md` (section "7. API and Proto Design") in this repository. See the source for original wording and examples.

## What to flag

### Undocumented proto fields (🔴 Must fix)

- New proto fields added without doc comments. Document all proto fields with comments — proto changes ship to every Temporal SDK and are effectively permanent.

### Field naming (🔴 Must fix)

- New proto fields using `camelCase` rather than `snake_case`. Use proper field names: `request_id` not `requestId`, `schedule_time` not `scheduledTime`.

### Internal concepts in user-facing errors (🔴 Must fix)

- User-facing error messages (returned via gRPC to clients) that name internal concepts, internal type names, or internal flag identifiers. Don't expose internal concepts in user-facing errors: "LowCardinalityKeyword is not a user facing concept."

### Growing function signatures (🟡 Should fix)

- RPC handlers, builders, or event constructors gaining additional scalar parameters where an attributes/options struct is the established pattern. Accept event attributes structs instead of growing function signatures.

### int/string for closed value spaces (🟡 Should fix)

- New fields typed as `int32`, `int64`, or `string` whose value space is closed (a known set of states, kinds, or modes). Prefer enums over int/string for well-known values.

## What to ignore

- Do not duplicate the built-in Correctness check.
- Do not flag issues already caught by buf lint or other proto linters configured in the repo.
- Ignore generated `*.pb.go` files, vendored dependencies, and lockfiles.
- Existing field naming inconsistencies — only flag fields added or renamed in this PR.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line. After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
