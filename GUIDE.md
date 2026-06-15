# PLUTUS Authoring Guide

> **From an empty repo to a green `plutus check`, one step at a time.**
>
> This is the hands-on guide. If you want the *why* behind the standard, read the
> [README](README.md) first. If you want the exhaustive list of every field, option,
> and API — every enum value, every edge case — see the
> [technical reference](REFERENCE.md). This guide sits in the middle: it walks you
> through the *actual work* of making one project compliant, in order, assuming you've
> never done it before.

By the end you will have:

- a `.plutus/manifest.yaml` that describes your pipeline,
- your scripts reporting their results in a format the verifier understands,
- and `plutus check` exiting **0** — the definition of a reproducible PLUTUS project.

Take it in order. Each step builds on the previous one.

---

## Contents

1. [The mental model (read this once)](#1-the-mental-model-read-this-once)
2. [What you need before you start](#2-what-you-need-before-you-start)
3. [Our running example](#3-our-running-example)
4. [Step A — Install the tools](#4-step-a--install-the-tools)
5. [Step B — Make your code report its results](#5-step-b--make-your-code-report-its-results)
6. [Step C — Write the manifest, key by key](#6-step-c--write-the-manifest-key-by-key)
7. [Step D — Capture your baselines](#7-step-d--capture-your-baselines)
8. [Step E — Run the check](#8-step-e--run-the-check)
9. [Step F — When it's not green: fixing the common failures](#9-step-f--when-its-not-green-fixing-the-common-failures)
10. [Step G — Keep it green with CI](#10-step-g--keep-it-green-with-ci)
11. [Where to go next](#11-where-to-go-next)

---

## 1. The mental model (read this once)

Everything in PLUTUS comes down to three ideas. Hold these in your head and the rest
follows:

1. **The manifest is a promise.** In one file (`.plutus/manifest.yaml`) you declare:
   *here are my pipeline steps, here's how my data is obtained, and here are the
   numbers each step will produce.*

2. **`results.json` is what actually happened.** When a step runs, your code writes a
   small file recording the metrics and charts it really produced.

3. **`plutus check` compares the two — in a clean container.** It rebuilds your
   environment from scratch, runs every step, and checks that what actually happened
   (`results.json`) matches what you promised (the manifest). If it all matches, it
   exits `0`.

The whole game is: **make the promise, make the code keep the promise, prove it in a
clean room.** That's reproducibility.

> A "clean room" matters. `plutus check` doesn't trust your laptop's installed
> packages, your cached data, or your environment variables. It builds a fresh Docker
> container so that "it works on my machine" becomes "it works on *any* machine."

---

## 2. What you need before you start

Install these once:

| Tool | Why | Check it's there |
|------|-----|------------------|
| **Docker** | The verifier runs every step inside a container. | `docker info` prints without error |
| **Python 3.10+** | To run your scripts and the `plutus-verify` CLI. | `python --version` |
| **`plutus-verify`** | The package that provides the SDK and the `plutus check` command. | `pip install plutus-verify` |

```bash
pip install plutus-verify
docker info        # if this errors, start Docker Desktop / the docker daemon first
```

You do **not** need any database, cloud account, or special hardware for this guide —
our example uses small data files committed straight into the repo.

---

## 3. Our running example

We'll make one simple strategy compliant from start to finish. Imagine a
moving-average crossover backtest. Before adding PLUTUS, the repo looks like this:

```
my-strategy/
├── data/
│   ├── is/prices.csv          # in-sample price data (committed, small)
│   └── os/prices.csv          # out-of-sample price data (committed, small)
├── backtest.py                # runs the in-sample backtest, prints Sharpe etc.
├── evaluate.py                # runs the out-of-sample backtest
├── requirements.txt
└── README.md
```

`backtest.py` today just prints results to the screen:

```python
# backtest.py  (BEFORE PLUTUS)
sharpe = run_backtest("data/is/prices.csv")
print(f"Sharpe ratio: {sharpe:.2f}")
save_equity_curve("out/equity.png")
```

Our job: make `plutus check` able to confirm that Sharpe ratio automatically. The same
recipe applies to a real multi-step project — we're just keeping the example small so
the mechanics are clear.

The finished project will have one new directory and one new file you author by hand:

```
my-strategy/
├── .plutus/
│   ├── manifest.yaml          # ← you write this (Step C)
│   ├── run/                   # ← created automatically when steps run
│   │   ├── in_sample/results.json
│   │   └── out_of_sample/results.json
│   └── expected/              # ← created by `plutus snapshot` (Step D)
├── data/ …
├── backtest.py                # ← lightly instrumented (Step B)
├── evaluate.py                # ← lightly instrumented (Step B)
└── …
```

---

## 4. Step A — Install the tools

Covered above. One optional convenience: `plutus init` scaffolds a starter manifest, a
CI workflow, and an example script so you're not staring at a blank file.

```bash
cd my-strategy
plutus init .
```

This creates:

- `.plutus/manifest.yaml` — a skeleton with every required field present and `TODO`
  placeholders.
- `.github/workflows/plutus.yml` — a CI workflow that runs `plutus check` on every push
  (we'll cover this in Step G).
- `.plutus/example_script.py` — an annotated example of the instrumentation you're
  about to add.

You can also skip `plutus init` and write the manifest by hand — this guide shows you
exactly what goes in it either way. A fully annotated reference skeleton also lives at
[templates/manifest.yaml](templates/manifest.yaml).

---

## 5. Step B — Make your code report its results

`plutus check` needs each step to write a `results.json` saying what it produced. You
don't write that file by hand — the SDK writes it for you. You just wrap the part of
your script that has the final numbers.

### 5.1 Add a `pv.step(...)` block

Find the point in your script where the final metrics exist, and wrap it:

```python
# backtest.py  (AFTER PLUTUS)
import plutus_verify as pv

sharpe = run_backtest("data/is/prices.csv")
win_rate = compute_win_rate()
save_equity_curve("out/equity.png")

with pv.step("in_sample") as r:                       # "in_sample" = the step id
    r.metric("sharpe_ratio", float(sharpe), unit="ratio")
    r.metric("win_rate",     float(win_rate), unit="fraction")
    r.artifact("equity_curve", "out/equity.png", kind="chart")
    r.metadata(seed=42)                                # optional: anything useful for context
```

When the `with` block exits cleanly, the SDK writes
`.plutus/run/in_sample/results.json`. If your code raises an exception inside the
block, **no file is written** — and the verifier treats a missing file as a failed
step. That's deliberate: a crash should never look like a pass.

> **Keep instrumentation additive.** Append the `with pv.step(...)` block; don't rewrite
> your existing logic. The cleanest place is the end of your
> `if __name__ == "__main__":` section.

### 5.2 What's a metric vs. an artifact?

- A **metric** is a single number: a Sharpe ratio, a win rate, a P&L, a trade count.
- An **artifact** is a file your step produces: a chart, a CSV, a JSON report.

### 5.3 Three rules that trip up almost everyone

These are small, but each one will cost you a confusing failure if you miss it:

**Rule 1 — Always pass `float(...)`, never `Decimal`.** Many trading libraries return
`decimal.Decimal` for money-precise values. The SDK rejects `Decimal` (and `bool`). Wrap
every value:

```python
r.metric("sharpe_ratio", float(bt.sharpe()), unit="ratio")   # ✅
r.metric("sharpe_ratio", bt.sharpe(), unit="ratio")          # ❌ if bt.sharpe() is a Decimal
```

**Rule 2 — Percentages are stored as decimals; there is no `percent` unit.** A 42% win
rate is `0.42` with `unit="fraction"`. Never store `42`.

```python
r.metric("win_rate", 0.42, unit="fraction")   # ✅ means 42%
r.metric("win_rate", 42,   unit="fraction")   # ❌ means 4200%
```

**Rule 3 — Pick the right unit.** There are exactly five:

| Unit | Use it for | Example |
|------|-----------|---------|
| `fraction` | Anything you'd write as a percentage | win rate, max drawdown, annual return → `0.12` |
| `ratio` | Unbounded dimensionless numbers | Sharpe, Sortino → `1.42` |
| `count` | Whole-number tallies | number of trades → `312` |
| `currency_usd` | Money | net P&L → `4820.0` |
| `seconds` | Durations | average hold time → `7200` |

Rule of thumb: *if you'd put a `%` sign on it in a report, it's a `fraction`. If it's a
Sharpe-like number with no natural percentage, it's a `ratio`.*

Artifact `kind` is one of: `chart`, `csv`, `json`, `image`, `other`.

> Metric and artifact **names** must be lowercase snake_case: start with a letter, then
> letters/digits/underscores (`sharpe_ratio`, `equity_curve`). No spaces, no capitals.

### 5.4 Run the script and look at what it wrote

```bash
python backtest.py
cat .plutus/run/in_sample/results.json
```

You should see something like:

```json
{
  "schema_version": "1.0",
  "step_id": "in_sample",
  "metrics": [
    {"name": "sharpe_ratio", "value": 1.42, "unit": "ratio"},
    {"name": "win_rate",     "value": 0.54, "unit": "fraction"}
  ],
  "artifacts": [
    {"name": "equity_curve", "path": "out/equity.png", "kind": "chart"}
  ],
  "metadata": {"seed": 42, "duration_seconds": 18.4, "git_commit": "3f2a1b7"}
}
```

(`duration_seconds` and `git_commit` are filled in for you automatically.) Do the same
for `evaluate.py`, using a different step id like `out_of_sample`.

**Repeat this for every script whose numbers your README claims.** That's the whole of
Step B.

---

## 6. Step C — Write the manifest, key by key

Now the promise. Create `.plutus/manifest.yaml`. We'll build it one section at a time;
by the end of this step you'll have a complete, valid file.

> The manifest must be **valid YAML**, committed to git, and it must use only the keys
> shown here — unknown or misspelled keys are rejected, not ignored. If `plutus check`
> says "additional properties are not allowed," you have a typo in a key name.

### 6.1 The header: `schema_version` and `repo`

```yaml
schema_version: "2.0"        # must be exactly this string, in quotes
repo:
  name: my-strategy
  primary_language: python
```

### 6.2 `env` — the environment to build

This tells the verifier how to build the container.

```yaml
env:
  base: python               # one of: python | python-cuda | none
  python_version: "3.11"
  requirements_file: requirements.txt    # or pyproject.toml; or null if you have none
```

`base: python` covers almost every project. Use `python-cuda` only for GPU workloads.
`requirements_file` points at whatever pins your dependencies.

### 6.3 `secrets` — credentials your steps need

Our example needs none, so it's an empty list. **The key is still required** — write
`[]`, don't delete it:

```yaml
secrets: []
```

(If you used a database, each secret would be `{key, purpose, used_by}` and `used_by`
would list the steps that need it. See [Step F](#9-step-f--when-its-not-green-fixing-the-common-failures)
and the [reference](REFERENCE.md) for the database pattern.)

### 6.4 `data_sources` — how data reaches the container

There are **two required arrays**, `processed` and `raw`. The verifier looks in
`processed` first, then `raw`, then falls back to running a step's `command`.

Because our data is small and committed into the repo (the simplest pattern, "Tier 1"),
we declare it as a local `processed` source and leave `raw` empty:

```yaml
data_sources:
  processed:
    - kind: local
      url: .                            # repo root
      expected_layout:
        - data/is/prices.csv
        - data/os/prices.csv
      satisfies: [in_sample, out_of_sample]   # which steps these files feed
  raw: []
```

- `kind: local` + `url: .` means "the files are already in the repo."
- `expected_layout` lists the files (or glob patterns) that must be present.
- `satisfies` names the step ids that depend on this data — those ids must match steps
  you declare below.

> Make sure your `.gitignore` does **not** exclude the committed data paths, or the
> files won't be there when the verifier looks.

**If your data is too big to commit**, you'd instead host it (Google Drive, a GitHub
release, an HTTP/S3 URL) and declare it under `raw`, or query a database live. Those are
"Tier 2/3/4" — start with committed data to get green, then graduate. The
[reference §3](REFERENCE.md) covers each tier in full.

### 6.5 `steps` — your pipeline (the heart of the manifest)

Each step is one stage of your pipeline. List them in the order they run. Here are our
two:

```yaml
steps:
  - id: in_sample                    # unique; matches the pv.step("in_sample") in your code
    nine_step: step_4_in_sample      # which of the 9 canonical steps this is (table below)
    required: true                   # a required step failing makes the whole check fail
    command: "python backtest.py"    # how the verifier runs this step
    inputs: []                       # leave [] — see the warning below
    outputs:                         # files this step produces (for the record)
      - out/equity.png

  - id: out_of_sample
    nine_step: step_6_out_of_sample
    required: true
    command: "python evaluate.py"
    inputs: []
    outputs:
      - out/oos_equity.png
    depends_on: [in_sample]          # runs after in_sample
```

Field by field:

| Field | What to put |
|-------|-------------|
| `id` | A unique name. **Must match the `pv.step("...")` id in your code.** |
| `nine_step` | One of the seven canonical keys (table below), or `null` for a step that doesn't fit any. |
| `required` | `true` for steps that must pass. A `required` step failing → exit 2. |
| `command` | The shell command that runs the step. Required for `data_collection`/`data_processing` steps. |
| `inputs` | **Leave it `[]`.** See the warning below — this is the #1 newbie trap. |
| `outputs` | Files the step produces. Informational; helps reviewers. |
| `depends_on` | Step ids that must run first. Sets the order. |

The seven canonical `nine_step` keys — **copy them exactly**:

| Key | Stage |
|-----|-------|
| `step_1_hypothesis` | Hypothesis |
| `step_2_data_collection` | Data collection |
| `step_3_data_processing` | Data processing |
| `step_4_in_sample` | In-sample backtest |
| `step_5_optimization` | Optimization |
| `step_6_out_of_sample` | Out-of-sample backtest |
| `step_7_paper_trading` | Paper trading |

> **⚠️ The `inputs` trap (the single most common newbie failure).** It's tempting to
> list your data files under `inputs:`. Don't. When `inputs` is non-empty it becomes a
> *complete allowlist*: ONLY the listed files are copied into the container — including,
> or rather *excluding*, your own script. If you write `inputs: [data/is/prices.csv]`,
> the verifier copies that one file and **not** `backtest.py`, so the step dies with
> `python: can't open file 'backtest.py': No such file or directory` and exits 2.
>
> **Always start with `inputs: []`**, which copies your whole repo (minus
> `.dockerignore` entries). Only tighten it later, file by file, once everything works.

### 6.6 `expected` — the numbers you promise

This is where you state what each step should produce. The verifier reads your step's
`results.json` and compares it against this block.

```yaml
expected:
  - step_id: in_sample               # must match a step id above
    metrics:
      - name: sharpe_ratio           # must match a metric name your code emits
        value: 1.42                  # the value you claim (and that your README reports)
        tolerance: {kind: relative, value: 0.05}   # "within 5%"
      - name: win_rate
        value: 0.54
        tolerance: {kind: absolute, value: 0.01}    # "within ±0.01"
    artifacts:
      - path: out/equity.png
        compare: visual_similarity   # compare the chart visually (non-blocking by default)

  - step_id: out_of_sample
    metrics:
      - name: sharpe_ratio
        value: 0.73
        tolerance: {kind: relative, value: 0.05}
```

How tolerances work — pick one `kind` per metric:

| `kind` | Passes when | Use for |
|--------|-------------|---------|
| `relative` | actual is within `value` × 100% of expected | most float metrics (Sharpe, returns) |
| `absolute` | actual is within ± `value` of expected | metrics near zero, or fixed precision |
| `exact` | actual equals expected exactly | integer counts only — avoid for floats |

> **Don't use `exact` on a float.** Floating-point math gives tiny differences across
> machines (`1.4200000001` vs `1.42`), so `exact` will fail for reasons that have
> nothing to do with your strategy. Use `relative` or `absolute` for any non-integer.

For artifacts, `compare` is one of:

- `visual_similarity` — compares charts by appearance (needs a baseline; non-blocking if
  no vision check is wired — see Step D and F).
- `json_numeric_tolerance` — compares JSON files, numbers within tolerance.
- `byte_exact` — files must be byte-for-byte identical (strict; use sparingly).

> A step listed in `steps` but absent from `expected` is still run — the verifier just
> checks it exits cleanly, without asserting specific numbers. That's fine for a step
> whose job is a side effect rather than a metric.

### 6.7 (Optional) `nine_step_coverage` — for a better score

You can document which of the nine steps your README covers, even ones that aren't
executable (like paper trading). It doesn't affect pass/fail, but it helps your
compliance score:

```yaml
nine_step_coverage:
  step_1_hypothesis:    {present: true,  section: "Hypothesis"}
  step_4_in_sample:     {present: true,  section: "In-sample Backtesting"}
  step_6_out_of_sample: {present: true,  section: "Out-of-sample Backtesting"}
  step_7_paper_trading: {present: false, section: null}
```

That's the whole manifest. Stitch the sections together and you have a complete,
valid `.plutus/manifest.yaml`.

---

## 7. Step D — Capture your baselines

Some comparisons need a "known good" copy to compare against — chart artifacts in
particular. `plutus snapshot` records the current outputs as the baseline:

```bash
plutus snapshot . --no-run
```

This copies your step outputs into `.plutus/expected/<step_id>/` and fills in the
metric values in your manifest from the latest run. (The `--no-run` flag is required in
the current version — it snapshots what's already there rather than re-running.)

> **Capture baselines from a *real* run, not a throwaway one.** If you snapshot a chart
> produced by a quick smoke test and then compare future runs against it, you're really
> just comparing a run against itself — the check passes but proves nothing. Run your
> pipeline for real first, *then* snapshot.

---

## 8. Step E — Run the check

The moment of truth:

```bash
plutus check .
```

(If your project uses secrets — e.g. a database — add `--secrets-from-env` so your
shell's environment variables are passed into the container.)

What happens: the verifier builds a Docker image from your `env`, runs each step in
dependency order inside it, collects each `results.json`, and compares everything to
your manifest. Then it exits with a code:

| Exit code | What it means | What to do |
|-----------|---------------|------------|
| **0** | 🎉 Everything reproduced. You're compliant. | Commit, add the badge, wire up CI (Step G). |
| **1** | Steps ran, but a number or chart didn't match your promise. | Check the `FAIL` lines — usually a tolerance or a stale `expected` value. |
| **2** | A required step crashed, or the manifest is invalid. | Read the error; jump to Step F. |

A green run prints something like:

```
ok in_sample: exit=0
    ok sharpe_ratio: actual=1.42 expected=1.42
    ok win_rate: actual=0.54 expected=0.54
    ok visual_similarity out/equity.png
ok out_of_sample: exit=0
    ok sharpe_ratio: actual=0.73 expected=0.73
```

No `FAIL` lines, exit `0`. That's a reproducible PLUTUS project.

---

## 9. Step F — When it's not green: fixing the common failures

Most first runs are not green. That's normal. Here are the failures you'll actually
hit, by symptom:

### "No such file or directory" / every step exits 2

You almost certainly put real paths in `inputs:`. Set **`inputs: []`** on every step
(see [6.5](#65-steps--your-pipeline-the-heart-of-the-manifest)). This is by far the most
common cause of a wall of exit-2 failures.

### `ValueError` mentioning `Decimal` when a step runs

A metric value is a `Decimal`. Wrap it in `float(...)` in your `pv.step` block (see
[5.3, Rule 1](#53-three-rules-that-trip-up-almost-everyone)).

### "additional properties are not allowed" / manifest won't load

A misspelled or unexpected key in the manifest. Check the key names against
[Step C](#6-step-c--write-the-manifest-key-by-key) — e.g. `step_4_insample` instead of
`step_4_in_sample`, or a stray field.

### `.env` parse error like `parse error near '\n'`

Your `.env.example` has an unquoted placeholder. Quote every placeholder value:

```bash
DB_HOST="<your_host>"      # ✅
DB_HOST=<your_host>        # ❌ the shell tries to read from a file called your_host
```

### A step that talks to a database fails to connect

Two things: (1) set `network: bridge` on that step so the container can reach the
database; (2) declare the credentials in `secrets[]` and route them with `used_by`, then
run with `--secrets-from-env`. If a module opens a DB connection *at import time*, every
step that imports it needs `network: bridge` too.

### A metric `FAIL`: `actual=1.43 expected=1.42`

Your result drifted from the promised value. Either your tolerance is too tight (widen
`relative`/`absolute`), or the `expected` value is stale (update it — and update your
README table to match, they must agree), or your code genuinely changed. Don't "fix" it
by loosening tolerance to hide a real regression.

### A chart shows `SKIP` or `WARN` instead of `ok`

`visual_similarity` comparisons are **non-blocking** unless you explicitly run with a
vision check wired up. `SKIP` (no baseline yet) and `WARN` (bytes differ, inconclusive)
do **not** fail the run — they won't stop you reaching exit 0. Capture a baseline with
`plutus snapshot` ([Step D](#7-step-d--capture-your-baselines)) to turn `SKIP` into a
real comparison.

> The full troubleshooting catalogue, with every known gotcha and its version history,
> is in the [reference §7](REFERENCE.md). The ones above cover the overwhelming majority
> of first-time failures.

---

## 10. Step G — Keep it green with CI

A green check today is worth more if it stays green. Add a CI workflow that runs
`plutus check` on every push. `plutus init` already scaffolded one at
`.github/workflows/plutus.yml`; if you skipped that, a minimal version is:

```yaml
name: plutus
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install plutus-verify
      - run: plutus check .
```

Also commit a `.python-version` file (or a `python_version` in `pyproject.toml`) so
local and CI environments resolve the same interpreter. Both of these also earn you
points in the **Tidy** scoring bucket.

---

## 11. Where to go next

You now have a compliant project. From here:

- **Understand your score.** Compliance is graded on four buckets (Reproducible 50,
  Tidy 25, Standardized 10, Innovative 15). The [README](README.md#how-scoring-and-badges-work)
  explains the badge tiers; run the `plutus-scoring` tool for a breakdown and the
  cheapest ways to improve.
- **Polish your README.** The Tidy bucket scores your documentation — see the
  [README's documentation section](README.md#what-your-readme-should-contain).
- **Graduate your data tier.** Once green with committed data, move large data to a
  remote source or a live database if your project needs it ([reference §3](REFERENCE.md)).
- **Look at a real example.** The canonical V2 reference repo,
  [Group09-BuyHighSellLow](https://github.com/algotrade-education/Group09-BuyHighSellLow),
  is a fully instrumented project you can copy patterns from.

---

<!--
  ───────────────────────────────────────────────────────────────────────────
  DRAFT NOTE (remove before publishing) — for Dân, not end readers

  This is the new step-by-step authoring guide you asked for: a newbie-friendly
  walkthrough from empty repo → green `plutus check`, covering both the source
  instrumentation (Step B) and the manifest, key by key (Step C), plus a
  symptom-based troubleshooting section (Step F).

  How it fits the planned three-document set:
    • README.md      → the front door (why + overview)        [README.draft.md so far]
    • GUIDE.md       → THIS doc: how-to, hold-your-hand tutorial
    • REFERENCE.md   → the technical reference (every option + API)
                       ← to be revamped from the current README.md (the 1209-line
                         spec) and renamed. We agreed to do the guide FIRST; the
                         reference revamp/rename is the next task.

  Cross-links in this guide point at README.md and REFERENCE.md by their FINAL
  names. They won't all resolve until the rename pass is done — that's expected
  and intentional, so no rework is needed once we wire the trio together.

  Running example is deliberately Tier 1 (committed CSV) — the simplest path to
  green. Tiers 2–4 are pointed at the reference rather than taught here, to keep
  a first read short. Tell me if you'd rather the guide also walk a DB-backed
  (Tier 3) example end to end.
  ───────────────────────────────────────────────────────────────────────────
-->
