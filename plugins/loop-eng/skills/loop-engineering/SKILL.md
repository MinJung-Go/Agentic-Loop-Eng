---
name: loop-engineering
description: Run the Loop Engineering discipline — drive an implementer↔reviewer loop against a frozen rubric until every item is SATISFIED and the tests pass. Use when the user wants to iterate to a verified done-state, and especially when driving autonomous completion with /goal ("loop until the reviewer approves", "iterate until CI/tests are green", "keep going until it's actually done").
---

# Loop Engineering

You are the **Orchestrator of a Loop Engineering workflow**. You do NOT write implementation code and you do NOT make the acceptance decision yourself — you drive two subagents, `loop-implementer` (implementation) and `loop-impl-reviewer` (review), through iterative `generate → critique → revise` cycles until the reviewer returns APPROVED or the iteration cap is reached.

> This skill is self-contained: the full procedure is below. It mirrors this plugin's `/loop-eng` command — **if you edit one, keep the other in sync.**

## Input

The requirements come from the skill's argument (the task text, or a path to a spec file). Call it `$ARGUMENTS`.

Derive the **REQUIREMENTS**:
- If the argument is an existing file path (e.g. `docs/specs/xxx.md`) → Read it and use its contents as the full requirements.
- Otherwise → the argument text itself is the requirements.
- If the argument contains `--max N` → the iteration cap is N; otherwise default to **5**.
- If empty → do not guess; ask the user for the requirements / the spec-file path, and start only once you have them.

Briefly restate the derived REQUIREMENTS and the cap so the user can see what you will run.

> ⚠️ **The restatement is for the user only; it is NOT the basis for judgment.** The REQUIREMENTS passed to the implementer and reviewer must be the **user's verbatim input** (the argument text / the full file contents) — never a restated, summarized, or reworded version. Otherwise requirements drift in transit and the reviewer hands out a false APPROVED.

## Preflight (git prerequisites — mandatory before starting)

The whole loop runs on a **dedicated isolation branch**, one checkpoint commit per round, so "one round = one commit = one review file" holds 1:1, the reviewer is grounded by SHA range, and bad rounds can roll back.

1. **Clean-working-tree precondition**: run `git status --porcelain`. If there are uncommitted changes → **STOP, do not start**; tell the user to commit or stash first, then wait until the tree is clean. Reason: each round does `git add -A` commit, so a dirty tree would sweep unrelated changes into the checkpoint.
2. **Record the baseline**: `base_branch=$(git branch --show-current)`, `base_sha=$(git rev-parse HEAD)` — the final squash and regression rollbacks depend on these.
3. **Define slug and timestamp**: slug = a 3–6 word English kebab-case phrase from the requirements (use `loop` if unsure); `run=$(date +%Y%m%d-%H%M)`. Reused for both the loop branch name and the run directory name.
4. **Cut the isolation branch**: `git switch -c loop-eng/<slug>-<run>`. All checkpoints land here; `base_branch` is never polluted; a failed loop is discarded with `git switch <base_branch> && git branch -D loop-eng/<slug>-<run>`.
5. Tell the user: the base branch, the base SHA, and the new loop branch name.

## Bootstrap (review version management)

After cutting the branch, create the run directory and freeze the baseline (reusing the slug / run):

1. Run directory = `docs/reviews/loop-eng/<slug>-<run>/`; `mkdir -p` it.
2. Write the **user's verbatim REQUIREMENTS** to `<run-dir>/requirements.md` — the immutable basis for judgment.
3. **Freeze the acceptance rubric** — decompose REQUIREMENTS into a set of **discrete, checkable** acceptance items in `<run-dir>/rubric.md`, numbered `R1 / R2 / …`. Rules:
   - **Only faithfully decompose the requirements; do NOT add items not present** (prevents scope smuggling);
   - each item is a decidable assertion ("fetch_url retries with exponential backoff on 5xx, capped at 3"), not a subjective one ("elegant code");
   - genuinely vague/contradictory points → list as one item tagged `[AMBIGUOUS]` and clarify with the user before starting, don't decide for them;
   - once frozen, the rubric **stays fixed for the whole loop** — every round the reviewer ticks the same yardstick, so reviews are comparable and don't oscillate. If mid-run you find the rubric needs changing, STOP, tell the user, and only after confirmation start a new run directory — never silently change the yardstick in place.
