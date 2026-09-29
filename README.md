# geospatial-co

**Explainable location scoring with H3, PostgreSQL, and LangGraph.**

geospatial-co is a Python backend and interactive CLI for evaluating potential business locations. It combines precomputed geographic data with a transparent, six-dimension scoring model and optional language-model guidance. Its five business profiles cover retail, EV charging, warehouses, telecom, and renewable energy.

The central question is: **how does a location perform against the priorities of a particular use case, and which factors drive that result?** Each score retains its input dimension scores, weights, and weighted contributions so the numerical result can be inspected independently of the generated narrative.

> **Project status:** an early backend implementation. The scoring functions, API routes, CLI, database migration, and graph are included. Geographic datasets, ingestion pipelines, a web frontend, and a production deployment are not. Some API workflows have known integration gaps, documented below; this repository is not a turnkey nationwide dataset or a validated investment model.

## Watch demo video

https://youtu.be/vNFSIz0yl6o?si=tGBLZ6kuQPjr6kHr

## Contents

- [Capabilities](#capabilities)
- [Quick start](#quick-start)
- [Architecture](#architecture)
- [Scoring methodology](#scoring-methodology)
- [Data contract](#data-contract)
- [API reference](#api-reference)
- [CLI and graph development](#cli-and-graph-development)
- [Configuration](#configuration)
- [Repository map](#repository-map)
- [Known limitations](#known-limitations)
- [Development and validation](#development-and-validation)
- [License](#license)

## Capabilities

| Capability | Implementation and scope |
| --- | --- |
| Site lookup | Converts coordinates to an H3 cell and fetches its stored feature row, or accepts an explicit cell ID. |
| Weighted scoring | Combines six precomputed dimension scores using supplied or recommended weights. |
| Explanation | Reports raw scores, weights, contributions, contribution ranks, and the top/bottom two contributors. |
| Weight sensitivity | Recomputes a stored cell under two weight configurations and returns total and per-dimension deltas. |
| Multi-site comparison | Contains ranking logic and a 2–5-site API schema; exactly two sites currently encounter a routing defect. |
| Hotspot ranking | Scores stored cells in a state and sorts them; the current API graph defaults to Gujarat and ten results. |
| LLM assistance | Recommends weights and writes summaries, with default-weight and template-text fallbacks. |
| Spatial utilities | Includes H3 neighbors and PostGIS radius queries as Python tools; these are not exposed as dedicated API routes. |

## Quick start

Use **Python 3.11 or later** and a PostgreSQL server with PostGIS installed. PostgreSQL 15+ is the setup target. You need a populated `site_features` table for database-backed analysis; migrations create the schema only.

```bash
git clone https://github.com/quixoticalcoder/geospatial-co.git
cd geospatial-co
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e .
cp .env.example .env
```

On Windows PowerShell, use `py -3.11 -m venv .venv`, activate with `.\.venv\Scripts\Activate.ps1`, and copy the template with `Copy-Item .env.example .env`.

Edit `.env` with your database connection and chosen LLM credentials. The application database URL uses the async driver:

```dotenv
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/geospatial_co
LLM_PROVIDER=groq
LLM_MODEL=llama-3.3-70b-versatile
GROQ_API_KEY=your_key_here
LANGCHAIN_TRACING_V2=false
```

Create the database using your PostgreSQL administrator or database tooling, then apply the migration:

```sql
CREATE DATABASE geospatial_co;
```

```bash
python -m alembic upgrade head
python -m alembic current
# Load your prepared geographic data before issuing scoring requests.
python -m uvicorn main:app --reload --port 8000
```

Open [interactive API docs](http://localhost:8000/docs), [ReDoc](http://localhost:8000/redoc), or the [service status endpoint](http://localhost:8000/). The status endpoint reports that the application is running; it does not verify database connectivity, data coverage, or LLM availability.

See [SETUP.md](SETUP.md) for installation details, data-loading requirements, and troubleshooting. For a numerical demonstration that needs neither a database nor an API key, see [Development and validation](#development-and-validation).

## Architecture

The project separates input validation, orchestration, storage access, scoring, and language generation.

```mermaid
flowchart TD
    API[FastAPI scoring, comparison, hotspots] --> O[orchestrator]
    CLI[Interactive CLI] --> O
    O -->|single site, no weights| A[advisory]
    O -->|single site with weights or comparison| F[fetch_features]
    O -->|no site input| G[geospatial]
    O -->|validation error| E[error_handler]
    A --> F
    F --> S[fetch_scores]
    S --> C[compute_score]
    C --> X[explainability]
    X --> I[insight]
    G --> I
    I --> END[End]
    E --> END
    F -.-> DB[(PostgreSQL / PostGIS)]
    S -.-> DB
    G -.-> DB
    A -.-> LLM[Configured LLM provider]
    I -.-> LLM
```

[`agents/graph.py`](agents/graph.py) defines nine nodes. The orchestrator makes decisions from request fields without an LLM. The advisory node can propose weights; the insight node converts structured results into prose. Database tools use asynchronous SQLAlchemy sessions, and [`scoring/engine.py`](scoring/engine.py) performs the numerical calculation.

The graph handles the main analysis workflows. Site lookup and what-if routes call tools directly, and the score route recomputes the score from graph output when constructing its response. Graph compilation accepts an optional checkpointer, but the default API, CLI, and module-level graph do **not** configure persistent checkpoints.

### Failure behavior

- Advisory failures fall back to the relevant default weight profile.
- Insight failures return a short template summary.
- Graph errors are surfaced by analysis routes, generally as HTTP 400 or 500.
- Missing lookup rows produce HTTP 404 in site routes. The what-if route currently maps any score-fetch exception to 404.
- Secondary comparison sites that fail to load are logged and skipped, so a returned ranking may contain fewer sites than requested.

## Scoring methodology

For dimension scores \(s_i\) and weights \(w_i\), the engine computes:

```text
contribution_i = score_i × weight_i
site_readiness_score = round(sum(contribution_i), 2)
```

The six dimensions are `demand_score`, `accessibility_score`, `competition_score`, `suitability_score`, `risk_score`, and `infrastructure_score`. Each stored dimension score must lie between 0 and 100. Each weight must lie between 0 and 1; the model accepts a weight sum between 0.99 and 1.01. Supply weights summing to exactly 1.0 to preserve the intended 0–100 range.

### Default profiles

These values are defined in [`scoring/weights.py`](scoring/weights.py).

| Use case | Demand | Accessibility | Competition | Suitability | Risk | Infrastructure |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `retail` | 0.30 | 0.20 | 0.20 | 0.10 | 0.10 | 0.10 |
| `ev_charging` | 0.25 | 0.30 | 0.05 | 0.10 | 0.10 | 0.20 |
| `warehouse` | 0.15 | 0.35 | 0.05 | 0.15 | 0.15 | 0.15 |
| `telecom` | 0.10 | 0.25 | 0.10 | 0.15 | 0.15 | 0.25 |
| `renewable` | 0.10 | 0.20 | 0.05 | 0.25 | 0.20 | 0.20 |

For illustrative scores of **80, 70, 60, 90, 50, 75**, the retail profile yields:

```text
80×0.30 + 70×0.20 + 60×0.20 + 90×0.10 + 50×0.10 + 75×0.10 = 71.50
```

This is an arithmetic example, not an observed site's result. With these weights, demand contributes 24.00 points and risk contributes 5.00 points.

### Interpretation and reproducibility

The numerical engine is deterministic for fixed dimension scores and weights. An end-to-end request that omits weights may receive LLM-recommended weights, so its score is not guaranteed to repeat. Supply explicit weights and preserve the underlying data snapshot when reproducibility matters.

“Strengths” and “weaknesses” mean the largest and smallest **weighted contributions**, not necessarily the largest and smallest raw scores. Contribution values and the total are rounded independently, which can produce small rounding differences.

The engine does not invert risk or competition at request time. Your data pipeline must orient every dimension so that a higher score means a more favorable outcome. Normalization, inversion, and distance-decay helpers exist under `scoring/`, but the live scoring path reads the six stored scores directly. It does not recompute them from raw features or dynamically replace competitor counts.

## Data contract

[`001_initial_schema.py`](migrations/versions/001_initial_schema.py) creates `site_features`, enables PostGIS, and adds geometry, state, and grid-ID indexes. The primary key is `grid_id`, an H3 cell identifier.

| Data group | Representative fields |
| --- | --- |
| Location | `grid_id`, `latitude`, `longitude`, `state`, `district`, `area_name`, `geom` |
| Demographics | Population at different radii, density, age distribution, literacy, income |
| Transportation | Road density, highway distance, intersections, travel-time measures |
| Economic activity | POI counts, shops, restaurants, competitors, footfall proxy |
| Land use | Commercial/residential/industrial ratios, building density, built-up area |
| Environment | Air quality, flood and earthquake risk, green space, temperature |
| Infrastructure | Power, water, and public transport indicators |
| Derived scores | The six normalized dimension scores used by the engine |
| Metadata | `last_updated` |

The schema comments reference possible upstream sources such as OSM, WorldPop, census data, and environmental datasets. This repository does not fetch, bundle, license, or validate those datasets. Data preparation and provenance remain the responsibility of the external pipeline.

Coordinates are mapped with `h3.latlng_to_cell` at `H3_RESOLUTION` (default 8). The resulting ID is used for an indexed equality lookup. Despite its name, `/sites/nearest/` does not search for the closest available row or fall back to neighboring cells if coverage is missing. Keep the configured resolution aligned with the stored data.

The graph validates the primary input against a bounding box of latitude −8 to 37 and longitude 68 to 97. This is a coarse regional restriction, not a national-boundary check. `SiteInput` requires `lat` and `lng` even when `h3_id` is supplied.

## API reference

Request and response models live in [`models/request.py`](models/request.py). Run the service and consult `/docs` for the generated schemas.

| Method | Route | Behavior |
| --- | --- | --- |
| `GET` | `/` | Static service status and version. |
| `GET` | `/sites/{h3_id}` | Stored features for an exact H3 cell. |
| `GET` | `/sites/nearest/?lat=…&lng=…` | Features for the cell containing the coordinates; keep the trailing slash. |
| `POST` | `/score` | A single site's score, contribution breakdown, insight, and optional advisory text. |
| `POST` | `/score/what-if` | Original/modified scores and contribution deltas for one stored cell. |
| `POST` | `/compare` | Ranked site scores and an insight; see the two-site routing limitation. |
| `POST` | `/hotspots` | Hotspot results and an insight; state and result-count inputs are not currently propagated. |

### Score a site with explicit weights

This request uses the retail defaults directly, bypassing weight recommendation. The insight node still attempts an LLM call. The example coordinates require a matching populated cell in your database.

```bash
curl -X POST http://localhost:8000/score \
  -H 'Content-Type: application/json' \
  -d '{
    "site_input": {"lat": 23.0225, "lng": 72.5714},
    "use_case": "retail",
    "weights": {
      "demand_score": 0.30,
      "accessibility_score": 0.20,
      "competition_score": 0.20,
      "suitability_score": 0.10,
      "risk_score": 0.10,
      "infrastructure_score": 0.10
    }
  }'
```

| Response field | Contents |
| --- | --- |
| `site_score` | Cell ID, coordinates, final score, contributions, original scores, and weights used. |
| `score_breakdown` | Per-dimension raw score, weight, contribution and rank, plus strengths/weaknesses. |
| `insight_text` | Generated narrative or fallback summary. |
| `advisory_text` | Weight recommendation rationale when advisory ran; otherwise optional. |

Omit `weights` to request LLM-assisted weight recommendation with fallback to the default profile.

### Compare weight scenarios

Replace `YOUR_STORED_H3_ID` with a cell present in the database. This route uses stored scores and does not call an LLM.

```bash
curl -X POST http://localhost:8000/score/what-if \
  -H 'Content-Type: application/json' \
  -d '{
    "h3_id": "YOUR_STORED_H3_ID",
    "current_weights": {
      "demand_score": 0.30, "accessibility_score": 0.20,
      "competition_score": 0.20, "suitability_score": 0.10,
      "risk_score": 0.10, "infrastructure_score": 0.10
    },
    "modified_weights": {
      "demand_score": 0.25, "accessibility_score": 0.30,
      "competition_score": 0.05, "suitability_score": 0.10,
      "risk_score": 0.10, "infrastructure_score": 0.20
    }
  }'
```

The response's `result` contains `original_score`, `modified_score`, `delta`, and `dimension_deltas`.

## CLI and graph development

```bash
# Installed console command
geospatial-co

# Equivalent source-module entry point
python -m cli
```

The Rich/Typer interface offers single-site scoring, comparisons, and hotspot discovery, with guided inputs and formatted results. It invokes the graph directly rather than requiring a running HTTP server. Each analysis gets a fresh thread ID; that identifier alone does not enable persistence.

For local graph inspection, install the optional development CLI:

```bash
python -m pip install 'langgraph-cli[inmem]'
langgraph dev
```

[`langgraph.json`](langgraph.json) registers the graph as `geospatial_co`, referencing `./agents/graph.py:graph`. The underscore is the machine-identifier form of the project name. Local graph tooling may have additional runtime requirements; the core package's Python requirement is 3.11+.

## Configuration

[`core/config.py`](core/config.py) loads case-insensitive environment variables and `.env` settings.

| Variable | Default / purpose |
| --- | --- |
| `DATABASE_URL` | Async PostgreSQL URL, with `geospatial_co` as the example database. |
| `LANGGRAPH_CHECKPOINT_URL` | Reserved PostgreSQL checkpointer setting; unused by default compilation. |
| `LLM_PROVIDER` | `groq`; also supports `openai`, `anthropic`, and `ollama`. |
| `LLM_MODEL` | `llama-3.3-70b-versatile`; choose a model compatible with your selected provider. |
| `LLM_TEMPERATURE` | `0.2`. |
| `LLM_MAX_TOKENS` | `1000`; the current Ollama branch does not pass this setting. |
| `GROQ_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` | Credential for the selected hosted provider. |
| `APP_ENV` | `development`; also accepts `staging` and `production`. |
| `LOG_LEVEL` | `INFO`. |
| `H3_RESOLUTION` | `8`; must match your stored H3 IDs. |

Optional tracing variables in `.env.example` are consumed by LangChain/LangSmith rather than the application's settings model. Tracing is disabled in the template; enable it and supply valid credentials if needed. Its project label is `geospatial-co`.

The editable package install includes Groq, OpenAI, and Anthropic integrations. `requirements.txt` is a separate installation list with OpenAI/Anthropic commented out. The Ollama branch additionally requires `langchain-community`, a running Ollama service, and a locally available model. No provider credentials or database content are included.

## Repository map

```text
geospatial-co/
├── main.py                         FastAPI app and service status
├── cli.py                          Interactive Rich/Typer interface
├── agents/                         Graph, state, routing, advisory, insights
├── api/routes/                     Lookup, scoring, comparison, hotspots
├── core/                           Settings, database, logging, exceptions
├── llm/                            Provider factory and prompts
├── models/                         Pydantic request, feature, score, weight models
├── scoring/                        Weighted sum, profiles, normalization, decay
├── tools/                          Database, spatial, explanation, weight tools
├── migrations/                     Alembic environment and initial SQL schema
├── .env.example                    Configuration template
├── alembic.ini                     Migration configuration
├── langgraph.json                  Local graph registration
├── pyproject.toml                  Package metadata and console entry point
├── requirements.txt                Alternative dependency list
├── SETUP.md                        Installation and troubleshooting
└── geospatial_agent_coding_prompt.md  Historical design brief
```

The historical design brief describes intended behavior and is not an implementation guarantee. This README describes the current source. Runtime logs are written to `logs/agent_app.log` with rotation and are excluded from Git.

## Known limitations

These are observable implementation gaps, not completed features:

1. **No bundled data or ETL.** An empty migration produces no usable scoring locations. All six derived scores must be populated and valid for single-site scoring.
2. **Timestamp/model mismatch.** The migration defines `last_updated` as SQL `TIMESTAMP`, while `SiteFeatures` expects a string. Normal database timestamp values can fail Pydantic validation during feature loading. The type needs reconciliation before relying on real-data workflows.
3. **Two-site comparison routing.** The route separates the primary site from comparison targets, but the orchestrator requires at least two remaining targets to select comparison mode. An exactly two-site request can return an empty ranking. Requests with three to five sites reach comparison mode; secondary failures may still be silently omitted from the ranking.
4. **Hotspot request fields are dropped.** The API accepts `state` and `top_n`, but does not pass them through graph state. The spatial node defaults to Gujarat and `top_n=10`.
5. **Checkpoints are not enabled.** Installing a PostgreSQL checkpointer dependency or setting its URL does not connect it to the running graph.
6. **Dependency ranges are broad.** The code uses H3 v4 functions while the dependency lists permit v3.7. Use H3 v4 for these interfaces. There is no lockfile to reproduce a complete environment.
7. **Validation and scaling need further work.** Weight sums slightly above 1 can yield an out-of-range total. Hotspot ranking fetches all matching rows and sorts in Python. Coordinate validation is not uniformly applied across all lookup and comparison paths.
8. **No production access controls or automated test suite.** Authentication is a placeholder; the repository includes no CI pipeline or committed tests. Startup and `/` do not establish service readiness.

The numerical score is a weighted index of supplied data and priorities. The repository provides no empirical accuracy study, predictive calibration, or decision-outcome benchmark.

## Development and validation

A self-contained scoring example exercises the core engine without database access or LLM calls:

```bash
python - <<'PY'
from models.site import PrecomputedScores
from scoring.engine import compute_final_score
from scoring.weights import USE_CASE_WEIGHTS

scores = PrecomputedScores(
    grid_id="illustrative-cell",
    demand_score=80, accessibility_score=70, competition_score=60,
    suitability_score=90, risk_score=50, infrastructure_score=75,
)
result = compute_final_score(scores, USE_CASE_WEIGHTS["retail"])
assert result.site_readiness_score == 71.5
print(result.model_dump_json(indent=2))
PY
```

Useful local checks after a change:

```bash
python -m compileall -q agents api core llm models scoring tools main.py cli.py
python -m pip check
geospatial-co --help
python -m alembic upgrade head --sql
```

Compilation checks syntax; migration SQL generation checks migration rendering. Neither substitutes for integration tests against populated PostgreSQL/PostGIS and a configured provider. When adding functionality, validate routing, missing data, invalid weights, provider failures, and data-model compatibility as well as the numerical result. Keep dependency metadata and setup instructions synchronized.

## License

The existing project notice identifies this software as proprietary. No standalone license grant is included. Contact the repository owner for permission to use or redistribute it.
