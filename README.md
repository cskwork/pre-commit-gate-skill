# pre-commit-gate — one commit workflow for every agent harness

[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

An agent skill that runs the same ordered gate before every commit, whichever harness you use (Claude Code, Codex CLI, Kiro, or any agent that reads `SKILL.md` skills).

The rule behind the order: **every step that edits code runs before every step that judges it.** A cleanup pass after verification ships code nobody verified.

| # | Gate | Edits code |
|---|------|-----------|
| 1 | Simplify (optional) | yes |
| 2 | Verify: build, static checks, real behaviour observed | no |
| 3 | Independent review of the staged change, then record the receipt | no |
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

Then add one line to the instructions file your harnesses read. Codex and Kiro read `AGENTS.md`; Claude Code reads `CLAUDE.md`, and `AGENTS.md` only when no `CLAUDE.md` exists in that folder or any parent. Keep the text in `AGENTS.md` and make `CLAUDE.md` a link to it (`ln -s AGENTS.md CLAUDE.md`) so every harness reads one file. Where a `CLAUDE.md` already exists, add the line `@AGENTS.md` to it instead, which Claude Code reads as an import:

```md
Before committing code, run the `pre-commit-gate` skill; it is the commit workflow and replaces any harness instruction to run a cleanup pass right before commit.
```

## Enforce it with a git hook

An instruction line can be forgotten; a git hook runs in every harness. `hooks/pre-commit` blocks a code commit unless HEAD and the staged tree match the receipt gate 3 records after verify and review pass, so code edited after review cannot be committed on a stale pass. Docs-only commits (`*.md`, `*.rst`, `*.adoc`, `LICENSE`) pass through; `*.mdx` counts as code because frameworks render it. A clean `git merge` does not run `pre-commit`; a merge finished by hand after conflicts carries new code and needs a receipt. If the repo's `pre-merge-commit` hook delegates to `pre-commit` (git's own `pre-merge-commit.sample` does), clean merges need a receipt too: record it on the merge result, or leave `pre-merge-commit` unset.

```sh
ln -s ~/src/pre-commit-gate-skill/hooks/pre-commit "$(git rev-parse --git-path hooks)/pre-commit"
```

Keep any existing `pre-commit` hook: chain the two, with hooks that rewrite the index (formatters that re-stage) first and this gate last. Install it only where `git rev-parse --git-path hooks` is inside the repo; a global `core.hooksPath` is shared, and a hook there blocks every repo that uses it.

### Limits

The hook guards `git commit`. These paths go around it, so the skill's gates cover them by hand:

- The receipt is written by the agent itself: the hook catches a skipped or stale gate, not a forged one.
- `git rebase --continue` commits resolved conflicts without running `pre-commit`. Verify and review the resolution before continuing.
- Git fixes an automatic merge's tree before `pre-merge-commit` runs, so a formatter there changes the index but not the merge commit. With index-rewriting hooks, merge with `--no-commit`, run the gates, then `git commit`.
- Markdown that a site generator compiles (VitePress `<script setup>` in `.md`) is code. Such repos drop `md` from the hook's docs pattern.
- With `core.symlinks=false` (common on Windows) a committed `CLAUDE.md` link checks out as a one-line text file. Teams on such checkouts use `@AGENTS.md` inside `CLAUDE.md` instead of a link.

## Project overrides

Per-project commands (test scope, review level, verify adapter, commit message rules) belong in that project's `AGENTS.md` or `CLAUDE.md`. The skill defers to them.

## 한국어 요약

커밋 전에 어느 하네스에서든 같은 순서로 도는 관문입니다. 코드를 바꾸는 단계(simplify)를 먼저, 판정하는 단계(verify → 독립 리뷰)를 나중에 둡니다. 리뷰나 검증에서 고친 코드는 검증부터 다시 거친 뒤 커밋·push합니다.

## License

MIT
