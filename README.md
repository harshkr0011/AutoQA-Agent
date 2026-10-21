# AutoQA Agent

AutoQA Agent is an AI-powered end-to-end browser testing platform. Define tests in natural language, organize them by product and suite, run them in a real browser, and review live progress and run artifacts from a web dashboard.

The application consists of a React frontend and a FastAPI backend. The backend stores catalog and run data in PostgreSQL, uses `browser-use` to operate a browser, and connects to a supported language model provider to interpret and execute test instructions.

## Contents

- [Features](#features)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Windows setup](#windows-setup)
- [macOS and Linux setup](#macos-and-linux-setup)
- [Configuration](#configuration)
- [Running the application](#running-the-application)
- [Using the application](#using-the-application)
- [API overview](#api-overview)
- [Docker](#docker)
- [Deploy to Render](#deploy-to-render)
- [Troubleshooting](#troubleshooting)
- [Project structure](#project-structure)
- [Security notes](#security-notes)
- [Contributing](#contributing)
- [License](#license)

## Features

- **Natural-language browser tests:** define ordered browser actions and a human-readable evaluation criterion.
- **Organized test catalog:** manage products, suites, tests, and test steps.
- **Real browser execution:** execute runs using `browser-use` with a selected model.
- **Single-test and suite runs:** run one test or execute a suite of tests.
- **Live run updates:** receive execution progress over Server-Sent Events (SSE).
- **Run history and reports:** inspect status, result, errors, and related artifacts.
- **Artifacts:** save screenshots and GIF recordings for browser runs.
- **Suite import and export:** import suite definitions, including Excel workflows in the frontend, and export suite data.
- **Multiple model providers:** OpenAI, Azure OpenAI, Anthropic Claude, and Google Gemini.

## How it works

```text
React 19 + Vite frontend
          |
          | HTTP API + SSE
          v
FastAPI backend -------- PostgreSQL
          |
          | browser-use Agent + LangChain model adapter
          v
      Web browser
```

The database stores products, suites, tests, ordered steps, runs, and artifact metadata. Screenshots and GIFs are stored under `backend/artifacts/` and exposed through the backend at `/artifacts`.

## Requirements

- Windows, macOS, or Linux
- Node.js 18 or later
- pnpm
- Python 3.11 or later (Python 3.12 is recommended)
- Astral `uv`
- PostgreSQL 15 or later
- An API key for at least one supported model provider to execute AI-driven runs

The frontend can be built without starting PostgreSQL or configuring an LLM key. Database features require PostgreSQL, and executing browser tests requires a working provider key.

## Windows setup

Open **Command Prompt** in the project root. If `pnpm` or `uv` is not recognized after installation, open a fresh terminal. For pnpm installed by npm, add its usual global command directory to the current Command Prompt:

```cmd
set "PATH=%APPDATA%\npm;%PATH%"
```

Install project dependencies:

```cmd
pnpm install:all
```

This installs the JavaScript workspace dependencies and creates/synchronizes the backend virtual environment using `uv`.

If `uv` is installed but not on PATH, run it using its full path, or use the already-created backend virtual environment for backend commands:

```cmd
cd /d C:\path\to\AutoQA-Agent\backend
.\.venv\Scripts\python.exe -m alembic upgrade head
```

See [Configuration](#configuration) to create `backend\.env`, configure PostgreSQL and add a provider key before starting the backend.

## macOS and Linux setup

From the project root:

```bash
pnpm install:all
cp backend/.env.example backend/.env
```

Edit `backend/.env` with your PostgreSQL connection and provider credentials, then apply database migrations and start the services:

```bash
cd backend
uv run alembic upgrade head
cd ..
pnpm dev
```

## Configuration

Create `backend/.env` from the template:

**Windows Command Prompt**
```cmd
copy backend\.env.example backend\.env
notepad backend\.env
```

**macOS / Linux**
```bash
cp backend/.env.example backend/.env
```

Set a PostgreSQL connection string. Replace the example username, password, host, port, and database name with values that match your PostgreSQL installation:

```env
DATABASE_URL=postgresql+psycopg://postgres:YOUR_PASSWORD@localhost:5432/qa_agent
```

Create the database before applying migrations. For example, when using `psql` connected to the `postgres` database as a PostgreSQL administrator:

```sql
CREATE DATABASE qa_agent OWNER postgres;
```

The SQL above belongs in `psql` or pgAdmin's Query Tool. The `DATABASE_URL=...` setting belongs in `backend/.env`, not in the SQL tool.

Choose one or more provider configurations and enter valid credentials:

```env
# OpenAI
OPENAI_API_KEY=

# Azure OpenAI (takes priority over OPENAI_API_KEY for OpenAI-compatible models)
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_API_VERSION=2024-12-01-preview
AZURE_OPENAI_DEPLOYMENT_NAME=

# Anthropic Claude
ANTHROPIC_API_KEY=

# Google Gemini
GOOGLE_API_KEY=
```

Additional settings:

```env
ENV=local
BROWSER_HEADLESS=false
MAX_CONCURRENT_RUNS=3
```

- `ENV=local` enables local development behavior. The default is `local`.
- `BROWSER_HEADLESS=false` shows the browser window; set it to `true` for headless environments.
- `MAX_CONCURRENT_RUNS` controls the maximum number of simultaneous browser sessions (default: `3`).

Provider selection is based on the selected model name:

- `gemini-*` uses `GOOGLE_API_KEY`.
- `claude-*` uses `ANTHROPIC_API_KEY`.
- Other model names use Azure OpenAI when `AZURE_OPENAI_API_KEY` is set; otherwise they use OpenAI via `OPENAI_API_KEY`.

If a database password contains URL-reserved characters (for example `@`, `:`, `/`, or `#`), percent-encode those characters in the connection URL. Keep `.env` private and never commit API keys or database passwords.

## Running the application

### Start both services

From the project root:

```bash
pnpm dev
```

Open:

- Frontend: <http://localhost:5173>
- Backend health check: <http://localhost:8000/api/health>
- Interactive API documentation: <http://localhost:8000/docs>

In local mode, the Vite development server runs the frontend and proxies API and artifact requests to the backend. If port 5173 is occupied, Vite may choose another available port and print it in the terminal; use the URL it prints.

### Start services separately

Frontend, from the project root:

```bash
pnpm dev:frontend
```

Backend, from the project root:

```bash
pnpm dev:backend
```

On Windows, if `uv` is not on PATH but `pnpm install:all` already created the virtual environment, start the backend from a separate Command Prompt:

```cmd
cd /d C:\path\to\AutoQA-Agent\backend
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Keep both terminal windows open while using the application. Apply database migrations after creating the database and before using database-backed features:

```bash
cd backend
uv run alembic upgrade head
```

Windows alternative:

```cmd
cd /d C:\path\to\AutoQA-Agent\backend
.\.venv\Scripts\python.exe -m alembic upgrade head
```

## Using the application

1. Open the frontend and confirm that the dashboard loads.
2. Create a **product** to represent the application or system under test.
3. Create a **suite** under the product.
4. Create **tests** in the suite. Add ordered steps describing what the browser should do and an evaluation criterion describing the expected outcome.
5. Select a supported model and start a test or suite run.
6. Follow live execution updates, then review the result, run history, screenshots, and GIF artifacts.

The browser agent interacts with the target site using the task and test information provided. Use a test environment and test accounts; do not put production credentials or sensitive personal data in test steps.

## API overview

The backend API is available under `/api`. Full request schemas and interactive examples are available at `/docs`.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Backend health check |
| `GET`, `POST` | `/api/products` | List and create products |
| `GET`, `PATCH`, `DELETE` | `/api/products/{product_id}` | Read, update, or delete a product |
| `GET`, `POST` | `/api/suites` | List and create suites |
| `GET`, `PATCH`, `DELETE` | `/api/suites/{suite_id}` | Read, update, or delete a suite |
| `GET` | `/api/suites/{suite_id}/export` | Export a suite definition |
| `POST` | `/api/suites/import` | Import a suite definition |
| `POST` | `/api/suites/{suite_id}/runs` | Start a suite run |
| `GET`, `POST` | `/api/tests` | List and create tests |
| `GET`, `PATCH`, `DELETE` | `/api/tests/{test_id}` | Read, update, or delete a test |
| `GET`, `PUT` | `/api/tests/{test_id}/steps` | List or replace test steps |
| `POST` | `/api/tests/{test_id}/duplicate` | Duplicate a test |
| `POST` | `/api/tests/{test_id}/runs` | Start a test run |
| `GET`, `POST` | `/api/browser-use/runs` | List runs or start an ad-hoc browser run |
| `GET` | `/api/browser-use/runs/{run_id}` | Read run status and result |
| `POST` | `/api/browser-use/runs/{run_id}/cancel` | Cancel a run |
| `GET` | `/api/browser-use/runs/{run_id}/stream` | Subscribe to live run events using SSE |
| `GET` | `/api/browser-use/runs/{run_id}/artifacts` | List run artifacts |
| `GET` | `/api/suite-runs` | List suite runs |
| `GET` | `/api/suite-runs/{suite_run_id}` | Read a suite run and its child runs |
| `GET` | `/api/reports/runs/{run_id}` | Read a run report |

Catalog endpoints support optional query filters such as `product_id` or `suite_id` where applicable.

## Docker

The repository includes a multi-stage `Dockerfile` for building the frontend and backend image. Configure the environment and PostgreSQL connection for your deployment, then build and run the image:

```bash
docker build -t autoqa-agent .
docker run --rm -p 8000:8000 \
  -e DATABASE_URL="postgresql+psycopg://USER:PASSWORD@HOST:5432/DATABASE" \
  -e OPENAI_API_KEY="YOUR_OPENAI_API_KEY" \
  autoqa-agent
```

Replace the placeholders with deployment-specific values. Do not put real secrets into source control or shell history. In the container, the backend serves the built frontend and API on port `8000`; configure PostgreSQL to be reachable from the container. Review the Dockerfile and deployment configuration before using the image in production.

## Deploy to Render

The repository includes a `render.yaml` Blueprint for the **free demo** setup you selected. It creates a Docker-based AutoQA Agent web service and a Render PostgreSQL database, connects the service to the database, configures headless browser execution, and uses `/api/health` as the health check.

### Publish with the Blueprint

1. Push the project to a GitHub or GitLab repository accessible to Render. Keep `backend/.env` and all credentials out of the repository.
2. Sign in at [Render](https://dashboard.render.com/), choose **New → Blueprint**, and connect your repository and deployment branch.
3. Render detects `render.yaml` at the repository root. Review the proposed web service and database, then create the Blueprint.
4. Wait for the PostgreSQL database and web service to finish provisioning and deploying. The Docker container installs Chromium, runs database migrations on startup, and starts the API and built frontend together.
5. In the Render dashboard, open the `autoqa-agent` service and go to **Environment**. Add a valid key for at least one provider:
   - `OPENAI_API_KEY`, or
   - `AZURE_OPENAI_API_KEY` and `AZURE_OPENAI_ENDPOINT`, or
   - `ANTHROPIC_API_KEY`, or
   - `GOOGLE_API_KEY`.
6. Save the environment changes and allow the service to redeploy. Open the service URL shown by Render. Check `<your-service-url>/api/health`; a healthy API returns `{"status":"ok","service":"backend"}`.

The Blueprint does not contain provider credentials. Add secrets through Render's dashboard after creation, and never commit API keys to source control.

### Free plan limitations

This configuration is intended for evaluation, not production:

- A free web service can spin down when idle, so its first request after inactivity may take about a minute.
- The free service filesystem is ephemeral. Screenshots and GIFs saved locally may disappear after restarts or deploys, even though their metadata remains in PostgreSQL.
- Render's free PostgreSQL offering is time-limited and has plan-specific restrictions. Check its expiration in Render and export any data you need before the database expires.
- Browser automation and LLM calls can exceed free instance CPU or memory limits.

For ongoing use, upgrade the web service and PostgreSQL database to paid plans. Attach a persistent disk mounted at `/app/backend/artifacts` to preserve screenshots and GIFs. Keep the database and web service in the same region. Check Render's current [free plan](https://render.com/docs/free), [disk](https://render.com/docs/disks), and [PostgreSQL](https://render.com/docs/postgresql) documentation for up-to-date limits and pricing.

## Troubleshooting

### `pnpm` is not recognized on Windows

Install pnpm and reopen Command Prompt. If npm installed it but the command remains unavailable, add npm's global command directory for the current terminal:

```cmd
set "PATH=%APPDATA%\npm;%PATH%"
pnpm --version
```

### `uv` is not recognized

Reopen the terminal after installing `uv`. If the project virtual environment already exists, use `backend\.venv\Scripts\python.exe` to run Alembic or Uvicorn as shown above.

### Database connection fails

- Confirm PostgreSQL is running and listening on the host and port in `DATABASE_URL`.
- Confirm the database exists and the configured role/password are valid.
- Put `DATABASE_URL` in `backend/.env`; it is not a SQL statement and should not be entered into pgAdmin's Query Tool.
- Run `alembic upgrade head` after the connection and database are ready.
- Restart the backend after changing `.env`.

### Frontend loads but catalog or run data fails

Check the backend terminal for errors and open <http://localhost:8000/api/health>. A successful health check only confirms that the API process is running; catalog endpoints also require a working database.

### Browser runs fail

Confirm that the configured provider key is valid and matches the selected model family. Check the backend terminal for provider or browser startup errors. Start with a harmless test site and non-sensitive test data.

### Port already in use

Close the other process using the port or use the alternative port printed by Vite. Keep the frontend's Vite proxy configuration and backend address consistent when changing ports.

## Project structure

```text
backend/
  alembic/              Database migrations
  app/
    routers/            FastAPI resource and run endpoints
    services/           Browser runner and artifact handling
    main.py             FastAPI application and local/prod frontend behavior
    models.py           SQLAlchemy data models
    schemas.py          Pydantic request and response schemas
  artifacts/            Run screenshots and GIFs (created at runtime)
  pyproject.toml        Python dependencies and project configuration
frontend/
  src/
    components/         Shared UI and run components
    lib/api.ts          Typed frontend API client
    pages/              Dashboard, catalog, run, and report pages
  package.json          Frontend dependencies and scripts
package.json            Workspace scripts
pnpm-workspace.yaml     pnpm workspace configuration
Dockerfile              Multi-stage container build
```

## Contributing

Contributions are welcome. Please keep changes focused, follow existing TypeScript and Python conventions, and run the relevant build or tests before submitting a change.

Common project commands:

```bash
pnpm build
pnpm test:backend
```

The backend test command requires the backend test dependencies and a suitable test environment. Database-dependent tests may require a configured PostgreSQL instance.

## License

This project is licensed under the **GNU Affero General Public License v3.0**. See [LICENSE](LICENSE) for details.
