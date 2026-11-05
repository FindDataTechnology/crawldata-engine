# crawldata-engine

> By [FindData](https://www.finddatatech.cloud) — powers the 柏讯 (Wire) product
line. MIT licensed.

**English** | [中文](README.zh-CN.md)

**crawldata-engine** is a data-stream acquisition and processing engine: a
crawl-control plane, an open-data ontology, and an analysis surface in one
deployable service.

You declare **what** you want by **concept + entity** ("price.close of Moutai",
"GDP of China"); the engine resolves concepts to physical columns in each
source, ranks candidate sources by quality and reachability, and fetches from
the best one (with failover). Around that core it runs the full crawl lifecycle:

- **Crawl control plane** — declarative policies (`crawl_policies`) → compiled
  plans (`CrawlPlan`) → scheduled k8s Jobs or a lease-queue worker fleet;
  per-source rate limiting, proxy pools, ban rules and circuit breakers built in.
- **Storage & governance** — a governed observation store (`semantic_observations`)
  with date-shape checks, month-bucket (`obs_month`) indexing, and O(Δ) incremental
  rollups feeding a storage watermark.
- **Event stream** — an at-least-once Redis Streams plane (envelopes with
  monotonic-ULID idempotency keys) landing realtime pushes with idempotent
  natural-key writes; cron stays as gapfill reconciliation.
- **Analysis plane** — pluggable `AnalysisBackend` (default: rollup-backed
  aggregates + time series) behind capped, fail-loud API endpoints.
- **Operations panel** — a bilingual (EN/中文) `/panel` console for coverage,
  scheduling, login stations and stream health, with permission tiers and a
  ships-dark gate.

## Quick start

**Zero-setup (Docker)** — Postgres + Redis + engine + operations panel:

```bash
docker compose -f docker-compose.dev.yaml up --build
# panel: http://localhost:8300/panel   MCP: http://localhost:8300/mcp
```

**CLI (your own Postgres or the default SQLite)**:

```bash
pip install "crawldata-engine"        # +[data] for akshare/yfinance SDKs

crawldata-engine migrate              # create the ontology tables
crawldata-engine import-catalog       # load the in-code example catalog
                                      # (World Bank wbgapi is key-free)
crawldata-engine serve                # MCP server (stdio)
```

The in-code seed catalog works with no workspace and no keys. Providers that
need API keys (`POLYGON_API_KEY`, `EDGAR_IDENTITY`, …) or a local datasource
workspace are skipped with a stated reason, never a crash. See CONFIG.md for
the full environment reference and docs/ for datasource integration guides.

## Architecture

```
 Datasources (pluggable)              Engine layer (this repo)
 akshare / yfinance / polygon /       concepts + concept_bindings  (column -> concept)
 EDGAR / World Bank / cn-gov /        entities + per-source identifiers
 datacommons / your own provider      source_rankings (quality x reach x freshness)
        │                             semantic_observations (governed store)
   Catalog loaders: import-catalog,   rollups + storage watermark
   consume-concepts, propose-         crawl: policies -> plans -> k8s Jobs
     bindings, seed-entities,               or lease-queue worker fleet
   generate-schedules                 stream: Redis Streams + idempotent landing
        │                             analysis: pluggable backends (aggregates,
   Runtime: read() -> cache hit? :          time series)
     dispatch (ranked + failover)     panel: /panel operations console
   Search: semantic_search + graph_search + ai_search
```

Eight capabilities (see `openspec/changes/add-crawldata-engine/specs/`):
`open-data-catalog`, `semantic-layer`, `entity-identity`, `source-ranking`,
`concept-fetch`, `scheduled-refresh`, `entity-graph`, `vector-search`.

## Install

```bash
cd crawldata-engine
uv sync                  # base install

# full datasource support (akshare, yfinance, edgar, world bank, ...)
uv sync --extra data
```

The database path defaults to `fd_open_data_mcp/metadata/daas.db` (SQLite),
overridable via `FD_OPEN_DATA_MCP_DATABASE_URL` (Postgres recommended for
production; the Docker quickstart provisions one). `FINDDATA_ROOT` (optional)
points at a local datasource workspace for providers that need it — the in-code
seed catalog works without it.

**Note**: before using SEC EDGAR data, set
`EDGAR_IDENTITY="your_email@example.com"` in the environment.

## Quick start

**Zero-setup option** — bring up Postgres + Redis + the engine (MCP server and
the `/panel` console on `http://localhost:8300/panel`) with one command. The
`init` service runs `migrate` + `import-catalog` automatically, loading the
in-code example catalog (World Bank `wbgapi` is key-free, so `fetch` works out
of the box; providers that need a local workspace or API keys are skipped with
a reason, never a crash):

```bash
docker compose -f docker-compose.dev.yaml up --build
# teardown + reset data:  docker compose -f docker-compose.dev.yaml down -v
```

Note: the panel ships **dark** (no auth) in this dev stack — never expose it
publicly without setting `PANEL_TOKEN` or `LOGTO_*` (see SECURITY.md).

Or run the CLI directly against your own Postgres:

```bash
# 1. Create the ontology tables
crawldata-engine migrate

# 2. Import catalogs (akshare 673, yfinance 12, cn-gov 11, cn-report 44, edgar 6, ...)
crawldata-engine import-catalog
# or a single provider: crawldata-engine import-catalog akshare

# 3. Consume the 926 indicator_defs as concepts and propose column->concept bindings
crawldata-engine consume-concepts
crawldata-engine propose-bindings

# 4. Seed per-source entity identifiers (stocks for akshare/yfinance, countries for worldbank)
crawldata-engine seed-entities

# 5. Generate refresh schedules from indicator_defs.frequency
crawldata-engine generate-schedules

# 6. Read by concept + entity (read-through cache + ranked dispatch + failover)
crawldata-engine read --concept-id 234 --entity-type stock --entity-id 1 --date 2024-07-26
```

## MCP server

```bash
crawldata-engine serve          # FastMCP, stdio transport
```

**71 tools**, registered across six files (49 `@mcp.tool` in `server.py` + 22 from the five `register_*` modules; count verified 2026-10-06 against the live instance's `tools/list` = 71):

| Group | Tools |
|-------|-------|
| Catalog/bindings (`server.py`) | `import_catalog`, `register_datasource`, `update_entity`, `update_concept`, `update_binding`, `consume_concepts`, `propose_bindings`, `list_concepts`, `list_concept_families`, `list_registry_entries`, `record_concept_mapping`, `list_concept_mappings`, `import_crosswalks`, `list_bindings`, `review_bindings`, `confirm_binding` |
| Entity identity (`server.py`) | `seed_entity_identifiers`, `resolve_entity`, `add_entity_identifier`, `list_entities`, `get_entity`, `add_entity` |
| Entity graph (`server.py`) | `list_relationships`, `add_relationship`, `graph_search` |
| Vector/semantic search (`server.py`) | `semantic_search`, `semantic_search_entities`, `semantic_search_unified`, `re_embed_concept`, `ai_search` |
| Ranking/read (`server.py`) | `rank_sources`, `read`, `read_series`, `fetch` |
| Search scopes (`server.py`) | `scope_create`, `scope_list`, `scope_update`, `scope_delete`, `scope_bind_caller`, `scope_unbind_caller`, `scope_list_bindings`, `scope_stats` |
| Refresh schedules/indexes (`server.py`) | `plan_crawl`, `generate_refresh_schedules`, `list_schedules`, `run_schedule`, `list_cnreport_rules`, `enumerate_wbgapi_indicators`, `register_discovered` |
| Crawl policy control plane (`policy_tools.py`) | `policy_create`, `policy_list`, `policy_get`, `policy_update`, `policy_enable`, `policy_disable`, `policy_delete`, `policy_trigger_now`, `policy_runs`, `policy_estimate`, `data_stats` |
| Crawl platform control (`platform_tools.py`) | `platform_sources`, `platform_runs`, `platform_trigger`, `platform_cancel_run`, `platform_cancel_pending` |
| Crawl visibility (`visibility_tools.py`) | `crawl_status` |
| Coverage gaps (`coverage_tools.py`) | `coverage_report` |
| Authenticated-crawling identity pool (`auth_tools.py`) | `auth_status`, `auth_events`, `auth_request_login`, `auth_launch_login` |

## Catalog expansion and the read boundary

The `list_concepts` family (MCP tools and the `list-concepts` CLI) also returns
entries from the **unified indicator registry** (`registry_entries` table) that
passed the naming-quality gate (verified), covering world_bank / gta_panel /
china_city_panel / the fd_open_data mirror; yearbook's numeric-placeholder
entries stay unverified and out of the catalog.

- **Merge semantics**: local concepts (`concepts` table) first, registry entries
  after (ordered by domain, semantic_code); entries whose semantic_code
  duplicates a local concept code are deduped (local row wins). Registry rows
  carry `native_code` and `source_db` identity fields, so retrieval works with
  either the native or the semantic code (the `query` parameter substring-matches
  code/name). Without that table in the local DB, the tool falls back to the
  pure local catalog — behavior unchanged.
- **Read boundary (important)**: this server does **not** read observation data
  for those external-registry indicators — forwarding reads through
  business-mcp is a later change; in this version `read`/`fetch` on registry
  indicators does not return external-source data. This expansion affects the
  indicator dictionary only, not any data-reading capability.

### Full-registry enumeration (`list_registry_entries`, v0.5.38)

The parallel **tiered-browsing channel** (ADR-0002): pages through ALL
registered entries — verified and not — ordered by `(source_db, native_code)`,
each row carrying the authoritative `verified` flag, with an optional
`status` filter (`verified` | `registered`; default all). The catalog/search/
read **verified gate is untouched**: `list_concepts`, `ai_search` and the read
paths still exclude unverified entries. Consumers: the Wire public catalog
mirror re-keyed its ids to `source_db:native_code` from this feed. An optional
`scope` filters rows by the indicator-scope rules (source_db / domain /
semantic_code dimensions); unscoped calls return a plain list (the Wire sync
contract).

## Data sources

The dispatcher `run_upstream()` routes by source name to adapters. The table
below is ordered by **verified reachability**, not self-reported status.

### Verified online ✅

| Source | Notes |
|--------|-------|
| **akshare** | A-share stocks/funds/financials |
| **yfinance** | Yahoo Finance global equities |
| **cn-report** | Chinese financial reports (44 tools, see [fd-cn-report](https://github.com/FindDataTechnology/fd-cn-report)) |
| **edgar** | SEC EDGAR filings (needs `EDGAR_IDENTITY`) |
| **wbgapi** | World Bank data API |
| **cnstats** | National Bureau of Statistics data |
| **ckan** | CKAN catalog data |
| **nbs-gdp** | NBS GDP data |
| **datacommons** | Google Data Commons (needs `DC_API_KEY`) |
| **polygon** | Polygon.io US equities (external package fd-polygon) |
| **edinet** | Japan EDINET filings |
| **dartlab** | Korea DARTLab disclosures |

> The `polygon` and `datacommons` runners live in external `fd-*` packages,
> lazily loaded via the manifest's `fetch.module` — `crawldata-engine` does not
> depend on `polygon-api-client` / `requests` unless a fetch actually happens.

### Stub adapters ⚠️ (sample data only, **no** network)

| Source | Status |
|--------|--------|
| **cisa-industry** | stub — static sample rows |
| **amac-fund** | stub |
| **shfe-metal-futures** | stub |
| **agriculture** | stub (DCE) |
| **cme-agricultural-futures** | stub (CME) |
| **chemicals** | stub |
| **electronics** | stub |
| **nonferrous** | stub |
| **flowers-kifc** | stub |
| **fin_platforms** | stub (Wind) |
| **sac-securities** | stub |

These adapters expose `run_<source>()` entries and manifest registrations, so
`list-sources` self-reports "✅ Full support" — but they return synthetic
sample rows, not real exchange or association data. Read the adapter file
before depending on any of them.

### Read-only catalogs

| Source | Status |
|--------|--------|
| **cn-gov** | manifest registration only (11 ministry catalogs) |
| **world** | CKAN + Chinese NBS statistical catalogs |

## Crawl control center (panel + reconciler)

A policy describes **what to crawl**: concept × entity scope × date range ×
frequency × mode. `CrawlPolicy` objects are created on the panel, compiled into
`CrawlPlan`s by the reconciler, and executed by `scraw-fd-open-data-mcp` writing
into `semantic_observations`.

```bash
# Start the control panel (default http://0.0.0.0:8000)
FD_OPEN_DATA_MCP_DATABASE_URL=<db url> crawldata-engine panel

# Run the reconciler once (due policies -> launch; close stale runs)
python -m fd_open_data_mcp.refresh.reconciler
```

**Environment variables:**
- `PANEL_TOKEN` — when set, `/panel/*` requires it (header `X-Panel-Token`,
  `?token=`, or cookie).
- `POLICY_MAX_FETCHES` (default `50000`) — plan-size guardrail; a due policy
  whose fetch estimate exceeds it is rejected (recorded as a failed run)
  unless the policy sets `force`.
- `RECONCILER_LAUNCHER` — `scrapyd` (default) or `k8s` (`K8sJobLauncher`).
- `SCRAPYD_URL` / `SCRAW_PLAN_DIR` (scrapyd launcher); `SCRAW_K8S_NAMESPACE` /
  `SCRAW_K8s_IMAGE` / `SCRAW_K8S_DATABASE_URL` / `SCRAW_K8S_REDIS_URL`
  (k8s launcher).
- `FD_PROXY_FORWARDER` — empty for local dev (the injection layer returns a
  direct sentinel → direct egress); cluster crawling uses the standalone
  `fd-proxy-service` forwarder for proxy selection. The legacy vars
  `FD_PROXY_POOL`/`FD_EGRESS_MODE` are no longer read.

## Tests

```bash
uv run --with pytest pytest -q
```

## Design notes / v1 limitations

- **Propose-confirm flow**: column→concept bindings carry `confidence` +
  `provenance`; below-threshold bindings stay out of dispatch (into the review
  queue). One real fetch promotes a binding to `sample-confirmed`.
- **Ranking** is per `(source × concept)`, self-tuning from `fetch_log`
  (bounded — one failure never removes a source).
- **Conflict policy**: one cached value per `(concept, entity, date)` with its
  `source_used`; cross-source values are never merged.
- **LLM providers** power semantic enrichment / cross-language concept
  mapping. v1 uses rule tables + `semantic_type` hints.

Full specifications: `openspec/changes/add-crawldata-engine/`.

## Proxy pool & circuit breaker

The fetch stack rotates IPs through a **proxy pool** to avoid source bans, with
per-(source, real_source×proxy_ip) circuit breakers and ban rules. Pool state is
a plain table (`proxies`), so any deployment that keeps the table fresh works;
`FD_PROXY_POOL` / `FD_PROXY_FORWARDER` select the pool implementation.

Key files: `fd_open_data_mcp/proxy/` (selector, breaker, ban rules, injection);
real-source failover in `fd_open_data_mcp/fetch/dispatch.py`.

## Standard real-source names

- `eastmoney` — Eastmoney (A-share primary, via akshare)
- `tencent` — Tencent Finance (A-share failover)
- `sina` — Sina Finance (A-share alternative)
- `yahoo_finance` — Yahoo Finance (global markets, via yfinance)

Libraries (`akshare`, `yfinance`) call several real sources underneath; the
breaker tracks health per real source, not per library, enabling smart
failover.

## LLM configuration (PDF report extraction)

`fd-cn-report` uses an LLM to extract financial indicators from annual-report
PDFs. The current provider is **DeepSeek on Ark** (see the
[fd-cn-report](https://github.com/FindDataTechnology/fd-cn-report) README):

```bash
# crawldata-engine/.env
LLM_BASE_URL=…/api/plan/v1     # Ark endpoint
LLM_API_KEY=…                  # Ark key
LLM_MODEL=deepseek-v4-flash
```

`LLM_API_KEY` and `OPENAI_API_KEY` (legacy) are both accepted;
`LLM_API_KEY` wins when both are set.

## Images

`docker build -t crawldata-engine .` produces the runtime image (the compose
quickstart builds it locally). CI (`.github/workflows/image.yml`) builds and
pushes per-commit `sha-<short>` + rolling `main` tags to the registry configured
in repository secrets; pin the sha tag in deployment manifests for immutable
rollouts.

## License

MIT
