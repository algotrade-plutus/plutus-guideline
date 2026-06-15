# Handoff: Upgrade the Plutus Standard Guideline V1 → V2

> **For the executor:** this is a self-contained handoff. You are running on the same machine as
> the `plutus-automation-scoring` repo, so every file path cited below is directly readable. Start
> with the "Source-of-truth map" — it tells you where each normative claim is enforced in code.

## Context

This repo (`plutus-automation-scoring`) shipped the **automation** for the Plutus Standard: the
`plutus-verify` framework (v0.2.10) plus the `plutus-transform` and `plutus-scoring` skills. The
automation now defines, mechanically, what "Plutus-compliant" means — a `.plutus/manifest.yaml`
contract, a per-step `results.json` contract, `plutus check` exit-code semantics, and a
50/25/10/15 scored rubric.

The **published standard** — the human-facing document at
`https://github.com/algotrade-plutus/plutus-guideline` — still describes V1: a prose standard built
around required `README.md` sections, "reproducibility by reading the README," a `DATA.md` spec, and
a single-percentage badge system (Bronze 50–69 / Gold 70–99 / Platinum 100). Nothing in the
published standard mentions the manifest, the results contract, the units vocabulary, the data
tiers, `plutus check`, or the four scoring buckets.

**The gap is the problem:** authors reading V1 cannot produce a repo the automation will pass, and
the badge percentage no longer corresponds to how the score is actually computed. This handoff
specifies — section by section — what the **V2 guideline document must say** so that what it asks
authors to do lines up exactly with what `plutus-verify` enforces, and so the badge number again
means something precise and reproducible.

**This is a plan-only handoff.** It does not draft the guideline prose. For each section of the new
guideline it states (a) what the section must require/say, (b) the normative MUST/SHOULD statements,
(c) the automation behavior it maps to with source file paths in this repo, and (d) the existing
internal doc to lift wording from. The executor (a human or a follow-up agent) writes the actual
prose into the external `plutus-guideline` repo.

### Decisions already made (with the user)
- **Structure:** reorganize around the two contracts + four buckets, with the 9-step process as the
  spine. **Preserve the V1 standard** in the repo as an archived, versioned document — do not delete it.
- **Scoring/badges:** keep the Bronze/Gold/Platinum badge tiers, but **redefine the percentage as the
  sum of the four weighted buckets** (Reproducible 50 + Tidy 25 + Standardized 10 + Innovative 15).
  No hard gate on `plutus check` (the user chose continuity, not the gate variant).
- **Compliance path:** **spec-first.** The manifest + results contracts ARE the normative
  requirement; any tool that emits them satisfies the standard. The CLI/skills are documented as a
  recommended convenience, not the definition of compliance.
- **Deliverable:** plan-only, section-by-section spec.

---

## Source-of-truth map (where each normative claim is enforced)

Every normative statement in the V2 guideline must be traceable to code in this repo. Authoritative
sources:

| Concern | Authoritative file(s) |
|---|---|
| Manifest schema (2.0), required keys, enums | `plutus_verify/spec/schema.py` |
| Cross-field invariants (unique ids, references, data-step commands) | `plutus_verify/spec/validator.py` |
| Manifest load funnel + error type | `plutus_verify/spec/loader.py` |
| `results.json` contract (1.0), units, kinds, atomic write | `plutus_verify/sdk/schema.py`, `plutus_verify/sdk/run.py` |
| Comparison & tolerance semantics | `plutus_verify/spec/runtime/artifact_compare.py` |
| `plutus check` flow + exit codes | `plutus_verify/__main__.py` (check), `plutus_verify/scaffold/check.py` |
| 9-step taxonomy keys | `plutus_verify/constants.py` |
| Scoring rubric (50/25/10/15) | `skills/plutus-scoring/references/compliance-rubric.md` |
| Data tiers, design rationale | `docs/design/v2-spec-and-execution.md` |
| User-facing manifest/results explainer (lift wording) | `docs/feature/v2-manifest.md` |
| Leak/inputs hardening (G11/G12) | `docs/design/secret-and-leak-hardening.md` |
| Free-form vs fixed template boundary | memory `project_form_filling_scope.md` |

