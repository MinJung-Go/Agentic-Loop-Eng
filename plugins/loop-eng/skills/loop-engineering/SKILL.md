---
name: loop-engineering
description: Run the Loop Engineering discipline — drive an implementer↔reviewer loop against a frozen rubric until every item is SATISFIED and the tests pass. Use when the user wants to iterate to a verified done-state, and especially when driving autonomous completion with /goal ("loop until the reviewer approves", "iterate until CI/tests are green", "keep going until it's actually done").
---

# Loop Engineering

This skill runs the Loop Engineering workflow: a frozen-rubric, git-checkpointed `generate → critique → revise` loop between two subagents — `loop-implementer` (implementation) and `loop-impl-reviewer` (verification) — that iterates until every acceptance item is independently verified. It is the "how"; `/goal` (or you, in a single turn) provide the "loop until".

## Authoritative procedure (single source of truth)

The full per-round procedure — Preflight (clean-tree gate, isolation branch), freezing `requirements.md` + a `rubric.md` (`R1…Rn`), cooperative parallel implementers on file-disjoint slices, checkpoint commits, SHA-scoped + tool-grounded review, the regression guard, and wrap-up/squash — is defined in this plugin's `/loop-eng` command.

**Read `${CLAUDE_PLUGIN_ROOT}/commands/loop-eng.md` and follow it exactly.** Do not paraphrase or re-derive it: that file is the single source of truth, so this skill and the command can never drift apart. Treat this skill's argument (requirements text, or a path to a spec file) as that command's `$ARGUMENTS`.

## Driving it with `/goal` (the reason this is a skill)

This skill exists so you don't have to paste the whole methodology into a `/goal` condition — a short goal can invoke it instead:

```
/goal Use the loop-engineering skill to implement <requirements>.
DONE WHEN: loop-impl-reviewer's latest verdict is APPROVED, every rubric item is SATISFIED, and the test command exits 0 — and that verdict appears verbatim in your final message.
```

**Completion sentinel — the one rule that makes `/goal` reliable.** `/goal`'s completion check is made by a small, fast evaluator reading the transcript; it cannot re-judge the whole methodology, and it must not be trusted to. So the *hard* judgment stays with `loop-impl-reviewer`, and the evaluator only pattern-matches an unambiguous sentinel:

- End the turn by echoing the reviewer's final verdict **verbatim** — the literal keyword `APPROVED`, the full rubric coverage table (every `R#` = SATISFIED), and the test command with its exit status.
- **Never emit `APPROVED` yourself as the orchestrator.** Relay it only when it genuinely came from `loop-impl-reviewer`; a fabricated sentinel is a false completion, which is exactly what this discipline exists to prevent.
- If the loop stops on the iteration cap, or on a blocked / ambiguous state, do NOT emit the sentinel — report what is still failing (by rubric ID) so `/goal` keeps iterating or hands back to the user.

Pair with **auto mode** for unattended runs.
