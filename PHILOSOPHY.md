# The Nine-Step Process

The concept behind the standard. This document defines the methodology a compliant
project is expected to follow; [STANDARD.md](STANDARD.md) defines how a repository is
packaged so that adherence can be checked mechanically.

The nine-step process is the *what*. The standard is the *how*.

## 1. Scientific principles

Developing an automated trading strategy that is both profitable and durable is hard,
and most ideas that look strong in theory do not survive contact with live markets.
Developing one is an iterative cycle: define a process, execute it, observe the outcome,
analyse it, adjust. For that cycle to converge on something dependable rather than
merely lucky, three principles govern it.

These principles are the reason the standard exists, and the standard is a mechanism for
enforcing them. Each has a direct counterpart in the manifest.

### Transparency

Every element of the process — input data, decision criteria, individual transactions,
and resulting performance — is recorded completely, clearly, and accurately.

The failure mode is the black box. A model or signal generator whose inputs and internal
logic are not understood cannot be debugged when performance decays: there is no way to
tell whether it is responding to signal, to noise, or to a relationship that has expired.
A process can look sophisticated and still be unimprovable, because nothing about it can
be questioned. This matters increasingly as learned models enter the pipeline, and it is
why the standard confines automated assistance to *authoring* a project and keeps it off
the verification path entirely — a verdict produced by an opaque judge would reintroduce
precisely the problem transparency exists to prevent.

Transparency is what makes the decision chain auditable, and it is the **foundation of
reproducibility**: without a complete record, a result cannot be re-derived, and a result
that cannot be re-derived cannot be verified by anyone. This is the principle that
separates systematic investing from speculation.

- **In the standard:** the manifest. A single committed declaration of environment, data
  sources, executable steps, and promised metrics — the complete record, in machine-
  readable form.

### Falsifiability

The process is constructed so that its effectiveness can be tested objectively against
real data. There must exist evidence that would show it does not work.

Consider decisions made for varied and unrecorded reasons — taking profit after a rise,
acting on a tip, reallocating on impulse. Because no decision is linked to a criterion or
to its outcome, no method can be evaluated: a gain cannot be distinguished from luck, and
a loss yields no diagnosis. Learning becomes anecdotal rather than empirical.

- **In the standard:** the promised metric values in the `expected` block, checked
  against produced output within a declared tolerance. Committing to a number in advance
  is what makes a comparison capable of failing.

### Consistency

The process is applied uniformly, across enough repetitions to produce statistically
meaningful evidence. A process executed rarely or unevenly yields results that describe
the circumstances rather than the method. Consistency also demands faithful execution:
the orders placed must match the decisions made.

Prefer the record built from ten thousand trades at a modest return over the one built
from a single trade at a spectacular one. Only the first is evidence.

If transparency is the foundation of reproducibility, consistency is its **substance** —
the property that makes a result hold on re-execution rather than only once.

- **In the standard:** deterministic containerized re-execution. The same pinned
  environment, the same steps, in the same order, every time.

### What the principles do and do not promise

Adherence does not guarantee returns. The principles govern the construction of the
process, not the profitability of the strategy it produces — a rigorous process can
yield a negative result, and reporting that plainly is a legitimate outcome.

They do determine whether anything can be learned. Investment approaches sit on a
spectrum by how far they comply: at one end, decisions rest on judgement that varies
case by case and cannot be evaluated; at the other, a predefined rule set is applied
uniformly and assessed quantitatively. Automated trading is the limit of the second,
and it is the only regime in which the three principles can be enforced mechanically
rather than promised.

> Process, not profit, is the leading indicator of durable results.

## 2. The nine steps

Each step names an **input**, an **output**, and a **validation criterion** — the
condition that must hold before the next step is entered. A completed study is therefore
a sequence of auditable artifacts, not a single reported number.

Steps 1–6 are the reproducible core. Steps 7–9 progressively introduce live market
conditions and real capital; their reproducibility properties are covered in §3.

### Step 1 — Forming the algorithm hypothesis

A candidate strategy is stated as an economic or behavioural claim about the market:
why an edge should exist and under what conditions. The statement is permitted to be
imprecise about parameters at this stage, but it must be specific enough to be wrong.

