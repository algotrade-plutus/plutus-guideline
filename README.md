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
no coverage claim. The verifier validates this block's shape but does not gate
exit codes on it; it serves scoring and documentation purposes — see §5.

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

`plutus check` is the normative verification command. Its exit code is the
machine-readable definition of "reproduced." Every other verdict in this
standard — the scoring rubric in §5, CI badge status, the Reproducible bucket
score — derives from this single contract.

---

## 4.1  Invocation

**Docker is the runtime prerequisite.** The verifier builds the environment
declared in `env` (§1.1) into a container image from a generated
`Dockerfile` and runs each pipeline step inside that container. A working
Docker daemon MUST be present on the host.

```
plutus check [REPO_PATH] [--secrets-from-env] [--data-tier {processed|raw|code|auto}] [--visual-check]
```

The minimal author-side invocation for a repo that uses environment secrets:

```
plutus check . --secrets-from-env
```

`--secrets-from-env` passes the caller's entire `os.environ` as secrets into
the container, satisfying every declared `secrets[].key` without a
secrets-management backend. `REPO_PATH` defaults to `.`; `--data-tier`
defaults to `auto` (processed → raw → command fall-through per §3).
When `--visual-check` is omitted, `visual_similarity` comparisons fall back
to byte comparison: bytes identical → `ok` (passes normally); bytes differ →
`WARN` (ok=False, skipped=True), which is non-blocking (see §4.4).

> **WARNING — stale run artifacts.** `plutus check` wipes `.plutus/run/`
> at the start of each invocation. Files left by a prior run are never
> consulted; comparisons always reflect what the current run produced.

---

## 4.2  The exit-code contract

This table is the normative definition of "reproduced":

| Exit code | Meaning | Condition |
|-----------|---------|-----------|
| **0** | All required steps executed cleanly AND all metric and artifact comparisons passed | Every required step exited 0 (no preflight error, not a hard execution failure), AND every metric comparison `ok`, AND every artifact comparison either `ok` or `skipped` |
| **1** | All required steps ran, but ≥ 1 comparison failed — including comparisons declared for optional steps (soft fail). | No required step hard-failed, but at least one metric returned `ok=False`, or at least one artifact returned `ok=False` and `skipped=False` |
| **2** | A required step failed execution, or a preflight / manifest-load error occurred | At least one required step exited non-zero (with no `skipped_reason`) OR has a `preflight_error`, OR manifest loading failed, OR the Docker image build failed |

**`required: true` gates exit 2** (§1.1): only a required step whose
execution hard-fails (non-zero exit, no `skipped_reason`) or whose preflight
check fails can push the exit code to 2. Exit 0 is the bar for the full
"reproducible" claim (see §5 Reproducible bucket for scoring implications).

**Exit 1 considers ALL declared comparisons**, including those belonging to
optional (`required: false`) steps. The `_exit_code` function in
`plutus_verify/scaffold/check.py` iterates `runtime.metric_results` and
`runtime.artifact_results` without filtering by `required`. A failed
comparison on an optional step will cause exit 1.

**Skipped steps are non-blocking for exit 2.** `skipped_reason` is a runtime field, not a manifest key; the verifier sets it when a step's work is satisfied without executing its command (e.g. because a data source already provides the required files — the value is `"satisfied_by_data_source"`). A step that records a
`skipped_reason` is not counted as a hard execution failure even if it is
marked `required`: `_exit_code` guards the exit-2 branch with
`sr.skipped_reason is None`. However, skipping does not suppress comparisons:
`_compare_metrics` in `plutus_verify/spec/runtime/orchestrator.py` does not
check `skipped_reason`. A skipped step produces no `results.json`; the
missing file triggers `MissingResultsError`, which returns `ok=False` for
every declared expected metric — contributing to exit 1. Authors SHOULD NOT
declare `expected.metrics` for steps that may be skipped (for example, a
`data_collection` step satisfied by a data source), because the absent
`results.json` will fail those comparisons.

The four-state status vocabulary used throughout this section:

| Status | ok | skipped | Meaning |
|--------|----|---------|---------|
| `ok` | True | False | Verified pass |
| `SKIP` | True | True | Not verified; no evidence of issue |
| `WARN` | False | True | Divergence detected but inconclusive (non-blocking) |
| `FAIL` | False | False | Verified divergence (blocks exit 0) |

---

## 4.3  Metric comparison semantics

Metrics are read from `.plutus/run/<step_id>/results.json` (§1.2) and
matched **by name** against `expected[].metrics[]` in the manifest. The
comparison is deterministic — no LLM involved; the `locate` directives used
by pre-v2 tooling do not appear in v2 manifests.

