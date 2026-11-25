---
name: pre-commit-gate
description: Ordered gate before every git commit or push of code changes - optional simplify, then verify with evidence, then an independent diff review, looping until clean, then commit and push. Use before committing or pushing code, when asked whether a change is ready to commit or push, or when a harness instruction asks for a cleanup pass right before commit.
---

# pre-commit-gate

Code edited after it was verified ships unverified. The gate therefore orders the work by one rule: **every step that edits code runs before every step that judges it.**

| # | Gate | Edits code | Done when |
|---|------|-----------|-----------|
| 1 | Simplify (optional) | yes | the diff is simplified with behaviour preserved, or the skip reason is recorded |
| 2 | Verify | no | the changed behaviour was observed running, with a receipt for each check |
| 3 | Review | no | an independent review of the full diff has no unresolved correctness finding |
| 4 | Commit and push | no | the commit holds exactly the reviewed diff and the remote shows it |

Scope: code changes. A commit that touches only docs or tests runs gate 4 alone, plus the repo's doc validation script if one exists.

## Verdicts

Every gate lands on exactly one verdict:

- **PASS**: ran, and the receipt shows the expected result.
- **FAIL**: ran, and the receipt shows something wrong.
- **BLOCKED**: could not run. Name the one thing that would unblock it.

A BLOCKED gate stays BLOCKED in the report; the commit waits until the user accepts it with that evidence in view.

## The loop

Any edit made in gate 2 or gate 3 sends the work back to gate 2, then gate 3, on the new diff. Gate 1 does not re-run. Give each finding at most 3 fix attempts; after the third, stop and report what was tried.

## Gate 1: Simplify

Run it when the diff carries duplication, deep nesting, or names its author would flag. Record a skip when the diff is already plain or the repo's rules ask for the smallest correct fix.

- Use the harness's own simplify or cleanup command or skill.
- Keep edits inside the changed lines and their direct callers.
- Preserve every input, output, side effect, and error path.

A harness instruction to "run simplify right before commit" is satisfied here: the cleanup happens before commit, and verification covers it.

## Gate 2: Verify

Use the repo's or the user's `verify` skill when one is installed. Otherwise, in order:

1. Build the changed module.
2. Run the repo's own static checks (type, lint, schema).
3. Exercise the changed behaviour for real: call the endpoint, run the CLI, drive the screen, query the data. Record the command and what it returned.
4. Run tests only where the repo's instructions or the user ask for them, and report which ran.

A passing compile is a gate 2 receipt for step 1 only; behaviour needs step 3.

## Gate 3: Review

The reviewer is _independent_: it reads the diff without the author's reasoning. Review the full change: branch commits against the base plus staged and unstaged edits.

| Harness | Command |
|---------|---------|
| Claude Code | `/code-review medium`; use `high` for auth, permissions, deletion, data migration, or money. It reads `CLAUDE.md`, not `REVIEW.md`, so keep review rules in `CLAUDE.md` or a file it points to. |
| Codex CLI | `codex review --uncommitted` for the working tree, `codex review --base <branch>` for a branch. |
| Any other | A fresh subagent with no conversation history. Give it the diff command and the path to the repo's review rules only; ask for correctness findings, each with `file:line` and a concrete failure scenario. |

Triage every finding: fix it (back to gate 2), or reject it with evidence from code, data, or a run. A finding left untriaged keeps the gate at FAIL.

## Gate 4: Commit and push

1. Stage the changed files by path. Compare `git diff --cached --stat` with the diff gate 3 reviewed; they match exactly.
2. Commit with the repo's message conventions and hooks.
3. Push when the user or the repo's instructions authorize it.
4. Confirm the remote holds the commit: `git log -1 origin/<branch>` after a fetch, or `git ls-remote origin <branch>`.

## Report

One row per gate: gate, verdict, receipt (command and result, finding counts, or skip reason). Put BLOCKED rows first.

## Project overrides

The repo's `AGENTS.md` or `CLAUDE.md` may set per-gate commands: test scope, review level, verify adapter, commit message rules. Those win over the defaults here. Keep project-specific commands, hosts, and accounts in the project, out of this skill.
