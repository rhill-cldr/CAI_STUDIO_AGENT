# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cloudera AI Agent Studio — a full-stack platform for building, testing, and deploying AI agents and workflows. Python gRPC backend + Next.js frontend. Supports CrewAI and LangGraph workflow frameworks. Deploys as a runtime image on Cloudera AI.

## Common Commands

### Testing
```bash
make test                        # Run all tests (studio + workflow engine)
make test-studio                 # Studio Python tests only
make test-workflow-engine        # Workflow engine tests only

# Run a single test file
uv run pytest -v -s tests/path/to/test_file.py

# Run a single test
uv run pytest -v -s tests/path/to/test_file.py::test_function_name
```

### Linting & Formatting
```bash
make lint-fix                    # Auto-fix all formatting (Python + JS)
make lint-check                  # Check-only (CI runs this)

# Python only
uv run -m ruff format studio/
uv run -m autoflake --in-place --remove-all-unused-imports --ignore-init-module-imports --recursive studio/

# JS/TS only
npm run format-fix               # Prettier
npm run lint-fix                  # ESLint --fix
```

### Development
```bash
./bin/local-dev.sh               # Full-stack dev server
npm run dev                      # Frontend only (Next.js + Turbopack)
```

### Protobuf
```bash
./bin/generate-proto.sh          # Regenerate Python + TypeScript from .proto
```
After modifying `studio/proto/agent_studio.proto`, always run this to regenerate `*_pb2.py`, `*_pb2_grpc.py`, `*_pb2.pyi`, and the TypeScript client.

### Database Migrations
```bash
uv run alembic revision --autogenerate -m "description"
uv run alembic upgrade head
uv run alembic downgrade -1
```

### Frontend Tests
```bash
npm run test                     # Jest
npm run test:watch               # Jest watch mode
```

## Architecture

### Backend (Python, `studio/`)
- **gRPC service** (`studio/service.py`): Main API implementing the `AgentStudio` service defined in `studio/proto/agent_studio.proto`. All backend operations are proto-first.
- **Database**: SQLAlchemy ORM + SQLite. Models in `studio/db/model.py`, DAO in `studio/db/dao.py`. Alembic migrations in `alembic/`.
- **Domain modules**: `studio/agents/`, `studio/tools/`, `studio/workflow/`, `studio/models/`, `studio/as_mcp/`, `studio/deployments/` — each handles its entity's CRUD and business logic, called by `service.py`.
- **Workflow engine** (`studio/workflow_engine/`): Separate Python package with its own virtualenv (`.venv` inside that directory). Runs as FastAPI/Uvicorn sidecars on ports 51000+. Supports CrewAI and LangGraph execution.

### Frontend (Next.js 15, `app/`)
- App Router with React 19 and TypeScript. Ant Design + TailwindCSS for UI.
- Next.js API routes in `app/api/` act as a gRPC proxy to the backend.
- State management via Redux Toolkit.
- Generated TypeScript proto client at `studio/proto/agent_studio.ts`.

### Key Ports
| Port | Service |
|------|---------|
| 8090 | Next.js app |
| 50051 | gRPC service |
| 50052 | Phoenix observability |
| 51000+ | Workflow engine runners |

## Code Style

- **Python**: `ruff format` (120 char line length), `autoflake` for unused imports. Targets `studio/` directory.
- **JS/TS**: Prettier (100 print width, single quotes, trailing commas) + ESLint. No `console.log` (use warn/error/info/debug).
- Always use `uv run` instead of bare `python` for Python commands.

## Dual Virtualenv Setup

The main app uses the root `.venv` (managed by `uv sync`). The workflow engine at `studio/workflow_engine/` has its own `.venv`. When running workflow engine tests, the test script handles the virtualenv switch automatically.
