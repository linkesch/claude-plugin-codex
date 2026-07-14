---
name: claude
description: "Delegate a task, investigation, or code review to Claude Code. Use when the user says claude, asks for a second opinion, wants Claude to review the diff or challenge a design, or asks to hand off / delegate work to another model."
---

# Claude Code

Delegate to the local `claude` CLI. You are a forwarder, not an orchestrator:
run one command, return its stdout verbatim.

## Preflight

If `claude` is not on PATH, stop and tell the user to install it:
`npm install -g @anthropic-ai/claude-code`, then `claude` once to log in.
Do not attempt the install yourself.

## "Not logged in · Please run /login"

This does not mean the user is logged out. On macOS, Claude Code's OAuth token
lives in the keychain, and the Codex sandbox blocks keychain access — so the
call fails even on a working login.

Re-run the same command with escalated permissions, outside the sandbox. That
is the only fix: an API key does not help, because the sandbox blocks outbound
network as well, and keychain access cannot be granted piecemeal.

If escalation is unavailable or the user declines, say so and stop. Do not
silently do the work yourself.

## Delegate a task

Read-only by default — Claude investigates and reports, it does not edit:

```bash
claude -p "<the request>"
```

Add `--permission-mode acceptEdits` only when the user asks Claude to actually
make the change ("have claude fix it", "let claude implement this"):

```bash
claude -p --permission-mode acceptEdits "<request>"
```

For anything beyond a one-line ask, read `references/prompting-claude.md` in
this skill's own directory before composing the prompt. Getting the prompt right
is the only Codex-side work allowed here.

## Review the diff

```bash
git diff HEAD | claude -p "Review this diff for correctness bugs, security issues, and edge cases. Open the files it touches and their callers before concluding a path is safe. Report findings as file:line + severity + what breaks + the fix, most severe first. Report only — do not edit anything."
```

Use `git diff <base>...HEAD` for a branch review. Check `git status --short
--untracked-files=all` first: untracked files are not in the diff, so name them
in the prompt or the review silently skips them. An empty diff with untracked
files present is not "nothing to review".

## Adversarial review

When the user wants the design challenged rather than the implementation
checked — "is this the right approach", "poke holes in this", "why shouldn't
this ship":

```bash
git diff HEAD | claude -p "$(cat <SKILL_DIR>/prompts/adversarial-review.md)

User focus: <the user's focus text, or 'none'>"
```

`<SKILL_DIR>` is the absolute path of the directory this SKILL.md lives in — the
working directory is the user's project, not the skill, so a relative `cat` will
not find the file. That prompt already carries the full review contract: pass it
as-is, do not rewrite it or soften the framing.

## Follow-ups

`claude -p -c "<follow-up>"` continues the most recent Claude conversation in
this directory. Use it for "keep going", "resume", "apply the top fix", "dig
deeper". Send only the delta, not a restatement. Omit `-c` to start fresh. When
it is ambiguous, ask which one.

## Options

- `--model opus|sonnet|haiku` — only when the user names a model. Leave unset otherwise.
- `--add-dir <path>` — when the task spans a directory outside the repo.

## Handling the result

- Return Claude's stdout as-is. No summary, no commentary, no verdict of your own.
- Keep findings in the order and severity Claude gave them. Use its file paths and line numbers exactly.
- Preserve evidence boundaries: if Claude marked something an inference, an uncertainty, or an open question, keep that label.
- No findings is a real answer. Report it plainly rather than hunting for something to say.
- If Claude made edits, say so and list the touched files.
- **After presenting review findings, STOP.** Do not fix anything. Ask which findings, if any, the user wants addressed. Auto-applying fixes from a review is forbidden, even when the fix looks obvious.
- If the run failed or Claude was never successfully invoked, report the error and stop. Do not substitute your own answer for the one Claude did not give.

## Rules

- One `claude` call per request. No retry loops, no chaining.
- Preserve the user's intent. You may tighten the wording into a better prompt, but do not answer the request yourself, investigate the repo first, or add your own analysis alongside Claude's.
- Runs are not backgrounded. Large reviews take a while; say so rather than killing the call.
