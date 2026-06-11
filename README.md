# The PLUTUS Standard — Version 2

> **Version:** 2.0 (draft) | **Previous version:** [versions/v1/README.md](versions/v1/README.md)

---

# 0  Introduction — what "compliant" means in v2

The PLUTUS Standard ensures the *reproducibility* of algorithmic-trading research
projects up to Step 7 (Paper Trading) of the
[9-step process](https://hub.algotrade.vn/knowledge-hub/steps-to-develop-a-trading-algorithm/).
By reproducibility, V1 meant "a human can follow the README and reproduce the
results." V2 sharpens that definition: a project is compliant when
**`plutus check` exits 0 against the project's declared contract** — no human
judgment on the verification path.

The README is still required. Its role in V2 is to document *why* the research
was conducted, explain the strategy and methodology, and give human readers the
context they need. But the README is no longer the authoritative source of truth
for *what* the pipeline does or what results it produces. That authority belongs
to the **manifest** (`.plutus/manifest.yaml`).

A compliant V2 project provides two machine-readable contracts: First, the author
declares a `.plutus/manifest.yaml` (`schema_version: "2.0"`) that names every
pipeline step, specifies how data is acquired, and states the `expected` metrics
and artifacts each step must produce (see §1.1). Second, each step's script emits
a strict `.plutus/run/<step_id>/results.json` (`schema_version: "1.0"`) at runtime
(see §1.2). The `plutus-verify` framework (invoked as `plutus check`) reads the
manifest, builds the declared environment in a container, runs the steps in
dependency order, and compares the produced `results.json` files against the
manifest's `expected` blocks. If every comparison passes within the declared
tolerances, `plutus check` exits 0 — a passing verification (see §4).

A compliant project MUST be self-contained and runnable without contacting the
author. All data sources needed to reproduce the results MUST be either committed
to the repository or declared in the manifest's `data_sources` block with
retrievable URLs or acquisition instructions, so that any reviewer can run
`plutus check` in isolation and obtain a reproducible outcome.

Project repositories MUST be hosted under an ALGOTRADE-managed GitHub
organization. Recognized organizations include `algotrade-research`,
`algotrade-course`, and `algotrade-education`. This list may grow. Hosting
outside these organizations does not satisfy the standard.

Compliance is graded by a scoring rubric and
expressed as a badge. The rubric and badge thresholds are defined in §5.

---

# 1  The two machine contracts

The two contracts described in this section are the normative core of V2. They
are specifications that the `plutus-verify` tooling implements — the contracts are the
requirement; `plutus-verify` is a convenience. A repo that satisfies both
contracts independently passes the standard regardless of which version of the
tooling validates it.

## 1.1  The manifest  (`.plutus/manifest.yaml`, `schema_version: "2.0"`)

### Normative requirement

A compliant V2 repo MUST contain the file `.plutus/manifest.yaml` committed to
git. The file MUST be valid YAML whose root is a mapping. It MUST validate
against the PLUTUS v2 JSON Schema (Draft 2020-12) and MUST pass all cross-field
invariants described below. Any file that fails either check is not a valid
manifest; `plutus check` will reject it with a load error before running any
step. The authoritative schema definition lives in the `plutus-verify` package
(module `plutus_verify/spec/schema.py`); any tool that validates against it
complies.

`schema_version` MUST be exactly the string `"2.0"` (a JSON Schema `const`).
There is no other accepted value.

> **Table convention:** In the tables below, **MUST** = required; *optional* = not required, but validated if present; *conditional* = required only under the stated condition.

> **Strict validation:** `additionalProperties: false` is set at every level of
> the schema. Unknown or misspelled keys are validation errors, not silently
> ignored. This applies to the manifest root, `repo`, `env`, every step, every
> data-source entry, and every expected block.

---

### Top-level keys

| Key | Required | Type / Shape | Notes |
|-----|----------|-------------|-------|
| `schema_version` | MUST | string | Exactly `"2.0"` — a schema `const` |
| `repo` | MUST | object | `{name, primary_language}` — both required |
| `env` | MUST | object | Runtime environment declaration — see sub-keys below |
| `secrets` | MUST | array | Declared secrets; may be `[]` |
| `data_sources` | MUST | object | `{processed: [...], raw: [...]}` — both arrays required (may be empty) |
| `steps` | MUST | array | At least one step (`minItems: 1`) |
| `expected` | MUST | array | Per-step expected outcomes; may be `[]` |
| `nine_step_coverage` | optional | object | Coverage map keyed by the nine-step keys (see §2) |

**`env` sub-keys:**

| Key | Required | Type / Shape | Notes |
|-----|----------|-------------|-------|
| `base` | MUST | string enum | One of `python`, `python-cuda`, `none` |
| `python_version` | MUST | string | e.g. `"3.11"` |
| `requirements_file` | optional | string \| null | Path to `requirements.txt` or `pyproject.toml`; `null` if none |
| `os_packages` | optional | array of strings | Additional `apt` packages to install |
| `gpu_required` | optional | boolean | Defaults to `false` |

---

### Step shape

Each entry in `steps[]` MUST have the following required fields, plus a set of
optional fields that gate runtime behavior:

| Field | Required | Default | Notes |
|-------|----------|---------|-------|
| `id` | MUST | — | Non-empty string; MUST be unique across all steps |
| `nine_step` | MUST | — | One of the 7 canonical keys or `null` for a free-form step — canonical keys enumerated in §2 |
| `required` | MUST | — | Boolean; gates the exit code — see §4 |
| `command` | conditional | `null` | MUST be non-empty when the step `id` is `data_collection` or `data_processing` (see invariants); otherwise may be `null`. |
| `label` | optional | `null` | Human-readable display name |
| `network` | optional | `"none"` | One of `none`, `bridge`, `host` |
| `timeout_seconds` | optional | `1800` | Integer ≥ 1; default `1800` s (30 min), applied by the loader |
| `inputs` | optional | `[]` | Allowlist of paths staged into the container — see the warning below |
| `outputs` | optional | `[]` | Paths copied back out of the container after the step |
| `depends_on` | optional | `[]` | Step IDs; declares topological-sort ordering edges |
| `verification_mode` | optional | `"execute"` | `execute` runs the command; `artifact_check` skips execution and only verifies declared output files exist |

> Note: the JSON Schema enforces only `minimum: 1`; the 30-minute default is applied by the loader when the key is omitted.

> **WARNING — `inputs` is a complete-coverage allowlist (gotcha G12 in the troubleshooting catalogue — see §7):**
> When `inputs` is non-empty it is not merely a list of "data inputs."
> It is the *complete* set of paths staged into the container for that step.
> Any file not matched — including the step's own script source — will be absent
> inside the container, causing `python <script>` to exit 2 with "No such file."
> **The recommended default is `inputs: []`**, which falls back to the full
> repository tree filtered only by `.dockerignore`. Tighten step-by-step only
> after confirming the manifest works at the looser setting.

---

### Cross-field invariants

Structural schema validation runs first; then the following invariants are
checked and MUST all hold:

1. **Unique step IDs.** No two steps may share the same `id` value.
2. **`data_collection` / `data_processing` steps MUST declare a non-empty
   `command`.** This is matched against the step's `id` (not its `nine_step`
   field): any step whose `id` is literally `"data_collection"` or
   `"data_processing"` carries this obligation even when a `data_sources`
   entry already provides a download URL. The rationale is that the download
   path may be unavailable; the command is the fallback.
