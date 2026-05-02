# Technical report — `vits_model_train` skill discovery (Stage 1: skeleton, facts only)

**Status:** Draft, Stage 1 (skeleton). No interpretation in this version. Discussion / patterns / hypotheses come in Stage 2 + 3 only after Peng signs off on the facts here.

**Supersedes:**
- `findings_v1_archived_20260428.md` — earliest version, retracted at top (2026-04-27) due to incorrect "Inductor noise floor 2.0" attribution. The real cause was validator seeding gap (np.random unseeded → layerdrop variability → 2.0 max diff). Do not act on its L/M/S/R reclassifications.
- `findings_v2_archived_20260429.md` — yesterday's draft, flagged 2026-04-29 by Peng for fabricated insights, unverified Dynamo-mechanism claims, content-free "Behaviors" section, and a strategy taxonomy built without grepping the actual catalog. Most of it is unsalvageable as written.

This report rebuilds from trial artifacts. Every claim must have either (a) a path to source data (file path, line number, or trial-id), (b) a citation in code, or (c) an explicit "not yet known" / "speculative" tag.

---

## Self-discipline rules (kept visible to enforce them)

1. *Every claim has a source citation.* If I can't cite it, I write "not yet known."
2. *Mechanism claims about Dynamo internals require either a source-line citation or an "unverified" tag.* No more "the agent's fix works because Dynamo can't trace X" without showing the break_reason and the fix diff.
3. *No "behaviors" section unless each behavior changes a skill-design decision.* Observation alone is not insight.
4. *Strategy taxonomy requires grep-verified provenance against `debug-graph-breaks/SKILL.md`.* "In catalog" / "in agent prior" / "novel composition" must be evidenced.
5. *Fullgraph (GB=0) is one signal, not the only one.* Partial GB reduction is also harvested and reported.
6. *Speculation is allowed but capped and labeled.* Stage 3 only, max 1-2 paragraphs, marked `[hypothesis]`.

---

## 1. Background

**Case:** `vits_model_train` — HuggingFace VITS (`VitsModel`) compiled with `torch.compile(fullgraph=True)` in train mode. Source: `discovery/cases/vits_model_train.py`.

**Question:** when the `debug-graph-breaks` skill is loaded into the discovery agent, does the agent's reasoning, fix-space, or fix-shape change vs. bare Claude (no skill)?

**Why this case:** VITS is a real HF model with multiple distinct break shapes in train mode. Multiple seeds + multiple variants give us enough cells to compare skill vs. no-skill behavior with seed-level variability visible.

**Catalog under test:** `skills/debug-graph-breaks/SKILL.md`. *Provenance: copied into pt2-skill-discovery from corpus's `discovery/skills/debug-graph-breaks/SKILL.md` (270 lines, last touched ~2026-04-22 per git log).*

---

## 2. Methodology

### 2.1 Variants

Source: `scripts/variants.py`. Each variant is a constraint added to the agent's prompt. Quote of each variant's constraint is in the source; below is a summary.

| ID | Name | Constraint | Source |
|---|---|---|---|
| V0 | baseline | No extra constraint beyond the case prompt | `variants.py` |
| V2 | (TODO: read variants.py to fill in) | TODO | `variants.py` |
| V4 | (TODO) | TODO | `variants.py` |
| V6 | no_config_flags | Disallow editing `_dynamo.config.*` flags | `variants.py` |
| V8 | model_layer_only | Disallow editing the test/baseline script; fix must live in model source | `variants.py` |
| V9 | no_setup_at_all | (TODO: confirm full constraint) | `variants.py` |

*Action item before Stage 2: fill in V0/V2/V4/V9 by reading the source. Do not infer; quote the constraint text.*

### 2.2 Skill conditions

Two arms per variant:
- `noskill` — bare Claude Code agent, no skill loaded.
- `SKILL` (sometimes labeled `debug_graph_breaks` in older runs) — same agent with `debug-graph-breaks/SKILL.md` loaded.

### 2.3 Trial counts

Two distinct runs of trial data exist; *both are cited explicitly because their schemas and methodology differ.*

**Original run** — `fingerprints.csv` in this directory. 30 trials. Run ids: `20260425-144345` (V0/V2/V4/V6 × {SKILL, noskill} × 3 seeds = 24), `20260426-014253` (V8 × {SKILL, noskill} × 3 seeds = 6 add-on). *Pre-determinism-fix*: validator used `torch.manual_seed` only; numpy + python.random were unseeded, so VITS layerdrop's `np.random.uniform` produced different layer-drop patterns across forward passes. This caused the "2.0 max_diff / 2.0 eager_self_diff" pattern visible in V6 and V8 cells.

