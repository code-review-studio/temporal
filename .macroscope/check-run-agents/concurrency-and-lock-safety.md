---
title: Concurrency & Lock Safety
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

Review changed Go code for lock misuse, data races, and proto-aliasing bugs. Flag patterns that risk shard stalls, races, or correctness violations in workflow state.

## What to flag

### No I/O Under Locks (🔴 Must fix)

- Database calls, RPCs, network round-trips, or other blocking I/O performed while a `sync.Mutex` or `sync.RWMutex` is held. This stalls every other caller waiting on the lock. Use side effect tasks instead.

**Cite as:** No I/O Under Locks
**Source:** [`.github/copilot-instructions.md` § 8. Concurrency and Safety](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#8-concurrency-and-safety)
> "Don't do IO while holding locks - use side effect tasks"

### Clone Protos Accessed Outside Workflow Lock (🔴 Must fix)

- Returning a proto message pointer (or storing it in a struct field reachable outside the workflow lock) without first cloning. Use `common.CloneProto(...)` rather than returning the pointer directly — aliased proto messages cause data races when the original is later mutated under the lock.

**Cite as:** Clone Protos Accessed Outside Workflow Lock
**Source:** [`.github/copilot-instructions.md` § 8. Concurrency and Safety](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#8-concurrency-and-safety)
> "Proto message fields accessed outside the workflow lock must be cloned, not aliased: use `common.CloneProto(...)` rather than returning the pointer directly."

### Clone Before Releasing Locks (🔴 Must fix)

- Releasing a lock and then continuing to use a value that may be modified by another caller under the lock. Clone the value before releasing.

**Cite as:** Clone Before Releasing Locks
**Source:** [`.github/copilot-instructions.md` § 8. Concurrency and Safety](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#8-concurrency-and-safety)
> "Clone data before releasing locks if it might be modified"

### Prefer sync.Mutex Over sync.RWMutex (🟡 Should fix)

- New `sync.RWMutex` usage where `sync.Mutex` would do. Prefer `sync.Mutex` almost always, except when reads are much more common than writes (>1000×) or readers hold the lock for significant time. If the PR adds an `RWMutex`, it should justify the read/write ratio.

**Cite as:** Prefer sync.Mutex Over sync.RWMutex
**Source:** [`.github/copilot-instructions.md` § 8. Concurrency and Safety](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#8-concurrency-and-safety)
> "Prefer `sync.Mutex` over `sync.RWMutex` almost always, except when reads are much more common than writes (>1000×) or readers hold the lock for significant time"

### Prefer Mutex Over Atomics (🟡 Should fix)

- Reach-for-`atomic` patterns where a plain `sync.Mutex` would be clearer. Default to `sync.Mutex` for synchronization; atomics are an advanced tool for specific patterns or performance concerns.

**Cite as:** Prefer Mutex Over Atomics
**Source:** [`.github/copilot-instructions.md` § 8. Concurrency and Safety](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#8-concurrency-and-safety)
> "Default to `sync.Mutex` for synchronization; atomics are an advanced tool for specific patterns or performance concerns"

### Prefer Immutable Data Patterns (🟡 Should fix)

- New mutable shared state (especially proto message fields) where an immutable pattern would eliminate the need for synchronization. Prefer immutable data patterns for normal structs and especially proto messages to avoid data races and synchronization.

**Cite as:** Prefer Immutable Data Patterns
**Source:** [`.github/copilot-instructions.md` § 8. Concurrency and Safety](https://github.com/temporalio/temporal/blob/main/.github/copilot-instructions.md#8-concurrency-and-safety)
> "Prefer immutable data patterns (for normal structs and especially proto messages) to avoid data races and synchronization"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (golangci-lint config).
- Ignore generated code, vendored dependencies, and lockfiles.
- Test files are out of scope for this agent — the Testify Suite & Goroutine Safety agent handles those.

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