3. **`depends_on` references.** Every step ID listed in any step's `depends_on`
   array MUST refer to an existing step ID.
4. **`expected.step_id` references.** Every `step_id` value in the `expected`
   array MUST refer to an existing step ID.
5. **`data_sources.*.satisfies` references.** Every step ID listed in any data
   source's `satisfies` array MUST refer to an existing step ID. (`satisfies`
   has `minItems: 1` — each data source must satisfy at least one step.)
6. **`secrets.*.used_by` references.** Every step ID listed in a secret's
   `used_by` array MUST refer to an existing step ID, with the exception of
   entries that begin with `"data_sources."` (a qualifier form for data-source
   level usage), which are exempt from this check.

A violation of any invariant causes `plutus check` to fail before any step is
executed.

---

### Illustrative minimal manifest fragment

The fragment below is illustrative only — it is not a template. A complete
annotated template with all optional fields is provided at
`templates/manifest.yaml` in this repository.

```yaml
schema_version: "2.0"
repo:
  name: my-strategy
  primary_language: python
env:
  base: python          # python | python-cuda | none
  python_version: "3.11"
secrets: []             # no secrets — required key, may be empty
data_sources:
  processed: []
  raw: []               # both arrays required, may be empty
steps:
  - id: in_sample
    nine_step: step_4_in_sample    # one of 7 canonical keys, or null
    required: true                  # gates exit code — see §4
    command: "python -m my_strategy.backtest"
    inputs: []          # [] recommended: falls back to .dockerignore
    outputs: ["out/metrics.json"]
expected:
  - step_id: in_sample
    metrics:
      - name: sharpe_ratio
        value: 0.85
        tolerance: {kind: relative, value: 0.05}
```

