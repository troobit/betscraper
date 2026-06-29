# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build / Test / Lint

The implementation is a single installable Python package, `betscraper/`, run from the VS Code
integrated terminal. No external database or services — SQLite only.

- Install: `pip install -e .` (Python 3.11+, pinned dependencies only).
- Run (CLI, Typer): `python -m betscraper <command>` — `init-db`, `scrape [--sport] [--loop]`,
  `arb [--stake]`, `serve`. The SQLite DB is created and seeded on first run.
- Test: `pytest tests/test_arbitrage.py` — only the arbitrage margin/stake math is unit-tested,
  a deliberate carve-out (error logging is the reliability mechanism everywhere else — see the
  spec Non-Goals). Property tests use Hypothesis.
- Lint / format: `ruff check .` and `ruff format .`.

> **Not yet implemented.** No `betscraper/` package exists yet — the commands above are the ones
> the design (`specs/data/cross-book-arbitrage/design.md`) targets. Replace them with the real
> invocations as the code lands.

## Conventions

- Local-first, single-user: no Postgres/Redis/Celery, no auth, no containers or deploy CI (see
  the spec Non-Goals). A single SQLite file is the only datastore.
- Money math uses `Decimal` for margins and integer cents for stakes — never float — so a
  borderline `margin < 1` is never misclassified by float drift, and rounding never turns a leg
  into a loss.
- Scrapers stay thin (fetch + parse → `ScrapedOdds`); all normalization, matching, and arbitrage
  live in the shared pipeline and engine. Adding a bookmaker is one fetch+parse method, no change
  to core logic.
- British/Irish English spelling (`tools/check_spelling.sh` and the `spelling` workflow). Exempt
  unavoidable US spellings in external identifiers with a `spelling-ignore` comment on that line.

## Stack preferences

- **IaC:** OpenTofu (`tofu`), not Terraform.
- **Cloud:** Azure over AWS where cloud hosting is required.
- **AI / compute:** prefer local and FOSS solutions over paid API or hosted-ML calls unless
  there's a clear reason the local option won't do.

## Spec-driven workflow

The full process is `specs/PROCESS.md` — read it before creating or changing a spec.

Feature work lives under `specs/<domain>/<capability>/`. The domain is one of the project's fixed
set (`specs/DECISIONS.md` MD-1: `platform`, `data`, `api`, `ui`, `ops`). The first and current
spec is `specs/data/cross-book-arbitrage/` — the project MVP, named for the capability it
delivers. Name future spec folders for the capability, never for the layer, the effort/phase
(`mvp`, `v2`), or the word `feature`.

Root `nextup.md` is the universal entry point: run `/nextup` at the start of a session to pick up
where you left off. `nextup.example.md` is the tracked seed; the live `nextup.md` is gitignored.
