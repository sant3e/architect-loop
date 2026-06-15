# Research Fan-Out Reference

Read this only when a slice-scale research trigger fires. The fan-out uses
Codex as parallel web-research subagents: read-only, live search, and no repo
writes. The architect keeps all judgment, verifies load-bearing claims, and
writes local `.scratch` artifacts.

## Fan Out

Decompose the question into 3-5 narrow, non-overlapping research questions.
Cover different angles, not the same angle multiple times: official
docs/reference, changelog/breaking changes, community failure reports,
alternatives/comparisons, security/operational constraints.

One fresh `codex exec` per question, all launched in parallel:

```bash
REPO=<repo-root>
SLICE=<slice>
mkdir -p "$REPO/.scratch/architect-loop/research/$SLICE"

codex exec -C "$REPO" --sandbox read-only -c web_search="live" \
  -m gpt-5.5 -c model_reasoning_effort="high" \
  -o ".scratch/architect-loop/research/$SLICE/<NN>-<topic>.md" \
  - < ".scratch/architect-loop/research/$SLICE/<NN>-<topic>.prompt.md"
```

Write each research block to a `.prompt.md` file and pass it via stdin (`-`),
never as a shell argument.

- `--sandbox read-only`: researchers never write implementation files.
- `-c web_search="live"`: forces fresh results. If the canary complains, use
  the version ladder from `dispatch.md`.
- Effort `high`, not `xhigh`; research is coverage work.
- Scope each researcher to <=5 subjects and add hard context rules. A
  researcher that fills its context window may die without writing output.

## Research Block Template

```text
You are a web research agent. Answer ONE question. Do not write code, do not
make recommendations; judgment belongs to the architect.

QUESTION: <one narrow question>

OUTPUT FORMAT - a markdown report:
- Findings as bullets. EVERY finding carries: source URL, source date if shown,
  the exact figure or a short direct quote, and a confidence tag
  (high = primary source / med = reputable secondary / low = single blog or
  forum post).
- Prefer primary sources: official docs, changelogs, release notes, source
  code.
- Record exact version numbers and dates.
- When sources disagree, report the disagreement; do not resolve it.
- If you cannot find evidence, write NOT FOUND. Never infer.
- End with the 2-3 findings most likely to change an implementation decision.
```

## Gather

1. Read findings under `.scratch/architect-loop/research/<slice>/`.
2. Identify load-bearing claims: API shape, version constraint, limit,
   deprecation, security constraint, operational behavior.
3. Verify each against a second independent source or the live dependency.
4. Discard single-source low-confidence claims or mark them as open questions.
5. Write distilled decisions either into:
   - `.scratch/<feature-slug>/PRD.md`, when shaping product/feature scope
   - `.scratch/architect-loop/state/<slice>/research.md`, when the facts are
     slice-local
6. Do not commit research files, PRDs, or raw findings unless the human
   explicitly asks.
