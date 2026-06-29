# Specs Overview

> How specs are written, tracked, and turned into code: [PROCESS.md](PROCESS.md).
> Cross-cutting decisions are distilled in the meta decision log: [DECISIONS.md](DECISIONS.md).
> Per-spec decision logs remain authoritative for detail.

> **Domain** ([PROCESS.md §3](PROCESS.md#3-directory-structure-spec-boundaries-and-naming)) — every
> spec lives at `specs/<domain>/<capability>/`; the domain is one of this project's fixed set
> (see [DECISIONS.md](DECISIONS.md) MD-1).
> **Mode** ([PROCESS.md §5](PROCESS.md#5-choosing-the-mode-full-spec-smolspec-or-iterative)) —
> `full` / `smol` / `iterative`.

> **This is a generated index.** Regenerate it from the spec folders with `/specs-overview` after
> adding or merging a spec — do not hand-merge it (PROCESS.md §9).

| Name | Domain | Created | Status | Mode | Summary |
|------|--------|---------|--------|------|---------|
| [Cross-Book Arbitrage](#cross-book-arbitrage) | data | 2026-06-22 | Draft | full | Local single-user tool that scrapes Australian bookmaker odds, lines up the same event/outcome across books, and reports guaranteed cross-book arbitrage with stake instructions. The project MVP. |

---

## Cross-Book Arbitrage

Uplifts a prototype into a local, single-user tool that scrapes three Australian bookmakers,
normalizes the same event/outcome across them, and reports guaranteed cross-book arbitrage with
stake instructions. SQLite-backed, run from the VS Code terminal via a CLI, a small REST API, and
a read-only dashboard. The dominant concern is odds-data ingestion and the arbitrage engine, so it
lives under `data` even though it scrapes external sites (`api`) and exposes CLI/API/dashboard
surfaces (`ui`), which it references (PROCESS.md §3). This is the project MVP, named for the
capability it delivers — not "mvp".

- [requirements.md](data/cross-book-arbitrage/requirements.md)
- [design.md](data/cross-book-arbitrage/design.md)
- [decision_log.md](data/cross-book-arbitrage/decision_log.md)