Each metric declares a `tolerance` block with a `kind` and `value`. Three
kinds are recognised, implemented in `_within_tolerance()` in
`plutus_verify/spec/runtime/orchestrator.py`:

| Kind | Formula | Implementation notes |
|------|---------|----------------------|
| `exact` | `actual == expected` | Plain numeric equality — no float fuzz. Recommended only for integer metrics; use `absolute` or `relative` for floating-point metrics because exact equality on floats is brittle across environments. |
| `absolute` | `|actual − expected| ≤ value` | Symmetric; units must match |
| `relative` | `|actual − expected| / |expected| ≤ value` | When `expected == 0` the check degrades to `|actual| ≤ value` (divide-by-zero is avoided by comparing the actual value against the tolerance bound directly) |

Both the manifest's `expected.metrics[].value` (§1.1) and the metric values in
`results.json` (§1.2) are typed `number` by their schemas, so in a conforming
repository every metric comparison is numeric.

A metric comparison that cannot locate the actual value (e.g. file missing)
is recorded as `ok=False` and contributes to exit 1.

---

## 4.4  Artifact comparison semantics

Artifacts are matched **by path** against `expected[].artifacts[]` in the
manifest. Three compare modes are supported, implemented in
`plutus_verify/spec/runtime/artifact_compare.py`:

| Mode | Behaviour | Missing baseline | Missing produced file |
|------|-----------|-----------------|----------------------|
| `byte_exact` | File bytes must be identical | `FAIL` (ok=False, skipped=False) | `FAIL` |
| `json_numeric_tolerance` | Deep-walk JSON; numeric values within relative tolerance (default 5%); non-numeric must be byte-equal. When `expected == 0`, degrades to absolute: `|produced| ≤ tol` | `FAIL` | `FAIL` |
| `visual_similarity` | LLM vision comparison at declared `threshold` (default 0.7) | **SKIP** (ok=True, skipped=True) — no reference yet; run `plutus snapshot` to enable | `FAIL` (ok=False, skipped=False) |

**`visual_similarity` without `--visual-check`.** When no vision client is
wired (the default), the verifier falls back to a byte comparison: bytes
identical → `ok` (passes normally); bytes differ → **WARN** (ok=False,
skipped=True). A WARN is non-blocking and does not contribute to exit 1
because `skipped=True` exempts it from the exit-code check.

---

## 4.5  What a green run looks like

A passing run (`plutus check . --secrets-from-env` → exit 0) prints a report
structured by the 9-step framework. A green run has no `FAIL` lines. Each
required step shows:

```
  ok <step_id>: exit=0
      ok <metric_name>: actual=<v> expected=<v>
      ok <compare_mode> <artifact_path>   # <compare_mode>: byte_exact | json_numeric_tolerance | visual_similarity | byte_identical
                                          # or SKIP / WARN — both non-blocking
```

Optional steps (`required: false`) and their comparisons appear in the same
format; a `FAIL` from an optional comparison still causes exit 1.

Any `FAIL` line in the report — whether from a metric, an artifact, or a
step execution — means the run exited 1 or 2. Exit 0 with no `FAIL` lines
is the definition of a reproduced result.

Back-references: `required` and `expected` fields → §1.1; `results.json`
schema → §1.2; data resolution (processed → raw → command) → §3. Forward
reference: how exit-code outcomes map to compliance scores → §5.

---

# 5  Compliance & scoring (50/25/10/15) and badges

The PLUTUS compliance score is a single percentage derived from four weighted
buckets. It replaces the hand-judged estimate used in V1 with a machine-derived
number produced by the `plutus-scoring` routine (see §7 for the tooling
overview). The only inputs to the score are the manifest, the scripts, the
README, and — for the Reproducible bucket — the outcome of `plutus check`.

## 5.1  The four buckets

| Bucket | Weight | What it measures |
|--------|--------|------------------|
| **Reproducible** | 50 pts | `plutus check` exits 0 and README-claimed metrics match script output within declared tolerance |
| **Tidy / well-documented** | 25 pts | README structure, install hygiene, no `<placeholder>` parse traps, code-hygiene patterns, docs match reality |
| **Standardized / template** | 10 pts | Follows canonical PLUTUS shape (7 nine-steps, predictable file paths, externalized parameters); could serve as a template |
| **Innovative** | 15 pts | New metrics, novel diagnostics, regime-tagged analytics, or strategy logic that is not textbook |

Maximum possible total: 100 pts.

---

### Reproducible (50)

This bucket's bar is identical to §4's exit-code contract: `plutus check`
exits 0. The tier scale below rewards clean reproducibility and penalises
workarounds:

- **50** — `plutus check` exits 0 cleanly; no workarounds applied; all required metrics inside tolerance.
- **45** — `plutus check` exits 0 but only after manifest-side workarounds (network/secret routing, snapshot seeding, etc.). Architectural smells were papered over, not fixed at source.
- **35** — `plutus check` exits 0 only after touching `requirements.txt` or other tracked config.
- **20** — Metrics match individually on host but `plutus check` cannot be made green within the session.
- **0** — Scripts don't produce the claimed metrics within tolerance.

---

### Tidy / well-documented (25)

Each of the five checklist items below is worth ~5 points; the bucket score is the sum of items satisfied. Unlike the other buckets, there are no named tiers.

- README structure clean, metric tables present and consistent
- `.env.example` parses with `source .env` (no unquoted `<placeholder>` lines)
- All data inputs documented (no surprise dependencies like a missing F2M leg)
- Optimization / parameter pipeline accurately described (script behavior matches docs)
- Has `.python-version` or equivalent pin, plus CI workflow

See §6 for the full documentation requirements that back this bucket.

---

### Standardized / template (10)

- **10** — Canonical 4-step shape (`data_collection` → `in_sample_backtest` → `optimization` → `out_of_sample_backtest`), parameters externalized to `parameter/*.json`, charts in predictable `result/{backtest,optimization}/` paths, no module-level side effects.
- **5** — Most of the above but one significant deviation (e.g., DB-at-import anti-pattern, divergent paper-trading script shape).
- **0** — Could not serve as a template without significant rework.

(The step names above are conventional step `id` values — the manifest `id` field — not the canonical `nine_step` keys of §2.)

---

### Innovative (15)

- **15** — Novel metrics (regime-tagged P&L, ADX-conditional return histograms, custom drawdown decomposition), novel diagnostics, or non-textbook strategy logic.
- **8** — Thoughtful strategy logic (regime-switching, opposite-extreme exits) but conventional analytical surface (standard Sharpe/Sortino/MDD/HPR + standard charts).
- **0** — Textbook backtest with no original instrumentation or analysis.

*(Bucket scores need not be multiples of 5; only the final sum is rounded.)*

---

## 5.2  Computing the score

**Percentage = sum of the four bucket scores, rounded to the nearest 5%.**

For example: Reproducible 45 + Tidy 20 + Standardized 10 + Innovative 8 = 83 pts → 83% → rounded to the nearest 5% = **85%**.

The `plutus-scoring` routine (§7) applies this rule and emits the rounded score.

---

## 5.3  No hard gate on `plutus check`

> **Note:** There is no hard pass/fail gate on `plutus check`. A repo that cannot make `plutus check` exit 0 is not disqualified from a badge — but Reproducible is 50% of the total, which caps a non-reproducing repo at the Bronze floor.

A repo that scores 0 on Reproducible can accumulate at most 25 + 10 + 15 = **50 pts**, which lands it exactly at the Bronze floor — and only if it is exemplary in every other dimension. The intent is to keep the door open for incremental improvement without implying that reproducibility is optional — it is simply priced in, not gated.

---

## 5.4  Badge tiers

Badge scores are now machine-derived via the `plutus-scoring` routine (§7),
not a hand-judged estimate as in V1. The thresholds and badge markup are
carried forward unchanged from V1. Repositories scoring below 50% do not qualify for a compliance badge.

**Standard compliance badges** (score shown in the badge label):

| Score range | Badge | Shields.io markup |
|-------------|-------|-------------------|
| 50%–69% | Bronze | `![Static Badge](https://img.shields.io/badge/PLUTUS-<score>-%23BA8E23)` |
| 70%–99% | Gold | `![Static Badge](https://img.shields.io/badge/PLUTUS-<score>-darkgreen)` |
| 100% | Platinum | `![Static Badge](https://img.shields.io/badge/PLUTUS-100%25-purple)` |

Replace `<score>` with the rounded number followed by `%25` (the URL-encoded `%` sign), e.g. `85%25` renders as `85%`. Platinum is always `100%25`; no placeholder needed.

**Special-designation badges** (awarded editorially, independent of score):

