# architect-loop

Run **Claude as your architect** and **GPT Codex as your builder** — on subscriptions,
not API tokens. The repo is the memory, the architect is the judgment, the builder is
the hands.

```
Claude (architect)          Codex (builder)             The repo (memory)
──────────────────          ───────────────             ─────────────────
reads docs/HANDOFF.md  ──►  PHASE 0: plan + mandatory   docs/HANDOFF.md
rules on disagreements      disagreements               docs/ frozen contracts
judges RAW results          PHASE 1: freeze contracts   docs/ frozen gates
writes next slice spec      PHASE 2: lane agents +
freezes gates BEFORE        reviewer agent, commit,
results exist          ◄──  update HANDOFF.md (raw
                            numbers only)
```

The loop: **the architect thinks, the builder builds, the repo remembers, you judge.**
Architect sessions cost minutes because state lives in `docs/HANDOFF.md`, not in chat.
Builder sessions run for hours under Codex Goal Mode.

## The five rules

1. Repo docs are the memory. **Not in `HANDOFF.md` = didn't happen.**
2. The builder **never grades its own work** — raw tables and numbers only.
3. **Disagreement is mandatory.** Silent compliance from the builder = failure.
4. Acceptance gates are **frozen before results exist**, never edited after.
5. Architect time is for **judgment**; builder time is for **typing**.

## Requirements

- [Claude Code](https://claude.com/claude-code) (any paid plan — the architect)
- [Codex CLI](https://developers.openai.com/codex/cli) v0.133+ signed into a ChatGPT
  subscription (the builder): `npm i -g @openai/codex@latest`

No API keys. Both halves run on flat-rate subscriptions.

## Install

Global (skill available in every repo):

```powershell
# Windows
Copy-Item -Recurse skills\architect "$env:USERPROFILE\.claude\skills\architect"
```

```bash
# macOS / Linux
cp -r skills/architect ~/.claude/skills/architect
```

Or per-project: copy `skills/architect/` into `<repo>/.claude/skills/architect/`.

## Usage

One architect session per work block:

```
/architect
```

Claude reads `docs/HANDOFF.md` (creating it from the template on first run), rules on
every open disagreement, judges the latest raw results against the frozen gates, writes
the next one-PR-sized slice spec, and ends with a paste-ready builder block.

Then either:

- **Manual** — paste the block into an interactive `codex` session. It opens with
  `/goal`, so Codex works the slice autonomously for hours.
- **Auto** — say `/architect run it` and Claude launches
  `codex exec --full-auto "<block>"` in the background, then verifies the handoff was
  updated and that the builder raised disagreements.

After the builder finishes, the next `/architect` session judges what landed in
`HANDOFF.md` — and only what landed in `HANDOFF.md`.

## What's in the box

| File | Purpose |
|------|---------|
| [skills/architect/SKILL.md](skills/architect/SKILL.md) | The architect role: hard rules, procedure, builder block template |
| [skills/architect/HANDOFF.template.md](skills/architect/HANDOFF.template.md) | The repo-memory file the builder maintains |

## Credits

Workflow concept from the "architect/builder" pattern circulating in the AI-coding
community: spend the strong reasoning model's time on judgment, the fast agentic
model's time on implementation, and make the repo — not the chat — the source of truth.

## License

MIT
