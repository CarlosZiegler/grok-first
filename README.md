# grok-first

A Claude Code skill that delegates scoped implementation work to [Grok Build CLI](https://docs.x.ai/build/overview), while Claude Code retains design, review, and final verification.

It is an original, Grok-specific alternative to a “codex-first” routing workflow: Claude writes an explicit work order; Grok executes it headlessly; Claude inspects the diff and proof before accepting it.

## Inspiration

This skill is inspired by [Peter Steinberger’s `codex-first` skill](https://github.com/steipete/agent-scripts/blob/main/skills/codex-first/SKILL.md) from [`steipete/agent-scripts`](https://github.com/steipete/agent-scripts). It adapts that delegation discipline for Grok Build CLI and deliberately replaces Codex-specific commands and routing behavior with verified Grok Build capabilities.

## What it does

- Routes clear implementation, repair, refactor, test, and exploration work to `grok`.
- Uses Grok's documented headless mode: `--prompt-file`, `--cwd`, JSON output, session resumption, and `--no-auto-update`.
- Pins Grok Build to `grok-4.5` with high reasoning effort by default.
- Requires an explicit repository scope and a self-contained work order.
- Keeps architecture, releases, secrets, destructive operations, and final review in Claude Code.

## Requirements

- Claude Code
- [Grok Build CLI](https://docs.x.ai/build/overview), authenticated locally or configured with `XAI_API_KEY`
- Git (for reviewing the worker's diff)

Verify Grok before installing the skill:

```bash
grok --version
grok --no-auto-update -p "Reply with exactly: GROK_OK" --output-format json
```

## Install

Install globally for Claude Code through [skills.sh](https://skills.sh):

```bash
npx skills add CarlosZiegler/grok-first \
  --skill grok-first \
  --agent claude-code \
  --global \
  --yes
```

This installs only the `grok-first` package for Claude Code. Restart Claude Code
afterward so it discovers the new skill.

To opt out of the skills CLI's anonymous install telemetry:

```bash
DISABLE_TELEMETRY=1 npx skills add CarlosZiegler/grok-first \
  --skill grok-first \
  --agent claude-code \
  --global \
  --yes
```

## Usage in Claude Code

Ask Claude Code to use the skill with an explicit work order, for example:

> Use grok-first to implement the agreed pagination specification in `apps/web`. Do not alter authentication. Run `bun test` and report the changed files and results.

The skill tells Claude to delegate through Grok Build headlessly, then to review the resulting diff and real test evidence itself.

## Security model

The skill uses `--always-approve` only for a fixed, trusted repository and an explicit task. It instructs Claude Code not to use it for secrets, unknown directories, publishing, destructive operations, or untrusted prompts. Review the generated diff before committing or pushing.

## Sources

- [Grok Build overview](https://docs.x.ai/build/overview)
- [Grok Build: Headless & Scripting](https://docs.x.ai/build/cli/headless-scripting)

## License

[MIT](LICENSE)
