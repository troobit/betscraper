# Decision Log: Cross-Book Arbitrage (Australian Betting Odds Scraper)

## Decision 1: Use the full spec workflow

**Date**: 2026-06-29
**Status**: accepted

### Context

The MVP is a from-scratch build spanning configuration, scraping, persistence, odds
normalisation and matching, an arbitrage engine, and a dashboard. It carries genuine
architectural choices (datastore, matching strategy, arbitrage correctness) rather than a
single narrow change.

### Decision

Use the full spec workflow (Requirements → Design → Tasks) rather than a smolspec.

### Rationale

Smolspec suits changes under ~80 lines across 1–3 files with narrow scope. This work crosses
many subsystems and has architectural decisions to make and record, so it exceeds that
threshold.

### Alternatives Considered

- **Smolspec**: One lightweight document - Rejected because the work spans many subsystems with
  architectural decisions.

### Consequences

**Positive:**
- Architectural decisions get explicit design; tasks break down for incremental verification.

**Negative:**
- More up-front spec effort than a smolspec.

---

## Decision 2: Greenfield, local-first, single datastore, no external services

**Date**: 2026-06-29
**Status**: accepted

### Context

The tool is built from scratch — there is no prior prototype to uplift. It is operated by one
person on one machine: a data-literate academic comfortable with SAS and basic Python but not a
software developer.

### Decision

Build the tool greenfield and local-first: a single local datastore, no external database,
cache, task queue, or background service. The whole workflow runs on one machine.

### Rationale

A single-user, single-machine tool has no need for server infrastructure. A single local
datastore is the simplest thing that supports persistence, the dashboard, and arbitrage
detection, and it keeps setup within reach of a non-developer.

### Alternatives Considered

- **External database / cache / queue (Postgres, Redis, Celery)**: Server-grade infrastructure -
  Rejected as unjustified complexity for one user on one machine.
- **No persistence (in-memory only)**: Simpler still - Rejected because the dashboard and
  freshness-windowed detection need stored odds history.

### Consequences

**Positive:**
- Minimal moving parts; a non-developer can install and run it.

**Negative:**
- No multi-user or networked operation; not a goal.

---

## Decision 3: Requirements are implementation-language-agnostic

**Date**: 2026-06-29
**Status**: accepted

### Context

The operator is SAS- and Python-familiar and can use, or be trained by an experienced developer
to use, the SDD workflow — but the build language is not fixed by the brief. Requirements should
not pre-empt that choice.

### Decision

State the requirements as behaviour and constraints only — "single local datastore", "declared
dependencies", "documented settings (file and/or environment variables)" — and decide the
implementation language, libraries, and stack in the design phase.

### Rationale

Local-first, single-user, no external services is a constraint, not a stack. Pinning a language
or framework in the requirements would pre-empt design and contradict the language-agnostic
intent. Money-correctness is kept as observable behaviour (no boundary misclassification, no leg
rounded into a loss) rather than by naming a numeric type.

### Alternatives Considered

- **Pin the stack in requirements (e.g. Python/SQLite)**: Less to decide later - Rejected; it
  prescribes implementation in the requirements phase and contradicts the agnostic intent.
- **Name a recommended stack as non-binding guidance**: A hint for the developer - Rejected for
  the requirements doc; such guidance belongs in design.

### Consequences

**Positive:**
- Design is free to choose the stack; requirements stay testable against behaviour.

**Negative:**
- The stack is undecided until design; `CLAUDE.md`'s current Python framing is provisional until
  design confirms or replaces it.

---

## Decision 4: Scope is guaranteed cross-bookmaker arbitrage only

**Date**: 2026-06-29
**Status**: accepted

### Context

"Arbitrage" can mean guaranteed cross-book arbitrage (lock in profit across bookmakers) or value
betting against an estimated fair probability. The two need very different machinery and carry
different risk.

### Decision