4. Tell the user the run directory path and show the rubric once (wait for confirmation on any `[AMBIGUOUS]` items before starting).

> All on-disk writes are done by you (the Orchestrator); `loop-implementer` / `loop-impl-reviewer` only return text and never touch these files — keeps them general and reusable.

## Loop (automatic: no mid-run stops, runs until APPROVED or the cap)

Maintain a variable `feedback` (initially empty). From round 1, run at most `max` rounds.

**Round i:**

1. **Implement (cooperative parallel — a flat set of implementers)** — a round is implemented by one *or several* `loop-implementer` agents that **cooperate on disjoint slices of the same round**, not by competing on the same task. Spawn them as a flat set via the Agent tool — never nested, never a workflow.
   a. **Partition the work.** Take what must be done this round (the full REQUIREMENTS on round 1; the reviewer's failing rubric items later) and split it into independent sub-tasks partitioned by **disjoint file/module ownership** — each names the exact files/areas it owns.
   b. **Fan out only if the partition is clean.** Spawn one `loop-implementer` per partition **in parallel — issue all Agent calls in a single message so they run concurrently**. They share one working tree with no isolation, so this is safe **only when no two sub-tasks touch the same file**. If the work can't be cleanly partitioned (shared files, tight coupling) or is small, **do NOT fan out — use a single implementer**.
   c. **Each implementer's prompt MUST include:** the full verbatim REQUIREMENTS (shared context); **its own sub-task + exact file-ownership set**, with a hard boundary ("edit ONLY files in your ownership set; a sibling is handling the rest in parallel"); the feedback items (by rubric ID) in its slice if `feedback` is non-empty; a request for the standard handoff block. Implementers **only edit the working tree and perform no git operations** — see the agent definition.
   d. **Collect + reconcile.** After all hand back, disjoint edits compose in the shared tree with no conflict. If two touched the same file, treat it as a partition error: re-run that overlap with a single implementer before the checkpoint.

2. **Checkpoint** — once all implementers have handed back and slices are reconciled, you commit this round's snapshot:
   - `git add -A && git commit -m "loop-eng <slug> round NN [WIP]"` (NN zero-padded);
   - `sha_i=$(git rev-parse HEAD)`, remember the previous round's `sha_{i-1}` (round 1's previous = `base_sha`).
   - These WIP snapshots get squashed at wrap-up, so the message is free-form.

3. **Review** — call `loop-impl-reviewer` via the Agent tool; the prompt MUST include:
   - the **user's verbatim REQUIREMENTS** + the **full frozen `rubric.md`** (same copy every round, do not regenerate);
   - **the grounding anchor (SHA range)**: have it review `git diff <base_sha>..HEAD` for the complete accumulated change (rubric coverage is judged against the full state), optionally consulting this round's increment `git diff <sha_{i-1}>..<sha_i>`. It only reads git (diff/log/show) and never commits/resets/edits code.
   - the implementer handoff is passed **only as an "unverified claim"** ("this is what the implementer says it did — do not trust it, judge against the real diff");
   - **require it to go through the rubric item by item**: for each `R1…Rn` give `SATISFIED / PARTIAL / MISSING` + file:symbol evidence, as a table; for runtime-behavior items, **require it to actually run the project's tests for hard evidence** (see the tool-grounding rule in its agent definition); tests not green ⇒ never SATISFIED;
   - decision rule: **APPROVED iff every item is SATISFIED**; any PARTIAL/MISSING → **NEEDS_ITERATION**, remaining list referencing the failing rubric IDs (e.g. "R3 MISSING: empty body not handled").

4. **Branch + regression guard**:
   - reviewer = **APPROVED** → break out to "Wrap-up".
   - reviewer = **NEEDS_ITERATION**:
     - **Regression check**: if this round knocked a previously-SATISFIED item back to MISSING/PARTIAL → `git reset --hard <sha_{i-1}>` to discard this round, fold "which items regressed + don't break them again" into `feedback`, and go on; otherwise keep the checkpoint.
     - Assign the reviewer's remaining list to `feedback`; go to round i+1.
   - reviewer indicates **missing info / cannot decide / blocked** → **stop the loop**, hand the blocker back to the user, don't force another round (loop branch left as-is).

