# Decision Log: Betscraper (Cross-Book Arbitrage Odds Scraper)

This is the repository-level **meta decision log**. It distils the load-bearing,
cross-cutting decisions that define the project from the per-spec decision logs under
`specs/<domain>/<capability>/decision_log.md`, and organises them by theme rather than by
feature.

It is a synthesis, not a replacement: each meta-decision (MD) cites the source per-spec
decisions that establish or refine it. When a meta-decision and a source decision disagree,
**the per-spec decision log is authoritative** — this file is a rolling summary and may lag the
detail. For the per-decision format see `rules/references/decision-log-format.md`.

For the spec index see [OVERVIEW.md](OVERVIEW.md). For how specs are developed see
[PROCESS.md](PROCESS.md).

---

## MD-1: Project domain set

**Date**: 2026-06-29
**Status**: accepted

### Context

Every spec lives at `specs/<domain>/<capability>/`, and the domain set is closed and
project-specific ([PROCESS.md §3](PROCESS.md#3-directory-structure-spec-boundaries-and-naming)).
Before writing any spec, the project must fix the set of domains its specs may use, so that
naming and placement are unambiguous and adding a domain later is a deliberate, logged act
rather than drift.

### Decision

This project's domains are:

| Domain | Owns |
|---|---|
| `platform` | app/runtime shell, build, packaging, local run path, runtime bindings |
| `data` | persistence, schemas, data models, migrations, data ingestion |
| `api` | programmatic interfaces and external integrations (HTTP, scraping, third-party I/O) |
| `ui` | user-facing surfaces, navigation, interaction, visual design (CLI, web dashboard) |
| `ops` | infrastructure, CI/CD, observability, runbooks |

The set is **closed**: adding, renaming, or removing a domain amends this decision.

### Rationale

The generic starter set fits this project unchanged: it ingests odds data and derives arbitrage
from it (`data`), scrapes external bookmaker sites and exposes a REST API (`api`), presents a
CLI and a read-only web dashboard (`ui`), packages and runs locally on SQLite (`platform`), and
has minimal ops. Domains are *layers of concern*, not features, so a small fixed set keeps each
capability's home obvious and resists inventing a domain per scraper or surface.

### Alternatives Considered

- **Flat `specs/<capability>/` (no domain layer)**: Simpler for a small repo - Rejected because
  the template mandates the domain layer up front so structure is not retrofitted once the
  project grows (this is exactly the drift being corrected — see `cross-book-arbitrage` D17).
- **A dedicated `arbitrage`/`core` domain for the engine**: More precise placement for the
  central logic - Rejected as premature; the engine's acceptance criteria are dominantly about
  ingested odds data, so `data` owns it (PROCESS.md §3). A new domain is added only when a
  genuine new layer of concern appears (PROCESS.md §10).

### Consequences

**Positive:**
- Every spec has an unambiguous home from day one.
- Adding a domain is a visible, reviewable change to this log.

**Negative:**
- A single-spec project carries a domain folder it could have done without.
- The arbitrage engine, which spans concerns, needs the "dominant domain" judgement (resolved to
  `data`).

---

<!-- Add further cross-cutting meta-decisions below as they emerge, citing the per-spec
     decision logs they distil. Example citation key:
       `cross-book-arbitrage D17` = Decision 17 in
       specs/data/cross-book-arbitrage/decision_log.md -->