- **Input:** market observation, prior research, or a structural assumption.
- **Output:** a written hypothesis, with any initial parameters that are already known.
- **Validation:** the hypothesis is falsifiable — it names conditions under which it
  would be rejected.

### Step 2 — Data preparation

Collecting, organizing, cleaning, and validating the inputs the strategy requires. For
most work, price series and fundamentals suffice; alternative datasets may be used where
the hypothesis calls for them. Cleaning — detecting and correcting missing values,
duplicates, and anomalies — is part of this step, not a preliminary to it, because
unclean data produces results that are misleading rather than merely noisy.

This step covers both collection and processing. The standard treats it as a single
canonical step (`step_2_data_preparation`) and allows the two activities to be
documented separately.

- **Input:** raw sources, with their provenance and access method recorded.
- **Output:** a validated, consistently formatted dataset, and the code that produced
  it from the raw sources.
- **Validation:** the dataset is complete and internally consistent over the intended
  period, and the transformation from raw to prepared is re-executable.

### Step 3 — Transforming the hypothesis into a set of rules

The hypothesis becomes an explicit, executable rule set: the logic that turns market
state into investment decisions, and decisions into orders. Decision logic yields target
positions; execution logic yields concrete orders with type, price, and size.

Proceeding directly from here to live trading is possible and inadvisable — without the
testing phases that follow, the result is indistinguishable from chance.

- **Input:** the hypothesis and the prepared dataset.
- **Output:** an implemented rule set with its parameters exposed rather than embedded.
- **Validation:** the rules express the hypothesis faithfully, and every free parameter
  is identified as such.

### Step 4 — In-sample backtesting

The dataset is partitioned chronologically into an in-sample window used for fitting and
an out-of-sample window reserved for evaluation — commonly around 70/30, though the
split is a declared choice, not a fixed rule. The rule set is evaluated on the in-sample
window to establish a reasonable expectation of behaviour and risk.

Realistic accounting is required: fees, taxes, slippage, and any other trading cost must
be included, or the result overstates performance by construction. A sample too small to
support inference — conventionally on the order of thirty transactions — cannot
establish a performance profile regardless of the metrics computed from it.

- **Input:** the rule set and the in-sample window.
- **Output:** in-sample performance metrics, at minimum a return measure, a
  risk-adjusted measure, and a drawdown measure.
- **Validation:** costs are modelled, the transaction count supports inference, and the
  out-of-sample window has not been touched. If performance is unsatisfactory, return
  to Step 1 with a revised hypothesis rather than adjusting parameters here.

### Step 5 — Optimization

Parameters are searched on the **in-sample window only**, seeking a stable region rather
than a maximum. A configuration that is optimal at a point but degrades sharply under
small perturbation is an artifact of the sample, not a property of the strategy.

Overfitting is the principal hazard of this step. Searching enough configurations will
produce a strong in-sample record by chance alone, and a result selected that way does
not transfer. Reporting an optimum discovered by exhaustive search, without disclosing
the search, misrepresents the evidence.

- **Input:** the in-sample window and a declared parameter space.
- **Output:** exactly one frozen configuration, plus the record of the search that
  produced it.
- **Validation:** the search touched no out-of-sample data, the chosen configuration
  lies in a stable region, and the number of configurations evaluated is disclosed —
  it is required to interpret the result that follows.

### Step 6 — Out-of-sample backtesting

The frozen configuration is evaluated once on the held-out window. This is the study's
headline claim and the first genuine test of the hypothesis: everything before it was
fitting.

Re-entering optimization after seeing these results converts the out-of-sample window
into in-sample data and invalidates the test. If the result is unsatisfactory, that is
a finding — report it.

- **Input:** the frozen configuration and the out-of-sample window.
- **Output:** out-of-sample performance metrics, comparable to the Step 4 measures.
- **Validation:** the configuration was unchanged from Step 5, the window was untouched
  before this step, and the degradation from in-sample to out-of-sample is reported
  rather than suppressed.

### Step 7 — Paper trading

The strategy runs forward against live market data with simulated capital, on data that
did not exist when the rule set was frozen. This tests the implementation as much as the
strategy: order handling, data feed behaviour, latency, and failure modes that
historical simulation does not exercise.

