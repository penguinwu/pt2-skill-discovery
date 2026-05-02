# pt2-skill-discovery — Project CLAUDE.md

## What this repo is

Research harness for compile-skill investigation. Per-case multi-trial experiments, parallel runner, validators, skill catalog under research. Sister repo: [penguinwu/oss-model-graph-break-corpus](https://github.com/penguinwu/oss-model-graph-break-corpus) supplies model targets + sweep infrastructure.

This repo split from corpus's `discovery/` dir on 2026-05-02 (see corpus's `discovery/MIGRATED.md` for the dual-run-period policy). Cold-start verified: this repo has zero runtime dependencies on the corpus repo. The `from sweep.explain import ...` import was duplicated to `scripts/explain_helper.py` (~75 LOC, self-contained).

## Layout

```
pt2-skill-discovery/
├── scripts/    — runner, validator, perf, parallel orchestrator, smoke (Python package)
├── cases/      — discovery cases (one .py per case)
├── experiments/— per-experiment dirs with plan.md + reports/
├── design/     — design.md, experiment_lifecycle.md, parallel-runner-design.md, …
├── skills/     — skill markdown (debug-graph-breaks, per-case-analysis, daily-briefing)
└── tools/      — (slated for Phase 5 Tier 5 port from corpus/tools/)
```

Imports inside this repo use `from scripts.X import …` and `from cases.Y import …`. Never import from corpus.

## Experimentation discipline (Tier A — full)

The only tier that lives here. Multi-trial investigations using the harness require:

| Stage | Mandatory artifact |
|---|---|
| Pre-launch smoke | `scripts/smoke_test.py` exits 0 (Layer 1 + Layer 2 both green) |
| Pre-launch plan | `experiments/<YYYY-MM-slug>/plan.md` with complete `## Pre-launch gates` section (Gates 0-4) |
| Launcher gate | `scripts/launch_parallel.py` refuses to start without the above. Bypass requires `--lifecycle-bypass --reason "<text>"` (writes the reason to plan.md as audit trail) |
| Lifecycle doc | `design/experiment_lifecycle.md` — full Gates 0-4 + closure rule |

Read `design/experiment_lifecycle.md` and walk through Gates 0-4 in strict order before any new experiment launch.

### Universal floor — closure rule

> Before launching any new run, every loose end from prior runs is CLOSED with verified fix or EXPLICITLY DEFERRED with written reason in plan.md. No silent drops. No "I'll fix it on the next run."

Hardened on 2026-04-27 after a chain of V8 experiments cascaded multi-hour failures because each "next run" papered over an unresolved loose end from the prior. The closure rule is the antidote.

### What is NOT an experiment

The lifecycle is for INVESTIGATIONS using the harness — "I want to learn X about model Y by running M variants × N trials." It is NOT for engineering work on the harness itself. Building or modifying `scripts/*.py` is engineering: apply ordinary code review + `scripts/smoke_test.py`. Adding cases is configuration: smoke_test Layer 2 catches schema regressions.

The line: ask "what investigation am I doing? what hypothesis am I testing?" If the honest answer is "I'm building / modifying the tool," it's not an experiment — don't scaffold one.

## Use the scaffold tools — do NOT hand-roll

Multi-trial experiments live under `experiments/<YYYY-MM-slug>/`. Never hand-roll the directory or the GitHub issues — both will drift from the convention.

(Scaffold tools — `new_experiment.py`, `new_case_issue.py`, `queue_task.py` — are slated to be ported from `corpus/tools/` in Phase 5 Tier 5 of the migration. Until then, invoke the corpus copies with `--root ~/projects/pt2-skill-discovery` to write to this repo's paths.)

Issue creation is *orthogonal* to scaffolding. Most local experiments don't warrant team-visible GitHub tracking; use `--with-umbrella-issue` only for experiments that produce shipped findings or need cross-team comments.

## Daily briefing

`skills/daily-briefing/SKILL.md` is this repo's morning brief. The brief reads `tools/brief_data.py` output (a JSON snapshot of plan / experiment / OPEN-LOOPS / commits / closed-issues / handoff state). Plans with `last_check` >3 days old (while `status: active`) get flagged STALE; `needs-input` items >=2 days old get a `←` flag.

Per Otter's local CLAUDE.md, an Otter-level meta-skill that orchestrates per-project briefs across both repos is a follow-up — not migration scope.

## Discovery Experiments — workflow

**Analysis outputs land on main directly + headline summary on the per-case issue.** Anything under `experiments/<exp>/reports/` (findings docs, fingerprint CSVs, synthesis docs) commits to main directly. After landing, post a headline summary as a comment on the per-case issue (or umbrella for cross-case synthesis). No PR-FIRST workflow — discontinued 2026-04-25 per Peng.

## Push permission

Otter is authorized to `git push` to this repo (extended from the corpus authorization on 2026-04-30 for migration Phase 7). Same guardrails:
- Only push to `main` — no remote branches
- Never force push (`--force`, `--force-with-lease`)
- Always run `git log --oneline origin/main..HEAD` before pushing and report the commit range
- If push is rejected (remote ahead), stop and report — do not force
- Log every push (timestamp + commit range) inline to Peng

Remote URL: SSH (`git@github.com:penguinwu/pt2-skill-discovery.git`). HTTPS is BPF-blocked at the proxy under agent identity — do not switch to HTTPS.

## Operating rules from shared CLAUDE.md

This repo inherits the org-wide rules at `~/.myclaw/CLAUDE.md` and Otter's local rules at `~/.myclaw/spaces/AAQANraxXE4/CLAUDE.md`. Where local rules conflict with shared rules on project-scoped work, local wins (narrower scope).
