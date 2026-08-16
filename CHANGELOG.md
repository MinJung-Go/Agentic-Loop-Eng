# Changelog

All notable changes to this plugin are documented here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/); versions track `plugin.json`.

## [0.2.0] - 2026-07-27

Adds an autonomous `/goal` path and simplifies the default loop.

### Added
- **`loop-engineering` skill** — a lightweight, `/goal`-native companion. `/goal`
  already supplies the loop *and* an independent completion evaluator; the skill
  just keeps each turn honest (advance → run the check → surface real output) so
  that evaluator can judge on facts. No rubric, no reviewer sub-agent, no git
  machinery — the quick "keep going until it works" path.
- **`/goal` usage docs** — a walkthrough and a control-flow diagram showing who
  owns the loop/verdict vs. the evidence.

### Changed
- **`/loop-eng` command translated to English.**
- **Single implementer per round by default.** The cooperative parallel fan-out
  is demoted to an optional note for large, cleanly file-disjoint work; the
  default path is now fully sequential (single implementer → single reviewer).

### Positioning
- **`/loop-eng` command** — strong, independent review (frozen rubric + real diff
  + reruns tests) for tasks whose "done" needs judgment.
- **`/goal` + skill** — light autonomous iteration for tasks with an objective,
  self-checkable oracle.

## [0.1.0] - 2026-07-27

Initial release: the Loop Engineering plugin + a single-plugin marketplace.

### Added
- **`/loop-eng` command** — the orchestrator. Drives an implementer↔reviewer loop
  on an isolated git branch until every frozen-rubric item is SATISFIED or the
  iteration cap is hit. Preflight (clean-tree gate, base SHA), a frozen
  `rubric.md`, per-round checkpoint commits, SHA-scoped + tool-grounded review, a
  regression guard, and a squash-on-approval wrap-up that never auto-pushes.
- **`loop-implementer` agent** — turns requirements + reviewer feedback into
  minimal, in-scope changes; never self-approves, never touches git.
- **`loop-impl-reviewer` agent** — the binary quality gate; judges the real
  `git diff` against the rubric and reruns the project's tests for grounding.
- **Marketplace `minjung-go`** — installable via
  `/plugin marketplace add MinJung-Go/Agentic-Loop-Eng`.

[0.2.0]: https://github.com/MinJung-Go/Agentic-Loop-Eng/releases/tag/v0.2.0
[0.1.0]: https://github.com/MinJung-Go/Agentic-Loop-Eng/releases/tag/v0.1.0
