# Builder Dispatch Reference

Verified live against Codex CLI 0.139 (June 2026). Facts that correct common
misinformation: the model slug is `gpt-5.5` (not `gpt-5.5-codex`); `--search`
and `-a/--ask-for-approval` are TUI-only flags that `codex exec` rejects; Goal
Mode's real subcommands are bare `/goal`, `/goal pause|resume|clear`.

**Preflight:** run `codex --version`. Need >= 0.133. On the first dispatch in a
new environment, launch one canary lane before fanning out.

## Storage Contract

All loop artifacts live under `.scratch/architect-loop/` and are expected to be
ignored by Git. Git is still used for implementation diffs and local lane
commits.

Implementation slices are selected from local planning artifacts under
`.scratch/<feature-slug>/`: a `PRD.md` plus at least one
`issues/<NN>-<slug>.md`. Research reports and handoffs are inputs to those
files, not replacements for them, unless the human explicitly skips local
PRD/issues for the run.

Per slice:

```text
.scratch/architect-loop/state/<slice>/
  manifest.json
  spec.md
  gates.md
  freeze/gates.md
  freeze/gates.sha256
  dispatch/<lane>.prompt.md
  reports/<lane>.md
  runs/<lane>.jsonl

.scratch/architect-loop/worktrees/<slice>-<lane>/
```

The authoritative slice state stays in the main checkout. Each lane worktree
gets a copy of the slice packet under its own `.scratch/architect-loop/packet/`
and writes its raw report/run log under its own `.scratch/architect-loop/`.
After the run, the architect ingests that report back into the main checkout's
`.scratch/architect-loop/state/<slice>/reports/`.

## Gate Freeze

Freeze before dispatch:

```bash
STATE=.scratch/architect-loop/state/<slice>
mkdir -p "$STATE/freeze"
cp "$STATE/gates.md" "$STATE/freeze/gates.md"
(cd "$STATE/freeze" && shasum -a 256 gates.md > gates.sha256)
```

Verify before judgment:

```bash
STATE=.scratch/architect-loop/state/<slice>
(cd "$STATE/freeze" && shasum -a 256 -c gates.sha256)
diff -u "$STATE/freeze/gates.md" "$STATE/gates.md"
```

The checksum proves the frozen artifact was not altered. It does not replace
implementation diff review.

## Canonical Headless Dispatch

Write the builder block to a file first, then pass it via stdin (`-`). Do not
put a large prompt block in a shell argument; shells mangle quotes and can make
Codex hang waiting on stdin.

Always use a worktree, including one-lane slices:

```bash
REPO=<repo-root>
SLICE=<slice>
LANE=<NN>
BASE=<base-sha>
STATE="$REPO/.scratch/architect-loop/state/$SLICE"
WT="$REPO/.scratch/architect-loop/worktrees/$SLICE-$LANE"

git -C "$REPO" worktree add "$WT" -b "tdiaconescu/lane/$SLICE-$LANE" "$BASE"

mkdir -p "$WT/.scratch/architect-loop/packet/$SLICE"
cp "$STATE/spec.md" "$WT/.scratch/architect-loop/packet/$SLICE/spec.md"
cp "$STATE/gates.md" "$WT/.scratch/architect-loop/packet/$SLICE/gates.md"
cp "$STATE/freeze/gates.md" "$WT/.scratch/architect-loop/packet/$SLICE/frozen-gates.md"
cp "$STATE/manifest.json" "$WT/.scratch/architect-loop/packet/$SLICE/manifest.json"

mkdir -p "$WT/.scratch/architect-loop/reports" "$WT/.scratch/architect-loop/runs"

codex exec -C "$WT" --sandbox workspace-write \
  -m gpt-5.5 -c model_reasoning_effort="xhigh" \
  --json -o ".scratch/architect-loop/runs/$SLICE-$LANE.jsonl" \
  - < "$STATE/dispatch/$LANE.prompt.md"
```

Launch 2-4 lanes in parallel only after file-touch sets are checked for overlap.
Most slices should remain one lane.

A worktree's `.git` is a pointer file and the resolved git dir is sandbox
protected. Builders cannot commit or touch shared history; the architect stages
and commits only after post-flight checks pass.

## Post-Flight Checks

Per lane, from the main checkout:

```bash
REPO=<repo-root>
SLICE=<slice>
LANE=<NN>
BASE=<base-sha>
STATE="$REPO/.scratch/architect-loop/state/$SLICE"
WT="$REPO/.scratch/architect-loop/worktrees/$SLICE-$LANE"

mkdir -p "$STATE/reports" "$STATE/runs"
cp "$WT/.scratch/architect-loop/reports/$SLICE-$LANE.md" "$STATE/reports/$LANE.md"
cp "$WT/.scratch/architect-loop/runs/$SLICE-$LANE.jsonl" "$STATE/runs/$LANE.jsonl"

(cd "$STATE/freeze" && shasum -a 256 -c gates.sha256)
diff -u "$STATE/freeze/gates.md" "$STATE/gates.md"

git -C "$WT" status --porcelain --untracked-files=all
git -C "$WT" diff --name-only "$BASE"
git -C "$WT" ls-files --others --exclude-standard
git -C "$WT" diff "$BASE" -- <allowed-files...>
```

Fail the lane if any changed or untracked implementation file is outside the
lane's declared allowed file set. Ignored `.scratch` files are lane artifacts
and must not be staged.

