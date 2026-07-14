# Prompting Claude

How to compose the prompt you pass to `claude -p`. Read this before writing a
non-trivial delegation prompt.

Claude's current models follow instructions literally and precisely. That cuts
both ways: a sloppy prompt is followed sloppily, and a contradictory prompt
produces a coin flip. Tighten the prompt before reaching for a bigger model.

## Core rules

- **One task per run.** Split unrelated asks into separate `claude -p` calls.
- **Say what done looks like.** Do not assume Claude infers the desired end state.
- **Ask for action, get action.** "Suggest changes to this function" returns
  suggestions. "Change this function" returns a change. Match the verb to what
  you actually want — this is the single most common miss.
- **Use XML tags for structure.** Claude is trained on them. Stable tag names
  beat prose paragraphs for anything with more than one constraint.
- **Tighten the contract before escalating.** A clearer prompt beats a bigger
  model or a longer explanation.
- **Remove contradictions.** Literal instruction-following means two rules that
  disagree produce inconsistent output. Delete the loser before sending.

## Blocks worth adding

Add only the ones the task needs. Empty ceremony costs tokens and dilutes the real constraints.

| Block | When |
|---|---|
| `<investigate_before_answering>` | Any claim about existing code. Cuts hallucination hard. |
| `<output_contract>` | You need a specific shape back — findings, a list, a verdict. |
| `<action_safety>` | Write-capable runs. Keeps Claude from wandering into unrelated refactors. |
| `<grounding_rules>` | Review, research, anything that could drift into unsupported claims. |
| `<verification_loop>` | Debugging or risky fixes. Tell Claude how to check its own work. |

Reusable text for the two that matter most:

```xml
<investigate_before_answering>
Never speculate about code you have not opened. Read the file before answering
any question about it. Never make claims about the codebase before investigating.
</investigate_before_answering>
```

```xml
<action_safety>
Avoid over-engineering. Only make changes that are directly requested or clearly
necessary. Do not refactor adjacent code, add abstractions, or expand scope.
</action_safety>
```

## Shape by task type

- **Review of local git changes** — use the prompt in `../prompts/adversarial-review.md`. It already carries the contract; do not rewrite it inline.
- **Diagnosis, planning, research** — read-only. Add `investigate_before_answering` and ask for observed-vs-inferred separation.
- **Implementation** — write-capable. Add `action_safety` and state the acceptance check ("the failing test passes", "the endpoint returns 404 on missing id").
- **Follow-up** — `claude -p -c "<delta>"`. Send only the change in direction, not a restatement of the whole prompt. Restate only if the direction changed materially.

## Anti-patterns

- Passing the user's raw words through when they are ambiguous about read vs write. Decide, or ask.
- Stacking `<thinking>`-style nudges. Claude's reasoning is adaptive; asking it to "think harder" mostly adds latency.
- Asking for JSON when a human reads the output next. You have no parser in this plugin — the JSON just gets rendered back to prose, badly.
- Restating the whole task on a `-c` follow-up. It re-anchors Claude on the original framing and buries the correction.
- Burying the actual ask under four paragraphs of context. Task first, context after.