**Rule for the executor:** if a normative claim cannot be cited to one of these, it does not belong
in the standard. Where code and prose disagree, code wins (and a skill-feedback note is filed per
`project_skill_feedback_test_bench.md`).

---

## Target document structure (the new `plutus-guideline` repo)

```
plutus-guideline/
  README.md                      ← V2 standard (normative), structure below
  AUTHORING.md (optional)        ← recommended workflow / tooling pointers (non-normative)
  data/DATA.md                   ← kept, lightly updated (data tiers cross-ref)
  data/data-field-description.md ← kept as-is
  templates/manifest.yaml        ← annotated reference manifest (skeleton, non-normative example)
  versions/
    v1/README.md                 ← the current V1 standard, moved here verbatim (archive)
  CHANGELOG.md                   ← "v2 supersedes v1" entry + version table
```

The new `README.md` sections (the spine is the 9-step process; the contracts and buckets are the
new normative core):

```
0  Introduction — what "compliant" means in v2
1  The two machine contracts
   1.1  The manifest  (.plutus/manifest.yaml, schema_version "2.0")
   1.2  The results contract  (.plutus/run/<step>/results.json, schema_version "1.0")
2  The 9-step taxonomy → manifest steps   (spine)
3  Data tiers
4  Verification — the `plutus check` contract (exit codes)
5  Compliance & scoring (50/25/10/15) and badges
6  README & documentation requirements (the Tidy bucket — V1's section rules migrate here)
7  Recommended workflow & tooling (non-normative)
8  Sample projects & badges
   Appendix A  Versioning & relationship to V1
```

---

## Section-by-section specification

### §0 Introduction — what "compliant" means in v2

- **Must say:** V2 keeps V1's mission (reproducibility up to Step 7 / Paper Trading) but moves the
  definition of "reproducible" from *"a human can follow the README"* to *"`plutus check` exits 0
  against a declared contract."* State plainly: the README is still required, but it now documents
  the *why*; the **manifest is the authoritative *what***.
- **Must state the two-contract model** in one paragraph: author declares a `.plutus/manifest.yaml`;
  each pipeline step emits a strict `results.json`; the verifier builds a container, runs the steps,
  and compares produced metrics/artifacts against the manifest's `expected` block.
- **Must keep** the hosting requirement (repos under the ALGOTRADE org) and the "self-contained,
  runnable without contacting the author" principle from V1 §0.
- **Maps to:** `docs/design/v2-spec-and-execution.md` ("the manifest is the plan"; inversion from v1
  LLM-extraction). Lift the framing from there.

### §1 The two machine contracts (NEW — normative core, spec-first)

#### §1.1 The manifest — `.plutus/manifest.yaml`
- **Normative MUST:** a compliant repo contains `.plutus/manifest.yaml`, committed to git, with
  `schema_version: "2.0"` (const), that validates against the JSON Schema (Draft 2020-12) and passes
  all cross-field invariants.
- **Must enumerate the required top-level keys** exactly as the schema does: `schema_version`,
  `repo {name, primary_language}`, `env {base, python_version, requirements_file, os_packages,
  gpu_required}`, `secrets [] ` (may be empty), `data_sources {processed[], raw[]}` (both arrays
  required), `steps[]` (≥1), `expected[]` (may be empty). `nine_step_coverage` is optional.
- **Must document the step shape** and which fields gate behavior: `id` (unique, non-empty),
  `nine_step` (one of the 7 keys or `null`), `required` (**gates the exit code**), `command`
  (**required for `data_collection`/`data_processing` steps**), `network` (`none`|`bridge`|`host`,
  default `none`), `verification_mode` (`execute`|`artifact_check`, default `execute`),
  `timeout_seconds` (default 1800), `inputs[]`, `outputs[]`, `depends_on[]`.
- **Must flag the `inputs` foot-gun (G12):** `inputs` is a **complete-coverage allowlist** of files
  staged into the container, not just "data inputs" — omitting the script binary causes `exit=2`.
  Recommend `inputs: []` (fall back to `.dockerignore`) unless the author deliberately scopes it.
- **Must document the invariants** a reader can violate accidentally: unique step ids; every
  `depends_on`, `expected.step_id`, `data_sources.*.satisfies`, and `secrets.*.used_by` must
  reference an existing step.