**Post-determinism-fix run** — 13 trials in `/tmp/runs/vits-r5v9-batch*/` and `/tmp/runs/vits-r5v9-parallel-relaunch-2026-04-28/`. Run after 2026-04-27 commit `0cd779c` (validator now uses HF `set_seed` covering torch + numpy + python.random). New schema: `result.json` per trial, with `validation.fix_status`, `validation.details.gb_under_canonical_inputs`, `validation.details.gb_call_sites`, `perf`, `perf_tier2`, `flags`. *Trials are uneven across cells* (1–3 per cell, not the planned 3 per cell). Reasons: some cells deferred, some manual relaunches.

### 2.4 Validation

Source: `scripts/validate_runner.py`. After the agent finishes editing, the validator runs the agent's modified model under `torch.compile(fullgraph=True)` against canonical inputs (defined in the case file). Outputs:

- `fix_status`: classification, source `validate_runner.py`. Values seen in trials: `general`, `partial`, `setup-required`, `none`, `runtime-error`. *Definitions need to be quoted from validate_runner.py — open action.*
- `gb_under_canonical_inputs`: number of graph breaks observed under canonical inputs.
- `gb_call_sites`: list of `{reason, file, line}` per break.
- `eager_deterministic` / `eager_self_diff`: two consecutive eager forward passes — diff between them. Should be ~0 with proper seeding.
- `max_diff_compiled_vs_eager`: max abs diff between compiled and eager output on canonical inputs.

### 2.5 Perf-tier-2

Source: `scripts/_measure_case.py`. Runs the compiled model at "realistic" input shapes (different from canonical) and (a) measures wall time, (b) checks that compiled output matches eager at those shapes.

