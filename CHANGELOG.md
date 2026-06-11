# CHANGELOG

All notable changes to the PLUTUS Standard are documented here.
Newest entries first.

---

## Version table

| Standard version | `plutus-verify` | Manifest `schema_version` | Results `schema_version` | Status |
|------------------|-----------------|---------------------------|--------------------------|--------|
| v2 (2026-06) | ≥ 0.2.10 | `"2.0"` | `"1.0"` | current |
| v1 | n/a (prose standard) | n/a | n/a | superseded (archived at [versions/v1/](versions/v1/)) |

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

- **README template carried into §6.** The prose structure previously implicit in V1 example
  projects is now a normative section (§6) with explicit heading requirements and a copyable
  template.

- **Data tiers replace the single Drive note.** A four-tier data-access model (Tier 1 committed data / Tier 2 Drive or GitHub release / Tier 3 database / Tier 4 layered) replaces the single Google Drive reference from V1 (§3).

- **`templates/manifest.yaml` added.** A commented manifest skeleton is provided at
  `templates/manifest.yaml` for bootstrapping new projects.

### Repository moves

- V1 README archived verbatim to `versions/v1/README.md`.
- `assessment/` (assessment guide + report forms) moved to `versions/v1/assessment/`.

### Badge continuity

<pending — canonical example re-score recorded here by the verification pass>
