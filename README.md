# Claude Code plugin for Codex

Use Claude Code from Codex to review code or delegate tasks.

The mirror image of [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc),
which does the same thing in the other direction. If you work in both, run both.

Delegation goes through your local `claude` CLI, so it reuses the auth, config,
MCP servers, and CLAUDE.md you already have. There is no separate account, no
API key to paste, and no second runtime.

## What You Get

| Mode | Ask for it like | What it does |
|---|---|---|
| **task** | "delegate this failing test to claude" | One `claude` run. Read-only unless you ask for edits. |
| **review** | "have claude review my working tree" | Correctness, security, and edge cases over local git state. |
| **adversarial review** | "have claude poke holes in this approach" | Challenges the design and its assumptions, not just the implementation. |

Codex picks the mode from your wording. After any review it stops and asks which
findings you want fixed — it never auto-applies them.

## Requirements

- [Codex CLI](https://developers.openai.com/codex) v0.117 or later (for the plugin system)
- [Claude Code CLI](https://claude.com/claude-code): `npm install -g @anthropic-ai/claude-code`
- A Claude subscription or an Anthropic API key, logged in once via `claude`

## Install

```bash
codex plugin marketplace add https://github.com/linkesch/claude-plugin-codex
codex plugin add claude@linkesch-claude
```

From a local clone instead:

```bash
git clone https://github.com/linkesch/claude-plugin-codex
codex plugin marketplace add ./claude-plugin-codex
codex plugin add claude@linkesch-claude
```

Verify it registered:

```bash
codex plugin list | grep claude@linkesch-claude
```

## Usage

Just ask, in words. There are no slash commands — Codex routes to the skill on
intent.

### Delegate a task

> delegate this failing test to claude

Runs read-only: Claude investigates and reports back, without touching files.
Ask for the edit explicitly to let it write:

> have claude fix the failing test

### Review

> have claude review my working tree

Reviews local git state for correctness bugs, security issues, and edge cases.
Untracked files aren't in `git diff`, so the skill checks `git status` first —
an empty diff with new files present is not "nothing to review".

### Adversarial review

> have claude poke holes in this approach
> is this the right design?
> why shouldn't this ship?

Uses the prompt in [`prompts/adversarial-review.md`](plugins/claude/skills/claude/prompts/adversarial-review.md).
Default stance is skepticism: it hunts for the strongest reasons the change
should not ship, prioritising expensive and hard-to-detect failures — trust
boundaries, data loss, rollback safety, races, version skew. It reports only,
never edits, and an empty finding list is a valid answer.

### Follow-ups

> keep going
> dig deeper into the second finding

Continues the last Claude conversation in that directory via `claude -p -c`.

## Typical Flows

**Review before shipping**

> have claude review my working tree

then, once you've picked what matters:

> have claude fix findings 1 and 3

**Second opinion on a design**

> have claude poke holes in this approach, focus on the retry logic

**Hand over a problem you're stuck on**

> delegate this to claude: the worker deadlocks under load, figure out why

## Troubleshooting: "Not logged in · Please run /login"

In the Codex app and normal interactive use, this does not come up — the plugin
works out of the box against your existing Claude login.

You can hit it when Codex runs `claude` **inside its sandbox**, most commonly
under `codex exec`, which defaults to a read-only sandbox with no approvals. On
macOS, Claude Code keeps its OAuth token in the keychain, and seatbelt blocks
keychain access, so the call fails even though you are perfectly logged in.

Run it outside the sandbox. An API key does not help — the sandbox blocks
outbound network too, so the call fails either way.

This is the one asymmetry with codex-plugin-cc, which has no such problem:
Codex keeps its auth in a plain `~/.codex/auth.json` that Claude Code can read
from inside its own sandbox.

## FAQ

### Do I need a separate Claude account for this plugin?

No. It shells out to the `claude` CLI you already logged into.

### Will it use my existing Claude config?

Yes — same config, same MCP servers, same CLAUDE.md, same permissions. The
plugin adds no configuration of its own.

### Which model does it use?

Whatever your `claude` CLI defaults to. Name one to override:

> ask claude with opus to review this

### Can it edit my files?

Only when you ask. Default is read-only; edits require explicit phrasing, which
maps to `--permission-mode acceptEdits`.

### Why are there no slash commands?

Codex plugins expose skills, not slash commands. Ask in plain language.

### Why is there no background mode or job tracking?

`codex-plugin-cc` needs a companion daemon because it drives Codex over
JSON-RPC. This direction doesn't: `claude -p` prints to stdout and `claude -p -c`
resumes the last conversation in a directory, so the session, job, and transfer
machinery collapse into flags. Runs are foreground. Large reviews take a while.

## Development

The plugin is four content files: a marketplace manifest, a plugin manifest, a
skill, and the prompt/reference the skill pulls in. No build, no dependencies.

Changes to source do **not** go live until you reinstall — Codex serves the
plugin from a cache under `~/.codex/plugins/`, even on an unchanged version:

```bash
codex plugin add claude@linkesch-claude
```

Bump `version` in [`plugin.json`](plugins/claude/.codex-plugin/plugin.json) and
add a `CHANGELOG.md` entry for anything user-visible.

## License

MIT — see [LICENSE](LICENSE).

Not affiliated with, endorsed by, or sponsored by OpenAI or Anthropic.

"Claude", "Claude Code", "Codex", and their logos are trademarks of their
respective owners. The Claude glyph in [`plugins/claude/assets/claude.svg`](plugins/claude/assets/claude.svg)
is Anthropic's, used unmodified and only to identify which tool this plugin
delegates to. The MIT license above covers this project's own code and docs, not
that mark.
