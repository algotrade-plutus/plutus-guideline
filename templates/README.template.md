> *Guidance — how to use this template.* This page is both a *template* you copy to your
> project root as `README.md` and a *guide* to filling it. Work top to bottom: each section
> explains what to write and leaves bracketed `[…]` slots you replace; the italic *Guidance*
> notes add conventions and gotchas.
>
> - *Replace every bracketed slot with your content.*
> - *Keep the section order — it follows the [9-step Development Process](https://www.algotrade.vn/knowledge/9-step-process/the-9-step)
>   and what `plutus check` verifies, so a hand-written README lines up with what the
>   `plutus-document` tool would generate from your manifest.*
> - *Drop any section that doesn't apply (e.g. no optimization step → omit §5) and renumber
>   the sections that follow.*
>
> *Sections fall into two kinds:*
> - *You write these (narrative): Abstract, Introduction, Hypothesis, Rules, References.*
> - *These come from your manifest and committed results (and the `plutus-document` tool can
>   fill them): the data Source/Period block, the metric tables, the chart embeds, and the
>   reproduction commands — whatever you write by hand must match the committed baseline.*
>
> *A Plutus score or project-type badge is granted by the verification and scoring process
> and added separately — you don't author it, so this template doesn't include one.*
>
> **Delete this Guidance block — and every other *Guidance* note below — once the README is
> completely filled in.**

---

# [Your Strategy Name]

> [One sentence: the instrument and the core mechanic — what the strategy does, plainly.]

> *Guidance — The title is your project name. The blockquote under it is a one-line summary
> a reader grasps in five seconds (for example: "Moving-average crossover on VN30 futures —
> go long when the fast average crosses above the slow one").*

## Abstract

[Two to four sentences, in plain language: what the strategy does and how it works.] Then
one sentence tying it to the data and the standard: developed and evaluated end-to-end on
[data source + period] following the
[9-step Development Process](https://www.algotrade.vn/knowledge/9-step-process/the-9-step)
and the [Plutus Reproducibility Standard](https://github.com/algotrade-plutus/plutus-guideline).
On the in-sample period it reaches [headline in-sample metrics]; on the out-of-sample
period it reaches [headline out-of-sample metrics]. Every reported number is reproducible
in an isolated Docker container via `plutus-verify` against a committed groundtruth
baseline (see [Implementation & Reproducibility](#implementation--reproducibility)).

> *Guidance — The most-read section. Lead with what the strategy does, then name the data
> source and period and your headline numbers (Sharpe + holding-period return is a good
> pair). Keep the two links and the reproducibility claim. The numbers here must match your
> §4/§6 tables and the manifest's `expected` block.*

## Introduction

[A paragraph or two: the problem the strategy addresses, the core idea behind it, and the
reference(s) it builds on. This is context for a reader new to the strategy.]

## 1. Forming Algorithm Hypothesis

[State the hypothesis: the specific market behaviour you expect to exploit and why you
expect it to hold. Make it concrete enough to be testable.]

> *Guidance — Where the signal, sizing, or entry/exit logic is clearer as a formula,
> include it and define every symbol; math renders between `$$ … $$`. Keep formulas in
> service of understanding — a reader should finish this section knowing exactly what the
> strategy decides and on what basis.*

## 2. Data Preparation

- **Source:** [data provider and instrument].
- **Period:** [in-sample and out-of-sample date ranges].
- **Fees:** [the fee/slippage model your backtest assumes, if any].

[One or two sentences on what is collected, how, and where it lands in the repo — for
example under `data/is/` (in-sample) and `data/os/` (out-of-sample).]

> *Guidance — Source / Period / Fees are the facts `plutus check` resolves data against;
> keep them accurate to your manifest's `data_sources`. The bold labels are deliberate —
> short, scannable field names earn the emphasis.*

### Obtaining the data

[Give a credential-free path first, so anyone can reproduce downstream steps without
private access.]

**Option 1 — Download from [Google Drive / release / URL] (no credentials needed).**
Download from [your link] and place the `data/` folder at the repository root so the
pipeline resolves `data/is/...` and `data/os/...`:

```
[render your actual data layout as a tree, e.g.:]
data
├── is
│   └── [your_dataset].csv
└── os
    └── [your_dataset].csv
```

**Option 2 — Collect from the source.** [Note the credentials this needs — see
[Environment setup](#environment-setup).] Then run:

```bash
[your data-preparation command]
```

> *Guidance — Omit whichever option doesn't apply. If your data is small and committed to
> the repo, this whole subsection can be a single sentence pointing at the committed path.*

## 3. Forming Set of Rules

[List the concrete, executable rules your backtest applies — unambiguous enough that a
reader could re-implement them. Cover the ones that apply to your strategy:]

- **Entry:** [the condition that opens a position].
- **Exit:** [the condition that closes it].
- **Sizing:** [how position size is decided].
- **Cadence:** [when the strategy evaluates and acts].
- **Costs:** [how fees and slippage enter the P&L].

### Evaluation Metrics

[List the metrics you report — these are the display names of your manifest's
`expected.metrics`, and the only ones `plutus check` verifies.] Common choices are the
Sharpe ratio, the Sortino ratio, maximum drawdown, and holding-period return.

> *Guidance — State any benchmark or risk-free rate the figures assume. If you show a
> display-only metric that isn't in `expected`, label it as such so readers know it isn't
> machine-verified.*

## Implementation & Reproducibility

The pipeline is packaged as `[your_package]` with console-script entry points
([your step commands]); each step below shows its own command.

> *Guidance — One sentence naming the package and its entry points, taken from your
> manifest's `steps[].command`.*

### Environment setup

```bash
uv sync     # create the env from the committed lockfile
```

[One line on the lockfile / how dependencies are pinned.]

[Include the block below only if your manifest declares `secrets`. List exactly the keys in
`secrets[].key`, each with a quoted placeholder so `source .env` doesn't break on the `<`.
Delete it entirely if your project needs no secrets.]

```env
[KEY_ONE]="<...>"
[KEY_TWO]="<...>"
```

### Reproducibility

This repo ships a `.plutus/manifest.yaml` declaring the environment, data sources, steps,
and expected metrics. Reproduce every result in an isolated Docker container with
[plutus-verify](https://github.com/algotrade-plutus/plutus-verify) **[pin the version you
blessed against]**, installed from the public release wheel:

```bash
# Requires Docker running. `uv venv` provisions a suitable Python (>= 3.11) automatically.
uv venv .plutus-venv && source .plutus-venv/bin/activate
uv pip install "plutus-verify[runner] @ [release wheel URL for the pinned version]"

plutus check .     # build -> run each step in-container -> compare vs baseline
```

`plutus check` builds the image, resolves the declared data sources, installs this package,
runs each step in-container, then compares the produced metrics and charts against the
committed baseline in `.plutus/expected/`. Exit code `0` = reproduced (within tolerance),
`1` = partial, `2` = failed.

> *Guidance — Pin the exact `plutus-verify` version and wheel URL you blessed against, and
> keep the exit-code legend. This block is what turns the README's numbers from a screenshot
> into a verifiable claim.*

## 4. In-sample Backtesting

[One line on the parameter/config file the step reads.] Run:

```bash
[your in-sample command]
```

[Note where the charts are written, e.g. `result/backtest/`.]

### In-sample result ([in-sample period])

[Report one row per `expected.metrics` entry. Every value must be identical to the
committed baseline and to the headline numbers in your Abstract.]

| Metric | Value |
|--------|-------|
| [metric display name] | [value from the baseline] |
| [...one row per metric] | [...] |

[Then embed each chart the step produces, with a caption naming what it shows:]

[what the chart shows] — `[path to the committed chart]`

![[short alt text]]([path to the committed chart])

> *Guidance — The table and charts are groundtruth: `plutus-document` can fill them from the
> baseline. If you write them by hand, they must match it exactly — a mismatch is a
> documentation failure even when `plutus check` is green.*

## 5. Optimization

[One line on the search space / config file and the fixed random seed that makes the search
reproducible.] Run:

```bash
[your optimization command]
```

The optimized parameters are written to [the parameter file the step produces]:

```json
{
  "[parameter name]": "[optimized value]"
}
```

> *Guidance — A fixed seed is what makes optimization reproducible — state it. Omit this
> whole section if your project has no optimization step, and renumber §6.*

## 6. Out-of-sample Backtesting

Using the optimized parameters from Step 5, evaluate on the out-of-sample period:

```bash
[your out-of-sample command]
```

### Out-of-sample result ([out-of-sample period])

[Same format as §4: one row per `expected.metrics` entry, values matching the baseline,
followed by the chart embeds.]

| Metric | Value |
|--------|-------|
| [metric display name] | [value from the baseline] |
| [...one row per metric] | [...] |

[what the chart shows] — `[path to the committed chart]`

![[short alt text]]([path to the committed chart])

> *Guidance — Out-of-sample numbers are the honest test of the strategy. Report them as
> they are, even when they're weaker than in-sample — that candour is part of what the
> standard rewards.*

## Reference

[Numbered citations for the sources your hypothesis and method build on.]

[1] [Author, *Title*, edition. Publisher, year. Online: link]
