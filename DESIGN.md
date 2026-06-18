# DESIGN - Architect Loop Local-Scratch Fork

This fork keeps the upstream architect/builder separation but changes the
storage and planning contract for projects that do not want loop artifacts in
their repository history or PRs.

## Roles

| Role | Who | Owns |
|---|---|---|
| Architect | Claude Opus 4.8 in Claude Code | grilling, arbitration, gates, verification, integration |
| Builder | GPT-5.5 via `codex exec` | implementation inside one declared lane |
| Researcher | GPT-5.5 via read-only `codex exec` | raw sourced findings |
| Memory | `.scratch` plus Git history | local loop state and source diffs |

## Core Invariants

1. **The architect does not write implementation code.** It writes specs,
   gates, prompts, verdicts, and local planning artifacts.
2. **All loop artifacts are ignored.** PRDs, issues, gates, prompts, reports,
   run logs, handoffs, and worktrees live under `.scratch`.
3. **Git remains the code-diff authority.** Implementation changes are reviewed
   with `git diff`, `git status`, and `git ls-files`. SHA-256 checks only
   protect frozen local artifacts.
4. **Gates freeze before results exist.** The architect snapshots gates under
   `.scratch/architect-loop/state/<slice>/freeze/` and records a checksum
   before dispatch.
5. **Builders never grade their own work.** They report raw command output; the
   architect runs gates and reads the implementation diff.
6. **Disagreement is mandatory.** Builders must challenge the spec before
   coding, citing real files.
7. **Every builder runs in a fresh worktree.** This applies even to one-lane
   slices, so generated state and source edits stay isolated.
8. **Only declared implementation files can be staged.** `git add -A` is
   forbidden in the loop.
9. **No external issue tracker writes by default.** Local PRDs and issue slices
   are `.scratch` files unless the human explicitly asks for publication.
10. **Grilling precedes slicing when scope is not already settled.** The
    architect asks one question at a time, inspects code when possible, updates
    glossary/ADRs only when warranted, then distills to local PRD and issues.

## Artifact Layout

```text
.scratch/<feature-slug>/PRD.md
.scratch/<feature-slug>/issues/<NN>-<slug>.md

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
.scratch/architect-loop/research/<topic>/
```

## Gate Integrity Vs Code Diff Review

The upstream design committed gate files and used `git diff` to detect
tampering. This fork cannot rely on committed gate artifacts because generated
loop files must not reach the deployment repository.

Replacement:

- Gate tamper detection: compare `.scratch` gate files against a frozen copy and
  checksum.
- Implementation review: keep using Git diffs from the recorded `base_sha`.

These are intentionally separate. A checksum answers whether the gate artifact
changed. It does not show implementation changes and must never be used as a
replacement for source diff review.

## Review Branch Policy

Builders cannot commit. The architect creates one local review branch commit,
but only after:

- the frozen gate snapshot still matches,
- changed and untracked files are inside the lane's declared file set,
- the implementation diff has been read,
- all frozen gate commands have been run by the architect on the review branch,
- only declared implementation files are staged.

The review branch is created in the primary checkout from an updated project
base and follows repo Git instructions for branch names and commit messages. If
the primary checkout has user work, the loop stops and asks instead of hiding
the result in another worktree. The verdict is written after the final commit
SHA exists; amending or replacing that commit makes the verdict stale and
requires regeneration.

The loop stops after reporting the local review branch and commit. Pushing,
opening a PR, squashing, amending, or continuing follow-up work requires an
explicit human request. `.scratch` artifacts remain ignored either way.