Detect only guaranteed cross-bookmaker arbitrage: for a complete outcome set, take the best odds
per outcome across bookmakers and flag when the sum of implied probabilities is below 1, with a
stake split that keeps every leg profitable.

### Rationale

Guaranteed arbitrage is objectively verifiable from the odds alone and matches the user's intent
to act with confidence. Value betting requires probability estimation the MVP deliberately avoids.

### Alternatives Considered

- **Value betting against fair-odds estimates**: Larger opportunity space - Rejected; needs a
  probability model and carries estimation risk the MVP does not take on.

### Consequences

**Positive:**
- Opportunities are provable from observed odds; no modelling risk.

**Negative:**
- Opportunities are rare; an empty result is the normal state.

---

## Decision 5: The dashboard is the only MVP interface

**Date**: 2026-06-29
**Status**: accepted

### Context

The MVP is "a dashboard floating arbitrage options and displaying odds." A non-developer operator
needs the whole workflow — trigger a scrape, view odds, view arbitrage — from one place.

### Decision

Make a single web dashboard the only interface. It lists arbitrage opportunities and a
current-odds table and triggers scrapes via a refresh action. No command-line interface and no
standalone REST API ship in the MVP.

### Rationale

One page covers the entire workflow for a non-developer with no UI investment beyond what the
brief asks for. A CLI and an API are additional surfaces the MVP does not need; they can return
as a later spec if a script-comfortable workflow is wanted.

### Alternatives Considered

- **CLI + REST API + dashboard**: Serves script users too - Rejected as out of scope for an MVP
  scoped to the dashboard.
- **Dashboard + read-only JSON API**: A data-pull seam behind the page - Rejected to keep the MVP
  minimal; revisit if programmatic access is later required.

### Consequences

**Positive:**
- Smallest interface scope; one page for refresh, odds, and arbitrage.

**Negative:**
- No programmatic access in the MVP; a script user reads the datastore directly or waits for a
  later API spec.

---

## Decision 6: Scrapes are triggered by a dashboard refresh action

**Date**: 2026-06-29
**Status**: accepted

### Context

With the dashboard as the only interface, the scrape trigger must be reachable from the page.

### Decision

The dashboard provides a refresh action that runs a scrape for the configured sports, reports a
per-bookmaker summary, and updates the displayed odds and opportunities. At most one scrape runs
at a time; a page reload alone does not scrape. No background interval loop ships in the MVP.

### Rationale

A manual refresh is the simplest trigger for a single page and a non-developer: no background
process to manage, and the operator controls when load is placed on bookmaker sites. Arbitrage is
rare and short-lived, so on-demand refresh fits the actual workflow. The single-scrape guard and
reload-does-not-scrape rule prevent accidental duplicate load on bookmaker sites.

### Alternatives Considered

- **Background auto-refresh on an interval**: Hands-off freshness - Rejected for the MVP as a
  running loop and lifecycle the single-user workflow does not need.
- **Both interval loop and manual button**: Maximum flexibility - Rejected as over-scoped; the
  manual button alone satisfies the workflow.

### Consequences

**Positive:**
- No background process; the operator controls scrape timing and site load.

**Negative:**
- Odds are only as fresh as the last manual refresh; the freshness window guards stale legs, and
  the dashboard re-applies the freshness gate at view time.

---

## Decision 7: Deterministic event matching, no fuzzy matching

**Date**: 2026-06-29
**Status**: accepted

### Context

The same real-world event is named differently across bookmakers. A false match would produce a
phantom arbitrage that loses money; a missed match only forgoes an opportunity.

### Decision

Match events deterministically on normalised participant identity plus event date/time within a
configured window. Where match confidence is ambiguous, do not merge the records and exclude them
from arbitrage. Do not use fuzzy or probabilistic matching.

### Rationale

A false positive is far more costly than a false negative here. Deterministic matching with an
explicit ambiguity rule is conservative and auditable; the operator can verify each leg against
the raw source event name before staking.

### Alternatives Considered

