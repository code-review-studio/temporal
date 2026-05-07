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

These rules are derived from `.github/copilot-instructions.md` (section "8. Concurrency and Safety") in this repository. See the source for original wording and examples.

## What to flag

### I/O while holding locks (🔴 Must fix)

- Any database call, RPC, network round-trip, or other blocking I/O performed while a `sync.Mutex` or `sync.RWMutex` is held. This stalls every other caller waiting on the lock. Use side effect tasks instead.

### Proto messages aliased outside the workflow lock (🔴 Must fix)

- Returning a proto message pointer (or storing it in a struct field reachable outside the workflow lock) without first cloning. Use `common.CloneProto(...)` rather than returning the pointer directly. Aliased proto messages cause data races when the original is later mutated under the lock.
- Releasing a lock and then continuing to use a value that may be modified — clone before releasing.

### Lock-type selection (🟡 Should fix)

- `sync.RWMutex` used where `sync.Mutex` would do. Prefer `sync.Mutex` over `sync.RWMutex` almost always, except when reads are much more common than writes (>1000×) or readers hold the lock for significant time. If the new code adds an `RWMutex`, the PR should justify the read/write ratio.
- Reach-for-`atomic` patterns where a plain `sync.Mutex` would be clearer. Default to `sync.Mutex` for synchronization; atomics are an advanced tool for specific patterns or performance concerns.

### Mutable shared state (🟡 Should fix)

- New mutable shared state (especially proto message fields) where an immutable pattern would eliminate the need for synchronization. Prefer immutable data patterns for normal structs and especially proto messages to avoid data races and synchronization.

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (golangci-lint config).
- Ignore generated code, vendored dependencies, and lockfiles.
- Test files are out of scope for this agent — the Testify Suite & Goroutine Safety agent handles those.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line. After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
