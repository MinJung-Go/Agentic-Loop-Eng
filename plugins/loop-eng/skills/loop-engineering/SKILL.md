---
name: loop-engineering
description: Reproduce the Loop Engineering discipline autonomously with Claude Code's /goal — iterate to a tool-verified done-state. Use when driving completion via /goal ("loop until tests/CI are green", "keep going until every acceptance check passes", "autonomously finish X"). This skill governs each /goal turn; /goal itself supplies the loop and the completion gate.
---

# Loop Engineering via `/goal`

`/goal <condition>` already gives you the two halves of a Loop Engineering loop:

- **the loop** — after each turn it automatically starts the next one until the condition holds;
- **the gate / reviewer** — after every turn a small evaluator model checks whether the condition is met.

So this skill deliberately does **not** bring its own loop or its own reviewer — that would just duplicate what `/goal` already does. It supplies the missing half: a disciplined **per-turn implementer**, plus the one thing that makes `/goal`'s built-in gate trustworthy — **objective, transcript-visible evidence**.

> This is the `/goal`-native reproduction of the discipline, independent of the `/loop-eng` command. Use `/loop-eng` for a manual single-shot run; use this skill + `/goal` for autonomous iteration. They intentionally differ (one carries its own loop, this one borrows `/goal`'s) — do not try to keep them identical.

## How to launch

Put the requirements in the directive and an **objective, checkable** acceptance predicate in `DONE WHEN` — every part verifiable from the turn's own transcript:

```
/goal <REQUIREMENTS>.
DONE WHEN: <objective checks, each visible in this turn — e.g. `npm run ci` exits 0 with its output shown, and no test is deleted or weakened>.
```

Pair with **auto mode** for unattended runs.

## What you do each `/goal` turn

1. **Freeze the yardstick (turn 1).** Restate the acceptance predicate as a short numbered checklist `C1…Cn`, and on turn 1 write it to `docs/loop-eng/<slug>/criteria.md` so every later turn judges against the **same fixed list** instead of re-deriving it. Prefer objective, tool-checkable items; flag any genuinely subjective one.
2. **Advance.** Make the next smallest effective increment toward the requirements. Match the project's conventions and stay in scope — no speculative refactoring.
3. **Verify with tools — this is what makes `/goal`'s gate real.** Run the objective checks (tests / lint / build / a quick repro) and paste the **exact command + full output + exit code** into the turn. `/goal`'s evaluator reads the transcript and *cannot run tools itself*, so if you don't surface the evidence its verdict is only a guess.
4. **Map evidence → checklist.** State each `C#` as PASS / FAIL with the concrete result (failing test name, exit code, offending diff).
5. **Do not declare done.** If any check fails, that failure output *is* your feedback — fix it next turn. Never assert completion in words; let the objective evidence stand and let `/goal`'s evaluator make the call.
6. *(optional, recommended)* Work on a throwaway branch and `git commit` a checkpoint each turn, so a bad turn can be rolled back with `git reset --hard`.

## Keeping `/goal`'s gate honest

- **Objective condition + real evidence.** The evaluator is a small, fast model reading the transcript. It is reliable for "`C1`: command X exits 0 (output shown above)"; it is *not* reliable for "the code is clean/elegant." Write `DONE WHEN` as objective checks and always show real output.
- **Never fabricate evidence.** If a check can't be run, say so plainly — the goal then correctly stays unmet rather than completing falsely. A faked "exit 0" is exactly the failure this discipline exists to prevent.
- **Never self-declare `APPROVED`/done to satisfy the evaluator.** Completion is the evaluator's call on real evidence, not yours.
- **Want a deterministic gate instead of a transcript-reading one?** Add a **Stop hook** that runs the checks as a script and blocks stopping until they pass — stronger grounding, though it's a separate mechanism (`/goal` still provides the turn loop).