- **Fuzzy / probabilistic matching**: Catches more matches - Rejected because a wrong match
  creates a money-losing phantom arbitrage; conservatism wins.

### Consequences

**Positive:**
- No phantom arbitrage from mismatched events; results are auditable.

**Negative:**
- Some genuine matches are missed when naming differs beyond the normalisation rules.

---

## Decision 8: Correctness via money-safe math plus a unit-tested arbitrage core

**Date**: 2026-06-29
**Status**: accepted

### Context

Arbitrage is the one place a calculation error directly loses money: a borderline margin
misclassified, or a stake rounded so a leg becomes a loss. Elsewhere, the cost of a fault is a
logged failure, not lost money.

### Decision

Make the arbitrage core money-safe — classify the 1.0 boundary deterministically and round stakes
so no leg's return falls below the total stake — and cover the margin and stake-split math with
unit tests for 2-way, 3-way, and N-way markets, including the boundary and a worked example. Rely
on error logging as the reliability mechanism for the rest of the system rather than a broad test
suite.

### Rationale

Concentrating testing on the arbitrage math targets the only money-losing failure mode; logging
gives enough diagnosis everywhere else for a single-user tool. A broad automated suite is effort
the MVP does not need.

### Alternatives Considered

- **Broad automated test suite**: More coverage - Rejected as MVP effort out of proportion to a
  single-user tool whose non-arbitrage failures are non-fatal and logged.
- **No tests, logging only**: Least effort - Rejected because the arbitrage math is the one place
  a silent error loses money.

### Consequences

**Positive:**
- The money-losing failure mode is directly tested; diagnosis elsewhere comes from logs.

**Negative:**
- Regressions outside the arbitrage core are caught by logs/observation, not by tests.

---

## Decision 9: Deliver 2–3 working scrapers behind one scraping contract; the rest are stubs

**Date**: 2026-06-29
**Status**: accepted

### Context

Arbitrage needs competing prices, so at least two real bookmakers must return odds; across only
two it is rare, so three is better. Scraper sites differ and some need a browser runtime.

### Decision

Ship working scrapers for at least two bookmakers (target three by default, e.g. Sportsbet, TAB,
Ladbrokes), each satisfying one scraping contract so adding a bookmaker does not touch matching or
arbitrage. The remaining listed bookmakers (Bet365, Neds, Unibet) are documented stubs. Default
scrapers require no browser runtime; any browser-dependent scraper is a documented exception off
the default install path.

### Rationale

Two competing books are the minimum for arbitrage and three materially increase the hit rate. A
single thin contract keeps the core logic untouched as books are added. Keeping the browser
runtime off the default path keeps install within reach of a non-developer.

### Alternatives Considered

- **All six bookmakers working**: Maximum coverage - Rejected as more scraper maintenance than the
  MVP needs; stubs document the extension path.
- **Browser-based scraping by default**: Handles JS-heavy sites - Rejected for the default path
  because of its setup cost for a non-developer.

### Consequences

**Positive:**
- Enough competing prices for real arbitrage; adding a book is one contract.

**Negative:**
- Coverage is limited to the working scrapers until stubs are implemented.

---

## Decision 10: Initial sports are UFC, AFL, soccer 1X2, and small-outcome golf

**Date**: 2026-06-29
**Status**: accepted

### Context

A starting sports/markets scope is needed. Golf full-field outright markets have many outcomes,
making complete cross-book outcome sets impractical and risky to treat as arbitrage.

### Decision

Collect match-winner / head-to-head odds for UFC and AFL (2-way), Football (soccer) 1X2 (3-way),
and small-outcome golf markets such as 2-ball/3-ball or head-to-head matchups. Exclude golf
full-field outright markets from arbitrage.

### Rationale

These give a mix of 2-way, 3-way, and matchup markets to exercise the engine while keeping
outcome sets small and completely observable across books. Full-field golf has too many outcomes
to assemble a reliable complete set.

### Alternatives Considered