*Misnomer noted (per Peng comment #20):* "perf_tier2" suggests pure performance measurement, but the harness also raises on numerical mismatch at realistic shapes. So it doubles as input-shape robustness testing. Action: rename or split in a follow-up.

### 2.6 Canonical input shapes

Source: `cases/vits_model_train.py`. *Open action: enumerate canonical input shapes and check for any size-1 dimensions (per Peng comment #18 — 0/1 specialization risk). Will fill in here.*

### 2.7 What `debug-graph-breaks` SKILL.md actually contains

Verified by reading `skills/debug-graph-breaks/SKILL.md` (270 lines). Skill structure: 3 phases (Detect → Diagnose → Fix). Concrete techniques the skill recommends:

- *Logging-related escape hatches:*
  - `torch._dynamo.config.reorderable_logging_functions.add(<fn>)` — keep logging, no break
  - `torch._dynamo.config.ignore_logging_functions = [<fn>]` — drop logging during compile
  - `torch._higher_order_ops.print(...)` — compile-safe replacement for `print()`
- *Tracing-control decorators:*
  - `@torch._dynamo.decorators.leaf_function` — keep function as opaque leaf
  - `@torch._dynamo.decorators.nonstrict_trace` — non-strict tracing
  - `@torch._dynamo.disable` — eager fallback for whole function
- *Data-dep handling:*
  - `torch._check(<cond>)` to give Dynamo value-range info on a SymInt
  - `torch._dynamo.config.capture_scalar_outputs = True` to capture `.item()`
- *Graph-break documentation lookup:* fetch `https://meta-pytorch.org/compile-graph-break-site/gb/gbXXXX.html` per encountered GB type
- *tlparse output parsing:* read `failures_and_restarts.html`, stack trie, per-compilation dirs
- *Prioritization rule:* easy wins first; CAUSED_BY_EARLIER_GRAPH_BREAK last

*This is the ground truth for "in the catalog" claims in §4. Strategies the agent uses that are NOT in this list are either prior knowledge or novel — to be classified per-trial.*

---

## 3. Results — original 30-trial run (pre-determinism-fix)

Source: `fingerprints.csv` (30 rows).

### 3.1 fix_status by cell

| cell | count | fix_status distribution |
|---|---|---|
| SKILL × V0 | 3 | setup-required: 3 |
| SKILL × V2 | 3 | setup-required: 3 |
| SKILL × V4 | 3 | setup-required: 3 |
| SKILL × V6 | 3 | general: 3 |
| SKILL × V8 | 3 | general: 3 |
| noskill × V0 | 3 | setup-required: 3 |
| noskill × V2 | 3 | setup-required: 3 |
| noskill × V4 | 3 | setup-required: 3 |
| noskill × V6 | 3 | general: 1, setup-required: 1, none: 1 |
| noskill × V8 | 3 | general: 3 |

*Read:* V0/V2/V4 are dominated by `setup-required` in both arms (agent edited the test/baseline script — fix didn't survive harness re-mounting it). V6 and V8 produce `general` outcomes (fix held up after harness re-mount). Single `none` in noskill_V6_seed=3 (gb_canonical=1 — agent claimed fix but a break remained).

### 3.2 Non-setup-required trials (detail)

Source: `fingerprints.csv` rows where `fix_status != 'setup-required'`. 11 trials.

| arm | variant | seed | fix_status | gb_canonical | max_diff | eager_self_diff |
|---|---|---|---|---|---|---|
| SKILL | V6 | 1 | general | 0 | 2.0 | 2.0 |
| SKILL | V6 | 2 | general | 0 | 2.0 | 2.0 |
| SKILL | V6 | 3 | general | 0 | 2.0 | 2.0 |
| SKILL | V8 | 1 | general | 0 | — | — |
| SKILL | V8 | 2 | general | 0 | — | — |
| SKILL | V8 | 3 | general | 0 | — | — |
| noskill | V6 | 1 | general | 0 | 2.0 | 2.0 |
| noskill | V6 | 3 | none | 1 | — | — |
| noskill | V8 | 1 | general | 0 | 2.0 | 2.0 |
| noskill | V8 | 2 | general | 0 | 2.0 | 2.0 |
| noskill | V8 | 3 | general | 0 | — | — |

Where `eager_self_diff = 2.0`: caused by validator seeding gap (np.random not seeded → VITS layerdrop varied across forward passes). After determinism fix (commit `0cd779c`), this should go to 0 across all VITS trials. Re-run pending. Where `max_diff = '—'`: validator hit the `_eager_self_check` shape-mismatch infra bug (commit `4538580` fixed it after these runs).

---

## 4. Results — post-determinism-fix run (13 trials)

Source: `result.json` files in `/tmp/runs/vits-r5v9-batch{,2}/` and `/tmp/runs/vits-r5v9-parallel-relaunch-2026-04-28/`.

### 4.1 fix_status by cell

| cell | count | fix_status |
|---|---|---|
| SKILL × V0 | 1 | general: 1 |
| SKILL × V2 | 1 | none: 1 |
| SKILL × V4 | 1 | general: 1 |
| SKILL × V6 | 1 | none: 1 |
| SKILL × V9 | 3 | none: 3 |
| noskill × V2 | 1 | general: 1 |
| noskill × V4 | 2 | none: 1, general: 1 |
| noskill × V6 | 2 | none: 1, general: 1 |
| noskill × V9 | 1 | general: 1 |

*Coverage gap:* V0/V8 noskill cells have no post-determinism-fix trials. V0/V8 SKILL has 1 each. Many cells are n=1 — *very sparse for any per-cell claim*. V9 is the only cell with n=3 (SKILL only); noskill V9 is n=1.

### 4.2 GB count under canonical (post-fix trials)

| trial | fix | gb_canonical | flags |
|---|---|---|---|
| SKILL_V0_1 | general | 0 | file-mutated:modeling_vits.py |
| SKILL_V2_1 | none | 2 | file-mutated:modeling_vits.py |
| SKILL_V4_1 | general | 0 | file-mutated:modeling_vits.py |
| SKILL_V6_1 | none | 10 | file-mutated:modeling_vits.py |
| SKILL_V9_1 (waveA) | none | 17 | file-mutated:modeling_vits.py, tier1-tier2-direction-mismatch |
| SKILL_V9_1 (waveB) | none | 1 | file-mutated:modeling_vits.py |
| SKILL_V9_parallel | none | 1 | file-mutated:modeling_vits.py |
| noskill_V2_1 | general | 0 | file-mutated:modeling_vits.py, perf_tier2-parse-error |
| noskill_V4_1 | none | 1 | file-mutated:modeling_vits.py, file-mutated:baseline_vits.py |
| noskill_V4_parallel | general | 0 | file-mutated:modeling_vits.py |
| noskill_V6_1 | none | 1 | file-mutated:modeling_vits.py, file-mutated:baseline_vits.py |
| noskill_V6_parallel | general | 0 | file-mutated:modeling_vits.py |
| noskill_V9_1 | general | 0 | file-mutated:modeling_vits.py |

*Note:* "fix=none, gb_canonical>0" trials *did* edit the model but the canonical re-run still shows breaks. SKILL_V9 waveA's gb_canonical=17 means the fix had essentially no effect (agent's edit didn't reduce break count meaningfully). SKILL_V6_1's gb_canonical=10 is similar — agent edited but didn't reach fullgraph.

### 4.3 Per-trial graph-break call sites and edits

*Open action for Stage 1 completion: for each post-determinism-fix trial, list the gb_call_sites (top 3 per trial) and what files+lines the agent edited (from `diff_path`). This is a mechanical exercise from `result.json` data; will populate before requesting Peng's review.*

---

## 5. Methodology gaps (acknowledged)

These are real gaps in the experiment design that Peng surfaced (commits #18, #19, #20) and a few I noticed:

1. *Fullgraph-only success criterion (#19).* Current `fix_status` taxonomy treats `gb_canonical > 0` as failure even if the agent reduced breaks substantially. A fix that goes 17 → 1 is real progress but currently classified `none`. Need a `partial_reduction` status with a quantified metric (initial GB count vs final GB count). Action: define this and re-classify post-determinism-fix trials accordingly in Stage 2.
2. *Canonical input may include size-1 dims (#18).* If any canonical input dim is size 1, Dynamo's 0/1 specialization can mask data-dep dynamism that would otherwise surface. Action: enumerate canonical shapes from `cases/vits_model_train.py`, flag any size-1 dims.
3. *perf_tier2 is a misnomer (#20).* It both measures perf AND tests numerical match at realistic shapes. Should be split or renamed.
4. *Single canonical shape vs single realistic shape (#21).* Need to confirm: how many distinct shapes does validation exercise? Are realistic shapes shapes the agent never saw during discovery? Action: confirm from `_measure_case.py` and `validate_runner.py`.
5. *Sparse cell coverage post-determinism-fix.* Only V9 SKILL has n=3; most cells are n=1. Per-cell behavioral claims at n=1 are unreliable.
6. *Confounded re-run (#4).* Three originally-contaminated trials were re-run with TWO things changed (new prompt + chmod-RO sandbox). I cannot attribute outcome change to either alone.
7. *fix_status definitions are not precisely quoted.* I described them from memory; should pull the exact branch logic from `validate_runner.py`.

---

## 6. Open questions (things I do NOT know)

These are honestly tagged "unknown" — not extrapolated, not speculated.

1. *Why does `is_compiling()` guard cause a graph break in our trials?* Per Peng (#11): if the guard returns a constant True at compile, the branch should fold. Either (a) it doesn't fold in this Dynamo version, (b) the guarded body has other untraceable ops and the guard isn't the proximate cause, or (c) my report misattributed. Need to read break_reasons in trial gb_call_sites to find out which.
2. *Why does `torch_compilable_check()` cause a break?* Olga's analysis says the mechanism is internal `torch._check_with(ValueError, ...)` — ValueError can't be proxied. Need to verify this matches the actual break_reason in our trials.
3. *Why ~2.0 max_diff between compiled and eager on the original V6/V8 trials?* The "Inductor float32 numerics on chained exp/log/tanh/spline" attribution in v2 was speculation, not measured. The real cause was layerdrop variability per the v1 retraction. After the determinism fix it should go to 0; pending re-run confirmation.
4. *Does `@torch.jit.script` actually cause a graph break in VITS trials?* I claimed it did in v2; cannot find the break_reason that supports this in trial data. Likely a fabricated claim — needs verification or removal.
5. *Does the SKILL change agent reasoning, or just the fix shape?* This is the main research question and we don't yet have a clean answer. Cell coverage is too sparse and the contamination history confounds early data. Stage 3 (with re-runs) may answer.
6. *Which strategies the agent uses are catalog-recommended vs prior knowledge vs novel?* The catalog content is now ground-truthed (§2.7). Per-trial strategy classification is a Stage 2 exercise.

---

## 7. What goes in Stage 2 + 3 (NOT in this skeleton)

- *Stage 2:* per-trial strategy classification against the catalog (§2.7), fix-mechanism explanation (with break_reason citations), variant-design exposition (Peng's gem #3 — what each variant probed and what we learned).
- *Stage 3:* discussion / hypotheses, marked `[hypothesis]`, capped at 1-2 paragraphs. Skill-design recommendations IF the data supports them.

---

## 8. Action items before Stage 1 sign-off

Concrete things I need to do before this skeleton is complete enough to send for Peng review:

- [ ] §2.1: read `scripts/variants.py` and quote each variant's constraint verbatim (V0, V2, V4, V9 missing).
- [ ] §2.4: pull `fix_status` value definitions from `validate_runner.py` source.
- [ ] §2.6: enumerate canonical input shapes from `cases/vits_model_train.py`; flag any size-1 dims.
- [ ] §4.3: per-trial gb_call_sites (top 3 per trial) and edited file:lines from diff_path.
- [ ] §6.4: grep trial data for `jit.script` in break_reasons to confirm or refute the v2 claim.
- [ ] §6.1, §6.2: read trial gb_call_sites to verify `is_compiling` and `torch_compilable_check` mechanism claims.
- [ ] §5.4: count distinct shapes used in validation vs perf_tier2 from `_measure_case.py`.

After these are filled in, the skeleton is ready for Peng's review of the *facts*. Only after that goes to Stage 2.