## 1.2  The results contract  (`.plutus/run/<step_id>/results.json`, `schema_version: "1.0"`)

### Normative requirement

Each step that the verifier executes (steps with `verification_mode: execute`,
the default) MUST write `.plutus/run/<step_id>/results.json` on a clean exit.
The file MUST be valid JSON and MUST validate against the PLUTUS results JSON
Schema (Draft 2020-12). The authoritative schema lives in
`plutus_verify/sdk/schema.py`; any tool or language that produces a conforming
file satisfies the contract.

The Python SDK (`import plutus_verify as pv` … `with pv.step(...) as r:`) is
the recommended producer — it validates at call time and writes atomically — one
convenience path, not the only one.

> **Table convention:** **MUST** = required; *optional* = not required, but validated if present; *conditional* = required only under the stated condition.

> **Strict validation:** `additionalProperties: false` is set at the root, metric, and artifact levels — unknown or misspelled keys are rejected there. `metadata` is the sole exception (deliberately open; see the shape table).

---

### Top-level shape

| Key | Required | Type / Constraint | Notes |
|-----|----------|-------------------|-------|
| `schema_version` | MUST | string | Exactly `"1.0"` — a schema `const` |
| `step_id` | MUST | string | Matches the manifest step's `id` |
| `metrics` | MUST | array | Array of metric objects (may be `[]`) |
| `artifacts` | MUST | array | Array of artifact objects (may be `[]`) |
| `metadata` | MUST | object | Free key/value map (the only level without `additionalProperties: false` — deliberately open for author-supplied context such as seeds or notes); auto-populated fields documented below |

`step_id` MUST equal the `id` of the manifest step that wrote the file; it MUST match `^[a-z][a-z0-9_]*$` — the same snake_case pattern enforced on metric and artifact names.

---

### Metric object

| Field | Required | Type / Constraint | Notes |
|-------|----------|-------------------|-------|
| `name` | MUST | string | Matches `^[a-z][a-z0-9_]*$`; unique within the step |
| `value` | MUST | number | Finite `int` or `float`; `Decimal` and `bool` are rejected |
| `unit` | MUST | string enum | One of `fraction`, `ratio`, `count`, `currency_usd`, `seconds` |

> **WARNING — `percent` is rejected; always store a decimal (a common authoring error).**
> The `percent` unit is deliberately absent from the schema. Store `0.42`,
> not `42`, and declare it as `unit: "fraction"`.