- **Maps to:** `spec/schema.py` (keys/enums/consts), `spec/validator.py` (invariants),
  `spec/loader.py` (`ManifestLoadError`). Lift the user-facing explanation from
  `docs/feature/v2-manifest.md`.

#### §1.2 The results contract — `.plutus/run/<step_id>/results.json`
- **Normative MUST:** each step the manifest marks for execution writes
  `.plutus/run/<step_id>/results.json` with `schema_version: "1.0"`, a matching `step_id`, and
  `metrics`/`artifacts`/`metadata`.
- **Must specify the metric rules:** `name` is `snake_case` (`^[a-z][a-z0-9_]*$`); `value` is a
  number; `unit` ∈ `{fraction, ratio, count, currency_usd, seconds}`. **`percent` is rejected** —
  state the canonical-decimal rule explicitly (0.42 = 42% as `fraction`; unbounded ratios like Sharpe
  as `ratio`). This is the single most common authoring error to call out.
- **Must specify artifact rules:** `name` snake_case, `path` relative to the step working dir, `kind`
  ∈ `{chart, csv, json, image, other}`.
- **Must state** that the write is atomic and happens only on a clean step exit (a raised exception
  produces no file → the step reads as failed), and that `duration_seconds` + `git_commit` are
  auto-injected.
- **Spec-first framing:** the contract is the requirement; the Python SDK
  (`import plutus_verify as pv; with pv.step(...) as r: r.metric(...)`) is **one convenience
  producer**. Any language/tool that writes a conforming `results.json` complies. Show the SDK
  snippet as the recommended path, not as the definition.
- **Maps to:** `sdk/schema.py` (units/kinds/regex/consts), `sdk/run.py` (atomic write, auto-fields).
  Lift wording from `docs/feature/v2-manifest.md`.

### §2 The 9-step taxonomy → manifest steps (the spine)

- **Must publish the 7 canonical keys verbatim** and map each to its V1 README section so existing
  authors recognize the lineage:
  `step_1_hypothesis`, `step_2_data_collection`, `step_3_data_processing`, `step_4_in_sample`,
  `step_5_optimization`, `step_6_out_of_sample`, `step_7_paper_trading`.
- **Must explain the escape hatch:** repo-specific steps that don't fit a canonical key use
  `nine_step: null` + a `label` (e.g. an ML model-training step). Per the
  `project_form_filling_scope` memory: the 7 keys + field names + enums are the **fixed** contract;
  step *count beyond the 7*, metric *names*, and section *content* are legitimately repo-specific.
  The standard fixes the frame, not the contents.
- **Must clarify scope:** reproducibility is expected through Step 6 (minimum) / Step 7 (ideal),
  carried over from V1 §0; Steps 8–9 remain out of scope. Paper trading is typically not
  reproducibly executable — document it via `nine_step_coverage` or an optional `artifact_check` step
  rather than forcing execution.
- **Maps to:** `plutus_verify/constants.py` (`NINE_STEP_KEYS`); `nine_step_coverage` handling in
  `spec/schema.py`.

### §3 Data tiers (NEW — replaces V1's single "put it on Google Drive" note)

- **Must define the four tiers** and which `data_sources` shape each uses:
  - **Tier 1 — committed CSV:** data in-repo; sources may be omitted; steps read directly.
  - **Tier 2 — Drive / GitHub release:** declared under `data_sources.raw`/`processed` with
    `kind`, `url`, `expected_layout` (globs the download must produce), `satisfies` (step ids).
  - **Tier 3 — database:** a `bridge`-network step with `secrets`; lazy connection; `data_collection`
    runs the query.
  - **Layered:** Drive primary + DB fallback (most complex; discourage for first-time authors).
- **Must explain `processed` vs `raw`:** `processed` (ready-to-run) is tried first and falls through
  to `raw` or the step command.
- **Must cross-reference** `data/DATA.md` and `data/data-field-description.md` (kept from V1) for the
  field spec.
- **Maps to:** `docs/design/v2-spec-and-execution.md` (data-tier section); `spec/schema.py`
  (`DataSource` shape).

### §4 Verification — the `plutus check` contract (NEW)

