# TSM — Auditable Assortment Observatory Integration Master Plan

**Status:** PLANNED  
**Date:** 2026-10-01  
**Canonical file:** `docs/TSM_INTEGRATION_MASTER_PLAN_2026-10-01.md`

## Purpose
Build TSM as an auditable observatory of the public men's assortment on tsum.ru while strictly separating observed facts, calculated metrics and hypotheses.

## Truth rule
Never present public availability as audited inventory, disappearance as confirmed sale, public counters as sales or a proxy as internal TSUM fact.

## Integration map

| Capability | Source | Decision |
|---|---|---|
| Collection | Crawlee + Playwright | ADOPT |
| Raw evidence | warcio | ADOPT |
| Dataset versioning | DVC | ADOPT |
| Analytics | DuckDB + Polars | ADOPT |
| Price normalization | price-parser | ADOPT |
| Entity reconciliation | RapidFuzz | ADOPT |
| Text similarity | sentence-transformers | ADAPT |
| Visual duplicate/similarity | imagehash, later Qdrant | ADAPT |
| Data contracts | Frictionless + Pandera | ADOPT |
| Quality monitoring | Great Expectations/Evidently patterns | ADAPT |
| Audit explorer | Datasette | OPTIONAL/ADOPT |

## Phase 0 — Collection policy
Document allowed public sources, access constraints, request-rate policy, collector version and evidence retention. Do not bypass access controls.

## Phase 1 — Daily snapshot
Capture source URL, observed_at, exposed source ID, brand/title/category, current/original price, colour/size options, observed availability/status, media URLs/hashes and named public counters.

Collector changes require regression fixtures.

## Phase 2 — WARC evidence
Use warcio to retain selected raw request/response evidence. Every normalized observation links to snapshot/evidence hash.

## Phase 3 — DVC dataset lineage
Large raw/processed artefacts live under DVC; Git stores code, schemas, pointers and metric definitions. Every report records dataset version + code SHA.

## Phase 4 — Normalisation and identity
Use price-parser for prices and RapidFuzz for candidate brand/product/category reconciliation. Low-confidence matches require review.

Create stable TSM product entities with observation links; changing URLs/titles do not automatically create new products.

## Phase 5 — Text and visual similarity
Use sentence-transformers for candidate model similarity. Start with imagehash for cheap near-duplicates; introduce Qdrant only when scale justifies it. Similarity is not proof of identical SKU.

## Phase 6 — Analytics
Use DuckDB + Polars for:
- assortment entry/exit;
- price/markdown change;
- brand/category structure;
- colour/size breadth;
- persistence duration;
- availability-change patterns.

All formulas and truth classes are versioned.

## Phase 7 — Quality gates
Frictionless/Pandera validate schema/types/uniqueness. Add monitoring for collector-wide shifts, gaps, impossible values and schema drift. Great Expectations/Evidently are scale-up tools, not new authorities.

## Phase 8 — Audit explorer
Datasette may expose read-only snapshots and calculations. Metric views should trace back to observations.

## Phase 9 — Hypothesis layer
Only after sufficient history, derive explicitly labelled hypotheses such as likely replenishment, markdown, exit or size depletion. Never label as sales without internal sales data.

## Prohibited
- bypassing access restrictions;
- equating public availability with stock;
- inferring revenue/sales as fact;
- losing raw lineage;
- making Qdrant product authority;
- overwriting historical observations.

## Issue order
1. TSM-INT-00 Collection policy
2. TSM-INT-01 Daily snapshots
3. TSM-INT-02 WARC evidence
4. TSM-INT-03 DVC lineage
5. TSM-INT-04 Normalisation/entity identity
6. TSM-INT-05 DuckDB/Polars analytics
7. TSM-INT-06 Similarity
8. TSM-INT-07 Quality gates
9. TSM-INT-08 Datasette
10. TSM-INT-09 Hypothesis layer

**Implementation instruction:** auditability and truth labels take priority over aggressive inference.

## Additional wave — structured-source extraction and rendered-page evidence

### extruct structured-data extraction — ADOPT

Reference: https://github.com/scrapinghub/extruct

Before relying on brittle DOM selectors, inspect public structured metadata exposed by the page:

- JSON-LD;
- microdata;
- RDFa;
- Open Graph;
- other supported embedded structures.

For product pages, preserve the raw structured block and parse candidate fields such as:

- product/name;
- brand;
- offers/price/currency;
- availability;
- colour/size where exposed;
- canonical URL;
- image.

Important: structured metadata is still a **public observed source**, not an internal TSUM fact. It can be stale or incomplete and must be cross-checked against rendered/server evidence.

Store:

- structured-data type/source;
- parser version;
- raw block hash;
- extracted fields;
- conflict indicators versus rendered/API observations.

### SingleFile rendered-page snapshot — ADOPT/CONDITIONAL

Reference: https://github.com/gildas-lormeau/SingleFile

Use for selected audit samples or important change events, not necessarily every page every day.

Capture a self-contained rendered HTML snapshot when:

- parser/schema changes;
- product materially changes;
- a disputed/important observation needs human review;
- collector regression fixtures are created.

This complements WARC:

- WARC = raw request/response evidence;
- SingleFile snapshot = human-reviewable rendered-page evidence.

Do not rely on SingleFile as the primary data extraction path.

### Structured-vs-rendered reconciliation

Add an explicit observation-quality layer:

`raw HTTP/WARC + structured metadata + rendered DOM -> normalized observation + conflicts`

Example conflict flags:

- JSON-LD price differs from visible price;
- structured availability says InStock but variant UI shows unavailable;
- canonical/product ID changed;
- structured metadata disappeared after collector release.

A conflict should downgrade confidence or queue review, not be silently resolved by whichever parser ran last.

### Acceptance extension

- important observations can be reconstructed from raw and rendered evidence;
- structured metadata parser version is recorded;
- conflicts are visible as data-quality facts;
- no structured field is promoted to internal-sales/inventory truth.

**Sequencing:** extruct can be added during normalisation; SingleFile follows the WARC/snapshot framework and should be sampling/event driven to control storage.

