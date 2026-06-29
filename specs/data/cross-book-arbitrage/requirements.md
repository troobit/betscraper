# Requirements: Cross-Book Arbitrage (Australian Betting Odds Scraper)

## Introduction

This MVP is a local, single-user tool, built from scratch, whose purpose is to find
**cross-bookmaker arbitrage** — sets of bets across different bookmakers that lock in a
guaranteed profit regardless of outcome — and surface them on a dashboard alongside the
current odds they are derived from. It runs entirely on one machine against a single local
datastore, with no external database, service, or task queue. It is operated by a
data-literate academic who is comfortable with SAS and basic Python but is not a software
developer, and who can use — or be trained by an experienced developer to use — the
spec-driven development (SDD) workflow and tools. The requirements describe behaviour and
constraints only; the implementation language and stack are chosen in the design phase.

## Non-Goals (Out of Scope)

- A command-line interface and a standalone REST API — the dashboard is the only interface in this MVP.
- API-key authentication, per-request rate limiting, and multi-user roles (Admin/API/Guest).
- External databases, caches, task queues, or background services — the tool uses a single local datastore only.
- Working scrapers for all six listed bookmakers — only 2–3 ship working; the rest are documented stubs.
- A broad automated test suite — error logging is the MVP's reliability mechanism, EXCEPT the arbitrage calculation, which is unit-tested (see 7.7).
- Value betting against an estimated "fair" probability (only guaranteed cross-book arbitrage).
- Golf *outright* (full-field) arbitrage — excluded from evaluation; only small-outcome golf markets (matchups) are considered.
- Fuzzy / probabilistic event matching — the MVP uses deterministic matching and excludes records it cannot match with confidence.
- Automated placing of bets, account integration, or any interaction with bookmaker accounts.
- Background/scheduled scraping — scrapes run only when the operator triggers a refresh.
- Historical trend analytics, alerting/notifications, and data export.
- Containerization, cloud deployment, and CI/CD.

## Requirements

### 1. Local-First Project Setup

**User Story:** As a non-developer academic, I want to install and run the tool locally without external services, so that I can get to a first result quickly.

**Acceptance Criteria:**

1. <a name="1.1"></a>The system SHALL run with no external database, cache, or background service, using a single local datastore.  
2. <a name="1.2"></a>The system SHALL start successfully with only its declared dependencies installed, with no dependency-version-conflict errors.  
3. <a name="1.3"></a>WHEN the application starts and its datastore does not yet exist, THEN the system SHALL create it and initialise the schema and seed data automatically.  
4. <a name="1.4"></a>The system SHALL read all configuration from documented settings (a configuration file and/or environment variables), and SHALL start using documented defaults when optional values are absent.  
5. <a name="1.5"></a>The system SHALL provide a single documented command to install dependencies and a single documented command to start the tool.  

### 2. Configuration Management

**User Story:** As the operator, I want to control which bookmakers, sports, intervals, and dashboard address are active through configuration, so that I can adjust behaviour without editing source code.

**Acceptance Criteria:**

1. <a name="2.1"></a>The system SHALL load configuration values for enabled bookmakers, enabled sports, request timeout, retry count, per-request delay, odds freshness window, event match window, minimum-profit threshold, default total stake, and the dashboard host and port.  
2. <a name="2.2"></a>WHEN a list-valued setting (e.g. enabled bookmakers) is provided as a delimited string, THEN the system SHALL parse it into the corresponding list without error.  
3. <a name="2.3"></a>IF a configuration value is missing or invalid, THEN the system SHALL log a clear message identifying the setting and SHALL fall back to the documented default rather than crashing.  
4. <a name="2.4"></a>The system SHALL hold bookmaker definitions (code, display name, base URL, per-request delay) in configuration such that adding a bookmaker entry requires no change to core scraping logic.  

### 3. Core Scraping Framework

**User Story:** As a developer extending the tool, I want each bookmaker to be added by satisfying one well-defined scraping contract, so that adding a bookmaker does not touch matching or arbitrage logic.

**Acceptance Criteria:**

1. <a name="3.1"></a>For each configured bookmaker and sport, the system SHALL produce odds records that each carry bookmaker, event identity, market, outcome/selection, decimal odds, and the scrape timestamp.  
2. <a name="3.2"></a>The system SHALL apply the configured timeout, retry count with backoff, and per-request delay to each fetch (the delay serving as a politeness measure to limit load on bookmaker sites).  
3. <a name="3.3"></a>WHEN a fetch fails after the configured number of retries, THEN the system SHALL log an error identifying the bookmaker, sport, and URL, and SHALL continue with the remaining work rather than aborting the run.  
4. <a name="3.4"></a>The system SHALL convert source odds to decimal odds, and WHEN an odds value cannot be parsed THEN it SHALL skip that selection and log a warning rather than store an invalid value.  
5. <a name="3.5"></a>WHEN a bookmaker is configured but has no working scraper, THEN the system SHALL log and skip it rather than silently ignoring it.  
6. <a name="3.6"></a>The default-enabled scrapers SHALL require no browser runtime to install or run; WHERE a site can only be scraped with a browser runtime, that scraper SHALL be documented as an exception with its added setup cost and SHALL NOT be on the default install path.  

