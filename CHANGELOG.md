# CHANGELOG

All notable changes to the Plutus Reproducibility Standard are documented here.
Newest entries first.

---

## Version table

| Standard version | `plutus-verify` | Manifest `schema_version` | Results `schema_version` | Status |
|------------------|-----------------|---------------------------|--------------------------|--------|
| v2 (2026-06) | ≥ 0.2.10 | `"2.0"` | `"1.0"` | current |
| v1 | n/a (prose standard) | n/a | n/a | superseded (archived at [archive/v1/](archive/v1/)) |

---

## v2 (2026-06)

**V2 supersedes V1.**

### Normative changes

- **Compliance redefined via two machine contracts.** Compliance is now determined by two
  machine-readable contracts — `.plutus/manifest.yaml` (`schema_version: "2.0"`) and
  `.plutus/run/<step_id>/results.json` (`schema_version: "1.0"`) — verified by `plutus check`
  exit codes (0 = pass, non-zero = fail). The V1 human-judged prose checklist is retired as a
  normative gate.

- **Scoring redefined as four weighted buckets.** The single percentage score is replaced by
  four weighted buckets summed and rounded to the nearest 5%:
  - **Reproducible** (50 pts)
  - **Tidy / well-documented** (25 pts)
  - **Standardized / template** (10 pts)
  - **Innovative** (15 pts)

  Badge tiers (percentage thresholds and colours) are unchanged from V1.

- **README template carried into §6.** The V1 README section template (V1 §1) carries over
  unchanged as §6, reframed as the evidence the Tidy bucket scores rather than the definition
  of compliance.

- **Data tiers replace the single Drive note.** A four-tier data-access model (Tier 1 committed data / Tier 2 Drive or GitHub release / Tier 3 database / Tier 4 layered) replaces the single Google Drive reference from V1 (§3).

- **`templates/manifest.yaml` added.** A commented manifest skeleton is provided at
  `templates/manifest.yaml` for bootstrapping new projects.

### Repository moves

- V1 README archived verbatim to `archive/v1/README.md`.
- `assessment/` (assessment guide + report forms) moved to `archive/v1/assessment/`.

### Badge continuity

The canonical v2 reference project
([Group09-BuyHighSellLow](https://github.com/algotrade-education/Group09-BuyHighSellLow))
was re-scored under the v2 buckets on 2026-06-11, against `plutus-verify` 0.2.10:

| Bucket | Score | Reasoning |
|--------|-------|-----------|
| Reproducible | 50/50 | `plutus check . --secrets-from-env` exits 0 cleanly — all 5 steps green, 30 metric comparisons and 11 artifact comparisons `ok`, no manifest-side workarounds, Tier 3 DB step passes via real connectivity |
| Tidy / well-documented | 22/25 | README metric tables match `expected` exactly; `.env.example` parses cleanly; data inputs and parameter pipeline fully documented; deductions: CI does not run `plutus check`, and three backtest steps declare `outputs: []` while producing report files |
| Standardized / template | 10/10 | Canonical step shape, parameters externalized to `config/strategy_params/*.json`, predictable `reports/<label>/plots/` output paths, no module-level side effects |
| Innovative | 8/15 | Thoughtful strategy logic (adaptive-volatility regimes, dual-session ranges) and extended diagnostics (MAE/MFE, exit-reason histogram, rolling Sharpe), but the headline metric surface is conventional |

**Total: 90 pts → 90% → Gold**, computed exactly as §5 defines (sum of buckets, rounded
to the nearest 5%, mapped to the kept V1 badge tiers). This confirms badge continuity: the
v2 machine-derived score lands in the same Gold band that comparable v1 sample projects
(70–80%) occupied, and the §5.3 arithmetic (a non-reproducing repo caps at 50%) holds in
practice — this repo's score would drop to 40% without its green `plutus check`.