- **Must publish the exit-code contract verbatim** as the normative definition of "reproduced":
  - **0** — all **required** steps executed cleanly **and** all metric/artifact comparisons passed.
  - **1** — all required steps ran, but ≥1 comparison failed (PARTIAL / soft fail).
  - **2** — a required step failed execution, or a preflight check failed.
- **Must state** that only `required: true` steps gate the exit code (optional steps can fail
  without blocking) and that **exit 0 is the bar for the full "reproducible" claim.**
- **Must document the comparison semantics:** metrics matched **by name** against `expected.metrics`;
  artifacts matched **by path** against `expected.artifacts`. Tolerance kinds: `exact`
  (`math.isclose`), `absolute` (`|a−e| ≤ v`), `relative` (`|a−e|/|e| ≤ v`, degrades to absolute when
  `e=0`). Artifact compare modes: `json_numeric_tolerance`, `visual_similarity` (needs a baseline +
  vision endpoint; falls back/SKIPs without one), `byte_exact`.
- **Must include** the minimal author-side command and what a green run looks like:
  `plutus check . --secrets-from-env` → `exit=0`, metrics `ok`, artifacts `ok`/`SKIP`/`WARN` (never
  `FAIL`).
- **Maps to:** `plutus_verify/__main__.py` (check subcommand), `scaffold/check.py` (exit codes),
  `spec/runtime/artifact_compare.py` (tolerance/compare). Note Docker is a runtime prerequisite.

### §5 Compliance & scoring (50/25/10/15) and badges (REPLACES V1's implicit %)

- **Must publish the four weighted buckets** and the per-bucket tier scales exactly as the rubric
  defines them:
  - **Reproducible — 50:** 50 clean `plutus check` exit 0 / 45 green after manifest-side workarounds /
    35 green only after touching tracked config / 20 host-only match / 0 not reproduced.
  - **Tidy / well-documented — 25:** ~5 pts each for README structure + metric tables, `.env.example`
    that parses, all data inputs documented, parameter pipeline accurately described,
    `.python-version` pin + CI workflow.
  - **Standardized / template — 10:** 10 canonical 4-step shape + externalized params + predictable
    output paths + no module-level side effects / 5 one deviation / 0 needs rework.
  - **Innovative — 15:** 15 novel metrics/diagnostics / 8 thoughtful but conventional / 0 textbook.
- **Must define the percentage** as the rounded (nearest 5%) sum of the four bucket scores, and map
  it onto the **kept** badge tiers: Bronze 50–69%, Gold 70–99%, Platinum 100%, plus the existing
  Sample / PROTO badges. State that the number is now machine-derived via the `plutus-scoring`
  routine, not a hand-judged estimate.
- **Must note** (since no hard gate was chosen) that Reproducible is 50% of the total, so a repo that
  fails `plutus check` is capped well inside Bronze in practice — make this explicit so the
  continuity-with-V1 choice doesn't read as "repro is optional."
- **Maps to:** `skills/plutus-scoring/references/compliance-rubric.md`;
  `docs/feature/plutus-scoring-skill.md`.

### §6 README & documentation requirements (the Tidy bucket — V1's §1 migrates here)

- **Must carry over V1's README section template** (Abstract, Introduction, Hypotheses, Data,
  Implementation/How-to-Run, In-sample, Optimization, Out-of-sample, Paper Trading, Results,
  Conclusion, Reference, Final Report link) — but reframe it as **the evidence the Tidy bucket scores**,
  not as the definition of compliance.
- **Must add the v2-specific documentation requirements** that the rubric actually checks: a
  metric table consistent with `expected`, a `.env.example` that parses under `source` (no unquoted
  `<placeholder>` — call out the G3 shell-parse failure), every data input documented (no surprise
  dependencies), an accurate parameter/optimization pipeline description, a `.python-version` pin, and
  a CI workflow running `plutus check`.
- **Must keep** the Final Report/Paper rules from V1 §1.1 (optional in README, fuller detail; LaTeX
  source + build instructions if used) and fold "completeness of docs" into the Tidy points.
- **Maps to:** `compliance-rubric.md` (Tidy sub-points); V1 §1 (carried structure); G3 in
  `skills/plutus-transform/references/known-gotchas.md`.

### §7 Recommended workflow & tooling (NEW, non-normative)