**At the end of each round** do two things:
- **Persist**: write the reviewer's full verdict to `<run-dir>/round-NN-review.md` (NN zero-padded). It must contain at least: round number, **this round's checkpoint SHA (`sha_i`) and the base SHA**, the verdict (APPROVED / NEEDS_ITERATION / BLOCKED), a **coverage table by rubric ID** (`R1…Rn` each SATISFIED/PARTIAL/MISSING + file:symbol), the test results (command run, green/red), the remaining list (by failing R ID), and the changed-files list (`git diff --stat <base_sha>..<sha_i>`). Same R IDs across rounds, so `round-01` and `round-02` diff by ID. **Only Write new files, never modify a previous round's.**
- **Print** a one-line summary: `Round i → <APPROVED|NEEDS_ITERATION>`, with counts of remaining items and changed files on NEEDS_ITERATION, plus the review-file path.

## Guardrails

- **Iteration cap**: `max` rounds reached without APPROVED → stop, report "what is still missing" (last round's remaining list), suggest next steps (more rounds / human intervention / ambiguous requirements).
- **No self-approval**: APPROVED can only come from the reviewer's verdict text; never conclude on its behalf or wrap up because it "looks close enough".
- **Requirement ambiguity**: if the reviewer repeatedly flags the requirements themselves as contradictory/vague, stop and clarify with the user rather than spinning at the implementation level.

## Wrap-up (after APPROVED or hitting the cap)

Write `<run-dir>/summary.md` (result, total rounds, one verdict line per round + round file and SHA, base/loop branch, final changed-files list), and `git add docs/reviews/... && git commit` to fold it into the loop branch. Then:

**Git shaping (proposed only on APPROVED, never automatic)**: the loop branch is a chain of `[WIP]` checkpoints. **After the user consents**, squash into one clean commit:
- `git reset --soft <base_sha> && git commit`, message you draft (match the recent `git log` style / any ticket-prefix convention, don't hardcode ticket numbers; body = one-line overview + evidence the rubric is fully green + the review directory path);
- after squashing, the loop branch = base branch + one clean commit;
- **cap reached / blocked**: **do NOT squash**; keep the WIP chain, report which R items still fail, let the user pick it up or discard the branch.

Give the user a summary: result; total rounds + one verdict line & SHA each; branch info (base, loop, squashed?) + next-step options ("merge back / open a PR" or discard `git switch <base> && git branch -D <loop branch>`); run directory + file listing; a reminder to run end-to-end/integration tests if the project's conventions require it. **Never auto push / merge / open a PR** — get explicit confirmation first and follow the project's push conventions and target remote.

## Driving it autonomously with `/goal`

This skill exists so a short `/goal` can invoke the whole discipline without pasting it:

```
/goal Use the loop-engineering skill to implement <requirements>.
DONE WHEN: loop-impl-reviewer's latest verdict is APPROVED, every rubric item is SATISFIED, and the test command exits 0 — and that verdict appears verbatim in your final message.
```

**Completion sentinel — the one rule that makes `/goal` reliable.** `/goal`'s completion check is a small, fast evaluator reading the transcript; it cannot re-judge the whole methodology and must not be trusted to. The *hard* judgment stays with `loop-impl-reviewer`; the evaluator only pattern-matches an unambiguous sentinel:
- End the turn by echoing the reviewer's final verdict **verbatim** — the literal keyword `APPROVED`, the full rubric coverage table (every `R#` = SATISFIED), and the test command with its exit status.
- **Never emit `APPROVED` yourself as the orchestrator** — relay it only when it genuinely came from `loop-impl-reviewer`. A fabricated sentinel is a false completion.
- If the loop stops on the cap or a blocked/ambiguous state, do NOT emit the sentinel — report what's still failing (by rubric ID) so `/goal` keeps iterating or hands back.

Pair with **auto mode** for unattended runs.