For the fraction-vs-ratio choice: use `fraction` for values that are bounded shares naturally rendered as a percentage (win rate, max drawdown, annual return); use `ratio` for unbounded dimensionless numbers like Sharpe or Sortino. The rule: **any value you would write "42%" on a report → store `0.42` with `unit="fraction"`; any Sharpe-like ratio with no natural percent interpretation → `unit="ratio"`.**

---

### Artifact object

| Field | Required | Type / Constraint | Notes |
|-------|----------|-------------------|-------|
| `name` | MUST | string | Matches `^[a-z][a-z0-9_]*$`; unique within the step |
| `path` | MUST | string | Path relative to the step's working directory |
| `kind` | MUST | string enum | One of `chart`, `csv`, `json`, `image`, `other` |

---

### Write semantics

The write is **atomic** (rename-based) — readers never observe a partial file.
(The SDK's implementation serializes to `results.json.tmp` then calls
`os.replace`; other producers may use any equivalent atomic-rename approach.)
A step that raises an exception produces **no** `results.json` — the verifier
treats the absent file as a failed step.

Two metadata fields are **auto-injected** at write time; user-supplied values take priority — the SDK sets these fields only if they are absent:

| Injected key | Value | Override? |
|--------------|-------|-----------|
| `duration_seconds` | Wall-clock seconds elapsed inside the `with` block (rounded to 3 dp) | User-supplied `r.metadata(duration_seconds=…)` wins |
| `git_commit` | Short 7-character HEAD SHA (best-effort; omitted if no `.git`) | User-supplied `r.metadata(git_commit=…)` wins |

---

### Python SDK (recommended path)

```python
import plutus_verify as pv

with pv.step("in_sample") as r:
    r.metric("sharpe_ratio", float(sharpe), unit="ratio")
    r.metric("win_rate",     float(win),    unit="fraction")  # 0.42 means 42%
    r.artifact("equity_curve", "out/equity.png", kind="chart")
    r.metadata(seed=42)
```

The `pv.step()` context manager validates each `r.metric()` and `r.artifact()`
call immediately (fail-fast at the offending line), then re-validates the
assembled payload against the JSON Schema before writing. Any language or tool
that writes a conforming `results.json` without the SDK is equally valid.

---

### Compact example — valid `results.json`

```json
{
  "schema_version": "1.0",
  "step_id": "in_sample",
  "metrics": [
    {"name": "sharpe_ratio",  "value": 1.42,  "unit": "ratio"},
    {"name": "win_rate",      "value": 0.54,  "unit": "fraction"},
    {"name": "total_trades",  "value": 312,   "unit": "count"},
    {"name": "net_pnl",       "value": 4820.0,"unit": "currency_usd"},
    {"name": "avg_hold_time", "value": 7200,  "unit": "seconds"}
  ],
  "artifacts": [
    {"name": "equity_curve", "path": "out/equity.png", "kind": "chart"},
    {"name": "trade_log",    "path": "out/trades.csv",  "kind": "csv"}
  ],
  "metadata": {
    "duration_seconds": 18.4,
    "git_commit": "3f2a1b7"
  }
}
```

---

> **Note — `Decimal` values are rejected by the SDK (gotcha G6 in the
> troubleshooting catalogue — see §7).** Metric helpers in many repos return
> `decimal.Decimal` for monetary precision. The SDK's `r.metric()` accepts only
> `int` or `float`; passing a `Decimal` raises `ValueError` immediately.
> Always wrap the value: `r.metric("sharpe_ratio", float(bt.metric.sharpe_ratio(...)))`.

How the verifier uses metrics and artifacts — tolerance semantics, comparison
modes, and the exit-code contract — is defined in §4.

---

# 2  The 9-step taxonomy → manifest steps

## 2.1  Lineage — V1 README sections → V2 manifest keys