- **Must be explicitly labeled non-normative** (spec-first: compliance = the contracts, not the
  tool). Then document the convenience path: `plutus init` to scaffold manifest + CI + example;
  instrument steps via the SDK; `plutus snapshot` to capture baselines; `plutus check` to verify;
  the `plutus-transform` skill for an end-to-end v1→v2 migration; the `plutus-scoring` skill for the
  score + ranked improvements + a copy-paste re-run command.
- **Should point** authors at `templates/manifest.yaml` (the annotated skeleton this handoff adds to
  the repo) and at the gotcha catalogue (G1–G7, G11–G12) as a troubleshooting reference.
- **Maps to:** `plutus_verify/__main__.py` (init/snapshot/transfer/bootstrap subcommands);
  `skills/plutus-transform/SKILL.md`; `skills/plutus-scoring/SKILL.md`.

### §8 Sample projects & badges

- **Must keep** the V1 sample-projects table and badge gallery, but add a column / note for the v2
  bucket breakdown where a repo has been re-scored, and mark which samples are v2-verified
  (`plutus check` green) vs v1-legacy.
- **Should** designate at least one repo as the canonical v2 reference example (candidate:
  `cs408-2026/Group09-BuyHighSellLow`, the test-bench repo from the
  `project_skill_feedback_test_bench` memory) and link its manifest.

### Appendix A — Versioning & relationship to V1

- **Must preserve V1 verbatim** under `versions/v1/README.md` and link it from the new README.
- **Must add a CHANGELOG/version table** stating: v2 supersedes v1; v2 corresponds to
  `plutus-verify` ≥ 0.2.10, manifest `schema_version "2.0"`, results `schema_version "1.0"`; and that
  v1-badged repos remain valid but are scored under the legacy prose criteria until re-verified.

---

## Repo-level changes the executor performs (outside the README prose)

1. Move the current V1 `README.md` → `versions/v1/README.md` (verbatim archive).
2. Write the new V2 `README.md` per §0–§8 + Appendix A above.
3. Add `templates/manifest.yaml` — an annotated skeleton covering every required key and the step /
   expected / data_sources shapes (derive from `skills/plutus-transform/references/manifest-templates/`).
4. Lightly update `data/DATA.md` to cross-reference the data tiers (§3); leave
   `data/data-field-description.md` unchanged.
5. Add `CHANGELOG.md` with the version table (Appendix A).
6. (Optional) Add `AUTHORING.md` if §7 grows beyond a section — but keep the normative contracts in
   the README.

---

## Verification (how to confirm the V2 guideline is faithful before publishing)

1. **Code cross-check (claim-by-claim):** for every normative MUST in §1–§5, open the cited source
   file and confirm the guideline's wording matches the code (enum values, key names, regex, exit
   codes, bucket weights). Any mismatch → fix the guideline, or file a `.plutus/skill-feedback.md`
   note if the *code* is wrong (per `project_skill_feedback_test_bench`).
2. **Round-trip a real repo:** take the test-bench repo (`Group09-BuyHighSellLow`), confirm its
   `.plutus/manifest.yaml` satisfies every requirement the guideline states, run
   `plutus check . --secrets-from-env` and confirm `exit=0`, then run the `plutus-scoring` skill and
   confirm the produced bucket scores + badge % match how §5 says the number is computed.
3. **Naive-author dry run:** have a reader who only read the new README produce a minimal manifest +
   one instrumented step from scratch and run `plutus check`. If they hit G1–G7/G12 without the
   guideline having warned them, add the warning. This is the real test of "spec-first" sufficiency.
4. **Badge continuity:** re-score one existing v1-badged sample repo under the new bucket sum and
   confirm its badge tier is explainable (and document any tier change in the CHANGELOG).

---

## Out of scope / explicitly deferred

- Changing any `plutus-verify` code or the scoring rubric — the guideline documents the automation as
  it is; discrepancies are filed as feedback, not fixed here.
- Re-scoring the full sample-projects table (do the canonical example + one continuity check; bulk
  re-scoring is a separate effort).
- Building the reference template *repo* with instrumented source (the user chose plan-only; the
  annotated `templates/manifest.yaml` skeleton is the in-scope artifact, full sample repo is not).