- **Include golf full-field outrights**: More markets - Rejected; large outcome sets make a
  complete cross-book set impractical and the "arbitrage" unreliable.
- **Single sport only**: Simpler - Rejected because a spread of 2-way/3-way/matchup markets better
  exercises matching and detection.

### Consequences

**Positive:**
- Covers 2-way, 3-way, and matchup shapes; outcome sets stay complete and verifiable.

**Negative:**
- A popular golf market (full-field) is out of scope.

---

## Decision 11: Spec lives at `specs/data/cross-book-arbitrage/`

**Date**: 2026-06-29
**Status**: accepted

### Context

`specs/PROCESS.md` requires each spec to live at `specs/<domain>/<capability>/`, named for a
durable capability from the fixed domain set in `DECISIONS.md` MD-1 (`platform`, `data`, `api`,
`ui`, `ops`), never for the effort or phase.

### Decision

Home the spec at `specs/data/cross-book-arbitrage/`. The domain is `data` — the dominant concern
is collecting, normalising, matching, and computing over odds data; the dashboard is a thin
reporting layer. The capability is `cross-book-arbitrage`.

### Rationale

Naming the folder for the capability under its dominant domain keeps intent durable and
branch-portable and the spec's home unambiguous as the project grows. Most acceptance criteria
concern odds data, so `data` wins the dominant-domain judgement.

### Alternatives Considered

- **Phase-named folder (e.g. `specs/mvp/`)**: Less ceremony - Rejected; `PROCESS.md §3` forbids
  naming a spec for the effort or phase.
- **Split into multiple capability specs now**: More granular - Rejected as premature per
  `PROCESS.md §10`; one spec is right until concerns acquire separate owners or change-cadences.

### Consequences

**Positive:**
- Durable capability name under a fixed domain; branch-portable, regenerable in the index.

**Negative:**
- None material at this stage.

---

## Decision 12: Implementation stack — Python, SQLite, Flask

**Date**: 2026-06-30
**Status**: accepted

### Context

Requirements are language-agnostic (Decision 3); the design must commit a concrete stack. The
operator is SAS- and Python-familiar, will maintain the tool, and the project tooling already
assumes Python. The MVP is one local user, one dashboard page, a single SQLite datastore, and no
public API (Decision 5, Non-Goals).

### Decision

Build in Python 3.11+ with SQLite (SQLAlchemy ORM) as the datastore, Flask + Jinja2 serving the
dashboard, `pydantic-settings` for configuration, `requests` for HTTP, `Decimal` plus integer cents
for money math, and `pytest` + `hypothesis` (dev only) for the arbitrage tests.

### Rationale

Python matches the operator's skills and the project conventions. SQLite is the simplest datastore
that satisfies persistence, history, and freshness-windowed detection on one machine. Flask is the
lightest fit for "serve one page plus a refresh endpoint" — its async/OpenAPI surface would go
unused because the Non-Goals exclude a standalone REST API.

### Alternatives Considered

- **FastAPI + Uvicorn**: Async, typed, auto `/docs` - Rejected: its strengths target a public API
  this MVP explicitly excludes, adding ASGI/OpenAPI weight for no gain on a single internal page.
- **A non-Python stack (e.g. Node/Go)**: - Rejected: contradicts the operator's Python familiarity
  and the project tooling, for no benefit at this scale.

### Consequences

**Positive:**
- Smallest dependency set for a dashboard-only tool; familiar to the maintainer.

**Negative:**
- Flask is run multithreaded (`threaded=True`) so an in-progress scrape does not block reads or the
  one-at-a-time guard; a single local user needs no more (see Decision 18).

---

## Decision 13: Greenfield installable `betscraper/` package

**Date**: 2026-06-30
**Status**: accepted

### Context

There is no prior prototype to uplift (Decision 2). The code is organised from scratch. Adding a
bookmaker must not touch matching or arbitrage logic (Req 2.4, 3).

### Decision

