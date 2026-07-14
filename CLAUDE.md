# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Codex CLI plugin that delegates work to the local `claude` CLI. The mirror
image of [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc),
which goes the other way.

There is no code. No build, no dependencies, no test suite, no CI. The entire
product is a manifest plus Markdown that tells Codex which `claude` command to
run. Treat prose in `SKILL.md` as the implementation — a careless wording change
is a behaviour change.

## The verification loop

There is nothing to compile and nothing to unit test. The only way to know a
change works is to install it and drive Codex:

```bash
codex plugin add claude@linkesch-claude          # reinstall (see below — required)
codex plugin list | grep claude@linkesch-claude  # expect: installed, enabled
codex exec --skip-git-repo-check "List your available skills by name only." | grep claude:claude
```

To test the skill actually routes and shells out correctly:

```bash
codex exec --skip-git-repo-check "Use the claude:claude skill to ask Claude Code to reply with exactly BRIDGE_OK."
```

To test a prompt in isolation, skip Codex entirely and pipe straight to Claude —
faster, and it isolates prompt quality from plugin wiring:

```bash
git diff HEAD | claude -p "$(cat plugins/claude/skills/claude/prompts/adversarial-review.md)"
```

## Edits are not live until you reinstall

**The single biggest time-waster in this repo.** Codex serves the plugin from a
cache under `~/.codex/plugins/cache/<marketplace>/claude/<version>/`, not from
your working tree. Editing a file changes nothing until:

```bash
codex plugin add claude@linkesch-claude
```

This refreshes even on an unchanged version number. If a change appears to have
no effect, check the cache before debugging anything else:

```bash
diff -r plugins/claude ~/.codex/plugins/cache/linkesch-claude/claude/0.1.0
```

If the marketplace points at GitHub rather than a local path, you must push
first, then `codex plugin marketplace upgrade`, then reinstall. To develop
against local files instead:

```bash
codex plugin marketplace remove linkesch-claude
codex plugin marketplace add .
codex plugin add claude@linkesch-claude
```

## Architecture

Four layers, each pointing at the next:

1. `.agents/plugins/marketplace.json` — marketplace `linkesch-claude`, lists the plugin and its local path.
2. `plugins/claude/.codex-plugin/plugin.json` — plugin `claude`. `skills: "./skills/"` is what loads the skill. The `interface` block is metadata the Codex desktop app renders (icon, blurb, prompt chips); it has no effect on CLI behaviour.
3. `plugins/claude/skills/claude/SKILL.md` — the product. Its frontmatter `description` is what makes Codex route to it, so that line is load-bearing for discovery.
4. `prompts/` and `references/` — pulled in by `SKILL.md` on demand, not loaded up front.

Codex plugins expose **skills, not slash commands**. Routing is intent-based, so
the skill description matters more than any command name.

### The design constraint worth preserving

The skill is a **forwarder, not an orchestrator**: compose one prompt, make one
`claude` call, return stdout verbatim. Codex must not investigate the repo, add
its own analysis, or answer alongside Claude.

`codex-plugin-cc` needs ~2,000 lines of companion JavaScript and an app-server
broker because it drives Codex over JSON-RPC. This direction needs none of it:
`claude -p` prints to stdout and `claude -p -c` resumes the last conversation in
a directory, so session state, job tracking, and transfer collapse into flags.

Resist rebuilding that machinery. If background runs or job tracking ever seem
necessary, that is a real design change, not a gap to fill by reflex.

## Non-obvious constraints

- **Skill files need absolute paths.** Codex's working directory is the user's project, not the skill directory. A relative `cat prompts/...` fails. `SKILL.md` says `<SKILL_DIR>` for this reason — do not "simplify" it to a relative path.
- **Read-only is the default, deliberately.** Plain `claude -p` cannot edit, because `-p` auto-denies permission prompts. Writes require `--permission-mode acceptEdits` and explicit user intent. Do not make write the default.
- **After a review, Codex must stop.** Never auto-apply review findings. Ask which the user wants fixed. This is stated in `SKILL.md` and is intentional.
- **`claude -p` reads piped stdin as context**, which is how diffs get in. Verified.
- **Marketplace naming is owner-subject** (`linkesch-claude`), matching OpenAI's `codex@openai-codex`. It was briefly `claude-cc`, which was wrong: "cc" means Claude Code, the *host* being plugged into, and here the host is Codex.
- **The icon is Anthropic's trademark**, sourced unmodified from `claude.com/favicon.svg`. Used to identify the delegation target only. Not covered by this repo's MIT license — see the README trademark note before changing or re-using it.

## Known unverified: the macOS keychain issue

The first delegation on macOS fails with `Not logged in · Please run /login`
even on a working Claude login. Claude Code keeps its OAuth token in the macOS
keychain, and Codex's seatbelt sandbox blocks keychain access.

Confirmed: `claude -p` works outside the sandbox and fails inside `codex sandbox`.
The sandbox also blocks outbound network, which is why an `ANTHROPIC_API_KEY`
fallback does **not** work — that was tried and removed.

**Not confirmed:** that approving Codex's escalation actually fixes it. The
reasoning is sound (escalation runs outside seatbelt; outside seatbelt works) but
no successful end-to-end Codex→Claude run has been observed. It needs an
interactive approval, which `codex exec` cannot give. If you can run this
interactively, verify it and correct the README if the diagnosis is wrong.

## Releasing

**GitHub releases are not required for updates.** Codex tracks the default
branch by commit SHA (`source_type = "git"`, `last_revision = <sha>` in
`~/.codex/config.toml`). Users get changes from `codex plugin marketplace
upgrade` the moment you push. A release changes nothing functionally.

We tag anyway, because `openai/codex-plugin-cc` tags every version (`v1.0.2` …
`v1.0.6`) matching its `plugin.json`, and an unbacked version claim in a manifest
is a claim nothing corroborates.

The version in `plugin.json` is display metadata only — Codex does not use it to
gate updates, and reinstalling over the same version still refreshes the cache.

For anything user-visible, keep these three in sync or the release list starts
lying:

1. `version` in `plugins/claude/.codex-plugin/plugin.json`
2. A `CHANGELOG.md` entry
3. A matching git tag: `git tag vX.Y.Z && git push origin vX.Y.Z`

Deliberately not automated — `codex-plugin-cc` has a bump script and CI because
it has real code to test. Four content files do not justify it.
