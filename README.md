# Loop Engineering for Claude Code

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
Each round  Implementer(s) edit → orchestrator checkpoint-commits → Reviewer verifies
              (one implementer, or several in parallel — each owning file-disjoint slices)
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
- **Cooperative parallelism, not a race** — when a round's work splits into file-disjoint slices, the orchestrator fans out several `loop-implementer`s in parallel (a flat set, no nesting, no separate workflow engine), each owning distinct files so their edits compose without conflict. They collaborate to cover the requirement together — it is *not* a tournament where one implementation wins. If the work can't be cleanly partitioned, it falls back to a single implementer.

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

Claude Code's `/goal <condition>` loops turn-by-turn on its own until a completion condition holds — a small, fast evaluator checks after each turn and starts the next one if it isn't met yet. Instead of pasting the whole methodology into the goal, invoke the bundled `loop-engineering` skill and let a crisp sentinel be the condition.

### Walkthrough

**1. Install the plugin** (once):

```
/plugin marketplace add MinJung-Go/claude-loop-eng
/plugin install loop-eng@minjung-go
/reload-plugins
```

**2. (Optional) enable auto mode** so the loop can run unattended without stopping for tool-permission prompts:

```
/auto
```

**3. Fire the goal.** Put the requirements in the directive and a checkable sentinel in `DONE WHEN`. Example — make the CI command pass:

```
/goal Use the loop-engineering skill to implement the following on a fresh isolation branch.
REQUIREMENTS: make `npm run ci` pass on this branch — fix the failing type-checks and unit tests; do not weaken or delete the tests.
DONE WHEN: loop-impl-reviewer's latest verdict is APPROVED, every rubric item is SATISFIED, and `npm run ci` exits 0 — and that verdict appears verbatim in the final message.
```

What now happens, hands-off, each turn = one round:
- turn 1 freezes a rubric (e.g. `R1: \`npm run ci\` exits 0`, `R2: no test is deleted or weakened`) and does the first implement → checkpoint → review pass;
- if the reviewer returns `NEEDS_ITERATION`, `/goal` automatically starts the next turn with the failing rubric IDs fed back;
- when `loop-impl-reviewer` returns `APPROVED` with every item SATISFIED and `npm run ci` green, the turn echoes that verdict verbatim, the evaluator matches the sentinel, and **`/goal` clears itself** — the loop stops.

**4. Watch or stop it:**

```
/goal            # show the active condition, runtime, turn count, token spend, last evaluator reason
/goal clear      # stop early
```

When it finishes you're left with a clean squashed commit on the loop branch plus the review trail under `docs/reviews/loop-eng/…`, ready for you to push and let real CI confirm.

### Why the sentinel matters

`/goal`'s completion check is a small, fast evaluator reading the transcript — it **can't** re-judge the whole methodology, and it shouldn't be trusted to. So the **hard judgment stays with `loop-impl-reviewer`** (frozen rubric + real `git diff` + real tests), and the evaluator only pattern-matches the `APPROVED` verdict the orchestrator echoes. The orchestrator must never emit `APPROVED` itself — only relay it when it genuinely came from the reviewer — otherwise a fabricated sentinel would end the loop early. That single rule is what keeps autonomous `/goal` runs honest.

> Any task works, not just CI — swap the `REQUIREMENTS` and `DONE WHEN` lines. For an interactive, single-shot run instead of an autonomous goal, use the `/loop-eng` command directly (see [Usage examples](#usage-examples)).

## Notes

- The command and agents are written in English, but the workflow is language-agnostic — the Reviewer matches the language of your requirements while keeping the `APPROVED` / `NEEDS_ITERATION` keyword in English so the loop can parse it.
- The Reviewer runs your project's own test command for grounding; point it at whatever your repo documents (in `CLAUDE.md` / `AGENTS.md` / `README`).

## License

MIT © MinJung-Go