### 4. Working Bookmaker Scrapers

**User Story:** As the operator, I want odds collected from at least two real bookmakers, so that there are competing prices to compare for arbitrage.

**Acceptance Criteria:**

1. <a name="4.1"></a>The system SHALL provide working scrapers for at least two bookmakers (e.g. Sportsbet and TAB) that return real odds for the configured sports, and SHOULD enable three by default, since arbitrage across only two bookmakers is rare.  
2. <a name="4.2"></a>The working scrapers SHALL collect match-winner / head-to-head odds for UFC, AFL, and Football (soccer) 1X2, and small-outcome golf markets (e.g. 2-ball/3-ball or head-to-head matchups) where offered; golf full-field outright markets SHALL NOT be collected for arbitrage.  
3. <a name="4.3"></a>WHEN a bookmaker does not offer a configured sport, THEN the scraper SHALL log that the sport is unavailable and return no records for it, without error.  
4. <a name="4.4"></a>The system SHALL include documented stub scrapers for the remaining listed bookmakers (Bet365, Neds, Unibet) that clearly indicate they are not yet implemented.  

### 5. Data Persistence

**User Story:** As the operator, I want scraped odds saved to the datastore, so that the dashboard and arbitrage detection work from stored data.

**Acceptance Criteria:**

1. <a name="5.1"></a>The system SHALL persist each scraped odds record, associating it with its bookmaker, event, market, and outcome.  
2. <a name="5.2"></a>The system SHALL record the UTC time each odds record was scraped, so that the most recent odds for an event/market/outcome/bookmaker can be retrieved.  
3. <a name="5.3"></a>WHEN scraping a sport completes, THEN the system SHALL record a scraping-run log entry capturing bookmaker, sport, status, record count, and any error message.  
4. <a name="5.4"></a>The system SHALL create its schema successfully on first run (per 1.3) and SHALL retain odds history so the most-recent odds within the freshness window can be selected.  
5. <a name="5.5"></a>The schema SHALL contain the models required for the MVP — bookmaker, event, market, outcome, odds record, and scraping-run log — and no others.  

### 6. Odds Normalisation and Event Matching

**User Story:** As the operator, I want the same real-world event and outcome to line up across bookmakers, so that arbitrage comparisons are valid and not false.

**Acceptance Criteria:**

1. <a name="6.1"></a>The system SHALL normalise event identity and outcome labels into a canonical form so that records for the same real-world event and outcome from different bookmakers map together.  
2. <a name="6.2"></a>The system SHALL normalise market naming to canonical market codes (e.g. head-to-head/match-winner variants map to one code).  
3. <a name="6.3"></a>The system SHALL match events using a deterministic rule (normalised participant identity plus event date/time within a configured window) and SHALL NOT use fuzzy matching; WHEN match confidence is ambiguous, it SHALL NOT merge the records and SHALL exclude them from arbitrage.  
4. <a name="6.4"></a>IF a scraped record cannot be matched to a canonical event or market, THEN the system SHALL retain the record but exclude it from arbitrage comparison and log that it was unmatched.  
5. <a name="6.5"></a>The system SHALL group matched records into the complete set of mutually-exclusive outcomes for an event+market, so that 2-way (UFC, AFL), 3-way (soccer 1X2), and matchup-style (golf) markets are all represented.  

### 7. Arbitrage Detection

**User Story:** As the operator, I want the tool to find guaranteed cross-bookmaker arbitrage and tell me how to stake it, so that I can act on profitable opportunities with confidence they are real.

**Acceptance Criteria:**

1. <a name="7.1"></a>For an event+market with a complete set of outcomes, the system SHALL select the best (highest) available decimal odds for each outcome across all bookmakers.  
2. <a name="7.2"></a>The system SHALL compute the arbitrage margin as the sum over outcomes of `1 / best_odds`, and SHALL flag the event+market as an arbitrage opportunity WHEN that sum is below 1.  
3. <a name="7.3"></a>WHEN an opportunity is flagged, THEN for the operator's total stake the system SHALL report the stake to place on each outcome (and at which bookmaker), rounded so every outcome remains profitable, together with the worst-case guaranteed return after rounding, the absolute profit, and the ROI percentage.  
4. <a name="7.4"></a>The system SHALL only evaluate a market for arbitrage WHEN best odds are present for every outcome in its complete outcome set, and SHALL otherwise skip and log it.  
5. <a name="7.5"></a>The system SHALL apply a configurable minimum-profit threshold, expressed as an ROI percentage, and SHALL exclude opportunities below it from reported results.  
6. <a name="7.6"></a>The system SHALL base arbitrage detection only on the most recent odds per bookmaker/outcome, and SHALL include an outcome's odds only WHEN scraped within the configured freshness window; IF any leg is older than the window, THEN the opportunity SHALL be excluded and logged as stale.  
7. <a name="7.7"></a>A margin that sits exactly on the 1.0 boundary SHALL be classified deterministically and correctly, and rounding a stake to the bettable currency unit SHALL never reduce any outcome's return below the total stake; the margin and stake-split calculations SHALL be covered by unit tests for 2-way, 3-way, and N-way markets, including the implied-probability-sum-equals-1 boundary and a known worked example.  
8. <a name="7.8"></a>Each reported opportunity SHALL include, per outcome: the bookmaker, the decimal odds, the stake, the scrape timestamp, and the raw source event name as seen at that bookmaker, so the user can verify the match before staking.  

