# Builder Dispatch Reference

Verified live against Codex CLI 0.139 (June 2026). Facts that correct common
misinformation: the model slug is `gpt-5.5` (not `gpt-5.5-codex`); `--search`
and `-a/--ask-for-approval` are TUI-only flags that `codex exec` rejects; Goal
Mode's real subcommands are bare `/goal`, `/goal pause|resume|clear`.

**Preflight:** run `codex --version`. Need >= 0.133. On the first dispatch in a
new environment, launch one canary lane before fanning out.

## Storage Contract

All loop artifacts live under `.scratch/architect-loop/` and are expected to be
ignored by Git. Git is still used for implementation diffs and the local review
branch commit.

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
  runs/<lane>.stderr.log
  runs/<lane>.last.md
  verdict.md

.scratch/architect-loop/worktrees/<slice>-<lane>/
```

The authoritative slice state stays in the main checkout. Each lane worktree
gets a copy of the slice packet under its own `.scratch/architect-loop/packet/`
and writes its raw report, Codex JSONL event stream, stderr log, and last
message under its own `.scratch/architect-loop/`.
After the run, the architect ingests that report back into the main checkout's
`.scratch/architect-loop/state/<slice>/reports/` and ingests the run artifacts
back into `.scratch/architect-loop/state/<slice>/runs/`.

## Gate Freeze

Freeze before dispatch:

```bash
STATE=.scratch/architect-loop/state/<slice>
mkdir -p "$STATE/freeze"
command cp "$STATE/gates.md" "$STATE/freeze/gates.md"
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
RUN_JSONL="$WT/.scratch/architect-loop/runs/$SLICE-$LANE.jsonl"
STDERR_LOG="$WT/.scratch/architect-loop/runs/$SLICE-$LANE.stderr.log"
LAST_MSG="$WT/.scratch/architect-loop/runs/$SLICE-$LANE.last.md"

git -C "$REPO" worktree add "$WT" -b "tdiaconescu/lane/$SLICE-$LANE" "$BASE"

mkdir -p "$WT/.scratch/architect-loop/packet/$SLICE"
command cp "$STATE/spec.md" "$WT/.scratch/architect-loop/packet/$SLICE/spec.md"
command cp "$STATE/gates.md" "$WT/.scratch/architect-loop/packet/$SLICE/gates.md"
command cp "$STATE/freeze/gates.md" "$WT/.scratch/architect-loop/packet/$SLICE/frozen-gates.md"
command cp "$STATE/manifest.json" "$WT/.scratch/architect-loop/packet/$SLICE/manifest.json"

mkdir -p "$WT/.scratch/architect-loop/reports" "$WT/.scratch/architect-loop/runs"

codex exec -C "$WT" --sandbox workspace-write \
  -m gpt-5.5 -c model_reasoning_effort="xhigh" \
  --json -o "$LAST_MSG" \
  - < "$STATE/dispatch/$LANE.prompt.md" > "$RUN_JSONL" 2> "$STDERR_LOG"
