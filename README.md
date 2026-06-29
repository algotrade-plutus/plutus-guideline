# The PLUTUS Standard

> **A standard for reproducible algorithmic-trading research.**
> Version 2 · targets `plutus-verify` ≥ 0.5.1
>
> *This is the front door, written for human readers. For a step-by-step walkthrough of
> making one project compliant, read the [GUIDE](GUIDE.md). The exhaustive technical
> contract (schemas, exit codes, tolerances) lives in the [REFERENCE](REFERENCE.md).*

---

## Why this standard exists

A trading-strategy result is only worth as much as your ability to **get the same
numbers again**. If a project reports a Sharpe ratio of 1.4 and a 12% annual return,
anyone should be able to take the code, run it, and see 1.4 and 12% — not "roughly,"
not "after emailing the author," but the actual numbers the report claims.

That property is **reproducibility**, and it is the entire point of the PLUTUS
Standard. Everything else in this document — the file formats, the scoring, the
badges — exists to make reproducibility *checkable* rather than something you take on
faith.

This is true of both versions of the standard. Version 1 and Version 2 share one
philosophy; they differ only in *how* a project proves it reproduces:

- **Version 1** asked a human to read the README and follow it. "Reproducible" meant
  "a reviewer could, in principle, re-run this by hand."
- **Version 2** makes the same claim *mechanical*. Reproducible now means **one
  command (`plutus check`) re-runs the whole pipeline in a clean container and
  confirms the numbers match** — no human judgment on the verification path.

If you are new here, you do not need to read Version 1. This document stands on its
own. (Version 1 is archived at [versions/v1/](versions/v1/) for projects badged under
the old criteria.)

---

## The two pillars

Reproducibility is supported by two things working together:

**1. The 9-step process — the *concept*.**
ALGOTRADE's [9-step process](https://www.algotrade.vn/knowledge/9-step-process)
describes how a trading strategy should be developed, from forming a hypothesis to
live trading. It gives every project the same skeleton:

> hypothesis → data preparation → forming the set of rules → in-sample backtest →
> optimization → out-of-sample backtest → paper trading → live trading →
> continuous improvement

PLUTUS cares about **Steps 1 through 7** (up to paper trading). Steps 8 and 9 (live
trading, continuous improvement) are out of scope — they cannot be re-run on demand.

**2. This guideline — the *practice*.**
The 9-step process tells you *what* good research looks like. This guideline tells you
*how to package a repository* so that its reproducibility can be checked
automatically. It defines the files you add, what they must contain, and how the
result is scored.

In short: the 9-step process is the methodology; this guideline is the contract that
makes the methodology verifiable.

---

## What "compliant" means, in one sentence

> A project is compliant when **`plutus check` exits 0** against the contract the
> project declares about itself.

Exit 0 means: every required step ran cleanly in a fresh container, and every metric
and chart it produced matched what the project claimed it would. That's it. The rest
of this document explains how you declare that contract and what the score on top of
it means.

---

## How a project proves it reproduces

You add a small amount of machine-readable bookkeeping to your repo. Two pieces:

### 1. A manifest — what the project *promises*

A single file, `.plutus/manifest.yaml`, committed to git. Think of it as the
project's **plan and promise in one place**: it lists each step of your pipeline, says
how the data is obtained, and states the metrics and charts each step is *expected* to
produce.