- **Input:** the frozen configuration and a live data feed.
- **Output:** a forward performance record and an operational log.
- **Validation:** performance is consistent with Step 6, and the system runs without
  intervention over the evaluation period. Inconsistency here indicates either an
  implementation defect or an unrealistic backtest — both require diagnosis before
  capital is committed.

### Step 8 — Small-account testing

The strategy is deployed with real capital at deliberately small size. This is the first
stage exposed to real execution: actual fills, actual slippage, actual market impact.
Its purpose is to compare live behaviour against the paper-trading record and to surface
technical problems that only appear when orders reach a venue.

The period must be long enough to accumulate a meaningful number of transactions and to
observe stability — on the order of two months in typical use.

- **Input:** the frozen configuration and a small allocation of real capital.
- **Output:** a live performance record at small size, and a reconciliation against
  Step 7.
- **Validation:** live results are consistent with the paper-trading record, and no
  unresolved operational defects remain.

### Step 9 — Real trading

Capital is scaled up. Close supervision is required initially, and technical faults are
expected to surface and be corrected. Only after a sustained period of stable
operation — on the order of three months — is the strategy treated as an established
production system rather than one under evaluation.

- **Input:** the validated strategy and a full allocation.
- **Output:** an ongoing production performance record.
- **Validation:** operation is stable and performance remains consistent with the
  evidence accumulated in Steps 6–8. Persistent divergence is grounds for reduction or
  withdrawal, not for re-optimization against live results.

## 3. What the standard verifies

The standard mechanizes this process only where mechanization is meaningful. The
boundary follows from the nature of each step, not from implementation convenience.

**Steps 1–6 — fully reproducible.** Every input is historical and every step is a
deterministic function of disclosed data and code. Given the repository, an independent
party can rebuild the environment, re-execute each step, and obtain the same numbers.
This is the range over which `plutus check` compares produced metrics against promised
ones. Reproducibility through Step 6 is the minimum for a compliant project.

**Step 7 — reproducible code, non-reproducible results.** Paper trading depends on
market data that arrives after the strategy is frozen, so its *results* cannot be
re-derived: a re-run encounters a different market. What can be verified is that the
paper-trading implementation is present and executable — that the code runs in the
paper-trading environment. The standard therefore treats Step 7 as a coverage and
executability claim rather than a metric-comparison claim, supported by
`verification_mode: artifact_check` and by the `nine_step_coverage` declaration. Step 7
is the ideal upper bound of verification, not part of the reproducible core.

**Steps 8–9 — out of scope.** Small-account testing and real trading commit real capital
under live market conditions. They cannot be re-executed on demand at all: the market
state, the fills, and the impact are unrepeatable events. Excluding them is a statement
about what reproducibility can mean, not a gap in the tooling.

This boundary is deliberately narrow, and stating it precisely is part of the claim. A
green verdict asserts computational reproducibility over Steps 1–6 and executability at
Step 7. It asserts nothing about whether a strategy is profitable, statistically sound,
or free of methodological error — a flawed study reproduces its flaws faithfully.
Judging whether a reproducible result is also a good one remains the work of review.

## 4. Mapping to the manifest

The seven canonical `nine_step` keys used in `.plutus/manifest.yaml` correspond to the
steps above. The exact strings and their constraints are specified in
[REFERENCE.md §3](REFERENCE.md); the mapping is:

| Step | `nine_step` key |
|------|-----------------|
| 1 — Forming the algorithm hypothesis | `step_1_hypothesis` |
| 2 — Data preparation | `step_2_data_preparation` |
| 3 — Transforming the hypothesis into rules | `step_3_forming_set_of_rules` |
| 4 — In-sample backtesting | `step_4_in_sample` |
| 5 — Optimization | `step_5_optimization` |
| 6 — Out-of-sample backtesting | `step_6_out_of_sample` |
| 7 — Paper trading | `step_7_paper_trading` |
| 8 — Small-account testing | *(out of scope)* |
| 9 — Real trading | *(out of scope)* |

The standard fixes the key strings, field names, and enums — not the number of steps or
their content. A project may declare additional steps with `nine_step: null` and a
`label`.
