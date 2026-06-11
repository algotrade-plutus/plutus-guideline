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
a strict `.plutus/run/<step>/results.json` (`schema_version: "1.0"`) at runtime
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

## 1.2  The results contract  (`.plutus/run/<step>/results.json`, `schema_version: "1.0"`)

<!-- TODO(T4) -->

---

# 2  The 9-step taxonomy → manifest steps

<!-- TODO(T5) -->

---

# 3  Data tiers

<!-- TODO(T6) -->

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
