# The Plutus Reproducibility Standard

> The canonical home of the Plutus Reproducibility Standard — the standard text plus the
> guide, reference, templates, and version archive that support it.

A trading-strategy result is only worth as much as your ability to get the same numbers
again. The Plutus Reproducibility Standard makes that property *mechanical*: a compliant
project re-runs end-to-end with one command, `plutus check`, in a clean container, and
confirms every reported number matches what the project claimed. This repository holds
the standard itself and everything you need to apply it.

## Start here

Pick the door that matches what you're doing:

| If you want to… | Read |
|-----------------|------|
| Understand the *why* and the rules | [STANDARD.md](STANDARD.md) |
| Make a repository compliant, step by step | [GUIDE.md](GUIDE.md) |
| Look up an exact field, exit code, or API | [REFERENCE.md](REFERENCE.md) |
| Write your project's README | [templates/README.template.md](templates/README.template.md) |
| Start a manifest | [templates/manifest.yaml](templates/manifest.yaml) |

New to the standard? Read [STANDARD.md](STANDARD.md) first — it stands on its own.

## What's in this repo

- [STANDARD.md](STANDARD.md) — the standard: the philosophy, the two pillars (the 9-step
  process and this guideline), what "compliant" means, scoring, and badges.
- [GUIDE.md](GUIDE.md) — a hands-on walkthrough from an empty repo to a green
  `plutus check`, one step at a time.
- [REFERENCE.md](REFERENCE.md) — the normative contract: every manifest and results
  field, the comparison semantics, exit codes, the CLI, and the SDK API.
- [CHANGELOG.md](CHANGELOG.md) — version history and the schema-version correspondence.
- [templates/](templates/) — a starter `manifest.yaml` and a `README.template.md` you
  copy and fill in for your own project.
- [archive/](archive/) — frozen snapshots of past versions of the standard.

## The toolkit

The standard is tool-agnostic about *how* you work: compliance is defined by two
artifacts — a manifest and a results contract — that you can produce any way you like.
([STANDARD.md](STANDARD.md) explains what they are; [GUIDE.md](GUIDE.md) walks through
producing them.) Those artifacts are then verified by
[plutus-verify](https://github.com/algotrade-plutus/plutus-verify), which rebuilds the
project in a clean container and confirms the numbers reproduce. The same project also
ships Claude Code skills that help you produce compliant artifacts in the first place:
`plutus-standardize` (bring a repo up to the standard), `plutus-scoring` (score it), and
`plutus-document` (generate the project README from verified results).

## Versioning

This repo tracks Version 2 of the standard. Past versions are preserved, read-only, under
[archive/](archive/); projects badged under an older release keep their badges until
re-verified. See [CHANGELOG.md](CHANGELOG.md) for the full history.