### 8. Web Dashboard

**User Story:** As the operator, I want a single web page that shows current arbitrage opportunities and odds and lets me refresh the data, so that I can run and review the whole workflow without writing code.

**Acceptance Criteria:**

1. <a name="8.1"></a>The system SHALL serve a single dashboard page, at a documented local URL bound to the configured host/port (defaulting to localhost), reading from the stored data with no separate configuration to view.  
2. <a name="8.2"></a>The dashboard SHALL list current arbitrage opportunities with their profit percentage and ROI and, per outcome, the bookmaker, decimal odds, stake, and the age/scrape time of each leg.  
3. <a name="8.3"></a>The dashboard SHALL provide a total-stake input used to compute the per-outcome stakes; WHEN it is empty or invalid, THEN the system SHALL use the configured default total stake.  
4. <a name="8.4"></a>The dashboard SHALL show a table of current odds grouped or filterable by sport.  
5. <a name="8.5"></a>The dashboard SHALL compute opportunities and apply the freshness gate (7.6) against the current time each time the page is loaded or refreshed, so a page left open does not present legs that have since gone stale.  
6. <a name="8.6"></a>The dashboard SHALL provide a refresh action that triggers a scrape for the configured sports and, on completion, reports a per-bookmaker summary of records collected and updates the displayed odds and opportunities; reloading the page in the browser SHALL NOT by itself trigger a scrape.  
7. <a name="8.7"></a>WHILE a triggered scrape is in progress, the system SHALL run at most one scrape at a time (a further refresh request is disabled or rejected) and SHALL indicate that a scrape is running.  
8. <a name="8.8"></a>WHEN a scrape fails wholly or partly, THEN the dashboard SHALL surface the per-bookmaker errors in the summary and SHALL keep the previously stored odds and opportunities visible rather than showing an error or blank page.  
9. <a name="8.9"></a>WHEN no scrape has ever run, THEN the dashboard SHALL show a clear prompt to refresh; WHEN a scrape has run but no opportunities qualify, THEN the dashboard SHALL show an empty-state message with a diagnostic summary — records scraped, bookmakers covered, matched events, count below threshold, and count excluded as stale.  

### 9. Error Logging and Observability

**User Story:** As the operator, I want clear logs of what was scraped and what failed, so that I can diagnose problems without reading the source code.

**Acceptance Criteria:**

1. <a name="9.1"></a>The system SHALL log to both the console and a log file at a configurable log level, with the log file bounded by rotation or a documented size cap.  
2. <a name="9.2"></a>The system SHALL log the start and end of each scraping run with per-bookmaker record counts and any errors.  
3. <a name="9.3"></a>WHEN any scrape, parse, normalisation, or persistence step fails, THEN the system SHALL log an error with enough context (bookmaker, sport, event/URL) to locate the cause, and SHALL continue processing remaining items.  
4. <a name="9.4"></a>The system SHALL not abort an entire scraping run because a single bookmaker, sport, or record failed.  

### 10. Documentation and Examples

**User Story:** As a non-developer academic, I want a step-by-step quickstart and examples written for my skill level, so that I can set up and use the tool unaided.

**Acceptance Criteria:**

1. <a name="10.1"></a>The system SHALL include a `docs/` quickstart that gives ordered, copy-pasteable steps to install dependencies, configure, start the tool, and open the dashboard, assuming no DevOps knowledge.  
2. <a name="10.2"></a>The documentation SHALL include a worked example of using the dashboard — refreshing data, reading the odds table, and reading an arbitrage opportunity — with example output.  
3. <a name="10.3"></a>The documentation SHALL include a guide for adding a new bookmaker scraper against the scraping contract.  
4. <a name="10.4"></a>The documentation SHALL explain the arbitrage calculation (best odds per outcome, implied-probability sum, stake split, rounding) and SHALL state practical limitations: the golf full-field caveat, bet minimums/maximums, account-limitation risk, bonus-bet exclusions, and that a displayed price may not be available at the required stake.  
5. <a name="10.5"></a>The documentation SHALL state that the tool is for local, personal use; that scraping and arbitrage betting may breach bookmaker terms of service and risk account limitation or closure; and that the user is responsible for compliance.  
6. <a name="10.6"></a>The documentation SHALL state that arbitrage opportunities are infrequent and that an empty result is the normal, expected state rather than a fault.  
