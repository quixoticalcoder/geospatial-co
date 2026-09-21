# geospatial-co — Setup Guide

This guide covers local installation of the backend and CLI. Read the [known limitations](README.md#known-limitations) before running analysis against real data.

## 1. Prerequisites

- Python 3.11+ and pip.
- PostgreSQL with PostGIS installed on the database server; PostgreSQL 15+ is the setup target.
- Database credentials with permission to create the application schema. Enabling PostGIS may require administrator assistance.
- Prepared geographic rows and six normalized scores per cell. Data ingestion is external to this repository.
- A provider API key for generated advice/insights, or a local Ollama setup. LLM failures have fallbacks.

## 2. Clone and install

```bash
git clone https://github.com/quixoticalcoder/geospatial-co.git
cd geospatial-co
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e .
cp .env.example .env
```

Windows PowerShell equivalents:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e .
Copy-Item .env.example .env
```

Run subsequent commands from the repository root. The editable install registers the `geospatial-co` console command. The application uses H3 v4 APIs; if your environment already contains H3 v3, install a compatible version with `python -m pip install 'h3>=4,<5'`.

## 3. Configure the environment

Set your own values in `.env`:

```dotenv
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/geospatial_co
LLM_PROVIDER=groq
LLM_MODEL=llama-3.3-70b-versatile
GROQ_API_KEY=your_key_here
APP_ENV=development
LOG_LEVEL=INFO
H3_RESOLUTION=8
LANGCHAIN_TRACING_V2=false
LANGCHAIN_PROJECT=geospatial-co
```

The database name is configurable. Existing installations may keep their current database by setting `DATABASE_URL` explicitly; changing the example name does not rename or migrate an existing database. Percent-encode reserved characters in database usernames/passwords when forming a URL.

| Provider | Configuration | Additional requirement |
| --- | --- | --- |
| Groq | `LLM_PROVIDER=groq`, `GROQ_API_KEY`, compatible `LLM_MODEL` | Included in package installation. |
| OpenAI | `LLM_PROVIDER=openai`, `OPENAI_API_KEY`, compatible `LLM_MODEL` | Included in package installation. |
| Anthropic | `LLM_PROVIDER=anthropic`, `ANTHROPIC_API_KEY`, compatible `LLM_MODEL` | Included in package installation. |
| Ollama | `LLM_PROVIDER=ollama`, locally available `LLM_MODEL` | Install `langchain-community` and run the local Ollama service. |

`LANGGRAPH_CHECKPOINT_URL` is reserved configuration: the default graph does not use it. Tracing is optional and requires its own valid credentials when enabled. Keep `.env` out of version control.

## 4. Create the schema

Using a suitable PostgreSQL administrative connection:

```sql
CREATE DATABASE geospatial_co;
```

Then, from the activated application environment:

```bash
python -m alembic upgrade head
python -m alembic current
```

The migration enables PostGIS and creates `site_features` and its indexes. Alembic reads `DATABASE_URL` and switches from asyncpg to psycopg2 for synchronous migrations.

To inspect SQL without connecting to a database:

```bash
python -m alembic upgrade head --sql
```

Do not use the initial migration's downgrade on a database whose data or PostGIS extension you need to preserve: it drops the table and then the extension.

## 5. Prepare and load data

There is no seed dataset or ETL command in this repository. Use your external pipeline and the [migration schema](migrations/versions/001_initial_schema.py) as the column reference.

- Store H3 IDs in `grid_id` at the configured resolution.
- Supply valid `latitude` and `longitude` values and consistent state/district labels.
- Populate all six derived scores in the 0–100 range, oriented so higher means better.
- For radius queries, populate `geom` as an SRID 4326 point, using longitude before latitude in `ST_MakePoint`.
- Reconcile `last_updated` before integration: PostgreSQL returns a datetime for this column, while `SiteFeatures` currently declares a string. This can prevent normal rows from loading.

The primary-key lookup requires the exact computed cell. A nearby row does not satisfy missing coverage. The score calculator does not derive the six scores from raw population, infrastructure, or risk measurements.

## 6. Start the service or CLI

```bash
python -m uvicorn main:app --reload --port 8000
```

Open [Swagger UI](http://localhost:8000/docs) for request schemas and [service status](http://localhost:8000/) for a basic response. A successful status response does not confirm database or LLM readiness.

In another activated terminal, run:

```bash
geospatial-co
# Or:
python -m cli
```

The CLI runs the graph directly and does not require the API server. See the [API reference](README.md#api-reference) for examples and workflow limitations.

## 7. Optional graph development

```bash
python -m pip install 'langgraph-cli[inmem]'
langgraph dev
```

The graph ID in `langgraph.json` is `geospatial_co`. This development tool is installed separately and may impose its own runtime requirements. It does not enable PostgreSQL checkpointing in the application's default graph.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Package or command not found | Activate the intended environment and run `python -m pip install -e .` from the repository root. |
| Connection refused / authentication failure | Verify PostgreSQL is running and the host, port, role, password, and database in `DATABASE_URL` are correct. |
| PostGIS extension unavailable | Install PostGIS on the server and ask the database administrator to enable it in the application database. |
| `site_features` missing | Apply `python -m alembic upgrade head` to the same database used by the service. |
| Site not found | Check H3 resolution and exact cell coverage; the nearest route does not search neighboring stored cells. |
| Timestamp validation error | Reconcile `SiteFeatures.last_updated` with the SQL timestamp type. |
| H3 function missing | The implementation calls v4 functions such as `latlng_to_cell`; use H3 v4. |
| Two-site comparison returns no ranking | Known orchestrator routing defect; see README limitations. |
| Hotspots ignore state or result count | Known graph-state propagation gap; current behavior defaults to Gujarat and ten results. |
| Default advice or fallback insight | Inspect provider credentials, model configuration, connectivity, and `logs/agent_app.log`. |
| No persisted graph history | No checkpointer is attached in the default compilation. |

For a database-free numerical check, run the [scoring demonstration](README.md#development-and-validation). There is no automated integration suite in the repository.