```

`--json` is the event stream and must be captured from stdout. `-o` is only the
last assistant message; do not name it `.jsonl`. Keep stderr separate so shell
warnings and tool errors cannot corrupt the JSONL event stream.

If required local skill files are linked outside the worktree or may be blocked
by the builder sandbox, copy task-sized excerpts into
`$WT/.scratch/architect-loop/packet/$SLICE/skills/` and reference those packet
paths in the dispatch prompt.

Launch 2-4 lanes in parallel only after file-touch sets are checked for overlap.
Most slices should remain one lane.

A worktree's `.git` is a pointer file and the resolved git dir is sandbox
protected. Builders cannot commit or touch shared history; the architect stages
and commits only on the final review branch after post-flight and frozen gates
pass.

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
command cp "$WT/.scratch/architect-loop/reports/$SLICE-$LANE.md" "$STATE/reports/$LANE.md"
command cp "$WT/.scratch/architect-loop/runs/$SLICE-$LANE.jsonl" "$STATE/runs/$LANE.jsonl"
command cp "$WT/.scratch/architect-loop/runs/$SLICE-$LANE.stderr.log" "$STATE/runs/$LANE.stderr.log"
command cp "$WT/.scratch/architect-loop/runs/$SLICE-$LANE.last.md" "$STATE/runs/$LANE.last.md"
test -s "$STATE/runs/$LANE.jsonl"
python3 -c 'import json,sys; lines=[l for l in open(sys.argv[1]) if l.strip()]; [json.loads(l) for l in lines]; assert lines' \
  "$STATE/runs/$LANE.jsonl"

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

Fail or re-run the lane if the JSONL event stream is missing, empty, or invalid.
The last message file is useful for summaries, but it is not a substitute for
the JSONL run log.

Fail or return the lane for clarification if its report does not list the
required local skills it loaded. Terraform/OpenTofu lanes must load
`terraform-skill`. Lanes that change AWS providers/resources must load
`aws-stuff`; lanes that run AWS SSO/login, AWS CLI, live AWS inspection, or
Terraform plans/refreshes against AWS-backed state/data sources must also load
`aws-stuff` for operational context.

Gate commands must match the frozen command or an explicitly frozen allowed
variant. Record actual command, cwd/env, exit status, output path, post-run
source diff, and PASS / FAIL / BLOCKED / DEVIATED for each gate. Do not call a
gate PASS if a subcommand was skipped, an env var changed, `-lock=false` was
added ad hoc, or the command rewrote lock files, generated config, or unrelated
source files. If a Terraform/OpenTofu init or validate step wants to update
`.terraform.lock.hcl`, rerun with readonly lock behavior when supported, such as
`terraform init -backend=false -lockfile=readonly`. If the gate cannot run
without changing out-of-scope files, stop and record the blocker instead of
including the mutation in the lane.

For new allowed files, inspect them explicitly and use intent-to-add only when
needed to make the lane patch complete. Do not commit in lane worktrees:

```bash
# only if the lane created new allowed files:
git -C "$WT" add -N -- <new-allowed-files...>
git -C "$WT" diff --binary "$BASE" -- <allowed-files...>
```

Never run `git add -A`. Never stage `.scratch`, `.architect`, `docs/gates`,
`docs/lanes`, generated PRDs, lane reports, run logs, or other loop artifacts.

## Review Branch Finalization

```bash
REPO=<repo-root>
SLICE=<slice>
BASE=<slice-base-sha>
STATE="$REPO/.scratch/architect-loop/state/$SLICE"
BASE_BRANCH=<project-base-branch>
REVIEW_BRANCH=<project-rule-compliant-feature-branch>

git -C "$REPO" fetch --prune origin "$BASE_BRANCH"
UPDATED_BASE="$(git -C "$REPO" rev-parse "origin/$BASE_BRANCH")"
git -C "$REPO" status --short --untracked-files=all
git -C "$REPO" diff --quiet
git -C "$REPO" diff --cached --quiet
test -z "$(git -C "$REPO" ls-files --others --exclude-standard)"
git -C "$REPO" switch -c "$REVIEW_BRANCH" "$UPDATED_BASE"

mkdir -p "$STATE/patches"

# per passing lane, sequentially:
LANE=<NN>
WT="$REPO/.scratch/architect-loop/worktrees/$SLICE-$LANE"
# only if the lane created new allowed files:
git -C "$WT" add -N -- <new-allowed-files...>
git -C "$WT" diff --binary "$BASE" -- <allowed-files...> > "$STATE/patches/$LANE.patch"
git -C "$REPO" apply --check "$STATE/patches/$LANE.patch"
git -C "$REPO" apply --index "$STATE/patches/$LANE.patch"

git -C "$REPO" diff --cached --name-only
git -C "$REPO" diff --cached
git -C "$REPO" diff --cached --binary > "$STATE/final-candidate.pre-gates.patch"
<run every frozen gate command in "$REPO">
git -C "$REPO" status --short --untracked-files=all
git -C "$REPO" diff --quiet
test -z "$(git -C "$REPO" ls-files --others --exclude-standard)"
git -C "$REPO" diff --cached --binary > "$STATE/final-candidate.post-gates.patch"
diff -u "$STATE/final-candidate.pre-gates.patch" "$STATE/final-candidate.post-gates.patch"
test -s "$STATE/commit-message.txt"
! grep -Ei '^(Co-Authored-By:|Generated[- ]with|Generated[- ]by)' "$STATE/commit-message.txt"
git -C "$REPO" commit -F "$STATE/commit-message.txt"
git -C "$REPO" rev-parse HEAD

