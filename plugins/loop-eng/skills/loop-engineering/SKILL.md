---
name: loop-engineering
description: Lightweight companion for driving a task to completion with Claude Code's /goal — keep each turn honest so /goal's own evaluator can decide when it's done. Use for autonomous "keep going until it works" runs ("loop until tests/CI are green", "keep iterating until X passes", "autonomously finish X"). Not a heavy verification harness — for strong, independent diff+test review use the /loop-eng command instead.
---

# Loop Engineering with `/goal` (lightweight)

`/goal <condition>` is already a complete loop: it re-runs turns until a fresh evaluator model decides the condition holds, and that evaluator is a *different* model from the one doing the work (so you can't rubber-stamp your own work). The one catch: **the evaluator only reads the conversation — it can't run commands or read files.**

This skill is the light companion for that loop. It deliberately adds **no** rubric, **no** reviewer sub-agent, and **no** git machinery — it just keeps each turn honest so `/goal`'s own evaluator can do its job. (Want strong, independent review that reads the real diff and reruns tests? That's the `/loop-eng` command, not this.)

## Launch

Put the work in the directive and a check that **your own output can demonstrate** in `DONE WHEN`:

```
/goal <REQUIREMENTS>. DONE WHEN: <a result visible in the turn — e.g. `npm test` exits 0 with its output shown>.
```

Pair with **auto mode** to run unattended.

## Each turn, just do this

1. **Advance** — make the next real increment toward the requirements.
2. **Show the evidence** — run the check and paste its **actual command + output + exit code** into the turn. The evaluator judges from the transcript, so anything you don't surface, it can't credit.
3. **Don't call it done yourself** — let the output stand; `/goal`'s evaluator makes the call. If the check fails, that failure is simply the next turn's cue — keep going.

That's the whole discipline. Don't fabricate results (a faked "exit 0" ends the loop on a lie); if a check genuinely can't run, say so and the goal correctly stays open.

## Optional

- If something **must not change** on the way (e.g. "don't delete or weaken tests"), add it to the `DONE WHEN` and show the relevant `git status` / `git diff` so the evaluator can see it. Note this is best-effort — a small evaluator reading a diff is a light check, not a guarantee. When you need that guaranteed, use `/loop-eng` (independent reviewer) or a script Stop hook.