The manifest is the authoritative description of what your pipeline does. (Your README
still matters — see [Documentation](#what-your-readme-should-contain) — but its job is
to explain the *why* to humans, not to be the source of truth for *what* the code
does.)

A stripped-down manifest looks like this:

```yaml
schema_version: "2.0"
repo:
  name: my-strategy
  primary_language: python
env:
  base: python
  python_version: "3.11"
secrets: []
data_sources:
  processed: []
  raw: []
steps:
  - id: in_sample
    nine_step: step_4_in_sample     # which of the 9 steps this is
    required: true                   # a "required" step failing means the project fails
    command: "python -m my_strategy.backtest"
expected:
  - step_id: in_sample
    metrics:
      - name: sharpe_ratio
        value: 0.85
        tolerance: {kind: relative, value: 0.05}   # "within 5% of 0.85"
```

That `expected` block is the promise: *"when you run my in-sample backtest, the Sharpe
ratio will be 0.85, give or take 5%."* `plutus check` holds you to it.

> The full list of fields, the validation rules, and every enum value are in
> [REFERENCE.md §1](REFERENCE.md#1-manifest-schema). A fully annotated starter file you
> can copy is at [templates/manifest.yaml](templates/manifest.yaml).

### 2. Results — what the project *actually produced*

When each step runs, it writes a small `results.json` recording the metrics and charts
it actually produced. `plutus check` compares these against the manifest's promises.

You don't have to hand-write these files. The Python SDK does it for you — wrap the end
of a script in a `with` block and report your numbers:

```python
import plutus_verify as pv

with pv.step("in_sample") as r:
    r.metric("sharpe_ratio", float(sharpe), unit="ratio")
    r.metric("win_rate",     float(win),    unit="fraction")  # 0.42 means 42%
    r.artifact("equity_curve", "out/equity.png", kind="chart")
```

> **One thing that trips everyone up:** store percentages as decimals. A 42% win rate
> is `0.42` with `unit="fraction"`, never `42`. There is no `percent` unit.

> Full rules — allowed units, file format, how the comparison tolerances work — are in
> [REFERENCE.md §2](REFERENCE.md#2-results-schema) (results) and
> [§5](REFERENCE.md#5-the-plutus-check-contract) (comparison).

### Putting it together

There are two commands, and they run the *same* pipeline — they differ only in the last
move:

```
  plutus snapshot .   # bless: run every step, write the baseline you commit
  plutus check .      # verify: run again, compare against that baseline
```

You **bless** once with `plutus snapshot`: it builds your declared environment in a Docker
container, runs every step, and records the numbers and charts as the committed
groundtruth (`.plutus/expected/` plus the values in your manifest). From then on `plutus
check` **verifies** — it rebuilds the same container, reruns every step, and compares the
fresh output against that groundtruth. If everything matches, it exits 0. A working Docker
installation is the only prerequisite.

The split exists for one reason: a baseline should change only when you *decide* it
should. So `check` is **read-only** — everything it produces lands in gitignored scratch
directories (`.plutus/results/`, `.plutus/cache/`, `.plutus/run/`), and it never edits a
committed file. Drift you didn't deliberately re-bless fails the check instead of quietly
overwriting the baseline. (Add those three directories to your `.gitignore`.)

> If your manifest declares `secrets` (a database password, an API key), add
> `--secrets-from-env` so `plutus check` injects them from your shell — only the keys you
> declared are passed in, nothing else. A project with no secrets (like committed data)
> needs no flag.

The exit code is the whole verdict:

| Exit code | Meaning |
|-----------|---------|
| **0** | Everything reproduced. Required steps ran clean; all numbers and charts matched. |
| **1** | Steps ran, but at least one metric or chart didn't match what was promised. |
| **2** | A required step crashed, or the manifest itself was invalid. |

> The precise definition of each code — what counts as "required," how skipped steps
> behave, the exact comparison math — is in
> [REFERENCE.md §5](REFERENCE.md#5-the-plutus-check-contract).

---

## Mapping your project to the 9 steps

Each step in your manifest declares which of the 9 steps it belongs to, using a fixed
key. These are the seven keys PLUTUS recognizes:

| Key | The step |
|-----|----------|
| `step_1_hypothesis` | Hypothesis formulation |
| `step_2_data_preparation` | Data preparation (collection **and** processing) |
| `step_3_forming_set_of_rules` | Forming the set of rules |
| `step_4_in_sample` | In-sample backtesting |
| `step_5_optimization` | Optimization |
| `step_6_out_of_sample` | Out-of-sample backtesting |
| `step_7_paper_trading` | Paper trading |

Copy these exactly — `step_4_in_sample`, not `step_4_insample`.

> The 2025 revision of the 9-step process merged data collection and processing into one
> step (`step_2_data_preparation`), and made step 3 "forming the set of rules." The older
> `step_2_data_collection` / `step_3_data_processing` keys no longer load — rename them if
> you're updating an existing manifest.

A few practical notes:

- **You can have extra steps.** A feature-engineering or model-training stage that
  doesn't fit any of the seven keys is fine — set `nine_step: null` and give it a
  readable `label`.
- **Reproducibility is expected through Step 6 (minimum) and Step 7 (ideal).** Paper
  trading often can't be re-run automatically (it needs a live broker and real-time
  feeds), so the standard lets you *document* it rather than execute it. See
  [REFERENCE.md §3](REFERENCE.md#3-the-nine-step-keys) for how.

---

## Getting your data to the verifier

`plutus check` runs in an isolated container, so it needs to know how to obtain your
data. You declare that in the manifest's `data_sources` block. There are four common
patterns (informally, "tiers"):

| Tier | When to use it |
|------|----------------|
| **1 — Committed data** | Small datasets (under ~50 MB) you can commit straight into the repo. Simplest. |
| **2 — Remote download** | Data lives on Google Drive, a GitHub release, or an HTTP/S3 URL the verifier downloads. |
| **3 — Database** | A step queries a live database at runtime using credentials passed in as secrets. |
| **4 — Layered** | A remote download with a database fallback. Most powerful, most fragile — start with Tier 2 or 3 first. |

If you're starting out, Tier 1 or Tier 2 is almost always the right choice.

> The exact `data_sources` shape, the `processed → raw → command` fallback logic, and
> how secrets are routed are in [REFERENCE.md §4](REFERENCE.md#4-data-resolution-semantics).
> ALGOTRADE's shared datasets are documented in [data/DATA.md](data/DATA.md).

---

## How scoring and badges work

Passing `plutus check` is the foundation, but the PLUTUS badge reflects more than a
pass/fail. A project earns a percentage score built from four weighted buckets:

| Bucket | Worth | What it rewards |
|--------|------:|-----------------|
| **Reproducible** | 50 | `plutus check` exits 0 cleanly, with no workarounds. |
| **Tidy / well-documented** | 25 | Clear README, honest metric tables, clean setup, no surprise dependencies. |
| **Standardized** | 10 | Follows the canonical PLUTUS shape — predictable paths, externalized parameters. Could serve as a template. |
| **Innovative** | 15 | Original metrics, novel diagnostics, or non-textbook strategy ideas. |

**Your score is the sum of the four buckets, rounded to the nearest 5%.** For example,
50 + 22 + 10 + 8 = 90 → **90%**. There's no pass/fail gate, but Reproducible is half the
score, so a non-reproducing project can't get past 50% however polished it is.

The score renders as a `PLUTUS-<n>%` badge (via `plutus-document`).

**Treat the score as a reference signal, not a grade.** Only Reproducible (50) is
objectively verified — `plutus check` exits 0. The other three buckets are scored by the
`plutus-scoring` skill **by LLM judgment**, so they vary by model and over time. The thing
you can rely on is the green check.

> **PLUTUS score is an LLM-assessed reference signal, not a certified quality grade, and
> is subject to change. The verified guarantee is reproducibility — `plutus check` exits 0.**

> The full point-by-point rubric and the badge markup are in
> [REFERENCE.md §6](REFERENCE.md#6-scoring-rubric).

---

## What your README should contain

The **Tidy** bucket (25 points) scores your documentation. A strong PLUTUS README
covers, at minimum:

- **Abstract** — the project in under five sentences.
- **Introduction** — the why, the how, and the goal, in a paragraph or two.
- **Trading Hypotheses** — your Step 1.
- **Data** — collection (Step 2) and processing (Step 3): what data, what period, how
  to get it, what you did to it.
- **Implementation / How to Run** — environment setup and the concrete steps to
  reproduce from in-sample backtesting through to out-of-sample or paper trading.
- **In-sample Backtesting**, **Optimization**, **Out-of-sample Backtesting** — each
  with a results subsection. *The numbers here must match the manifest's `expected`
  block and what the code actually produces.*
- **Reference** — works cited.

A few documentation rules that catch people out:

- **Metric tables must match reality.** If the README says Sharpe 1.42 but the manifest
  promises 1.35, that's a Tidy failure even if `plutus check` is green.
- **`.env.example` must be sourceable.** Wrap placeholder values in quotes —
  `HOST="<your_host>"`, not `HOST=<your_host>` — or `source .env` breaks on the `<`.
- **No surprise data.** Every file or database a step reads must be declared in the
  manifest or committed to the repo.
- **Pin your Python and run CI.** Include a `.python-version` (or equivalent) and a CI
  workflow that runs `plutus check` on every push.

> The complete documentation checklist (the Tidy bucket criteria) is in
> [REFERENCE.md §6](REFERENCE.md#6-scoring-rubric).

---

## The toolkit (optional but recommended)

Compliance is defined by the two files and a green `plutus check` — *not* by any
particular tool. That said, the `plutus-verify` package makes the path easy. It's a
uv-managed project (not yet on PyPI), so install the CLI with [uv](https://docs.astral.sh/uv/)
from a release wheel:

```bash
uv tool install ./plutus_verify-<version>-py3-none-any.whl
```

(`pip install -e .` from a checkout also works.) uv is the recommended manager elsewhere
too: pin your *own* strategy's environment with `env.manager: uv` and a committed
lockfile so the container rebuilds identically every time — see [the GUIDE](GUIDE.md#62-env--the-environment-to-build).

A typical first-time flow:

1. `plutus init` — scaffold the manifest, a CI workflow, an example script, and a
   `.dockerignore`.
2. Instrument your scripts with the SDK (`with pv.step(...) as r:`).
3. Run them, then `plutus bootstrap` to auto-fill most of the manifest from the output.
4. `plutus snapshot` — bless your baselines (builds and runs in-container, then writes
   the committed groundtruth).
5. `plutus check .` — verify locally (read-only). Add `--secrets-from-env` only if your
   manifest declares secrets.

There are also three AI-assisted Claude Code skills, chained in this order:

- **`plutus-standardize`** — brings an existing repo up to the standard end-to-end
  (Survey → Decide → Instrument → Verify), anchored on a green `plutus check`. *(Formerly
  `plutus-transform`; it still answers to the old name.)*
- **`plutus-scoring`** — scores a compliant repo against the rubric and ranks the cheapest
  improvements.
- **`plutus-document`** — renders the standard README from the verified groundtruth
  (metric tables, chart embeds, reproduction block) plus your strategy narrative.

> Command details, the troubleshooting catalogue of known gotchas, and version notes
> are in [REFERENCE.md §7](REFERENCE.md#7-cli-reference) and
> [§9](REFERENCE.md#9-troubleshooting-catalogue).

---

## Sample projects & badges

A growing set of projects carry PLUTUS badges. The canonical **Version 2** reference is
[**Group09-BuyHighSellLow**](https://github.com/algotrade-education/Group09-BuyHighSellLow)
— a fully instrumented repo that passes `plutus check` and scores 90%. Start
from its `.plutus/` directory if you want a concrete working example.

Projects badged under V1's criteria remain valid until re-verified under V2. The exact
badge markup is in [REFERENCE.md §6.2](REFERENCE.md#62-badges). More educational examples
live at [github.com/algotrade-course](https://github.com/algotrade-course).

Projects must be hosted under an ALGOTRADE-managed GitHub organization
(`algotrade-research`, `algotrade-course`, `algotrade-education`, and others as the
program grows).

---

## Versioning

This is **Version 2** of the PLUTUS Standard, targeting `plutus-verify` ≥ 0.5.1
(manifest `schema_version: "2.0"`, results `schema_version: "1.0"`). The schema versions
have held steady since 0.2.10; what changed inside V2 is the nine-step taxonomy (0.3.0)
and the read-only `check` / in-container `snapshot` data flow (0.5.0). See
[REFERENCE.md §10](REFERENCE.md#10-versioning--schema-correspondence) for the details.

Version 1 is preserved verbatim at [versions/v1/README.md](versions/v1/README.md).
Projects badged under V1 keep their badges as **v1-legacy** until a maintainer
re-verifies them under V2. See [CHANGELOG.md](CHANGELOG.md) for the full history.