Gate commands must not rewrite lock files, generated config, or unrelated
source files. If a Terraform/OpenTofu init or validate step wants to update
`.terraform.lock.hcl`, rerun with readonly lock behavior when supported, such as
`terraform init -backend=false -lockfile=readonly`. If the gate cannot run
without changing out-of-scope files, stop and record the blocker instead of
including the mutation in the lane.

For new allowed files, inspect them explicitly and then stage declared files
only:

```bash
git -C "$WT" add -- <allowed-files...>
git -C "$WT" diff --cached --name-only
git -C "$WT" diff --cached
git -C "$WT" commit -m "lane <NN>: <what>"
```

Never run `git add -A`. Never stage `.scratch`, `.architect`, `docs/gates`,
`docs/lanes`, generated PRDs, lane reports, run logs, or other loop artifacts.

## Integration

```bash
git -C "$REPO" checkout -b "tdiaconescu/slice/$SLICE" "$BASE"
# per passing lane, sequentially:
git -C "$REPO" merge --no-ff "tdiaconescu/lane/$SLICE-$LANE"
<run the gate commands>

# cleanup:
git -C "$REPO" worktree remove "$WT"
git -C "$REPO" branch -d "tdiaconescu/lane/$SLICE-$LANE"
```

A merge conflict means the lane plan was not disjoint. Kill the conflicting
lane and re-spec; do not hand-resolve builder conflicts.

Before pushing or opening a PR, the human may squash local lane/integration
commits into one commit. Loop artifacts remain ignored either way.

## Stall Detection And Rescue

A dispatched run is STALLED when its `--json` output file has not grown for
15+ minutes and the last event is an `in_progress` command_execution. Silent
gaps between events are normal model thinking; a shell command that should take
seconds sitting in flight for 15+ minutes is not.

Diagnose before killing: find the command's child under the Codex PID
(codex -> shell -> child). Kill the narrowest thing: the stuck child process,
not the whole Codex run. Kill the whole run only when the builder re-enters the
same hang or the worktree is broken; then discard the lane and re-dispatch.

Known sandbox hang sources: `asyncio.create_subprocess_exec` and anything built
on it, including Playwright browser launch and anyio subprocess pools, can fail
or hang under workspace-write while plain `subprocess.run` works. When a gate
needs a runtime the sandbox cannot execute, the builder records the exact
failure and the architect runs that gate at judgment time.

## Manual Alternative

Paste the builder block into an interactive `codex` session prefixed with
`/goal ` when the human wants to watch or steer the run.

## Builder Block Template

```text
Execute the architect spec below. Operating rules:

PHASE 0 - Before any code: reply with your plan and EVERY disagreement you have
with this spec, with reasons, citing real files in this repo. Silent compliance
is a failure. Silent scope additions are a failure. If you have no
disagreements, state what you checked before concluding the spec is sound.
Verify the named APIs/formats/versions against the live dependencies before
planning around them.

PHASE 1 - Treat shared contracts listed in the spec as frozen. The acceptance
gates in .scratch/architect-loop/packet/<slice>/frozen-gates.md are read-only
for you; editing gate artifacts or regenerating criteria fails the lane.

PHASE 2 - Build YOUR LANE ONLY: exactly the files listed in BOUNDARIES. You
are one of several parallel lane agents working in isolated worktrees; files
outside your lane belong to other agents. Touching them fails your lane.
No placeholder implementations. Search the codebase before implementing.
Full implementations only.

Verify your work by running the lane's gate commands and record verbatim
output. Do NOT commit. Do NOT delete or update lock files. If Terraform/OpenTofu
or another tool wants to rewrite a lock file outside your declared boundary,
record the exact command and failure/blocker instead of accepting the change. Do
NOT escalate privileges if a git command fails; record the exact error and
continue. Give every potentially long command an explicit timeout. If a runtime
will not start under the sandbox, record the exact failure in your lane report
and route around it; never busy-wait or retry in a loop.

When done, write your lane report to:
.scratch/architect-loop/reports/<slice>-<lane>.md

Use RAW results only: tables, numbers, command output, file paths, SHAs. No
interpretation, no "promising". Every status claim must be backed by a command
result from this run. Keep the report compact. End it with exactly one status
line:
STATUS: COMPLETE | COMPLETE_WITH_CONCERNS (list them) | BLOCKED (exact blocker
and what you tried).

Verdicts belong to the architect and human. Persist until your lane is fully
handled end-to-end; do not stop at analysis or partial fixes.

=== OBJECTIVE (and why) ===
...

=== OUTPUT FORMAT ===
...

=== TOOL GUIDANCE (verification commands; verify-against-reality list) ===
...

=== BOUNDARIES (may touch / must not touch / out of scope) ===
...

=== DISAGREEMENT RULINGS (from last session) ===
...

=== ACCEPTANCE GATES (frozen in .scratch/architect-loop/packet/<slice>/frozen-gates.md) ===
...
```

## Builder-Side Standing Setup

- `~/.codex/config.toml`: `model = "gpt-5.5"`.
- Parallelism is architect-orchestrated worktrees. It does not depend on
  Codex's experimental `[features] multi_agent` config.
- Repo `AGENTS.md`: exact build/test commands and repo gotchas only. The
  loop's PHASE rules stay in the dispatch block so they version with the skill.