The ALGOTRADE 9-step trading-research process is the conceptual lineage that V1
formalised as required README sections. V2 carries those same seven steps forward
as a fixed set of canonical `nine_step` values that every manifest step can
declare. The table below is the authoritative mapping. The key strings are defined in
`plutus_verify/constants.py`; authors MUST copy them exactly (e.g.
`step_4_in_sample`, not `step_4_insample`).

| `nine_step` key | 9-step process | V1 README section |
|---|---|---|
| `step_1_hypothesis` | Step 1: Hypothesis formulation | `Trading (Algorithm) Hypotheses` |
| `step_2_data_collection` | Step 2: Data collection | `Data Collection` |
| `step_3_data_processing` | Step 3: Data processing | `Data Processing` |
| `step_4_in_sample` | Step 4: In-sample backtesting | `In-sample Backtesting` |
| `step_5_optimization` | Step 5: Optimization | `Optimization` |
| `step_6_out_of_sample` | Step 6: Out-of-sample backtesting | `Out-of-sample Backtesting` |
| `step_7_paper_trading` | Step 7: Paper trading | `Paper Trading` |

Steps 8 and 9 of the 9-step process (live trading and continuous improvement)
remain **out of scope** for `plutus check` verification. Reproducibility is expected through Step 6 (minimum) and
Step 7 (ideal), as under V1.

## 2.2  The fixed-frame / free-content boundary

The V2 standard fixes the **frame** — the seven canonical key strings, the
manifest field names, and the enums — not the **contents**. In particular:

- The **number of steps** beyond the canonical seven is legitimately
  repo-specific. A repository may declare additional steps (e.g. a feature
  engineering stage, an ML model-training stage) without any of them mapping to
  a canonical key.
- **Metric names** and their values are entirely author-defined; the standard
  does not prescribe what metrics a step must emit.

> Note: section content inside the human README is the author's responsibility and is not validated by `plutus check` — see §6.

## 2.3  Repo-specific steps (`nine_step: null`)

When a pipeline step fits none of the seven canonical keys, set `nine_step: null`
and provide a human-readable `label`. The `label` field is validated by the
schema (`type: ["string", "null"]`); it is optional but strongly recommended
whenever `nine_step` is `null` so that reports and dashboards can display a
meaningful name.

Example — an ML model-training step that precedes in-sample backtesting:

```yaml
steps:
  - id: model_training
    nine_step: null
    label: "ML model training"
    required: true
    command: "python -m my_strategy.train"
```

See §1.1 for the full step schema.

## 2.4  Steps outside the executable pipeline — `nine_step_coverage`

Some canonical steps — most commonly `step_7_paper_trading` — exist in the repository as documented work but cannot be re-executed automatically. Paper trading is typically **not reproducibly executable** in a CI environment: live-broker connectivity, real-time feeds, and wall-clock time make automated re-execution impractical. Two patterns are supported:

Two complementary patterns exist — they MAY be combined:

1. **`verification_mode: artifact_check`** — declare the paper-trading step but
   skip command execution; the verifier only checks that the declared output
   files exist.
2. **`nine_step_coverage` block** — document which canonical steps are covered
   by the repository, without requiring those steps to appear as executable
   manifest steps.

Pattern 1 declares a manifest step that skips execution; Pattern 2 is a top-level declaration that needs no step at all.

The optional top-level `nine_step_coverage` object is keyed by the canonical
nine-step key strings. Each entry has exactly two keys:

| Key | Required | Type | Notes |
|-----|----------|------|-------|
| `present` | MUST | boolean | Whether this step is present in the repository |
| `section` | optional | string \| null | The README section that documents the step |

Example:

```yaml
nine_step_coverage:
  step_7_paper_trading:
    present: true
    section: "Paper Trading"
  step_5_optimization:
    present: false
    section: null
```

Any subset of the seven canonical keys may appear; keys that are omitted imply
no coverage claim. See §4 for how the verifier uses this block at runtime.

---

# 3  Data tiers

