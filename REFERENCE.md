# Plutus Technical Reference

> The complete, normative contract — every field, enum, default, exit code, and API.
>
> This document is for lookup, not reading front-to-back. For *why* the standard exists,
> read the [STANDARD](STANDARD.md). For a *step-by-step walkthrough* of making a project
> compliant, read the [GUIDE](GUIDE.md). This reference is the authority for the exact
> rules those two documents summarize.
>
> *Targets* `plutus-verify` 0.5.1 (manifest `schema_version: "2.0"`, results
> `schema_version: "1.0"`). The schema versions are unchanged since 0.2.10, but the
> nine-step taxonomy changed at 0.3.0 (see [§3](#3-the-nine-step-keys)) and the
> `snapshot`/`check` data flow changed at 0.5.0 (see [§5](#5-the-plutus-check-contract)).
> The authoritative schemas live in the `plutus-verify` package
> (`plutus_verify/spec/schema.py` and `plutus_verify/sdk/schema.py`); any tool that
> validates against them complies. Where this document and the code disagree, the code
> wins — report the discrepancy.

Conventions used in every table below:

- **MUST** = required.
- *optional* = not required, but validated if present.
- *conditional* = required only under a stated condition.
- `additionalProperties: false` is set at every level of both schemas. Unknown or
  misspelled keys are validation errors, not silently ignored. The sole exception is
  the results file's `metadata` object, which is deliberately open.

---

## Contents

1. [Manifest schema](#1-manifest-schema) — `.plutus/manifest.yaml`
2. [Results schema](#2-results-schema) — `.plutus/run/<step_id>/results.json`
3. [The nine-step keys](#3-the-nine-step-keys)
4. [Data resolution semantics](#4-data-resolution-semantics)
5. [The `plutus check` contract](#5-the-plutus-check-contract)
6. [Scoring rubric](#6-scoring-rubric)
7. [CLI reference](#7-cli-reference)
8. [SDK API reference](#8-sdk-api-reference)
9. [Troubleshooting catalogue](#9-troubleshooting-catalogue)
10. [Versioning & schema correspondence](#10-versioning--schema-correspondence)

---

## 1. Manifest schema

`.plutus/manifest.yaml`, committed to git, valid YAML with a mapping root, validating
against the Plutus v2 JSON Schema (Draft 2020-12) and all cross-field invariants
([§1.7](#17-cross-field-invariants)). A file failing either check is rejected with a
load error before any step runs (`plutus check` exit 2).

`schema_version` MUST be exactly the string `"2.0"` (a JSON Schema `const`). No other
value is accepted.

### 1.1 Top-level keys

| Key | Required | Type / Shape | Notes |
|-----|----------|--------------|-------|
| `schema_version` | MUST | string | Exactly `"2.0"` |
| `repo` | MUST | object | `{name, primary_language}` — both required strings |
| `env` | MUST | object | See [§1.2](#12-env-sub-keys) |
| `secrets` | MUST | array | See [§1.3](#13-secrets-entry); may be `[]` |
| `data_sources` | MUST | object | `{processed: [...], raw: [...]}` — both arrays required, may be empty |
| `steps` | MUST | array | At least one step (`minItems: 1`); see [§1.4](#14-step-object) |
| `expected` | MUST | array | Per-step expected outcomes; may be `[]`; see [§1.5](#15-expected-entry) |
| `nine_step_coverage` | optional | object | See [§1.6](#16-nine_step_coverage-entry) |

### 1.2 `env` sub-keys

| Key | Required | Type | Notes |
|-----|----------|------|-------|
| `base` | MUST | string enum | One of `python`, `python-cuda`, `none` |
| `python_version` | MUST | string | e.g. `"3.11"` |
| `requirements_file` | optional | string \| null | Path to `requirements.txt` or `pyproject.toml`; `null` if none |
| `manager` | optional | string enum | `uv` or `pip` (default `pip`). See note below |
| `lockfile` | optional | string \| null | Path to the committed lockfile (e.g. `uv.lock`). Required when `manager: uv` (invariant 7) |
| `install_project` | optional | boolean | Default `false`. Installs the repo's *own* package into the image. uv-only (invariant 8). See note below |
| `os_packages` | optional | array of strings | Extra `apt` packages to install |
| `gpu_required` | optional | boolean | Default `false` |

Why `manager`/`lockfile` exist (reproducible environments, added 0.4.0). `pip` with
a loose `requirements.txt` resolves whatever versions exist at build time, so two builds
weeks apart can install different package versions and produce different numbers — the
opposite of reproducible. `manager: uv` + a committed `lockfile` pins every transitive
dependency to an exact version, so the container is rebuilt identically every time. A
`pip`/lockfile-less env is reported as `env: NOT reproducible` in `plutus check` output
with a deprecation note; the exit code is *not* affected yet (warn-only). A locked env
is also what makes `byte_exact` artifact comparison ([§5.4](#54-artifact-comparison))
trustworthy — without it, byte-level drift is expected and you should prefer
`json_numeric_tolerance` or `visual_similarity`.

Why `install_project` exists (installable-package repos, added 0.5.0). By default the
build installs only your *dependencies*; your own package isn't installed, so a step
command like `pmm-backtest` (a console script) or `python -m your_package.…` fails with
"command not found" / `ModuleNotFoundError`. Set `install_project: true` to also install
the repo's package into the image. It is *uv-only* and requires a `pyproject.toml` at
the repo root plus a committed lockfile (invariant 8). Dependencies stay in a cached
build layer (`uv sync --frozen --no-install-project`); the project is installed last,
after the source is copied, with `uv pip install --no-deps .` — a non-editable install
into `/opt/venv` that survives the runtime bind-mount of `/srv/repo`. Default `false`
means no behavior change for existing repos.

### 1.3 `secrets` entry

Each entry in the `secrets` array:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `key` | MUST | string | Environment-variable name injected into steps |
| `purpose` | optional | string | Human-readable description |
| `used_by` | optional | array of strings | Step IDs that need the secret; see invariant 6 ([§1.7](#17-cross-field-invariants)) |

### 1.4 Step object

Each entry in `steps[]`:

| Field | Required | Default | Type / Notes |
|-------|----------|---------|--------------|
| `id` | MUST | — | Non-empty string; unique across all steps |
| `nine_step` | MUST | — | One of the 7 canonical keys ([§3](#3-the-nine-step-keys)) or `null` for a free-form step |
| `required` | MUST | — | Boolean; gates the exit code ([§5](#5-the-plutus-check-contract)) |
| `command` | conditional | `null` | MUST be non-empty when the step `id` is `data_preparation` (invariant 2); otherwise may be `null` |
| `label` | optional | `null` | Human-readable display name (`type: ["string","null"]`) |
| `network` | optional | `"none"` | Enum: `none`, `bridge`, `host` |
| `timeout_seconds` | optional | `1800` | Integer ≥ 1. The schema enforces only `minimum: 1`; the 30-minute default is applied by the loader when the key is omitted |
| `inputs` | optional | `[]` | Path allowlist staged into the container — see warning below |
| `outputs` | optional | `[]` | Paths copied back out of the container after the step |
| `depends_on` | optional | `[]` | Step IDs; declares topological-sort ordering edges |
| `verification_mode` | optional | `"execute"` | Enum: `execute` (runs `command`, compares results) or `artifact_check` (skips execution, only verifies declared output files exist) |
| `sub_processes` | optional | `null` | Documentation-only breakdown of the data-preparation step into `collection` + `processing` slots. Only valid on the step whose `nine_step` is `step_2_data_preparation` (invariant 9). Never executed. See [§3](#3-the-nine-step-keys) |

> `inputs` is a complete-coverage allowlist (gotcha G12). When non-empty, `inputs`
> is *not* a list of "data inputs" — it is the complete set of paths staged into the
> container for that step. Any file not matched — including the step's own script —
> will be absent inside the container, causing `python <script>` to exit 2 with "No such
> file." The recommended default is `inputs: []`, which falls back to the full
> repository tree filtered only by `.dockerignore`. Tighten step-by-step only after
> confirming the manifest works at the looser setting.

### 1.5 `expected` entry

Each entry in `expected[]` describes the asserted outcome of one step. A step with no
`expected` entry is run but has no numeric or artifact assertions (only its exit code is
checked).

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `step_id` | MUST | string | MUST match an existing step `id` (invariant 4) |
| `metrics` | optional | array | Metric assertions — see below |
| `artifacts` | optional | array | Artifact assertions — see below |

`expected.metrics[]` object:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | MUST | string | Matches `^[a-z][a-z0-9_]*$`; must match a metric the step emits |
| `value` | MUST | number | The claimed value |
| `tolerance` | MUST | object | `{kind, value}` — see below |
| `display_name` | optional | string | Shown in reports |

`tolerance.kind` is one of `relative`, `absolute`, `exact`; `tolerance.value` is a
non-negative number. Semantics in [§5.3](#53-metric-comparison). For `exact`, the
`value` field is ignored by the verifier (set it to `0`).

`expected.artifacts[]` object:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `path` | MUST | string | Relative path of the output file |
| `compare` | MUST | string enum | One of `json_numeric_tolerance`, `visual_similarity`, `byte_exact` |
| `threshold` | optional | number \| null | Interpretation depends on `compare`; `null`/omitted uses the mode default (visual similarity default `0.7`). Semantics in [§5.4](#54-artifact-comparison) |

### 1.6 `nine_step_coverage` entry

Optional top-level object keyed by the canonical nine-step keys ([§3](#3-the-nine-step-keys)).
Validated for shape but does not gate exit codes; it feeds scoring ([§6](#6-scoring-rubric)).
Each entry:

| Key | Required | Type | Notes |
|-----|----------|------|-------|
| `present` | MUST | boolean | Whether this step is present in the repository |
| `section` | optional | string \| null | README section that documents the step |

Any subset of the seven keys may appear; omitted keys imply no coverage claim.

### 1.7 Cross-field invariants

Structural validation runs first; then these invariants MUST all hold. A violation
fails `plutus check` before any step executes.

1. *Unique step IDs* — no two steps share an `id`.
2. The `data_preparation` step **MUST** declare a non-empty `command`. Matched against
   the step's `id` (not its `nine_step`): any step whose `id` is literally
   `"data_preparation"` carries this obligation, even when a `data_sources` entry already
   provides a download URL (the command is the fallback when the download path is
   unavailable). *(Before 0.3.0 this rule keyed off the two ids `data_collection` /
   `data_processing`; the v2025 taxonomy folded those into one — see [§3](#3-the-nine-step-keys).)*
3. `depends_on` references — every ID in any `depends_on` array refers to an
   existing step ID.
4. `expected.step_id` references — every `step_id` in `expected` refers to an
   existing step ID.
5. `data_sources.*.satisfies` references — every step ID in any source's `satisfies`
   array refers to an existing step ID (`satisfies` has `minItems: 1`).
6. `secrets.*.used_by` references — every step ID in a secret's `used_by` refers to
   an existing step ID, except entries beginning with `"data_sources."` (a qualifier
   form for data-source-level usage), which are exempt.
7. `manager: uv` requires `lockfile`. A uv-managed env with no committed lockfile is
   rejected — uv can only build reproducibly from a lockfile.
8. `install_project: true` requires `manager: uv` (and, transitively, a `lockfile`)
   plus a `pyproject.toml` at the repo root. The validator errors clearly otherwise.
9. `sub_processes` is only valid on the data-preparation step — declaring it on any
   step whose `nine_step` is not `step_2_data_preparation` is rejected.

### 1.8 Data-source object (`_DATA_SOURCE`)

Each entry in `data_sources.processed[]` or `data_sources.raw[]`
(`additionalProperties: false`):

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `kind` | MUST | string | Free-form; known values: `local`, `google_drive`, `github_release`, `http`, `s3`, `manual`. Typos pass schema validation and surface only as a runtime download failure |
| `url` | MUST | string | Full URL for remote kinds; `.` (repo root) for `kind: local` |
| `expected_layout` | MUST | array of strings | Relative paths or glob patterns the source must provide |
| `satisfies` | MUST | array of strings (`minItems: 1`) | Step IDs whose data requirement this source satisfies |
| `secrets_required` | optional | array of strings | Secret keys for authenticated access; should name keys in the top-level `secrets[]` |
| `label` | optional | string \| null | Human-readable label |

---

## 2. Results schema

`.plutus/run/<step_id>/results.json`, written by each executed step
(`verification_mode: execute`) on a clean exit. Valid JSON, validating against the
Plutus results JSON Schema (Draft 2020-12). `schema_version` MUST be exactly `"1.0"`.

`additionalProperties: false` applies at the root, metric, and artifact levels.
`metadata` is the sole open object.

### 2.1 Top-level shape

| Key | Required | Type | Notes |
|-----|----------|------|-------|
| `schema_version` | MUST | string | Exactly `"1.0"` |
| `step_id` | MUST | string | Equals the manifest step's `id`; matches `^[a-z][a-z0-9_]*$` |
| `metrics` | MUST | array | Metric objects (may be `[]`) |
| `artifacts` | MUST | array | Artifact objects (may be `[]`) |
| `metadata` | MUST | object | Free key/value map (the only level without `additionalProperties: false`); auto-populated fields below |

### 2.2 Metric object

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | MUST | string | Matches `^[a-z][a-z0-9_]*$`; unique within the step |
| `value` | MUST | number | Finite `int` or `float`. `Decimal` and `bool` are rejected (gotcha G6) |
| `unit` | MUST | string enum | One of `fraction`, `ratio`, `count`, `currency_usd`, `seconds`. `percent` is deliberately absent — store `0.42` as `fraction`, never `42` |

Unit guidance: use `fraction` for bounded shares written as percentages (win rate, max
drawdown, annual return); `ratio` for unbounded dimensionless numbers (Sharpe, Sortino).

### 2.3 Artifact object

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | MUST | string | Matches `^[a-z][a-z0-9_]*$`; unique within the step |
| `path` | MUST | string | Relative to the step's working directory |
| `kind` | MUST | string enum | One of `chart`, `csv`, `json`, `image`, `other` |

### 2.4 Write semantics

- *Atomic write.* The SDK serializes to `results.json.tmp` then `os.replace`; readers
  never observe a partial file. Any equivalent atomic-rename approach complies.
- *No file on exception.* A step that raises inside the `with` block produces no
  `results.json`; the verifier treats the absent file as a failed step.
- *Auto-injected metadata* (set only if absent — user-supplied values win):

  | Injected key | Value |
  |--------------|-------|
  | `duration_seconds` | Wall-clock seconds inside the `with` block (rounded to 3 dp) |
  | `git_commit` | Short 7-char HEAD SHA (best-effort; omitted if no `.git`) |

---

## 3. The nine-step keys

The canonical `nine_step` values. The strings are defined in
`plutus_verify/constants.py`; copy them exactly (`step_4_in_sample`, not
`step_4_insample`).

| `nine_step` key | Stage |
|-----------------|-------|
| `step_1_hypothesis` | Hypothesis formulation |
| `step_2_data_preparation` | Data preparation (collection and processing) |
| `step_3_forming_set_of_rules` | Forming the set of rules |
| `step_4_in_sample` | In-sample backtesting |
| `step_5_optimization` | Optimization |
| `step_6_out_of_sample` | Out-of-sample backtesting |
| `step_7_paper_trading` | Paper trading |

> Taxonomy changed at 0.3.0 (v2025 nine-step process), breaking. The 2025 revision of
> the 9-step process collapsed *data collection* and *data processing* into one canonical
> step, *data preparation*, and renamed the third step to *forming the set of rules*.
> Concretely: `step_2_data_collection` → `step_2_data_preparation`, and
> `step_3_data_processing` → `step_3_forming_set_of_rules`. This is a hard cutover — a
> manifest using the old keys fails to load with `ManifestLoadError` (exit 2). The
> manifest `schema_version` stayed `"2.0"`; only the `nine_step` enum values changed.

Documenting data preparation in detail. Because one step now covers both collecting
and processing data, the data-preparation step accepts an optional, documentation-only
`sub_processes` block with two slots, `collection` and `processing` (each: a required
`description`, plus optional `command`/`inputs`/`outputs`). It is never executed — it
only records *what* the data preparation does for a reader — and is valid only on the
step whose `nine_step` is `step_2_data_preparation` (invariant 9).

*Fixed frame, free content.* The standard fixes the seven key strings, the manifest
field names, and the enums — not the number of steps or their content. A repo may
declare additional steps with `nine_step: null` (provide a `label`). Steps 8–9 of the
9-step process (live trading, continuous improvement) are out of scope for `plutus
check`. Reproducibility is expected through Step 6 (minimum) and Step 7 (ideal).

Paper trading, which usually can't be re-executed automatically, has two supported
patterns (which MAY be combined):

1. `verification_mode: artifact_check` on a declared step — skips command execution;
   only checks declared output files exist.
2. `nine_step_coverage` ([§1.6](#16-nine_step_coverage-entry)) — documents coverage
   without requiring an executable step.

---

## 4. Data resolution semantics

`data_sources` has two required arrays, `processed` and `raw`. For every step listed in
a source's `satisfies` array, the native runtime resolver tries, in order:

1. `processed` first. If a `processed` entry satisfies the step, the resolver checks
   whether its `expected_layout` files are already present. If so, it uses them and
   skips any download/command for that step; if not, it attempts to download them.
2. `raw` next. If no `processed` source satisfies the step, the resolver checks
   `raw`. A `raw` source requires downloading/processing before use.
3. `command` fallback. If neither array provides a usable source — or the download is
   unreachable — the resolver runs the step's declared `command`. This is why the
   `data_preparation` step MUST carry a non-empty `command` (invariant 2): the command is
   the unconditional fallback.

> Where downloads land (0.5.0). Fetched data sources are cached in the gitignored
> `.plutus/cache/`, not your working tree, and overlaid into each step's sandbox. Caching
> persists across runs; the working tree stays clean. Presence checks consider both
> committed data and the cache.

Both arrays are always required; an empty array is written `[]`, not omitted.

*Tiers* are an informal vocabulary, not named in the schema. They are combinations of
the above:

| Tier | Shape |
|------|-------|
| 1 — Committed | `processed` has one `kind: local`, `url: .` entry; `raw: []`. `data_preparation` step typically omitted |
| 2 — Remote | `processed: []`; `raw` carries a remote source (`google_drive`/`github_release`/`http`/`s3`/`manual`); a `data_preparation` step with a fallback `command` is declared |
| 3 — Database | both arrays `[]`; a `data_preparation` step queries a DB at runtime with `network: bridge` and credentials via `secrets[]` |
| 4 — Layered | a remote `raw` entry (primary) plus DB secrets (fallback) simultaneously. Highest authoring risk; secrets must cover both paths and the remote `expected_layout` must match the DB query's output paths |

`--data-tier` ([§7](#7-cli-reference)) can force a specific tier; default `auto` follows
the fall-through above.

---

## 5. The `plutus check` contract

The exit code of `plutus check` is the machine-readable definition of "reproduced."
Every other verdict (scoring, badge) derives from it.

`plutus check` builds the `env` ([§1.2](#12-env-sub-keys)) into a Docker image from a
generated `Dockerfile`, runs each step in dependency order inside the container, and
compares produced `results.json` files against the manifest's `expected` blocks. A
working Docker daemon is required.

### 5.0 Bless vs. verify, and the read-only guarantee (0.5.0)

`plutus` separates *blessing* a baseline (`snapshot`) from *verifying* against it
(`check`). Both run the identical in-container pipeline; they differ only in the final
move. The point of the split: a baseline should change only when you deliberately decide
it should, never as an accidental side effect of running the check. So `check` is
*read-only* — it never writes `.plutus/expected/` or any committed working-tree file.
A forgotten `snapshot` is then caught (missing groundtruth → fail), and output drift from
a code change you never re-blessed fails on purpose instead of silently overwriting the
baseline.

Three stores keep the roles separate:

| Store | Role | Written by | Commit it? |
|-------|------|-----------|------------|
| `.plutus/expected/<step>/…` + `manifest.yaml` metric `value`s | frozen groundtruth | `snapshot` only | yes |
| `result/…` (your declared output paths) | human-facing copy for the README | `snapshot` only | yes |
| `.plutus/results/<step>/…` | per-run scratch + inter-step data bus | `snapshot` and `check` | no — gitignore |
| `.plutus/cache/…` | fetched data sources | `snapshot` and `check` | no — gitignore |
| `.plutus/run/<step>/…` | per-step bookkeeping + captured `stdout`/`stderr` | `snapshot` and `check` | no — gitignore |

Because produced bytes land in the gitignored `.plutus/results/<step>/` (and are compared
*there*), `check` leaves the working tree clean — safe to re-run repeatedly. Earlier
steps reach later ones through that same buffer (the inter-step data bus), so a multi-step
pipeline reproduces end-to-end without writing the working tree. A failing step's
`stdout`/`stderr` persist to `.plutus/run/<step>/` and a stderr tail is printed in the
report — no need to re-run the container by hand to recover a traceback.

Because `snapshot` writes the groundtruth from the *container's* output (not the laptop's),
`byte_exact` baselines now work for build-sensitive artifacts (charts, `*.parquet`,
`model.pkl`) that a laptop baseline could never match — provided the env is locked
([§1.2](#12-env-sub-keys)).

### 5.1 Exit codes

| Exit | Meaning | Condition |
|------|---------|-----------|
| 0 | All required steps ran cleanly AND all comparisons passed | Every required step exited 0 (no preflight error, not a hard execution failure), AND every metric comparison `ok`, AND every artifact comparison `ok` or `skipped` |
| 1 | Required steps ran, but ≥ 1 comparison failed (soft fail) | No required step hard-failed, but ≥ 1 metric `ok=False`, or ≥ 1 artifact `ok=False` and `skipped=False`. Considers ALL declared comparisons, including those on optional (`required: false`) steps |
| 2 | A required step failed execution, or a preflight/manifest-load error | ≥ 1 required step exited non-zero (no `skipped_reason`) OR has a `preflight_error`, OR manifest loading failed, OR the Docker image build failed |

- Only `required: true` gates exit 2. Only a required step whose execution hard-fails
  or whose preflight check fails can push the exit code to 2.
- Exit 1 considers all declared comparisons — the exit-code function iterates
  `metric_results` and `artifact_results` without filtering by `required`, so a failed
  comparison on an *optional* step still causes exit 1.
- Skipped steps are non-blocking for exit 2. `skipped_reason` is a runtime field (set
  when a step's work is satisfied without executing its command, e.g.
  `"satisfied_by_data_source"`). A step with a `skipped_reason` is not a hard failure
  even if `required`. But skipping does *not* suppress comparisons: a skipped step
  writes no `results.json`, so any declared `expected.metrics` for it fail (contributing
  to exit 1). Do not declare `expected.metrics` for steps that may be skipped.

### 5.2 Status vocabulary

| Status | ok | skipped | Meaning |
|--------|----|---------|---------|
| `ok` | True | False | Verified pass |
| `SKIP` | True | True | Not verified; no evidence of issue (non-blocking) |
| `WARN` | False | True | Divergence detected but inconclusive (non-blocking) |
| `FAIL` | False | False | Verified divergence (blocks exit 0) |

Exit 0 is exactly "no `FAIL` lines."

### 5.3 Metric comparison

Metrics are read from `results.json` and matched *by name* against
`expected[].metrics[]`. The comparison is deterministic (no LLM). Three tolerance kinds,
implemented in `_within_tolerance()` (`plutus_verify/spec/runtime/orchestrator.py`):

| Kind | Formula | Notes |
|------|---------|-------|
| `exact` | `actual == expected` | Plain numeric equality, *no float fuzz*. Use only for integer metrics — exact equality on floats is brittle across environments |
| `absolute` | `|actual − expected| ≤ value` | Symmetric; units must match |
| `relative` | `|actual − expected| / |expected| ≤ value` | When `expected == 0`, degrades to `|actual| ≤ value` (avoids divide-by-zero) |

Both `expected.metrics[].value` and the `results.json` metric values are typed `number`,
so every comparison in a conforming repo is numeric. A comparison that can't locate the
actual value (missing file) is `ok=False` and contributes to exit 1.

### 5.4 Artifact comparison

Artifacts are matched *by path* against `expected[].artifacts[]`. Three modes,
implemented in `plutus_verify/spec/runtime/artifact_compare.py`:

| Mode | Behaviour | Missing baseline | Missing produced file |
|------|-----------|------------------|-----------------------|
| `byte_exact` | File bytes must be identical | `FAIL` | `FAIL` |
| `json_numeric_tolerance` | Deep-walk JSON; numeric values within relative tolerance (default 5%); non-numeric compared as parsed values. When `expected == 0`, degrades to absolute `|produced| ≤ tol` | `FAIL` | `FAIL` |
| `visual_similarity` | LLM-vision comparison at `threshold` (default `0.7`) | SKIP (no reference yet; run `plutus snapshot`) | `FAIL` |

`visual_similarity` without `--visual-check` (the default — no vision client wired):
falls back to a byte comparison. Bytes identical → `ok`; bytes differ → *WARN*
(`ok=False, skipped=True`), which is non-blocking and does not contribute to exit 1.

### 5.5 Report shape

A green run (exit 0) prints, per required step, no `FAIL` lines:

```
ok <step_id>: exit=0
    ok <metric_name>: actual=<v> expected=<v>
    ok <compare_mode> <artifact_path>     # or SKIP / WARN — both non-blocking
```

Any `FAIL` line — metric, artifact, or step execution — means exit 1 or 2.

---

## 6. Scoring rubric

The compliance score is a single percentage from four weighted buckets, produced by the
read-only `plutus-scoring` skill. It inspects the manifest, the scripts, the README, and
any `plutus check` output already present, then applies the rubric *by LLM judgment*. It
does *not* run `plutus check` itself.

> The score is reference-only. Only the *Reproducible* bucket (50) is anchored to an
> objective fact — `plutus check` exits 0, verified mechanically. The other three buckets
> are advisory LLM assessments: they vary with the scoring model and the day, and the
> rubric itself is subject to change. Canonical disclaimer (emitted with every score, and
> rendered under the README badge by `plutus-document`):
>
> *Plutus score is an LLM-assessed reference signal, not a certified quality grade, and is
> subject to change. The verified guarantee is reproducibility — `plutus check` exits 0.*

| Bucket | Weight | Anchor | Measures |
|--------|-------:|--------|----------|
| Reproducible | 50 | objective (`plutus check` exit 0) | `plutus check` exits 0; README-claimed metrics match script output within tolerance |
| Tidy / well-documented | 25 | LLM judgment | README structure, install hygiene, no `<placeholder>` parse traps, docs match reality |
| Standardized / template | 10 | LLM judgment | Canonical Plutus shape; could serve as a template |
| Innovative | 15 | LLM judgment | New metrics, novel diagnostics, non-textbook strategy logic |

*Reproducible (50):*

- 50 — `plutus check` exits 0 cleanly; no workarounds; all required metrics in tolerance.
- 45 — exits 0 only after manifest-side workarounds (network/secret routing, snapshot seeding).
- 35 — exits 0 only after touching `requirements.txt` or other tracked config.
- 20 — metrics match individually on host but `plutus check` can't be made green in-session.
- 0 — scripts don't produce the claimed metrics within tolerance.

*Tidy (25)* — five checklist items, ~5 pts each (no named tiers):

- README structure clean, metric tables present and consistent
- `.env.example` parses under `source` (no unquoted `<placeholder>` lines)
- all data inputs documented (no surprise dependencies)
- optimization / parameter pipeline accurately described
- has `.python-version` (or equivalent pin)

*Standardized (10):*

- 10 — canonical step shape, parameters externalized, predictable output paths, no module-level side effects.
- 5 — most of the above with one significant deviation.
- 0 — could not serve as a template without significant rework.

*Innovative (15):*

- 15 — novel metrics/diagnostics or non-textbook strategy logic.
- 8 — thoughtful strategy logic but conventional analytical surface.
- 0 — textbook backtest, no original instrumentation.

### 6.1 Computing the score

Percentage = sum of the four bucket scores, rounded to the nearest 5%. (Bucket
scores need not be multiples of 5; only the final sum is rounded.) Example: 45 + 20 + 10
+ 8 = 83 → 85%.

*No hard gate.* A repo that can't make `plutus check` exit 0 isn't disqualified, but
Reproducible is 50% of the total, so a non-reproducing repo caps at 25 + 10 + 15 = 50
pts, and only if exemplary everywhere else.

### 6.2 Badges

The score renders as a single `PLUTUS-<score>%` badge — the *only data-driven badge*,
derived from the score above (so it inherits the reference-only caveat in
[§6](#6-scoring-rubric); render the disclaimer near it). `plutus-document` emits it. There
are *no tiers or ranks* — the badge shows the percentage, nothing more:

```
![Static Badge](https://img.shields.io/badge/PLUTUS-<score>%25-<color>)
```

Replace `<score>` with the rounded number; the literal `%25` is the URL-encoded `%`, so
`85%25` renders as `85%`. `plutus-document` picks `<color>` from the score purely as a
visual cue — it carries no grade or rank.

*Special-designation badges* (config, not computed — awarded editorially, independent of
score):

| Designation | Shields.io markup |
|-------------|-------------------|
| Sample Project | `![Static Badge](https://img.shields.io/badge/PLUTUS-Sample-darkblue)` |
| PROTO series | `![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88)` |

---

## 7. CLI reference

`plutus-verify` exposes a Click command group. It is a uv-managed project and is not on PyPI; install the CLI with uv from a release wheel
(`uv tool install ./plutus_verify-<version>-py3-none-any.whl`) or from an editable
checkout (`pip install -e ".[runner]"`). Every subcommand takes an optional `REPO_PATH`
positional argument (default `.`).

#### `plutus init [REPO_PATH] [--force]`

Scaffolds three files: `.plutus/manifest.yaml` (skeleton with every required field and
`TODO` placeholders), `.plutus/example_script.py` (annotated SDK instrumentation example),
and a `.dockerignore` (so `check` doesn't surprise-write one into the repo root at build
time — 0.5.0). `--force` overwrites existing files.

#### `plutus check [REPO_PATH] [--secrets-from-env] [--data-tier {processed|raw|code|auto}] [--visual-check]`

Runs the v2 pipeline locally (the verification command — [§5](#5-the-plutus-check-contract)).

| Option | Default | Effect |
|--------|---------|--------|
| `--secrets-from-env` | off | Resolves *only* the manifest's declared `secrets[].key`s from the caller's environment and injects each into the steps that name it in `used_by`. It does *not* forward the whole `os.environ` (fixed in 0.4.2 — the old behavior leaked the host `PATH`/`HOME`/editor vars into the "reproducible" container and broke uv envs). A reserved-key denylist (`PATH`, `HOME`, `LD_LIBRARY_PATH`, `PYTHONPATH`, `VIRTUAL_ENV`, `UV_PROJECT_ENVIRONMENT`) is rejected even if declared. With `secrets: []`, nothing is injected |
| `--data-tier` | `auto` | Forces a data tier; `auto` follows the processed → raw → command fall-through ([§4](#4-data-resolution-semantics)) |
| `--visual-check` | off | Enables `visual_similarity` reference comparisons. Requires `PLUTUS_VISION_ENDPOINT` and `PLUTUS_VISION_MODEL` env vars (optionally `PLUTUS_VISION_API_KEY`); errors with exit 2 if they're unset. When off, `visual_similarity` entries fall back to byte comparison ([§5.4](#54-artifact-comparison)) |

Exit code is the verdict ([§5.1](#51-exit-codes)). Manifest-load and Docker-build
failures exit 2.

#### `plutus snapshot [REPO_PATH] [--no-run] [--no-artifacts] [--no-metrics]`

The *bless* command ([§5.0](#50-bless-vs-verify-and-the-read-only-guarantee-050)). By
default (0.5.0+) it builds the image, runs every step *in-container*, then writes the
groundtruth — artifact files → `.plutus/expected/<step_id>/`, metric numbers →
`expected.metrics[].value` in the manifest — plus a human-facing copy to your declared
output paths (`result/…`). Snapshotting from the container's own output is what lets
`byte_exact` baselines match what `check` later reproduces.

| Option | Effect |
|--------|--------|
| `--no-run` | Opt out of the in-container run; bless the outputs already present locally instead. Use it when a step needs a secret the in-container `snapshot` path doesn't yet inject — run the step locally with the secret first, then `snapshot --no-run` |
| `--no-artifacts` | Don't copy output files into `.plutus/expected/` |
| `--no-metrics` | Don't write metric values into the manifest |

> Secrets during `snapshot`. The in-container `snapshot` path currently injects no
> secrets. If a step needs one to produce its baseline, either run it locally with the
> secret and `snapshot --no-run`, or `plutus check . --secrets-from-env` first and
> snapshot from there. First-class secret flags on `snapshot` are a planned follow-up.

#### `plutus bootstrap [REPO_PATH] [--force]`

Run *after* instrumenting scripts and producing `.plutus/run/<step_id>/results.json`.
Reads `.plutus/run/`, auto-fills ~70% of `manifest.yaml.draft` from the filesystem and
results, and leaves `TODO_*` sentinels (explained in a companion `manifest_TODO.md`) on
fields needing domain knowledge. `--force` overwrites an existing draft.

#### `plutus transfer [REPO_PATH] [--llm-endpoint URL] [--llm-model MODEL] [--config PATH] [--no-prewarm] [--force]`

Converts a legacy README-based repo into a v2 draft manifest via an LLM (four calls;
requires an OpenAI-compatible endpoint). Produces `.plutus/manifest.yaml.draft` and
`.plutus/instrument_TODO.md` (a checklist of scripts still needing SDK instrumentation).

| Option | Default | Effect |
|--------|---------|--------|
| `--llm-endpoint` | config or `http://localhost:11434/v1` | OpenAI-compatible endpoint |
| `--llm-model` | config default | Model name |
| `--config PATH` | none | YAML config overriding LLM defaults (timeouts, `num_ctx`, etc.) |
| `--no-prewarm` | off | Skip the LLM prewarm step |
| `--force` | off | Overwrite an existing `instrument_TODO.md` |

> Legacy `verify` subcommand. `plutus verify` is the original pre-v2,
> LLM-orchestrated single-command path (stages `extract|build|execute|compare|report`,
> `--data-loader {google_drive|db_loader|auto}`, `--dry-run`, `--batch`, etc.). It is
> *not* the v2 contract path and is out of scope for this standard; use `plutus check`
> for v2 verification.

---

## 8. SDK API reference

The Python SDK is the recommended `results.json` producer — it validates at call time
and writes atomically. Any tool or language that writes a conforming file ([§2](#2-results-schema))
is equally valid. Public exports: `plutus_verify.step`, `plutus_verify.Run`,
`plutus_verify.__version__`.

### `pv.step(step_id, *, repo_path=None) -> Run`

Module-level factory returning a `Run` context manager. `step_id` MUST match a manifest
step `id` and the `^[a-z][a-z0-9_]*$` pattern. `repo_path` defaults to the current
working directory (where `.plutus/` is found). Use as a context manager:

```python
import plutus_verify as pv

with pv.step("in_sample") as r:
    r.metric("sharpe_ratio", float(sharpe), unit="ratio")
    r.metric("win_rate",     float(win),    unit="fraction")   # 0.42 == 42%
    r.artifact("equity_curve", "out/equity.png", kind="chart")
    r.metadata(seed=42)
```

On clean `__exit__`, the SDK re-validates the assembled payload against the JSON Schema
and writes `.plutus/run/<step_id>/results.json` atomically. On exception, no file is
written.

#### `Run.metric(name, value, *, unit="ratio") -> None`

Records one metric and validates immediately (fail-fast at the offending line).

- `name` — `^[a-z][a-z0-9_]*$`, unique within the step.
- `value` — `int` or `float`. `Decimal` and `bool` raise `ValueError` (gotcha G6) —
  always wrap library values in `float(...)`.
- `unit` — keyword-only; one of `fraction`, `ratio`, `count`, `currency_usd`, `seconds`
  (default `"ratio"`). No `percent`.

### `Run.artifact(name, path, *, kind="chart") -> None`

Records one artifact.

- `name` — `^[a-z][a-z0-9_]*$`, unique within the step.
- `path` — `str` or `Path`, relative to the step's working directory.
- `kind` — keyword-only; one of `chart`, `csv`, `json`, `image`, `other` (default
  `"chart"`).

#### `Run.metadata(**kwargs) -> None`

Merges arbitrary key/value pairs into the results `metadata` object (the one schema level
without `additionalProperties: false`). User-supplied `duration_seconds` / `git_commit`
override the auto-injected values ([§2.4](#24-write-semantics)).

---

## 9. Troubleshooting catalogue

Known failure patterns. Detailed Symptom → Diagnosis → Fix entries live in
`skills/plutus-standardize/references/known-gotchas.md` in the `plutus-automation-scoring`
repository (the skill was named `plutus-transform` before 0.5.0).

| ID | Title | Applies to |
|----|-------|-----------|
| G1 | Module-level DB connection forces network/secret routing | All versions — current |
| G2 | Internally-conflicting dependency file | All versions — current |
| G3 | `.env` placeholders crash shell sourcing | All versions — current |
| G4 | `visual_similarity` artifacts silently failed | Historical — fixed in 0.2.6 |
| G5 | Container stderr/stdout swallowed | Historical — resolved in 0.5.0 (stdout/stderr persisted, stderr tail printed) |
| G6 | SDK rejects `Decimal` metric values | All versions — current |
| G7 | Chart baseline captured after smoke-run is tautological | All versions — current |
| G11 | Runtime volume mount bypasses `.dockerignore` | Historical — v0.2.9 only; fixed in 0.2.10 |
| G12 | All execution steps FAIL exit=2 after declaring `step.inputs` | v0.2.10+ — current |

On `plutus-verify` 0.5.1, G4, G5, and G11 describe fixed behavior and can be ignored — a
failing step now persists `stdout`/`stderr` to `.plutus/run/<step>/` and the report prints
a stderr tail. The active pitfalls to watch are *G12* (the `inputs` allowlist —
[§1.4](#14-step-object)), *G6* (`Decimal` — wrap in `float()`), *G1* (`network: bridge`
for DB-at-import), *G3* (quote `.env` placeholders), and *G7* (capture chart baselines
from a real run, not a smoke run).

---

## 10. Versioning & schema correspondence

| Standard version | `plutus-verify` | Manifest `schema_version` | Results `schema_version` |
|------------------|-----------------|---------------------------|--------------------------|
| V2 (this document) | ≥ 0.5.1 | `"2.0"` | `"1.0"` |
| V1 | n/a (prose standard) | n/a | n/a |

This document tracks `plutus-verify` 0.5.1. The manifest and results `schema_version`
strings have held at `"2.0"` / `"1.0"` since 0.2.10, but two changes inside that envelope
matter for authoring:

- *0.3.0 (breaking, taxonomy):* the `nine_step` enum adopted the v2025 keys
  (`step_2_data_preparation`, `step_3_forming_set_of_rules`). A manifest with the old
  `step_2_data_collection` / `step_3_data_processing` keys no longer loads. Re-author the
  keys; no schema-version bump was issued.
- *0.5.0 (non-breaking, data flow):* `check` became read-only and `snapshot` runs
  in-container ([§5.0](#50-bless-vs-verify-and-the-read-only-guarantee-050)); `env` gained
  the optional `manager` / `lockfile` / `install_project` keys ([§1.2](#12-env-sub-keys)).
  Existing manifests keep working unchanged.

A future breaking change will increment the manifest or results schema version. V1 is
archived verbatim at [archive/v1/README.md](archive/v1/README.md); V1-badged repos
remain valid as v1-legacy until re-verified under V2. See [CHANGELOG.md](CHANGELOG.md) for
the full history.