Ship a single installable `betscraper/` package (`pip install -e .`) with thin scrapers in
`scrape/`, all canonicalisation/matching/persistence in `pipeline/`, the pure arbitrage engine in
`arbitrage.py`, and the Flask dashboard in `web/`. Scrapers return `ScrapedOdds`; everything
downstream is shared.

### Rationale

Concentrating normalisation, matching, and detection behind a thin scraper contract means a new
book is one fetch+parse method. A flat single-source layout would entangle scraping with core
logic and make extension risky.

### Alternatives Considered

- **Flat script(s) with no package boundary**: Less ceremony - Rejected: no seam between thin
  scrapers and shared logic, so adding a book would touch core code.

### Consequences

**Positive:**
- Adding a bookmaker is isolated to one module; core logic is untouched and unit-testable.

**Negative:**
- A package and `pyproject.toml` are slightly more setup than a single script.

---

## Decision 14: Plain HTTP/JSON scrapers; no Selenium on the default path

**Date**: 2026-06-30
**Status**: accepted

### Context

Some bookmaker sites are JS-heavy and would need a browser runtime. A non-developer must be able to
install the tool without that cost (Req 3.6).

### Decision

The three shipping scrapers use `requests` against each site's JSON endpoint (or HTML where
necessary). No Selenium/headless-browser path is built. The browser-runtime exception is documented
in `docs/adding-a-scraper` as guidance for a future scraper that needs it.

### Rationale

Building a browser path before any shipping scraper needs it is unjustified complexity and adds an
install burden the default path must avoid. Documenting the exception preserves the extension route.

### Alternatives Considered

- **Selenium-capable base scraper by default**: Handles JS-heavy sites - Rejected: setup cost for a
  non-developer and dead weight until a scraper actually needs it.

### Consequences

**Positive:**
- Default install needs no browser; scrapers stay simple.

**Negative:**
- A book that genuinely requires a browser is a stub until the documented exception is implemented.

---

## Decision 15: Deterministic matching key without date; a separate match window

**Date**: 2026-06-30
**Status**: accepted

### Context

The same event is named differently and ordered differently (home/away) across books, and listed
times differ by minutes — sometimes across a UTC-midnight boundary. Matching must be deterministic
(Decision 7) and must not split one real event into two.

### Decision

The canonical event key is `sport | sorted(normalized participants)` — no date in the key. Sorting
cancels participant order. `resolve_event` then matches a candidate sharing that key whose
`start_time` is within a configured **match window** (a few hours), distinct from the freshness
window. Multiple candidates inside the window → not merged, stored unmatched and logged.

### Rationale

A date in the key would split an event listed at 23:58 by one book and 00:03 by another. Sorting
plus an alias table covers naming and ordering differences without fuzzy matching, while the match
window still separates two fixtures between the same participants on different days.

### Alternatives Considered

- **Include the event date in the key**: Simpler lookup - Rejected: the UTC-midnight boundary
  splits a single event into two unmatched records.
- **Fuzzy/probabilistic matching**: Catches more - Rejected by Decision 7 (a false match loses
  money).

### Consequences

**Positive:**
- Robust to ordering and near-midnight listing differences; still deterministic and auditable.

**Negative:**
- A new configured value (match window) the operator may need to tune for sports with frequent
  rematches.

---

## Decision 16: Append-only odds; latest via `max(scraped_at)`, no history table

**Date**: 2026-06-30
**Status**: accepted

### Context

The tool must retain odds history so the most recent odds within the freshness window can be
selected (Req 5.2, 5.4), but the schema is limited to the models in Req 5.5.

### Decision

`OddsRecord` is append-only. "Latest odds" for a (bookmaker, outcome) is `max(scraped_at)`,
tie-broken by highest `id`. History is the accumulation of rows; no separate history model exists.

### Rationale

A separate history table would be a seventh model Req 5.5 does not list. Append-only rows give
history and freshness selection for free with one model.

### Alternatives Considered

- **Mutable "current odds" row + separate history table**: Explicit current state - Rejected: adds
  a model outside Req 5.5 and write complexity for no MVP benefit.