V1's data guidance amounted to "put data on Google Drive." V2 replaces that with
a structured model: a repository declares how it provisions data in the manifest's
`data_sources` block, and the runtime resolves data automatically before executing
pipeline steps. This section defines the four recognised provisioning patterns
(informally called tiers), explains the `processed`-before-`raw` fall-through
semantics, and documents the `_DATA_SOURCE` field shape.

For ALGOTRADE-provided datasets see [data/DATA.md](data/DATA.md). For a
description of individual data fields see
[data/data-field-description.md](data/data-field-description.md). How the
verifier uses `data_sources` during execution is described in §4.

---

## 3.1  The `processed`→`raw`→`command` fall-through

`data_sources` has two required arrays: **`processed`** and **`raw`**. The
native runtime resolver tries them in this order for every step listed in a
source's `satisfies` array:

1. **`processed` first.** If a `processed` entry satisfies the step, the
   resolver checks whether its `expected_layout` files are already present in
   the repo. If so, it uses those files and skips any download or command for
   that step; if not, it attempts to download them. Only when no `processed`
   source can satisfy the step does the resolver proceed to `raw`.
2. **`raw` next.** If no `processed` source satisfies the step, the resolver
   checks `raw`. A `raw` source requires downloading or processing before it can
   be used.
3. **`command` fallback.** If neither array provides a usable source — or if the
   download is unreachable — the resolver runs the step's declared `command`. This
   is why any step with `id: data_collection` or `id: data_processing` MUST carry
   a non-empty `command` (see the cross-field invariant in §1.1): the command is
   the unconditional fallback, not an optional extra.

Both arrays are always required in the manifest root; an array with no entries is
written as `[]`, not omitted.

---

## 3.2  The four data-provisioning tiers

The tiers are an informal vocabulary; they are not named in the schema. They are
expressed by the combination of `processed`-vs-`raw`, the `kind` value, and
(for Tiers 3 and 4) declared secrets; Tier 4 additionally carries a remote `raw` entry.

### Tier 1 — Committed CSV (data in-repo)

**What it is.** Data files are small enough to commit to the repository (typically
under 50 MB total). No download step is needed; pipeline steps read the committed
files directly.

**Manifest shape.** `data_sources.processed` carries a single entry with
`kind: local` and `url: .` (the repo root). `data_sources.raw` is `[]`. The
`data_collection` step is typically omitted entirely — backtest steps depend on
each other instead.

```yaml
data_sources:
  processed:
    - kind: local
      url: .
      expected_layout:
        - data/is/prices.csv
        - data/os/prices.csv
      satisfies: [in_sample, out_of_sample]   # must match steps[].id — see §1.1 invariant 5
  raw: []
```

> Note: `.gitignore` MUST NOT exclude the committed data paths.

---

### Tier 2 — Drive / GitHub release (declared remote source)

**What it is.** Data is hosted externally (Google Drive folder, GitHub release
asset, plain HTTP download, or S3 bucket) and declared in `data_sources.raw`.
The verifier downloads the source before running the step. A `data_collection`
step MUST still be declared with a fallback `command` for use when the remote
source is unreachable.

**Manifest shape.** `data_sources.processed` is `[]`; `data_sources.raw` carries
the remote source. `kind` is one of the known values: `google_drive`,
`github_release`, `http`, `s3`, or `manual`. For authenticated sources add
`secrets_required` to the entry and route the secret key through `secrets[].used_by`.

```yaml
data_sources:
  processed: []
  raw:
    - kind: google_drive
      url: https://drive.google.com/drive/folders/<ID>
      expected_layout:
        - data/is/prices.csv
        - data/os/prices.csv
      satisfies: [data_collection]
      # secrets_required: [DRIVE_TOKEN]   # authenticated sources only — see Tier 3 secrets[] syntax

steps:
  - id: data_collection
    nine_step: step_2_data_collection    # canonical key — see §2
    required: true
    network: bridge
    timeout_seconds: 1800
    command: "python data_loader.py"   # fallback if the Drive source is unreachable
```

