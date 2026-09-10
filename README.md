# Insight Orchestra

Insight Orchestra is a self-hostable AI data analysis workspace. Upload a
dataset or connect a database, let a coordinated group of agents clean and
analyze the data, inspect the evidence behind the findings, and ask follow-up
questions in natural language.

The application is designed to keep data processing under your control. You
can use a hosted LLM provider such as OpenAI, Anthropic, or DeepSeek, or run
the LLM locally with Ollama so that data does not need to leave your machine.

## What the project does

Insight Orchestra turns a CSV, spreadsheet, JSON/Parquet file, or supported
database table into an interactive analysis session:

1. **Ingest** - Upload a local file, choose a bundled demo dataset, or connect
   to PostgreSQL, MySQL, SQLite, or DuckDB.
2. **Clean** - Detect duplicates, missing values, outliers, and possible data
   quality or bias issues.
3. **Generate hypotheses** - Produce specific, directional insights grounded
   in descriptive statistics, correlations, and real column values.
4. **Debate and rank** - Score candidate hypotheses for confidence and
   business value, then select the strongest findings.
5. **Visualize** - Generate interactive Plotly charts that explain the
   selected insights.
6. **Ask follow-up questions** - Describe a question in plain English. The NLQ
   agent generates pandas code and runs it in a restricted execution
   environment.
7. **Save and share** - Keep workspaces on the server, export results, or
   create time-limited read-only share links.

Agent progress is streamed to the browser over Server-Sent Events (SSE), so
the UI shows what each stage is doing while a run is in progress.

## Main features

- Natural-language data analysis and follow-up questions
- OpenAI, Anthropic, DeepSeek, and Ollama provider support
- CSV, TSV, Excel, JSON, and Parquet file uploads
- PostgreSQL, MySQL, SQLite, and DuckDB connectors
- Five bundled demo datasets for trying the application immediately
- Data cleaning and quality checks
- Evidence-backed hypothesis generation and LLM-assisted ranking
- Interactive Plotly visualizations
- RestrictedPython sandbox for generated analysis code
- Real-time agent status and output streaming
- Workspace history and server-side session persistence
- HTML, Markdown, and CSV exports
- Read-only share links with an expiration time
- Docker Compose deployment with Redis and optional local Ollama
- Local development overrides for rebuilding the backend and frontend

## Architecture

The project has three application layers and two supporting services:

```text
Browser
  |
  | REST API + Server-Sent Events
  v
Next.js frontend (port 8501)
  |
  v
FastAPI backend (port 8000)
  |
  +-- Agent orchestration and LLM provider abstraction
  +-- Data connectors and upload storage
  +-- Restricted code execution
  +-- Workspace, sharing, and export services
  |
  +--> Redis (session/cache support)
  +--> Ollama (optional local LLM)
  +--> Hosted LLM API (optional alternative)
```

### Agent workflow

| Stage | Agent or service | Responsibility |
| --- | --- | --- |
| 1 | Data Janitor | Cleans data and reports missing values, duplicates, outliers, and quality warnings |
| 2 | Hypothesis Bot | Creates concrete, evidence-backed hypotheses from the cleaned data |
| 3 | Debate Manager | Scores and ranks hypotheses by confidence and business value |
| 4 | Viz Whiz | Selects useful columns and produces Plotly visualizations |
| 5 | Insight Summarizer | Summarizes findings and suggests follow-up questions |
| - | NLQ Agent | Converts natural-language questions into sandboxed pandas analysis |

For the detailed component and data-flow view, see
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md). For the agent implementation and
prompts, see [docs/AGENTS.md](docs/AGENTS.md).

## Prerequisites

### Docker setup

For the recommended setup, install:

- Docker Desktop with Docker Compose v2
- Git
- At least 4 GB RAM; 8 GB or more is recommended when using Ollama
- At least 5 GB free disk space, plus space for the selected Ollama model

On Windows, Docker Desktop must be running. Run the shell scripts from
Git Bash or WSL. PowerShell users can run the equivalent `docker compose`
commands directly.

### Local development

For commands that run outside Docker, use:

- Python 3.11
- Node.js and npm compatible with the installed Next.js version
- GNU Make, or run the commands in the Makefile manually

## Quick start with Docker

### Option 1: clone and run the setup wizard

```bash
git clone https://github.com/alinshaikh/insight-orchestra.git
cd insight-orchestra
./setup.sh
```