| Designation | Badge | Shields.io markup |
|-------------|-------|-------------------|
| Sample Project | ![Static Badge](https://img.shields.io/badge/PLUTUS-Sample-darkblue) | `![Static Badge](https://img.shields.io/badge/PLUTUS-Sample-darkblue)` |
| PROTO series | ![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88) | `![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88)` |

The Sample badge is given to projects selected as PLUTUS Sample Projects
featured in this guideline. The PROTO badge is given to projects in the PROTO
algorithm series developed by ALGOTRADE. Both may be combined with a compliance
badge. See §8 for the badge gallery and the current list of badged projects.

---

# 6  README & documentation requirements (the Tidy bucket)

The requirements in this section are what the **Tidy / well-documented** bucket (25 pts, §5)
scores. They are **not** the definition of compliance — that belongs to the two machine contracts (§1.1, §1.2).
A repository that meets every requirement in this section but whose `plutus check` exits non-zero
still scores 0 on the Reproducible bucket. Conversely, a repository that passes `plutus check`
cleanly but whose README is sparse will be capped at around 75 pts. Good documentation and
machine-verified reproducibility are complementary, not interchangeable.

A legacy subjective assessment guide is preserved at
[versions/v1/assessment/plutus-assessment-guide.md](versions/v1/assessment/plutus-assessment-guide.md)
for historical context; it does not define V2 scoring.

---

## 6.1  The required README section template

A Tidy V2 README contains the following sections. The structure is carried over directly from V1 §1;
the constraints stated there are preserved.

| Section | Required | Notes |
|---------|----------|-------|
| **Abstract** | MUST | Summarizes the project, motivation, methods, and findings in **fewer than five sentences** |
| **Introduction** | MUST | Briefly discusses the project: the motivation ("Why?"), an overview of the method ("How?"), and the goal ("What?"). Should be 1–2 paragraphs; details belong in the final report |
| **Related Work** or **Background** | optional | Brief prerequisite reading if the audience needs context before exploring the project |
| **Trading (Algorithm) Hypotheses** | MUST | Discusses Step 1 of the 9-step process |
| **Data** | MUST | Covers Steps 2–3; MUST contain two subsections: **Data Collection** (Step 2 — what data, period, format, structure, type; how to obtain it) and **Data Processing** (Step 3 — transformations applied, output location and format) |
| **Implementation** (or **How to Run**) | MUST | Environment setup; concrete steps to replicate from In-sample Backtesting (Step 4) through Out-of-sample Backtesting (Step 6) or Paper Trading (Step 7); how to change algorithm configurations |
| **In-sample Backtesting** | MUST | Standard parameter set, input data, and a **Result** subsection showing output tables/charts; link to the final report for the full experiment set |
| **Optimization** | MUST | What was optimized, over what ranges, and a **Result** subsection; the final parameter values used downstream MUST match what the pipeline actually runs |
| **Out-of-sample Backtesting** | MUST | Same structure as In-sample; results MUST reflect the post-optimization parameter set |
| **Paper Trading** | optional | Step 7; same structure; include data source, period, instruments, and results |
| **Conclusion** | optional | Summary of findings |
| **Reference** | MUST | All cited works |
| Link to Final Report/Paper | conditional | A hyperlink to the document. Required only when a final report exists; if in LaTeX, source + build instructions MUST be in the repo. |

> **Structure may deviate from this template when the situation requires it**, as under V1 §1 — but
> the README must still give a reader clear instructions for replicating results and an accurate
> project overview.

The complete set of parameters, input data, and outputs for all experiments is preferred in the
**Final Report or Paper**, not in the README. The README's Result subsections show the *standard*
(representative) run.

---

## 6.2  The five Tidy documentation checks

These are the five checklist items the scoring rubric evaluates (~5 pts each). Each is actionable
and independently scorable.

### 6.2.1  README structure and metric-table consistency

The README follows the section template above, **and** every metric table in the In-sample,
Optimization, and Out-of-sample sections is **numerically consistent with the manifest's `expected`
block** (§1.1). A discrepancy — for example, a Sharpe ratio of 1.42 in the README but an
`expected` value of 1.35 in the manifest — is a Tidy failure regardless of whether `plutus check`
exits 0. Authors MUST update both the README tables and the `expected` block together whenever
results change.

### 6.2.2  `.env.example` that parses under `source`

The repository MUST include a `.env.example` file that exports every secret key declared in
`secrets[]` (§1.1 and §3 data tiers). This file MUST parse without error under `source .env.example`
(or `source .env` after copying).

> **WARNING — unquoted `<angle-bracket>` placeholders break `source` (gotcha G3 in the
> troubleshooting catalogue — see §7).** A line such as
>
> ```bash
> # WRONG — the shell tries to evaluate <redis_host> as input-redirection
> MARKET_REDIS_HOST=<redis_host>
> ```
>
> causes `source .env` to fail with a parse error (e.g. `.env:8: parse error near '\n'`) on every shell
> that interprets `<` as input-redirection. The correct form quotes the placeholder value:
>
> ```bash
> # CORRECT — shell parses this as a plain string assignment
> MARKET_REDIS_HOST="<redis_host>"
> ```
>
> Every placeholder value in `.env.example` MUST be wrapped in double quotes so the file parses cleanly
> under `source`.

If a reviewer encounters an already-broken `.env` during setup, they can export only the needed keys (e.g. `eval "$(grep -E '^MARKET_' .env | sed 's/^/export /')"` — see §7 for the full workaround).

### 6.2.3  All data inputs documented

Every data input the pipeline reads — raw files, processed files, database tables, external API
calls — MUST be accounted for: either declared in the manifest's `data_sources` block (§3) or committed to the repository (Tier 1, §3) and described in the README's Data section. A reviewer who reads the README and inspects the manifest must be able to run `plutus check` without discovering undeclared data dependencies ("surprise dependencies"). A file a step reads that appears in neither place is a surprise dependency. Common examples include an F2M data leg that is queried at import time without a declared secret or `data_sources` entry, or a static file read by a script that is neither committed nor listed in `expected_layout`.

### 6.2.4  Optimization and parameter pipeline accurately described

The Optimization section of the README MUST accurately describe what the optimization step actually
does: which parameters were searched, over what ranges, by which method, and which values were
selected for the final run. The parameters the Out-of-sample and Paper Trading steps use MUST match
the values reported in the README — and those same values MUST be what the scripts actually pass to
the pipeline. A README that describes a grid search over `[10, 20, 30]` while the script runs a
single fixed value, or that reports a final window of 20 while the parameter file reads 15, is a
Tidy failure.

### 6.2.5  `.python-version` pin and CI workflow

The repository MUST contain a `.python-version` file (or an equivalent version pin in
`pyproject.toml` or a `python_version` field that is consistent with `env.python_version` in the
manifest) so that local `pyenv` and CI environments resolve the same interpreter. Additionally, a
CI workflow (GitHub Actions or equivalent) MUST be present that runs `plutus check` on each push or
pull request. The workflow ensures that reproducibility is verified automatically rather than only
at submission time. `plutus init` (§7) scaffolds a minimal `.github/workflows/plutus.yml` that
satisfies this requirement. CI green confirms the Reproducible contract; it does not substitute for the README metric-table consistency check in §6.2.1.

---

## 6.3  Final Report / Paper rules (carried from V1 §1.1)

A final report or paper is **optional** but strongly encouraged. When present:

- It contains all the same sections as the README but in **more detail**. The `Related Work`
  (or `Background`) and `Conclusion` sections are **required** in the final report (optional in the README only).
- It should contain adequate documentation to reproduce the result up to:
  - **Minimally** — Step 6: Out-of-sample Backtesting
  - **Ideally** — Step 7: Paper Trading
  of the [9-step process](https://hub.algotrade.vn/knowledge-hub/steps-to-develop-a-trading-algorithm/).
- If the document is written in LaTeX (or any other compiled markup), the **source files and
  compilation instructions MUST be committed to the repository**. A PDF-only submission without
  source does not satisfy this requirement.
- The complete set of parameters, input data, and outputs for all experiments belongs in the final
  report, not in the README.

---

# 7  Recommended workflow & tooling (non-normative)

> **Non-normative.** Compliance is defined entirely by the two contracts in §1: a valid `.plutus/manifest.yaml` (§1.1) and a passing `plutus check` exit code (§1.2). This section documents the recommended convenience path using the `plutus-verify` package (≥ 0.2.10, the version this standard targets; steps below reflect the actual subcommand set in `plutus-verify` 0.2.10). Any workflow that produces a conforming manifest and a green `plutus check` satisfies the standard; the specific commands below are suggestions, not requirements.

Install `plutus-verify` (≥ 0.2.10, the version this standard targets; see Appendix A for the version table):

```bash
pip install plutus-verify
```

## 7.1  Recommended author workflow

Run the steps below in order on a fresh repo.

1. **Scaffold with `plutus init`.**

   ```bash
   plutus init [REPO_PATH]
   ```

   Generates three files: `.plutus/manifest.yaml` (a manifest skeleton with every required field present and `TODO` placeholders), `.github/workflows/plutus.yml` (a minimal CI workflow that runs `plutus check` on every push and pull request — satisfies §6.2.5), and `.plutus/example_script.py` (an annotated illustration of SDK instrumentation). Use `--force` to overwrite an existing manifest or workflow.

   The [templates/manifest.yaml](templates/manifest.yaml) in this repository is the fully annotated reference skeleton; it covers every field — required and optional — with inline comments. It is the best starting point when you want more guidance than `plutus init` provides.

2. **Instrument your scripts with the SDK.**

   Add `pv.step(...)` blocks to each metric-emitting script (§1.2 — the SDK contract). Keep instrumentation additive: append the `with pv.step(...) as r:` block at the end of `if __name__ == "__main__"`, never modifying existing lines. Cast every metric value to `float(...)` (see G6 in §7.3).

3. **Run your scripts to produce `.plutus/run/<step_id>/results.json`**, then generate a manifest draft from the captured output:

   ```bash
   plutus bootstrap [REPO_PATH]
   ```

   `bootstrap` reads `.plutus/run/` (produced by your instrumented scripts) and auto-fills roughly 70% of `manifest.yaml.draft`, leaving `TODO_*` sentinels on fields that require domain knowledge, with a companion `manifest_TODO.md` explaining each sentinel. Edit the draft, fill the sentinels, then rename it to `manifest.yaml`.

   Alternatively, if your repo has a legacy README-based description, `plutus transfer` uses an LLM to extract a v2 draft manifest from the README:

   ```bash
   plutus transfer [REPO_PATH] [--llm-endpoint URL] [--llm-model MODEL]
   ```

   `transfer` produces `.plutus/manifest.yaml.draft` and `.plutus/instrument_TODO.md` — a checklist of scripts that still need SDK instrumentation. It uses four LLM calls and requires an OpenAI-compatible endpoint.

4. **Capture baselines with `plutus snapshot`.**

   After running your instrumented scripts locally, snapshot the outputs into `.plutus/expected/` so `plutus check` has a baseline to compare against:

   ```bash
   plutus snapshot [REPO_PATH] --no-run
   ```

   `snapshot` copies step output files into `.plutus/expected/<step_id>/` and fills `expected.metrics[].value` into `manifest.yaml` from the current run. Use `--no-artifacts` or `--no-metrics` to copy only one of the two. Currently `--no-run` is required — omitting it exits with an error (auto-run before snapshot is not yet wired in 0.2.10).

5. **Verify locally with `plutus check`.**

   ```bash
   plutus check . --secrets-from-env
   ```

   Runs the v2 pipeline locally against your working copy (§4). Builds a Docker image, executes each step in isolation, compares outputs to `.plutus/expected/`, and exits 0 on full pass. `--secrets-from-env` forwards the current shell environment into the container for any secrets declared in the manifest's `secrets[]` block. See §4 for the full check semantics and G3 in §7.3 if `.env` sourcing causes shell parse errors.

## 7.2  Automation skills

Two Claude Code skills provide AI-assisted paths for common workflows.

**`plutus-transform`** is an end-to-end v1→v2 migration skill. It runs four sequential phases (Survey → Decide → Instrument → Verify), anchored on `plutus check` exiting 0 with all README-claimed metrics matching script output within declared tolerance. Phase 2 is the only interactive phase; the rest run to completion unless a hard error appears. After a clean transform, the skill automatically chains into `plutus-scoring` for the compliance score and re-run command.

**`plutus-scoring`** applies the 50/25/10/15 compliance rubric (§5) to any v2-compliant repo and emits three outputs: per-bucket scores with one-line reasoning, ranked improvement paths (concrete file/line/change, cheapest wins first), and a copy-pasteable re-run command the maintainer can use any time. It is standalone-invokable — no transform required.

Invoke either skill via the Skill tool in Claude Code, or via `/plutus-transform` and `/plutus-scoring` slash commands.

## 7.3  Troubleshooting catalogue

The table below lists known failure patterns. Detailed **Symptom → Diagnosis → Fix** entries live in the `plutus-transform` skill's reference file at `skills/plutus-transform/references/known-gotchas.md` in the `plutus-automation-scoring` repository.

| ID | Title | Version applicability |
|----|---------------------------------------------------------|-----------------------|
| G1 | Module-level DB connection forces network/secret routing | All versions — current |
| G2 | Internally-conflicting dependency file | All versions — current |
| G3 | `.env` placeholders crash shell sourcing | All versions — current |
| G4 | `visual_similarity` artifacts silently failed | **Historical** — fixed in 0.2.6; not present on ≥ 0.2.6 |
| G5 | Container stderr/stdout swallowed | **Historical** — fixed in 0.2.6; not present on ≥ 0.2.6 |
| G6 | SDK rejects `Decimal` metric values | All versions — current |
| G7 | Chart baseline captured after smoke-run is tautological | All versions — current |
| G11 | Runtime volume mount bypasses `.dockerignore` | **Historical** — v0.2.9 only; fixed in 0.2.10 |
| G12 | All execution steps FAIL exit=2 after declaring `step.inputs` | v0.2.10+ — current behavior |

Authors on `plutus-verify` ≥ 0.2.10 can ignore G4, G5, and G11 — they describe bugs in earlier releases. G12 is the most common pitfall on the current release: when `step.inputs` is non-empty it is a complete-coverage allowlist, so the step's own script must be listed or the container will exit 2 with "No such file or directory." The recommended default is `inputs: []` on every step; tighten step-by-step after confirming a working baseline (see §1.1 and [templates/manifest.yaml](templates/manifest.yaml)). G7 is the subtlest active pitfall: capture chart baselines from a real run before any smoke-run overwrites them, or the comparison becomes tautological.

> **Version note.** This standard corresponds to `plutus-verify` ≥ 0.2.10. See Appendix A for the full version table and the changelog of normative behavior changes across minor releases.

---

# 8  Sample projects & badges

The projects listed in §8.1 were reviewed and badged under V1's criteria — a human-judged
compliance estimate against the V1 README requirements. Per Appendix A, they remain
valid as **v1-legacy** until a maintainer re-runs `plutus check` and the `plutus-scoring`
routine against a V2 manifest. When a project is re-verified under V2, its Standard column
entry is updated to `v2` and the badge score is replaced with the machine-derived
bucket breakdown (50/25/10/15) from §5. Badge markup follows §5.4.

## 8.1  Badged projects

| **ID** | **Project Name** | **Strategy Type** | **Author/Maintainer** | **Program** | **PLUTUS Badge** | **Standard** |
|--------|------------------|-------------------|-----------------------|:-----------:|:---------------------:|:------------:|
| 1 | [PROTO:SmartBeta](https://github.com/algotrade-research/ProtoSmartBeta) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/) | [Tạ Quang Khôi](https://github.com/khoi-ta), [Nguyễn An Dân](https://github.com/dan-algo) | ALGOTRADE PROTO | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen)![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88) | v1-legacy |
| 2 | [PROTO:Market Maker](https://github.com/algotrade-research/ProtoMarketMaker) | [Market-Making](https://hub.algotrade.vn/knowledge-hub/market-making-strategy/) | [Tạ Quang Khôi](https://github.com/khoi-ta), [Nguyễn An Dân](https://github.com/dan-algo) | ALGOTRADE PROTO | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen)![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88) | v1-legacy |
| 3 | [InstiFund](https://github.com/algotrade-research/InstiFund) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/), [ETF Front-Runner Strategy](https://hub.algotrade.vn/knowledge-hub/front-running-etf-strategy/) | [Đặng Minh Nhựt](https://github.com/BJMinhNhut) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-80%25-darkgreen) | v1-legacy |
| 4 | [DynamicGrid](https://github.com/algotrade-course/DynamicGrid) | [Grid](https://hub.algotrade.vn/knowledge-hub/grid-strategy/) | [Võ Hoàng Phúc Khang](https://github.com/vokhang1412), [Nguyễn Thanh Thảo Ly](https://github.com/sxweetlollipop2912), [Nguyễn Tuấn Khanh](https://github.com/ng-tuan-khanh) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-80%25-darkgreen) | v1-legacy |
| 5 | [SearchingTA](https://github.com/algotrade-research/SearchingTA) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/), [Mean-Reversion](https://hub.algotrade.vn/knowledge-hub/mean-reversion-strategy/) | [Tống Thiên Phước](https://github.com/tphuoc04/) | ALGOTRADE Internship 6.24 |  ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) | v1-legacy |
| 6 | [FinTrip](https://github.com/algotrade-research/FinTrip) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/) | [Tạ Quang Khôi](https://github.com/khoi-ta) | ALGOTRADE Internship 11.23 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) | v1-legacy |
| 7 | [Statisical-Arbitrage](https://github.com/algotrade-research/Statisical-Arbitrage) | [Market-Neutral: Statistical Arbitrage](https://hub.algotrade.vn/knowledge-hub/market-neutral-strategy/) | [Cao Quang Hiếu](https://github.com/HieuQCao) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) | v1-legacy |
| 8 | [TruTrend](https://github.com/algotrade-course/TruTrend) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Phạm Võ Quỳnh Như](https://github.com/pvqn), [Hồ Việt Bảo Long](https://github.com/lob23), [Nguyễn Phúc Bảo Uyên](https://github.com/Hollyuyen) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) | v1-legacy |
| 9 | [Intraday-Momentum](https://github.com/algotrade-research/Intraday-Momentum) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Lâm Thành Duy](https://github.com/ltduy6) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) | v1-legacy |
| 10 | [DuoMA](https://github.com/algotrade-course/DuoMA) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Nguyễn An Dân](https://github.com/dan-algo) (Maintainer) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) | v1-legacy |
| 11 | [QuarterOscillate](https://github.com/algotrade-course/QuarterOscillate) | [Mean-Reversion](https://hub.algotrade.vn/knowledge-hub/mean-reversion-strategy/) | [Đặng Đức Khiêm](https://github.com/duckhiemdang), [Nguyễn Lộc An](https://github.com/LucidNg), [Đặng Minh Triết](https://github.com/triet0612) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) | v1-legacy |
| 12 | [MomMean](https://github.com/algotrade-course/MomMean) | [Mean-Reversion](https://hub.algotrade.vn/knowledge-hub/mean-reversion-strategy/), [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Nguyễn Đức Hưng](https://github.com/DucHungGithub), [Hoàng Thiên Đức](https://github.com/Locopaly), Hoàng Nghĩa Việt |   CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) | v1-legacy |
| 13 | [SmartBeta](https://github.com/algotrade-research/SmartBeta) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/) | [Lê Đức Phú](https://github.com/dphu2609) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) | v1-legacy |
| 14 | [scalping-strategy](https://github.com/algotrade-research/scalping-strategy) | [Scalping](https://hub.algotrade.vn/knowledge-hub/scalping-strategy/) | [Trần Thị Phương Linh](https://github.com/ttplinh) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) | v1-legacy |
| 15 | [DeepMM](https://github.com/algotrade-research/deepmm) | [Market-Making](https://hub.algotrade.vn/knowledge-hub/market-making-strategy/) | [Lê Thanh Danh](https://github.com/danhleth), [Lương Thanh Anh Đức](https://github.com/luongthanhanhduc)| ALGOTRADE Internship 11.23 | ![Static Badge](https://img.shields.io/badge/PLUTUS-50%25-%23BA8E23) | v1-legacy |

### Other example projects

There is also a repertoire of projects that adhere to the PLUTUS Standard, which comes from the
educational program offered by ALGOTRADE, that can be explored
[here](https://github.com/algotrade-course).

Listed are the projects from:
- [CS408: Computational Finance, Algorithmic Trading - APCS, HCMUS, 2025](https://github.com/algotrade-course#cs408---apcs-hcmus---2025-project)

## 8.2  Canonical V2 reference project

[**Group09-BuyHighSellLow**](https://github.com/algotrade-education/Group09-BuyHighSellLow)
(`algotrade-education/Group09-BuyHighSellLow`) is the first designated canonical V2 reference
repository. It carries a committed `.plutus/manifest.yaml` (`schema_version: "2.0"`) with fully
instrumented pipeline steps and a `plutus check` exit 0 verified against the declared
expected-metrics contract (§1.1, §4). Its manifest and results files serve as the worked example
for all annotations in this standard; its bucket breakdown is recorded in CHANGELOG. New projects
that want a concrete V2 template should start from this repository's `.plutus/` directory structure
and SDK instrumentation pattern (§7.1).

| Field | Value |
|-------|-------|
| Repository | [algotrade-education/Group09-BuyHighSellLow](https://github.com/algotrade-education/Group09-BuyHighSellLow) |
| Manifest | `.plutus/manifest.yaml` (`schema_version: "2.0"`) |
| Verification | `plutus check` exit 0 |
| Standard | v2 |
| Score | scored in CHANGELOG |

---

# Appendix A  Versioning & relationship to V1

**V1 preserved verbatim.** The original V1 standard is archived without modification at
[versions/v1/README.md](versions/v1/README.md), including its sample project table and badge
percentage rules. The legacy assessment guide and report forms that previously lived at the
repository root have moved to [versions/v1/assessment/](versions/v1/assessment/).

**V1-badged repos remain valid.** Projects that earned a badge under V1 keep their badges and
are considered **v1-legacy** — scored under the original human-judged prose criteria — until a
maintainer re-verifies the project under V2. Re-verification requires a committed
`.plutus/manifest.yaml` (`schema_version: "2.0"`) and a clean `plutus check` exit 0; once
confirmed, the project's Standard column in §8 is updated to `v2` and the badge score is
replaced with the machine-derived bucket breakdown (50/25/10/15) from §5.

**Version correspondence.**

| Standard version | `plutus-verify` | Manifest `schema_version` | Results `schema_version` |
|------------------|-----------------|---------------------------|--------------------------|
| V2 (2026-06, this document) | ≥ 0.2.10 | `"2.0"` | `"1.0"` |
| V1 | n/a (prose standard) | n/a | n/a |

V2 of this standard was designed and validated against `plutus-verify` 0.2.10. Any release
≥ 0.2.10 and < 0.3.0 is expected to remain compatible; breaking changes will increment the
manifest or results schema version and be reflected in this table.

See [CHANGELOG.md](CHANGELOG.md) for the full version table and a summary of normative behavior
changes introduced in V2.
