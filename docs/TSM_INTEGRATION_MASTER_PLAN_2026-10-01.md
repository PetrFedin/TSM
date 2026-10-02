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

## Additional wave — assortment change points, survival analysis and reproducible analytical reports

This wave adds stronger longitudinal analysis without crossing the line from observable public behaviour to internal sales/stock claims.

### Change-point detection — ADAPT

Reference: https://github.com/deepcharles/ruptures

Use change-point methods on sufficiently long, quality-controlled time series such as:

- active assortment count by brand/category;
- observed price/discount depth;
- share of sizes observed available;
- new/removed product rate;
- category/brand mix.

Purpose:

- flag structural assortment/markdown regime changes;
- prioritize analyst review;
- compare periods more objectively.

A detected change point is a statistical signal, not proof of a commercial event/cause. Always show the underlying series and data-quality state.

### Product / Assortment Survival Analysis — ADAPT

Reference: https://github.com/CamDavidsonPilon/lifelines

Use survival methods for observable durations such as:

- days a product remains publicly listed;
- time until first observed markdown;
- time until observed disappearance;
- duration of a size/variant's observed availability.

Important labels:

- right-censored when the observation window ends;
- source/collector coverage;
- event definition.

Never label disappearance as sale or sell-through. It is an observed website event only.

### Collector Canary & Schema Drift Gate — ADOPT

Before full daily collection, run a small fixed canary set of representative pages/endpoints.

Canary checks:

- HTTP/render success;
- expected selectors/structured-data fields;
- price availability;
- product ID/canonical link;
- variant extraction;
- media;
- extraction conflict rate.

If the canary crosses failure thresholds, freeze publication of new analytical deltas and mark the collection run degraded rather than silently generating false market movements.

### Reproducible Analytical Report Layer — ADOPT/ADAPT

Reference: https://github.com/evidence-dev/evidence

Use Evidence or the same pattern for version-controlled analyst-facing reports built from DuckDB/Parquet/validated snapshots.

Every published report states:

- observation window;
- dataset/DVC version;
- code SHA;
- collector versions;
- truth definitions;
- caveats;
- source/evidence links where practical.

Evidence is a presentation layer; it must not become a new data authority.

### Event Annotation Registry — ADOPT

Create an analyst-maintained annotation table for known external/context events:

- sale campaign;
- website redesign;
- assortment launch;
- collection change;
- collector/parser change;
- public event/season boundary.

This lets charts distinguish a statistical change from a known technical/content event without inventing causal claims.

### Additional acceptance

- change points are reproducible from a declared series/version;
- survival analysis correctly treats censoring/observation gaps;
- degraded canary stops misleading daily deltas;
- analytical report links to exact dataset/code version;
- annotations never silently transform correlation into causality.

**Sequencing:** stable daily history + quality gate -> canary -> longitudinal metrics -> change-point/survival -> reproducible reports.

## Additional wave — fashion visual embedding benchmark and visual assortment map

TSM already plans visual similarity. This wave defines how to choose the embedding model honestly instead of assuming generic similarity is good enough for fashion assortment analysis.

### OpenCLIP baseline — ADAPT

Reference: https://github.com/mlfoundations/open_clip

Create an offline embedding worker over admitted public product-image derivatives.

Store only:

- TSM product/observation ID;
- image checksum;
- model ID/version;
- preprocessing version;
- embedding/vector reference;
- generated_at.

Raw image evidence stays in the existing evidence/media layer.

### Fashion-specific benchmark candidate — EVALUATE

Reference: https://github.com/aimagelab/open-fashion-clip

Evaluate a fashion-domain embedding model against the same labelled benchmark rather than adopting it automatically.

Because upstream model/code licensing and model-weight terms may differ, confirm both before production use.

### Human-labelled visual benchmark — ADOPT

Build a small reviewed evaluation set covering questions TSM actually cares about:

- same/near-identical model;
- same silhouette/type;
- visually similar style;
- same colour family;
- clearly unrelated.

Record ambiguous cases separately.

Compare:

- cheap imagehash duplicate baseline;
- generic OpenCLIP;
- fashion-specific candidate(s).

Choose the model per use case, not one universal score.

### Visual Assortment Map — ADOPT

Create a derived 2D/cluster analytical view from embeddings for:

- assortment density;
- near-duplicate clusters;
- style whitespace;
- brand/category overlap;
- new item distance from existing assortment.

The projection must always link back to real product observations/images.

2D position is an analytical visualization, not a categorical truth or sales recommendation.

### Cross-period visual change — ADOPT

For stable embedding versions, measure:

- cluster composition changes;
- entrance of new style groups;
- repeated/near-identical designs;
- visual breadth by brand/category.

Do not compare coordinates/vectors across changed model versions without a declared migration/re-embedding.

### Additional acceptance

- embedding result traces to exact image checksum + model/preprocess version;
- human benchmark and ambiguity are retained;
- model choice is justified by task-level metrics, not marketing claims;
- similarity/cluster labels never become SKU identity automatically;
- all longitudinal comparisons use compatible embedding versions.

**Sequencing:** evidence/media snapshots -> imagehash baseline -> labelled benchmark -> OpenCLIP/fashion-model evaluation -> chosen vector index -> visual assortment map.

**Dependency note:** confirm current model-weight and code licenses separately before production adoption.

## Additional wave — visual-regression canary for collector integrity

This wave detects site-layout changes that can break a collector even when HTTP requests still succeed.

### Pixelmatch screenshot-diff canary — ADOPT

Reference:

https://github.com/mapbox/pixelmatch

For a small fixed canary set of representative product/listing pages:

Playwright render -> normalized screenshot -> compare with accepted baseline -> diff ratio/regions -> collector quality gate

Use this alongside semantic/schema checks, not instead of them.

### Baseline Authority — ADOPT

Each visual baseline stores:

- page/canary ID;
- viewport/device profile;
- locale;
- captured_at;
- site state/context;
- screenshot checksum;
- collector/browser version;
- approved_by;
- superseded_by.

A new site design intentionally changes the baseline only after review.

### Dynamic-region masking — ADOPT

Mask/ignore known unstable regions where possible:

- rotating banners;
- timestamps;
- personalised/recommended carousels;
- cookie overlays;
- dynamic counters.

Otherwise visual diff noise will hide meaningful breakage.

### Visual Breakage Classification — ADOPT

Differentiate:

- harmless visual/style change;
- DOM/selector structure change;
- price/availability region missing;
- product image/content missing;
- consent/interstitial blocking;
- collector rendering failure.

A severe canary failure should downgrade/freeze analytical publication for the affected collector path.

### Human Review Artifact — ADOPT

Store baseline/current/diff images for failed canaries so an analyst can quickly understand whether the website or collector changed.

These images are QA evidence, not product observations unless separately admitted into the evidence pipeline.

### Additional acceptance

- canary suite uses fixed viewport/locale/browser settings;
- approved baseline changes are versioned;
- dynamic regions are controlled to reduce false positives;
- visual failure cannot silently generate market deltas;
- screenshot diff complements WARC/structured-data/schema checks;
- report identifies exact collector/browser release.

**Sequencing:** current collector canary -> screenshot baselines -> Pixelmatch diff -> severity rules -> publication gate.

**Dependency note:** Pixelmatch is currently ISC-licensed upstream and should be used only on the small canary set, not as a replacement for structured collection.

