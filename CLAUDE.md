# CLAUDE.md

> This file stacks on top of the workspace root at `C:\Code\GitHub\`:
> - Root [`CLAUDE.md`](../../CLAUDE.md) -- voice, rules, routing map, references, skills, slash commands, conventions.
> - Root [`MEMORY.md`](../../MEMORY.md) -- live facts across repos.
> - Root [`STATUS.md`](../../STATUS.md) -- live PR/CI/security dashboard.
> - [`.claude/resources/`](../../.claude/resources/README.md) -- deep reference for collaboration, workflow, git, OSS, debugging, voice.
>
> Read those first. The guidance below only adds **repo-specific context** -- it does not override anything in the root.


This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

StockSage-AI is an AI/ML-powered Indian stock market (NSE/BSE) prediction, paper trading, and competitive leaderboard platform. Users paper trade with Rs 1,00,000 virtual capital, compete on leaderboards, and benchmark against an autonomous AI trader. See `PLAN.md` for the comprehensive project plan with database schema, API design, ML pipeline, and implementation phases.

## Development Environment

All development runs inside **WSL2 (Ubuntu)** on Windows 11. Do NOT use Windows paths or Windows-native commands. GPU training uses NVIDIA CUDA passthrough from the Windows host into WSL2.

## Commands

### Infrastructure
```bash
docker compose up -d                    # Start PostgreSQL, Redis, API, Frontend
docker compose down                     # Stop all services
docker compose logs -f api              # Tail backend logs
```

### Backend (Python / FastAPI)
```bash
cd backend
uv venv && source .venv/bin/activate    # Create/activate venv
uv pip install -r requirements.txt      # Install backend deps
uv pip install -r requirements-ml.txt   # Install ML deps (PyTorch CUDA, XGBoost, etc.)
uvicorn app.main:app --reload --port 8000  # Run dev server
alembic upgrade head                    # Apply database migrations
alembic revision --autogenerate -m "description"  # Generate migration
ruff check .                            # Lint
ruff format .                           # Format
pytest                                  # Run all tests
pytest tests/unit/test_trading_service.py  # Run single test file
pytest -k "test_market_hours"           # Run tests matching name
```

### Frontend (Next.js)
```bash
cd frontend
pnpm install                            # Install deps
pnpm dev                                # Run dev server on :3000
pnpm build                              # Production build
pnpm biome check .                      # Lint + format check
pnpm biome check --write .             # Lint + format fix
```

### ML Training (GPU)
```bash
cd backend
python -m ml.training.train             # Train directly in WSL2 (fastest for iteration)
docker compose --profile training run --rm ml-trainer  # Train via Docker with GPU
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"  # Verify GPU
```

## Architecture

This is a monorepo with two main applications and a separate ML pipeline:

### Backend (`backend/`)
- **Framework**: FastAPI with Pydantic v2 schemas
- **Pattern**: Routers → Services → Models (3-layer separation)
  - `app/routers/` -- HTTP endpoint definitions, request/response handling
  - `app/services/` -- Business logic (trading rules, portfolio calculations, leaderboard ranking)
  - `app/models/` -- SQLAlchemy ORM models (PostgreSQL)
  - `app/schemas/` -- Pydantic request/response schemas (separate from ORM models)
  - `app/utils/` -- Cross-cutting concerns (JWT auth, market hours/holidays, exceptions)
  - `app/dependencies.py` -- FastAPI dependency injection (get_db, get_current_user)
  - `app/config.py` -- Pydantic Settings for environment variable management
- **Background tasks**: Celery workers (`tasks/`) for scheduled market data fetching, leaderboard recalculation, news sentiment scoring, model retraining
- **Database**: PostgreSQL 18 with TimescaleDB hypertables for `stock_prices` time-series data
- **Auth**: PyJWT (access + refresh tokens), pwdlib with Argon2 for password hashing, Redis for token blacklist

### ML Pipeline (`backend/ml/`)
Separate from the web app, imported by `app/services/ml_service.py` for inference.
- `data/` -- Fetching from yfinance, jugaad-data, Google News RSS
- `features/` -- Technical indicators (pandas-ta), sentiment scores (FinBERT), feature matrix builder
- `models/` -- Model definitions: baseline (sklearn), XGBoost (`device="cuda"`), LSTM (PyTorch CUDA)
- `training/` -- Training pipeline, evaluation metrics, CLI entry point (`python -m ml.training.train`)
- `inference/` -- Model loading + prediction serving, Redis caching
- `artifacts/` -- Saved model files (gitignored, `.pkl`/`.pt` files)

### Frontend (`frontend/`)
- **Framework**: Next.js 16 with App Router, React 19, TypeScript
- **Route groups**: `(auth)/` for login/signup (no sidebar), `(dashboard)/` for protected pages (sidebar layout)
- **Stock charts**: TradingView Lightweight Charts v5 (candlestick/OHLC)
- **Dashboard charts**: Recharts 3.x (portfolio analytics, leaderboard)
- **State**: Zustand stores (`stores/`) for client state, TanStack React Query for server state
- **Styling**: TailwindCSS v4 (CSS-first config, no `tailwind.config.ts`), shadcn/ui components
- **Auth**: Auth.js v5 (next-auth) with middleware protecting dashboard routes

## Key Technical Decisions

- **Indian market specifics**: NSE market hours 9:15 AM - 3:30 PM IST, Mon-Fri. Holiday calendar in `app/utils/market_hours.py`. NSE symbols as primary identifiers (e.g., `RELIANCE`, not BSE numeric codes). yfinance uses `.NS` suffix for NSE, `.BO` for BSE.
- **Time-series splits**: ML training MUST use time-based train/test splits, never random splits (causes data leakage).
- **ML model serving**: Predictions are cached in Redis with 1-hour TTL during market hours. Inference runs on CPU (fast enough for single predictions); training runs on GPU.
- **Paper trading**: Market orders execute at LTP (Last Traded Price). No short selling. Day orders expire at 3:30 PM IST. Orders outside market hours queue for next open.
- **XGBoost GPU**: Use `device="cuda"`, `tree_method="hist"` (XGBoost 3.x syntax). Do NOT use deprecated `gpu_hist` tree method.
- **Password hashing**: Use `pwdlib[argon2]`, NOT passlib (abandoned) or bcrypt. This is the FastAPI-recommended library.
- **JWT**: Use `PyJWT`, NOT `python-jose`. This is the FastAPI-recommended library.
- **Docker Compose**: Use `docker compose` (space, v2 plugin), NOT `docker-compose` (hyphen, legacy).
- **Package managers**: `uv` for Python (not pip), `pnpm` for Node.js (not npm).
- **Linters**: `Ruff` for Python (not black/flake8), `Biome` for TypeScript (not ESLint/Prettier).
- **Legal**: Every prediction page must display a disclaimer that this is not financial advice and is for educational purposes only.