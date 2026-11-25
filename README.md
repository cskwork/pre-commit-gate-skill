# pre-commit-gate — one commit workflow for every agent harness

[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

An agent skill that runs the same ordered gate before every commit, whichever harness you use (Claude Code, Codex CLI, Kiro, or any agent that reads `SKILL.md` skills).

The rule behind the order: **every step that edits code runs before every step that judges it.** A cleanup pass after verification ships code nobody verified.

| # | Gate | Edits code |
|---|------|-----------|
| 1 | Simplify (optional) | yes |
| 2 | Verify: build, static checks, real behaviour observed | no |
| 3 | Independent review of the full diff | no |
| 4 | Commit, push, confirm the remote | no |

A fix made in gate 2 or 3 loops back to gate 2. Each gate ends on PASS, FAIL, or BLOCKED; BLOCKED never reads as PASS.

Gate 3 maps to each harness's own reviewer: `/code-review` in Claude Code, `codex review` in Codex CLI, and a fresh context-free subagent elsewhere. See [`SKILL.md`](SKILL.md).

## Install

Clone once and link it into each harness's skills directory:

```sh
git clone https://github.com/cskwork/pre-commit-gate-skill ~/src/pre-commit-gate-skill
ln -s ~/src/pre-commit-gate-skill ~/.claude/skills/pre-commit-gate
ln -s ~/src/pre-commit-gate-skill ~/.codex/skills/pre-commit-gate
```

If your harnesses share one skills directory (for example `~/.agents/skills` linked from each), one link covers all of them.

Then add one line to the instructions file your harnesses read (`AGENTS.md`, `CLAUDE.md`):

```md
Before committing code, run the `pre-commit-gate` skill; it is the commit workflow and replaces any harness instruction to run a cleanup pass right before commit.
```

## Project overrides

Per-project commands (test scope, review level, verify adapter, commit message rules) belong in that project's `AGENTS.md` or `CLAUDE.md`. The skill defers to them.

## 한국어 요약

커밋 전에 어느 하네스에서든 같은 순서로 도는 관문입니다. 코드를 바꾸는 단계(simplify)를 먼저, 판정하는 단계(verify → 독립 리뷰)를 나중에 둡니다. 리뷰나 검증에서 고친 코드는 검증부터 다시 거친 뒤 커밋·push합니다.

## License

MIT
