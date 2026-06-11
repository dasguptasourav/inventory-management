# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded into memory at startup via `server/mock_data.py` — changes don't persist across restarts

## Commands

```bash
# Backend
cd server && uv run python main.py

# Frontend
cd client && npm run dev

# Tests (run from tests/ directory)
cd tests && uv run pytest backend/ -v

# Single test file
cd tests && uv run pytest backend/test_inventory.py -v

# Single test
cd tests && uv run pytest backend/test_inventory.py::test_function_name -v
```

## Architecture

### Filter System (cross-cutting concern)

`useFilters.js` is a module-level singleton — refs are declared outside the function, so all components share one filter state. When filters change, each view re-calls the API with `getCurrentFilters()` which maps `selectedPeriod` → `month` query param and `selectedLocation` → `warehouse` query param.

**Filter support by endpoint:**
- `/api/inventory` — warehouse, category (no month/time dimension)
- `/api/orders` — warehouse, category, status, month
- `/api/dashboard/summary` — all four filters
- `/api/demand`, `/api/backlog` — no filters

### i18n System

`useI18n.js` is also a module-level singleton persisted to `localStorage`. Currency switches automatically with locale (en → USD, ja → JPY). Product names, customer names, and warehouse names require explicit translation via `translateProductName()`, `translateCustomerName()`, `translateWarehouse()` — they are not handled by the `t()` key lookup.

### Data Flow

Vue view → `client/src/api.js` (axios, hardcoded to `http://localhost:8001/api`) → FastAPI → in-memory filtering via `apply_filters()` + `filter_by_month()` → Pydantic response validation

### Backend Filtering

`main.py` has two shared filter utilities: `apply_filters()` for warehouse/category/status and `filter_by_month()` for month/quarter. Month filtering matches against `order_date` using string containment. Quarter support maps Q1–Q4 2025 to month lists via `QUARTER_MAP`.

### Tests

Tests use FastAPI `TestClient` (synchronous). `conftest.py` adds `server/` to `sys.path` and imports `app` directly. `pytest.ini` sets `testpaths = backend` so always run from the `tests/` directory.

## Subagents

- **vue-expert**: Mandatory for creating or significantly modifying `.vue` files
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase structure questions

## Code Style

- Always document non-obvious logic changes with comments

## Key Constraints

- Inventory filters don't support month — no time dimension in inventory data
- Revenue goals: $800K/month single month, $9.6M YTD all months
- Use unique keys in `v-for` (sku, month, id — never index)
- Validate dates before calling `.getMonth()` — some order dates may be null/missing
- Update Pydantic models in `main.py` when changing JSON data structure in `server/data/`