### Consequences

**Positive:**
- History and freshness selection from one append-only model.

**Negative:**
- The table grows unbounded; acceptable for a single-user local tool, prunable later if needed.

---

## Decision 17: Schema is exactly Req 5.5's six models; Sport is config, not a model

**Date**: 2026-06-30
**Status**: accepted

### Context

Req 5.5 enumerates the schema models — bookmaker, event, market, outcome, odds record, scraping-run
log — "and no others." Sport is referenced throughout the requirements but is not in that list.

### Decision

Model exactly those six as DB tables, with `Market` and `Outcome` as first-class tables (not
collapsed into columns on the odds row). Sport is a configuration reference (`SPORTS` in
`config.py`) carried as a `sport_code` column, not a DB model.

### Rationale

Req 5.5 is restrictive ("and no others"), so the schema follows its enumeration literally. `Market`
(kind: 2-way/3-way/N-way) and `Outcome` (the mutually-exclusive slot set, Req 6.5) are exactly the
listed concepts and make the required-outcome-set check natural. Sport is a fixed, config-driven
enum like the bookmaker list, so it lives in config; the asymmetry with `Bookmaker` (which Req 5.5
does list) follows the requirement, not preference.

### Alternatives Considered

- **Collapse market/outcome into columns on the odds row**: Fewer tables - Rejected: contradicts
  Req 5.5, which names market and outcome as models.
- **Add a `Sport` table**: Symmetry with `Bookmaker` - Rejected: Req 5.5 omits sport and forbids
  models beyond its list.

### Consequences

**Positive:**
- Schema maps one-to-one onto Req 5.5; the outcome-set check reads off `Outcome` rows.

**Negative:**
- Sport and bookmaker are handled asymmetrically (config-only vs seeded table), which must be
  documented to avoid confusion.

---

## Decision 18: Dashboard backend — synchronous lock-guarded refresh; reload never scrapes

**Date**: 2026-06-30
**Status**: accepted

### Context

The dashboard must trigger scrapes via a refresh action, run at most one scrape at a time, indicate
that one is running, and ensure a browser reload does not itself scrape (Req 8.5–8.7). Scraping puts
real load on bookmaker sites; viewing is cheap and safe.

### Decision

`GET /` recomputes opportunities from stored odds and renders — reload-safe, never scrapes. The
refresh button issues `POST /scrape`, which runs the scrape **synchronously** and returns the
re-rendered page with a per-book summary. One-at-a-time is enforced by a module-level non-blocking
lock; a concurrent POST is rejected (`409`). The in-progress state is the browser's loading state
during the blocking request. The server runs `threaded=True` (so the lock can reject a concurrent
POST and `GET /` stays responsive during a scrape) with SQLite in WAL mode plus a busy-timeout (so
the view read does not collide with the scrape write).

### Rationale

Separating view (GET) from scrape (POST) is what stops an accidental reload, back/forward, or
left-open tab from hammering bookmaker sites — the guard Req 8.6 demands. The synchronous form
satisfies Req 8.6–8.8 with no status endpoint, polling, or background thread: the only code beyond a
naive "reload = scrape" design is a second route and a lock. A background-thread-plus-poll design
was considered and rejected as more machinery than a single local user needs.

### Alternatives Considered

- **Background thread + status-poll endpoint + JS**: Richer in-progress UX, non-blocking - Rejected:
  a status endpoint and client polling are more complexity than one local user needs.
- **Reload triggers the scrape (conflate GET and POST)**: Fewer routes - Rejected: every page view
  would hit the bookmaker sites, violating Req 8.6 and risking account limitation.

### Consequences

**Positive:**
- Satisfies Req 8.6–8.8 with minimal code; viewing stays cheap and safe to repeat.

**Negative:**
- The browser blocks for the scrape duration and "running" is only the browser spinner; acceptable
  for one local user, revisitable if a richer UX is wanted later.

---
