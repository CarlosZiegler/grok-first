---
name: grok-first
description: "Use when Claude Code should delegate scoped coding work to Grok Build CLI."
version: 1.0.0
author: Carlos Ziegler
license: MIT
metadata:
  tags: [claude-code, grok, grok-build, delegation, coding-agent]
---

# Grok First

## Purpose

Use Grok Build CLI as Claude Code's implementation worker. Claude Code owns the
brief, architecture, constraints, review, and final decision; Grok owns a
well-scoped execution pass in the target repository.

This is delegation, not a hidden model swap: Claude Code remains the active
harness and must review the resulting diff and test evidence before reporting
completion.

## When to Use

Delegate to Grok by default when the task is a clear work order:

- implementation from an agreed specification
- bounded bug fixes with a known reproduction or diagnosis target
- refactors, mechanical migrations, dependency/tooling work
- focused test writing and test/lint/typecheck repairs
- read-heavy exploration where the answer is grounded in repository files

Keep the work in Claude Code when it requires architectural decisions, product
or UX judgment, secrets or account access, releases/publishing, destructive
operations, or an edit so small that delegation costs more than doing it.

For a mixed task, first decide the design in Claude Code, then hand Grok a
frozen specification.

## Preconditions

Before delegation, verify all of the following:

1. The active harness is Claude Code and `grok` is available on `PATH`.
2. `grok --version` succeeds.
3. The target directory is explicit and is the only repository Grok may alter.
4. The work order states the goal, relevant paths, constraints, non-goals,
   proof command(s), and the requested final report.

If any prerequisite fails, do not substitute a guessed command. Explain the
blocker or continue directly in Claude Code.

## Delegation Contract

Grok starts a fresh headless session with no knowledge of the Claude Code
conversation. Write a self-contained work order. Include:

- **Goal:** the observable desired outcome.
- **Scope:** repository path and files/modules to inspect or change.
- **Constraints:** conventions, compatibility requirements, and files that must
  not change.
- **Non-goals:** explicitly exclude follow-on cleanup and unrelated changes.
- **Proof:** exact tests, typechecks, linters, or manual checks to run.
- **Report:** files changed, commands run with results, and unresolved risks.
- **Stop condition:** finish only this work order; do not start adjacent tasks.

Every hard prohibition needs an honest exit: if the worker cannot comply, it
must stop and report why rather than bypassing the constraint.

## Invoke Grok Headlessly

Use a temporary prompt file; do not put a multi-line work order in shell
history or create a repository file merely to transport it.

```bash
REPO="/absolute/path/to/repository"
PROMPT="$(mktemp -t grok-first-prompt.XXXXXX)"
OUT="$(mktemp -t grok-first-result.XXXXXX)"

cat >"$PROMPT" <<'EOF'
Work only in <REPO>.

Goal: <observable outcome>
Scope: <paths/modules>
Constraints: <rules; do not touch X>
Non-goals: <what not to do>
Proof: run <exact command(s)>.

Finish this work order only. In the final report list files changed, commands
run with their results, and remaining risks or blockers.
EOF

grok --no-auto-update \
  --cwd "$REPO" \
  --model grok-4.5 \
  --reasoning-effort high \
  --max-turns 30 \
  --always-approve \
  --output-format json \
  --prompt-file "$PROMPT" >"$OUT"

cat "$OUT"
rm -f "$PROMPT"
```

- `--prompt-file` avoids fragile shell quoting.
- `--output-format json` yields one machine-readable completion result; inspect
  the result file rather than parsing terminal chatter.
- `--no-auto-update` prevents update checks from interrupting scripting or CI.
- `--always-approve` is appropriate only for a scoped, trusted repository. Do
  not use it for untrusted prompts, unknown directories, secrets, publishing,
  or destructive work.
- Pin `--model grok-4.5` and `--reasoning-effort high` unless the work order
  deliberately chooses a different installed model.

For an interactive or multi-turn investigation, start `grok --cwd "$REPO"`
in a dedicated terminal instead. Do not use a stale headless session for an
unrelated new task.

## Review and Verification

Grok's report is evidence, not authority. Claude Code must:

1. Inspect `git status --short` and the full diff.
2. Check each changed file against the work order and repository instructions.
3. Run the stated focused verification or inspect its real output.
4. Ask Grok for one targeted follow-up only if needed; after repeated failed
   attempts, take over directly rather than looping.
5. Report completion only after the diff and evidence satisfy the original
   request.

## Follow-up Work

For a correction that depends on the prior Grok result, use the session ID in
the JSON output:

```bash
grok --no-auto-update \
  --cwd "$REPO" \
  --resume "<session-id>" \
  --always-approve \
  --output-format json \
  -p "Fix only <specific issue>. Re-run <exact verification> and report the result."
```

Use a fresh session for a new work order. A resumed session should only repair
or complete the same bounded task.

## Common Pitfalls

1. **Delegating an ambiguous task.** Resolve design choices before delegating;
   otherwise Grok must invent product or architecture decisions.
2. **Giving Grok no proof command.** A diff without real verification is not
   complete. State the command or desired evidence in the work order.
3. **Blindly trusting JSON success.** `stopReason: EndTurn` means the model
   stopped; it does not prove that the repository change is correct.
4. **Using `--always-approve` outside the stated scope.** It allows Grok to run
   tools freely. Keep its `--cwd` fixed and never include secrets in prompts.
5. **Running concurrently in the same checkout.** Use separate worktrees for
   independent write tasks. Parallel read-only exploration is safe.
6. **Letting auto-updates disturb automation.** Keep `--no-auto-update` on all
   headless invocations.

## Completion Checklist

- [ ] Grok was launched with an explicit `--cwd` and self-contained prompt.
- [ ] The output JSON was captured and its session ID retained when follow-up is likely.
- [ ] Claude Code inspected the actual diff, not just Grok's summary.
- [ ] Required verification was run and its result is known.
- [ ] No work outside the specified scope remains unreviewed.