The wizard:

- Checks Docker, Docker Compose, the daemon, and required ports.
- Creates `backend/.env` from `backend/.env.example`.
- Lets you choose Ollama, OpenAI, Anthropic, or DeepSeek.
- Starts the backend, frontend, Redis, and Ollama containers.
- Pulls the configured Ollama model when Ollama is selected.

The default Ollama setup is local and does not require an API key:

```bash
./setup.sh --provider ollama -y
```

Cloud provider examples:

```bash
./setup.sh --provider openai --api-key YOUR_OPENAI_KEY -y
./setup.sh --provider anthropic --api-key YOUR_ANTHROPIC_KEY -y
./setup.sh --provider deepseek --api-key YOUR_DEEPSEEK_KEY -y
```

Do not commit API keys or `backend/.env` to source control.

### Option 2: use published images directly

If `backend/.env` already exists, the stack can be started with the published
images from GitHub Container Registry:

```bash
docker compose up -d
```

To use a specific image tag:

```bash
IO_IMAGE_TAG=v1.0.0 docker compose up -d
```

### Option 3: build the application from source

Use the development Compose override when you are changing application code:

```bash
./setup.sh --build
```

This is equivalent to:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

### Open the application

After the containers become healthy:

| Service | Address |
| --- | --- |
| Web application | [http://localhost:8501](http://localhost:8501) |
| Backend API | [http://localhost:8000](http://localhost:8000) |
| Swagger API docs | [http://localhost:8000/docs](http://localhost:8000/docs) |
| Backend health | [http://localhost:8000/health](http://localhost:8000/health) |
| Ollama host port | `http://localhost:11435` |

The first backend start can take a couple of minutes while Python dependencies
load. The Compose health check allows for this cold-start time before the
frontend is started.

## Configuration

The setup script creates `backend/.env`. To configure manually:

```bash
cp backend/.env.example backend/.env
```

Then set one provider and its corresponding credentials.

### LLM providers

```dotenv
# Choose one: openai, anthropic, deepseek, ollama
LLM_PROVIDER=ollama

# Ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=qwen2.5:1.5b

# OpenAI
OPENAI_API_KEY=...
OPENAI_MODEL=gpt-4o-mini
OPENAI_MODEL_FALLBACK=gpt-4o

# Anthropic
ANTHROPIC_API_KEY=...
ANTHROPIC_MODEL=claude-haiku-4-5-20251001

# DeepSeek
DEEPSEEK_API_KEY=...
DEEPSEEK_MODEL=deepseek-chat
DEEPSEEK_BASE_URL=https://api.deepseek.com
```

### Application settings

Common settings include:

| Variable | Default | Purpose |
| --- | --- | --- |
| `BACKEND_HOST` | `0.0.0.0` | Backend bind address |
| `BACKEND_PORT` | `8000` | Backend port |
| `FRONTEND_PORT` | `8501` | Frontend port |
| `ENVIRONMENT` | `production` | Application environment |
| `ALLOWED_ORIGINS` | localhost origins | Comma-separated CORS origins |
| `MAX_FILE_SIZE` | `52428800` | Maximum upload size in bytes |
| `UPLOAD_DIR` | `backend/uploads` | Upload storage directory |
| `LOG_LEVEL` | `INFO` | Backend log level |
| `SANDBOX_ENABLED` | `true` | Enable restricted code execution |
| `SANDBOX_TIMEOUT` | `30` | Generated-code timeout in seconds |
| `SANDBOX_MEMORY_LIMIT` | `256` | Generated-code memory limit in MB |
| `REQUEST_TIMEOUT` | `600` | LLM request timeout in seconds |
| `MAX_RETRIES` | `1` | LLM retry count |

The Compose file supplies container-to-container values for Redis and Ollama.
For a deployment where the browser is not on the same machine as the stack,
set the browser-reachable backend URL:

```bash
PUBLIC_API_URL=http://192.168.1.50:8000 docker compose up -d
```

Set `ALLOWED_ORIGINS` to the real frontend origin in production instead of
leaving broad local-development origins enabled.

## Using the application

1. Open the web application at `http://localhost:8501`.
2. Select one of the demo datasets or upload your own file.
3. For database data, open the database connection flow, enter the connection
   details, inspect the available schema, and choose a table.
4. Start the analysis and follow the agent progress panel.
5. Review the ranked insights and charts in the workspace.
6. Ask a follow-up question in the chat panel.
7. Export the result as HTML, Markdown, or CSV, or create a read-only share
   link when needed.

Supported upload formats are CSV, TSV, Excel, JSON, and Parquet. Database
connections are intended for read-only analysis; use appropriate database
credentials and network restrictions.

## Services and persistent data

The default Compose stack contains:

| Service | Purpose | Container port | Host port |
| --- | --- | --- | --- |
| `backend` | FastAPI API and agent execution | 8000 | 8000 |
| `frontend` | Next.js browser application | 8501 | 8501 |
| `redis` | Session and cache support | 6379 | not published |
| `ollama` | Optional local LLM runtime | 11434 | 11435 |

Persistent data is stored in:

- `backend/uploads` for uploaded datasets and generated local artifacts
- `ollama_data` for downloaded Ollama models
- `redis_data` for Redis persistence

Stop the services without deleting volumes:

```bash
docker compose down
```

Stop the services and remove Compose-managed volumes:

```bash
docker compose down -v
```

The second command deletes local Redis and Ollama data, including downloaded
models. Use it only when that data can be discarded.

## Backend API

The FastAPI service exposes interactive documentation at
[http://localhost:8000/docs](http://localhost:8000/docs).

Important routes include:

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Service health check |
| `GET` | `/config` | Read active non-secret configuration |
| `POST` | `/config` | Update supported runtime configuration |
| `POST` | `/upload` | Upload a dataset |
| `POST` | `/process` | Run the full analysis pipeline |
| `POST` | `/nlq` | Ask a natural-language data question |
| `GET` | `/agents/stream/{session_id}` | Stream agent progress using SSE |
| `GET` | `/demo/list` | List bundled demo datasets |
| `GET` | `/demo/load` | Load a demo dataset |
| `POST` | `/connectors/connect` | Create a database connection |
| `GET` | `/connectors/schema` | Inspect connector schema |
| `POST` | `/connectors/load-table` | Materialize a selected table |
| `POST` | `/connectors/query` | Run a connector query |
| `DELETE` | `/connectors/{connection_id}` | Close a database connection |
| `GET` | `/workspaces` | List saved workspaces |
| `GET` | `/workspaces/{workspace_id}` | Read a workspace |
| `PUT` | `/workspaces/{workspace_id}` | Update a workspace |
| `DELETE` | `/workspaces/{workspace_id}` | Delete a workspace |
| `POST` | `/sessions/share` | Create a share token |
| `GET` | `/sessions/shared/{token}` | Read a shared session |
| `GET` | `/export/{session_id}/html` | Export an HTML report |
| `GET` | `/export/{session_id}/markdown` | Export Markdown |
| `GET` | `/export/{session_id}/csv` | Export tabular results |

The complete request and response reference is in
[docs/API_REFERENCE.md](docs/API_REFERENCE.md).

## Local development

The Makefile provides the common development commands. The default local
backend virtual environment path is `backend/venv`.

```bash
make install
make dev
make test
make lint
make frontend-build
make down
```

`make install` creates a Python 3.11 virtual environment, installs backend
dependencies and test tools, and runs `npm install` in `frontend/`.

If GNU Make is not available, run the equivalent commands manually:

```bash
# Backend
python3.11 -m venv backend/venv
backend/venv/bin/python -m pip install -r backend/requirements.txt
backend/venv/bin/python -m pip install ruff mypy pytest pytest-asyncio pytest-cov

# Frontend
cd frontend
npm install
cd ..
```

On Windows, the virtual environment executable is
`backend\venv\Scripts\python.exe` rather than
`backend/venv/bin/python`. The Docker workflow remains the simplest
cross-platform development path.

### Development commands

| Command | Description |
| --- | --- |
| `make dev` | Build and start the full stack from local source |
| `make up` | Start the published-image stack |
| `make down` | Stop the Compose stack |
| `make logs` | Follow logs from all services |
| `make test` | Run the Python test suite |
| `make lint` | Run Ruff, mypy, ESLint, and TypeScript checks |
| `make format` | Format Python with Ruff |
| `make frontend-build` | Build the Next.js frontend |
| `make clean` | Remove the local virtual environment and caches |

### Test suite

Tests live in [tests/](tests/) and use pytest:

```bash
backend/venv/bin/python -m pytest tests/ -v
```

Use Python 3.11 and the project virtual environment. Running the suite with a
different global Python installation can cause dependency-version collection
errors before tests execute.

### Lint and type checks

```bash
backend/venv/bin/python -m ruff check .
backend/venv/bin/python -m ruff format --check .
backend/venv/bin/python -m mypy backend/app/ --ignore-missing-imports
cd frontend
npm run lint
npx tsc --noEmit
```

## Project structure

```text
insight-orchestra/
├── backend/
│   ├── app/
│   │   ├── api/             # FastAPI routes, connectors, sessions, exports
│   │   ├── services/        # Agents, LLM abstraction, sandbox, workspaces
│   │   ├── connectors/      # Database connector implementations
│   │   └── utils/           # Demo data, file and BigQuery helpers
│   ├── uploads/             # Local upload and generated-artifact directory
│   ├── Dockerfile
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   ├── app/                 # Next.js App Router pages and layout
│   ├── components/          # Upload, chat, agents, charts, export UI
│   ├── package.json
│   └── Dockerfile
├── docs/
│   ├── AGENTS.md            # Agent pipeline details
│   ├── API_REFERENCE.md     # API request and response reference
│   ├── ARCHITECTURE.md      # System architecture
│   └── SETUP.md             # Additional setup and troubleshooting
├── tests/                   # Backend tests
├── docker-compose.yml        # Published-image stack
├── docker-compose.dev.yml    # Local-build override
├── setup.sh                 # Interactive setup and doctor command
├── install.sh               # One-line clone-and-install wrapper
└── Makefile                 # Local development shortcuts
```

## Troubleshooting

### Check the environment

```bash
./setup.sh doctor
```

This checks Docker, Compose, the daemon, ports 8000 and 8501, the environment
file, backend health, and the configured Ollama model.

### View service logs

```bash
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f ollama
```

### Backend is unhealthy

Check the backend logs and health endpoint:

```bash
docker compose ps
curl http://localhost:8000/health
docker compose logs --tail=200 backend
```

The backend imports data-science libraries during startup. Allow the
configured health-check start period to complete before restarting containers.

### Ollama model is missing

Check the configured model and pull it inside the Ollama container:

```bash
docker compose exec ollama ollama list
docker compose exec ollama ollama pull qwen2.5:1.5b
```

If you use a different value for `OLLAMA_MODEL`, pull that exact model name.

### Port 8000 or 8501 is already in use

Stop the process using the port, or change the published port mapping in
`docker-compose.yml` and keep the browser-facing `PUBLIC_API_URL` consistent.

### The browser cannot reach the backend

`PUBLIC_API_URL` is resolved by the browser, not by the frontend container.
When opening the UI from another machine, set it to the backend host's
reachable address:

```bash
PUBLIC_API_URL=http://HOST_OR_IP:8000 docker compose up -d
```

Also add the frontend origin to `ALLOWED_ORIGINS` in `backend/.env`.

## Security notes

- Keep API keys in `backend/.env` or a secret manager; never commit them.
- Use least-privilege, read-only credentials for database analysis.
- Restrict `ALLOWED_ORIGINS` to trusted frontend origins in production.
- Keep `SANDBOX_ENABLED=true` for generated-code execution.
- Do not expose Redis or Ollama publicly unless you have added appropriate
  authentication and network controls.
- Uploaded files and generated artifacts are stored on the backend host.
  Protect the upload directory and back it up according to your data policy.
- The sandbox reduces the capabilities of generated code, but it is not a
  substitute for container, network, identity, and host-level isolation.

## Documentation

- [Setup guide](docs/SETUP.md)
- [Architecture overview](docs/ARCHITECTURE.md)
- [Agent pipeline](docs/AGENTS.md)
- [API reference](docs/API_REFERENCE.md)
- [Contributing guide](CONTRIBUTING.md)
- [License](LICENSE)

## Contributing

1. Fork and clone the repository.
2. Create a feature branch.
3. Run the Docker development stack with `./setup.sh --build`.
4. Install local tooling with `make install`.
5. Run tests and checks with `make test` and `make lint`.
6. Use Conventional Commits, for example:

   ```text
   feat: add a new dataset connector
   fix: handle empty dataframes in the janitor
   docs: clarify Ollama setup
   test: cover connector error handling
   ```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the complete development,
testing, release, and demo-recording workflow.

## License

Insight Orchestra is released under the Apache License 2.0. See
[LICENSE](LICENSE) for the full text.