The ALGOTRADE-provided Backtest Data Suite and sample-project datasets hosted on
Google Drive are Tier 2 raw sources (see [data/DATA.md](data/DATA.md)).

---

### Tier 3 — Database (live query via secrets)

**What it is.** A `data_collection` step connects to a database at runtime using
credentials injected as secrets. Both `data_sources` arrays are empty. The step
MUST use `network: bridge` so the container can reach the database host. Backtest
steps that import a module which opens a database connection at import time also
need `network: bridge` (see the G1 gotcha in §7).

**Manifest shape.** `data_sources.processed` and `raw` are both `[]`. Secrets
carry the connection credentials; their `used_by` arrays list every step that
touches the database layer.

```yaml
secrets:
  - key: HOST
    purpose: Database host
    used_by: [data_collection, in_sample_backtest, out_of_sample_backtest]
  - key: PASSWORD
    purpose: Database password
    used_by: [data_collection, in_sample_backtest, out_of_sample_backtest]

data_sources:
  processed: []
  raw: []

steps:
  - id: data_collection
    nine_step: step_2_data_collection
    required: true
    network: bridge
    timeout_seconds: 1800
    command: "python data_loader.py"
```

---

### Tier 4 — Layered (Drive primary + DB fallback)

**What it is.** A `data_sources.raw` entry provides a Drive (or HTTP) source as
the primary path; if that download fails, the `data_collection` command falls
back to a live database query. Both a remote `raw` entry and database secrets are
present simultaneously.

This is the most complex pattern and carries the highest authoring risk: secrets
must cover both the download-path steps and the live-query fallback, and the
`expected_layout` of the remote source must exactly match the file paths the DB
query would produce. **First-time authors should start with Tier 2 or Tier 3 and
upgrade to Tier 4 only when both paths are confirmed independently working.**

---

## 3.3  `_DATA_SOURCE` field reference

Each entry in `data_sources.processed[]` or `data_sources.raw[]` MUST conform to
the following shape. All keys listed here and no others are accepted
(`additionalProperties: false`). Cross-references: the `satisfies` step IDs must
resolve against declared steps (§1.1 invariant 5); `secrets_required` key names
must match entries in the top-level `secrets` array (§1.1).

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `kind` | MUST | string | Backing-store identifier — free-form string; known values: `local`, `google_drive`, `github_release`, `http`, `s3`, `manual` (conventional label for human-provisioned data; no downloader is invoked) |
| `url` | MUST | string | Full URL for remote kinds (`google_drive`, `github_release`, `http`, `s3`); for `kind: local` use `.` (repo root) |
| `expected_layout` | MUST | array of strings | Glob patterns the downloaded/committed files must satisfy |
| `satisfies` | MUST | array of strings (`minItems: 1`) | Step IDs whose data requirement this source satisfies |
| `secrets_required` | optional | array of strings | Secret keys required to access this source; must match top-level `secrets[].key` values |
| `label` | optional | string \| null | Human-readable label for reports and dashboards |

> **Note on `kind`:** the JSON Schema validates `kind` as a free-form string with
> no enum constraint. The values listed above are conventional; typos pass schema
> validation and will only surface as a runtime download failure.

---

# 4  Verification — the `plutus check` contract (exit codes)

<!-- TODO(T8) -->

---

# 5  Compliance & scoring (50/25/10/15) and badges

<!-- TODO(T9) -->

---

# 6  README & documentation requirements (the Tidy bucket)

<!-- TODO(T10) -->

---

# 7  Recommended workflow & tooling (non-normative)

<!-- TODO(T11) -->

---

# 8  Sample projects & badges

<!-- TODO(T12) -->

---

# Appendix A  Versioning & relationship to V1

<!-- TODO(T13) -->
