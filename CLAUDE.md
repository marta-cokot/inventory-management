# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for ANY creation or significant modification of `.vue` files
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase structure questions and pattern searches
- **general-purpose**: Use for complex multi-step tasks

### Skills
- **backend-api-test**: Use when writing or modifying tests in `tests/backend/`

### MCP Tools
- **Always use GitHub MCP tools** (`mcp__github__*`) for all GitHub operations
  - Exception: Local-only branches — use `git checkout -b` instead of `mcp__github__create_branch`
- **Always use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Frontend: `http://localhost:3000` | API: `http://localhost:8001`

## Stack
- **Frontend**: Vue 3 + Composition API + Vue Router + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded into memory at startup via `server/mock_data.py`

## Commands

```bash
# Backend — uv is not available in PATH; use the venv directly
cd server
python3 -m venv .venv          # first time only
.venv/bin/pip install -r requirements.txt  # first time only
.venv/bin/python main.py

# Frontend
cd client
npm install                    # first time only
./node_modules/.bin/vite       # or: npm run dev

# Kill existing servers
lsof -ti:3000,8001 | xargs kill -9 2>/dev/null || true

# Backend tests (run from tests/ directory)
cd tests
../server/.venv/bin/pytest backend/ -v

# Run a single test file
../server/.venv/bin/pytest backend/test_inventory.py -v

# Run a single test by name
../server/.venv/bin/pytest backend/test_inventory.py::test_name -v
```

## Architecture

### Data Flow
```
FilterBar (useFilters composable)
  └─→ view components watch filter state
      └─→ api.js builds query params and calls FastAPI
          └─→ main.py applies_filters() / filter_by_month()
              └─→ returns filtered in-memory data as Pydantic-validated JSON
                  └─→ view stores raw data in refs, exposes derived data via computed
```

### Filter System
Four global filters live as module-level `ref`s in `client/src/composables/useFilters.js` (singleton pattern — state is shared across all components that import the composable). Filters: `selectedPeriod` (maps to `month` in API), `selectedLocation` (maps to `warehouse`), `selectedCategory`, `selectedStatus`.

- `getCurrentFilters()` returns the API-ready object; views pass this to `api.js` calls
- Inventory endpoint does **not** support `month` — only `warehouse` and `category`
- Backend date filter supports direct month (`2025-01`) and quarters (`Q1-2025` → `['2025-01','2025-02','2025-03']`)

### Frontend Structure
- **`client/src/views/`** — page-level components (one per route): Dashboard, Inventory, Orders, Demand, Spending, Reports
- **`client/src/components/`** — modal dialogs and shared UI (BacklogDetailModal, CostDetailModal, InventoryDetailModal, ProductDetailModal, ProfileMenu, FilterBar, etc.)
- **`client/src/composables/`** — three singletons: `useFilters` (global filter state), `useAuth` (mock user, always authenticated), `useI18n` (locale + translations)
- **`client/src/api.js`** — all HTTP calls; each method accepts an optional `filters` object and skips params whose value is `'all'`
- **`client/src/utils/currency.js`** — `formatCurrency(amount, currency)` with fixed USD→JPY rate of 150

### i18n System
Custom composable (`useI18n`) — no external library. Locale stored in `localStorage` under `app-locale`. Supported locales: `en`, `ja`. Switching locale also switches displayed currency (USD ↔ JPY). All UI strings use `t('key.path')`. Product names, customer names, and warehouse names have dedicated translation helpers (`translateProductName`, `translateCustomerName`, `translateWarehouse`).

### Backend Structure
All endpoint logic is in `server/main.py`. `server/mock_data.py` loads JSON files from `server/data/` at import time. All data is filtered in Python list comprehensions — no mutations to global lists.

Two shared helpers in `main.py`:
- `apply_filters(items, warehouse, category, status)` — common field filters
- `filter_by_month(items, month)` — handles both `YYYY-MM` and `QN-YYYY`

Pydantic models defined inline in `main.py`; update them whenever `server/data/*.json` structure changes.

### API Endpoints
| Endpoint | Filters supported |
|---|---|
| `GET /api/inventory` | warehouse, category |
| `GET /api/inventory/{id}` | — |
| `GET /api/orders` | warehouse, category, status, month |
| `GET /api/orders/{id}` | — |
| `GET /api/dashboard/summary` | warehouse, category, status, month |
| `GET /api/demand` | — |
| `GET /api/backlog` | — |
| `GET /api/spending/summary` | — |
| `GET /api/spending/monthly` | — |
| `GET /api/spending/categories` | — |
| `GET /api/spending/transactions` | — |
| `GET/POST /api/tasks` | — |
| `PATCH/DELETE /api/tasks/{id}` | — |
| `POST /api/purchase-orders` | — |
| `GET /api/purchase-orders/{backlog_item_id}` | — |

### Tests
Tests use FastAPI `TestClient` via `conftest.py` which adds `server/` to `sys.path`. No mocking — tests hit real in-memory data. `tests/pytest.ini` sets `testpaths = backend`.

## Design System
- **Colors**: Slate/gray palette (`#0f172a`, `#64748b`, `#e2e8f0`)
- **Status colors**: green (delivered/good), blue (in-transit), yellow (pending/warning), red (cancelled/critical)
- **Charts**: Custom SVG; CSS Grid for layouts
- **No emojis in UI**
- **Revenue goals**: $800K/month (single month filter), $9.6M YTD (all months)

## Common Gotchas
1. Use unique keys in `v-for` (`sku`, `id`, `month`) — never array index
2. Validate dates before `.getMonth()`: `const d = new Date(str); if (!isNaN(d)) { ... }`
3. Inventory endpoint ignores `month` filter — don't pass it
4. Updating JSON data structure requires updating the matching Pydantic model in `main.py`
5. `useFilters`, `useAuth`, `useI18n` are singletons — module-level refs persist across component mounts