# cleanup:
git -C "$REPO" worktree remove "$WT"
```

Choose `REVIEW_BRANCH` from repo Git instructions; default to
`tdiaconescu/<feature-slug>` only when the repo has no stricter rule. If that
branch already exists, create a brand new suffixed branch or ask; never
force-reset it without explicit human approval.

The final review branch belongs in the primary checkout so the human is left on
the branch to evaluate. Before `git switch -c`, inspect the status output. If
the primary checkout contains tracked changes or untracked implementation files,
stop and ask instead of stashing, overwriting, or creating a hidden review
worktree. Ignored `.scratch` loop artifacts are expected.

If patch apply, final gates, post-gate cleanliness checks, or staged-patch
comparison fail, stop without committing and report the blocker. If the base
update fails, stop and ask instead of using a stale base. Do not hand-resolve
builder conflicts.

Create `$STATE/commit-message.txt` from the repo's commit rules and inspect it
before committing. Do not include AI co-author trailers, generated-by footers,
or tool branding unless the repo explicitly requires them.

After the commit, write `$STATE/verdict.md` with review branch, updated base
SHA, final commit SHA, checkout path, gate ledger, and changed implementation
files. If the commit is amended or replaced after that, regenerate
`verdict.md`; otherwise it is stale. Stop after reporting the local branch,
commit SHA, and primary checkout path. Do not push, open a PR, squash, amend,
or continue follow-up work unless the human asks.

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

REQUIRED LOCAL SKILLS - Before PHASE 0:
Read every skill file listed in the REQUIRED LOCAL SKILLS section below. If the
slice is Terraform/OpenTofu and `terraform-skill` is available, you must read it
and apply its response contract, risk categories, validation rules, and rollback
expectations. If the lane changes AWS providers/resources or runs AWS
SSO/login, AWS CLI, live AWS inspection, or Terraform plans/refreshes against
AWS-backed state/data sources and `aws-stuff` is available, you must read it and
apply its credential, profile, query filtering, preview, and
destructive-operation rules. If a required skill path is missing or unreadable,
state that in PHASE 0 and mark the lane
COMPLETE_WITH_CONCERNS or BLOCKED depending on risk.

PHASE 0 - Before any code: reply with your plan and EVERY disagreement you have
with this spec, with reasons, citing real files in this repo. Silent compliance
is a failure. Silent scope additions are a failure. If you have no
disagreements, state what you checked before concluding the spec is sound.
Verify the named APIs/formats/versions against the live dependencies before
planning around them.

PHASE 1 - Treat shared contracts listed in the spec as frozen. The slice packet
and frozen gates under .scratch/architect-loop/packet/<slice>/ are read-only
for you; editing packet artifacts or regenerating criteria fails the lane.

PHASE 2 - Build YOUR LANE ONLY. Implementation changes may touch only the
implementation files listed in BOUNDARIES. Required loop artifacts are outside
that implementation boundary: write the lane report under
.scratch/architect-loop/reports/, and the dispatch wrapper writes run logs
under .scratch/architect-loop/runs/. Keep all .scratch artifacts ignored and
untracked. Files outside your lane belong to other agents; touching them fails
your lane. No placeholder implementations. Search the codebase before
implementing. Full implementations only.

Verify your work by running the lane's gate commands and record verbatim
output plus a gate ledger: frozen command, actual command, cwd/env, exit status,
output path, post-run source diff, and PASS / FAIL / BLOCKED / DEVIATED. Do NOT
commit. Do NOT delete or update lock files. If Terraform/OpenTofu or another
tool wants to rewrite a lock file outside your declared boundary, record the
exact command and failure/blocker instead of accepting the change. Do NOT
escalate privileges if a git command fails; record the exact error and continue.
Give every potentially long command an explicit timeout. If a runtime will not
start under the sandbox, record the exact failure in your lane report and route
around it; never busy-wait or retry in a loop.

When done, write your lane report to:
.scratch/architect-loop/reports/<slice>-<lane>.md

Use RAW results only: tables, numbers, command output, file paths, SHAs. No
interpretation, no "promising". Every status claim must be backed by a command
result from this run. Include `Skills used:` and `Extra context loaded:` lines
covering every required local skill. Keep the report compact. End it with
exactly one status line:
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

=== REQUIRED LOCAL SKILLS ===
...

=== BOUNDARIES (implementation allowlist / forbidden implementation files / allowed artifacts / out of scope) ===
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
