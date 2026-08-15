# Loop Engineering for Claude Code

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-D97757?logo=anthropic&logoColor=white)](https://www.anthropic.com/claude-code)
[![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen.svg)](https://github.com/MinJung-Go/claude-loop-eng/issues)

[What's inside](#whats-inside) • [How the loop works](#how-the-loop-works) • [Install](#install) • [Usage examples](#usage-examples) • [Driving it autonomously with /goal](#driving-it-autonomously-with-goal) • [Notes](#notes) • [License](#license)

---

A Claude Code **plugin + marketplace** that packages a disciplined *Loop Engineering* workflow: a `/loop-eng` command drives an **Implementer** and a **Reviewer** subagent through `generate → critique → revise` cycles until every acceptance-criterion is met — grounded in git and real test runs, not vibes.

> Loop Engineering is the 2026 name for the evaluator-optimizer / reflect-refine pattern (research lineage: Self-Refine → Reflexion → CRITIC). One agent generates, a *different* agent critiques against a fixed rubric, and the loop repeats until it passes or hits a limit.

## What's inside

| Component | Role |
|---|---|
| `/loop-eng` command | The **orchestrator**. Owns control flow, git, the frozen rubric, and all on-disk review artifacts. |
| `loop-implementer` agent | The **Implementer**. Turns requirements + reviewer feedback into minimal, in-scope changes. Never self-approves, never touches git. |
| `loop-impl-reviewer` agent | The **Reviewer**. A binary quality gate — judges the *actual git diff* against a frozen rubric, runs the project's tests for external grounding, and returns `APPROVED` / `NEEDS_ITERATION`. |
| `loop-engineering` skill | A thin, model-loadable entry point that routes to the same procedure as `/loop-eng`. Its purpose is composition — so a short `/goal` can invoke the whole discipline without pasting it. |

## How the loop works

```
/loop-eng <requirements | path/to/spec.md> [--max N]

Preflight   clean working tree? → record base SHA → cut an isolated branch loop-eng/<slug>-<run>
Freeze      write verbatim requirements.md + a frozen rubric.md (R1..Rn) — the fixed yardstick
Each round  Implementer edits → orchestrator checkpoint-commits → Reviewer verifies
              (a single implementer by default; optional parallel fan-out for large, file-disjoint work)
              git diff <base>..HEAD  +  runs the project's tests
              per-rubric verdict: R1 SATISFIED / R3 MISSING ...
            APPROVED  → done      NEEDS_ITERATION → feed failing R-ids back, next round
            regression? (a passing item went red) → git reset to last-good checkpoint
Stop        every rubric item SATISFIED & tests green, OR max rounds, OR reviewer blocked
Wrap-up     write summary.md; offer to squash the WIP checkpoints into one clean commit
            (never auto-pushes, never auto-merges)
```

Every run leaves a diffable audit trail:

```
docs/reviews/loop-eng/<slug>-<run>/
  requirements.md      # verbatim user input — the immutable baseline
  rubric.md            # frozen R1..Rn acceptance criteria
  round-01-review.md   # per-round verdict, SHA, rubric coverage table, test result
  round-02-review.md
  summary.md
```

## Design principles baked in

- **Ground the reviewer in reality** — it reviews the real `git diff` by SHA range and *runs the tests*; the Implementer's self-report is treated as an unverified claim, never as evidence.
- **Freeze the rubric** — the same yardstick every round, so verdicts are comparable and don't drift round-to-round.
- **Separate roles** — the Implementer never self-approves; the Reviewer never edits code. Approval is a gate, not a self-assessment.
- **Git-native** — one round = one checkpoint = one review file, all on a throwaway branch. Bad rounds roll back; the human owns the merge.
- **Single implementer by default** — a round uses one `loop-implementer`. Only for large work that splits cleanly into **file-disjoint** slices may the orchestrator optionally fan out several in parallel (a flat set that cooperate on distinct files — not a race). Small or coupled work stays single.

## Install

```bash
# add this marketplace
/plugin marketplace add MinJung-Go/claude-loop-eng

# install the plugin
/plugin install loop-eng@minjung-go

# reload to activate
/reload-plugins
```

## Usage examples

### 1. Implement to a spec file
```bash
/loop-eng docs/specs/new-feature.md
```

### 2. Fix a bug with an explicit acceptance bar
```bash
/loop-eng Add retry-with-exponential-backoff to fetch_url on 5xx responses, cap 3 retries --max 4
```

### 3. Loop until the build / CI is green

The Reviewer's gate is **tool-grounded**: every round it runs your project's own build/test command and only returns `APPROVED` when that command exits clean. So to "iterate until CI passes," make the CI command itself the acceptance bar:

```bash
/loop-eng Make `npm run ci` pass on this branch — fix the failing type-checks and unit tests. Do not weaken or delete the tests. --max 6
```

What happens each round:

1. the rubric freezes an item like **`R1: \`npm run ci\` exits 0`**;
2. the Implementer makes the smallest fix; the orchestrator checkpoint-commits it;
3. the Reviewer **actually runs `npm run ci`** on the checkpoint —
   - ❌ red → `NEEDS_ITERATION`, the failing output is fed back as the next directive;
   - ✅ green **and** every rubric item SATISFIED → `APPROVED`.

Because the Reviewer runs the **same command your GitHub Action runs**, a green loop means the pipeline should pass too — *before you ever push*. When it approves, you get one clean squashed commit ready to push and let CI confirm.

> **Scope note:** the loop runs locally on an isolated branch and never auto-pushes, so it drives your code to a state that *will* pass CI rather than polling live GitHub Actions runs. A heavier variant that pushes each round and gates on the real Actions result (`gh run watch`) is possible but opt-in — [open an issue](https://github.com/MinJung-Go/claude-loop-eng/issues) if you want it.

## Driving it autonomously with `/goal`

A **lightweight, independent** way to run the discipline — it doesn't use the `/loop-eng` command. The insight: `/goal` already *is* a loop. `/goal <condition>` supplies both halves:

- **the loop** — after each turn it automatically starts the next until the condition holds;
- **the judge** — a small, fast evaluator (a *different* model from the one working) checks the condition after every turn, reading only the transcript.

So you don't reimplement either. The bundled `loop-engineering` skill just keeps each turn honest — make an increment, run the check, **surface the real output** — so `/goal`'s evaluator has facts to read. This is the quick "keep going until it works" path; it is **not** strong verification. For an independent reviewer that reads the real diff and reruns tests, use the [`/loop-eng` command](#usage-examples) instead.

### How it works

```
  /goal "<REQUIREMENTS>.  DONE WHEN: <a check your own output can show>"
        │
        ▼
  ┌─────────────────────────  one turn  ──────────────────────────┐
  │                                                                │
  │  ▸ the skill keeps the turn honest                             │
  │      1. advance one real increment toward the requirements     │
  │      2. run the check → paste the real command + output +      │
  │         exit code into the turn                                │
  │      3. never declare "done" yourself                          │
  │                              │                                 │
  │  ▸ /goal judges  (a *different* evaluator model)               │
  │      reads only the transcript — cannot run tools / read files │
  │                              │                                 │
  │                       condition met?                           │
  └──────────────┬────────────────────────────────┬───────────────┘
                 │ no                              │ yes
                 ▼                                 ▼
     /goal auto-starts the next turn        /goal clears itself
     (evaluator's reason carried               → the loop stops
      forward as the next cue)
```

The division of labor: **`/goal` owns the loop and the (independent) verdict; the skill only owns surfacing honest evidence** — because the evaluator can judge nothing it can't see in the transcript.

### Walkthrough

**1. Install the plugin** (once):

```
/plugin marketplace add MinJung-Go/claude-loop-eng
/plugin install loop-eng@minjung-go
/reload-plugins
```

**2. (Optional) enable auto mode** so the loop runs unattended without stopping for tool-permission prompts:

```
/auto
```

**3. Fire the goal.** Requirements in the directive; a `DONE WHEN` your own output can demonstrate. Example — make the CI command pass:

```
/goal Follow the loop-engineering skill. REQUIREMENTS: make `npm run ci` pass on this branch — fix the failing type-checks and unit tests.
DONE WHEN: the latest turn runs `npm run ci`, shows its full output, and it exits 0.
```

What happens, hands-off:
- **each turn**, the skill makes the next increment, then actually runs `npm run ci` and pastes the command + full output + exit code into the turn;
- **`/goal`'s evaluator** reads that transcript and decides: not green → it automatically starts another turn (the failing output is the next turn's feedback);
- green → `/goal` **clears itself** and stops. No custom reviewer, no custom loop — `/goal` is both.

Want to guard a constraint too (e.g. "don't delete or weaken tests")? Add it to the `DONE WHEN` and have the turn show `git diff` — but keep in mind a small evaluator reading a diff is a light check. When that guarantee matters, reach for `/loop-eng`.

**4. Watch or stop it:**

```
/goal            # show the active condition, runtime, turn count, token spend, last evaluator reason
/goal clear      # stop early
```

### Why it stays honest

`/goal`'s evaluator reads the transcript and **cannot run tools itself** — so the skill's job is to write the `DONE WHEN` as an *objective* check and to **surface real command output every turn**. A condition like "`npm run ci` exits 0 (output shown)" is something a small model can verify reliably; "the code is clean" is not. The skill also forbids the agent from fabricating evidence or self-declaring done — completion is the evaluator's call on real output. (For a *deterministic* gate rather than a transcript-reading one, add a **Stop hook** that runs the checks as a script; that's a separate mechanism, with `/goal` still driving the loop.)

> This `/goal` path is independent of `/loop-eng`. Use `/loop-eng` for a manual single-shot run (it carries its own loop, rubric, and reviewer agent); use `/goal` + the skill for autonomous iteration. They intentionally differ.

### Why the sentinel matters

`/goal`'s completion check is a small, fast evaluator reading the transcript — it **can't** re-judge the whole methodology, and it shouldn't be trusted to. So the **hard judgment stays with `loop-impl-reviewer`** (frozen rubric + real `git diff` + real tests), and the evaluator only pattern-matches the `APPROVED` verdict the orchestrator echoes. The orchestrator must never emit `APPROVED` itself — only relay it when it genuinely came from the reviewer — otherwise a fabricated sentinel would end the loop early. That single rule is what keeps autonomous `/goal` runs honest.

> Any task works, not just CI — swap the `REQUIREMENTS` and `DONE WHEN` lines. For an interactive, single-shot run instead of an autonomous goal, use the `/loop-eng` command directly (see [Usage examples](#usage-examples)).

## Notes

- The command and agents are written in English, but the workflow is language-agnostic — the Reviewer matches the language of your requirements while keeping the `APPROVED` / `NEEDS_ITERATION` keyword in English so the loop can parse it.
- The Reviewer runs your project's own test command for grounding; point it at whatever your repo documents (in `CLAUDE.md` / `AGENTS.md` / `README`).

## Contributing

Contributions are welcome! Please feel free to submit an [Issue](https://github.com/MinJung-Go/Agentic-Loop-Eng/issues) or Pull Request.

## License

[MIT License](LICENSE), © 2026 MinJung-Go.

---

<div align="center">

**If this project helps you, please give it a Star!**

Questions or suggestions? Open an [Issue](https://github.com/MinJung-Go/Agentic-Loop-Eng/issues)

Made by [MinJung-Go](https://github.com/MinJung-Go)

</div>
