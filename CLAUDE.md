# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System Demo - Full-stack application with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
Use the Task tool with these specialized subagents for appropriate tasks:

- **vue-expert**: Use for Vue 3 frontend features, UI components, styling, and client-side functionality
  - Examples: Creating components, fixing reactivity issues, performance optimization, complex state management
  - **MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert**
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **Explore**: Use for understanding codebase structure, searching for patterns, or answering questions about how components work
- **general-purpose**: Use for complex multi-step tasks or when other agents don't fit

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend` directory with pytest and FastAPI TestClient

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only - use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded via `server/mock_data.py`

## Commands

```bash
# Start both servers at once
./scripts/start.sh

# Backend only
cd server && uv run python main.py

# Frontend only
cd client && npm install && npm run dev

# Run all backend tests
cd tests && uv run pytest -v

# Run a single test file
cd tests && uv run pytest backend/test_inventory.py -v

# Run a single test by name
cd tests && uv run pytest backend/test_dashboard.py::test_dashboard_summary -v

# Run tests with coverage
cd tests && uv run pytest --cov=../server -v
```

API docs are available at `http://localhost:8001/docs` while the server is running.

## Architecture

### Data Flow
Vue filters → `client/src/api.js` → FastAPI → In-memory JSON filtering → Pydantic validation → Computed properties

### Filter System
4 global filters (Time Period, Warehouse, Category, Order Status) are managed by the `useFilters` composable (`client/src/composables/useFilters.js`). All views share this filter state; `api.js` omits any filter set to `'all'` from query params.

- Month filter supports both `YYYY-MM` and quarter format (`Q1-2025` maps to months 1–3)
- Inventory endpoints do NOT support month filtering (no time dimension in inventory data)
- Category matching is case-insensitive on the backend

### Frontend (client/src/)
- **Views** (`views/*.vue`): 7 page-level views — Dashboard is the most complex (~38KB)
- **Components** (`components/`): Reusable modals (BacklogDetail, CostDetail, InventoryDetail, ProductDetail, ProfileDetails, Tasks) + FilterBar, LanguageSwitcher, ProfileMenu
- **Composables**: `useFilters` (shared filter state), `useAuth` (user/auth state), `useI18n` (active language + translation lookup)
- **i18n**: Bilingual EN/JA support via `locales/en.js` and `locales/ja.js`. Currency display converts USD↔JPY at a fixed 150 exchange rate using helpers in `client/src/utils/currency.js` (`formatCurrency`, `formatCurrencyWithDecimals`, `convertAmount`)
- **Reactivity pattern**: Raw API data stored in `ref()` (`allOrders`, `inventoryItems`), derived/filtered data in `computed()`

### Backend (server/)
- All JSON data is loaded into memory at startup; changes do not persist across restarts
- `apply_filters()` in `main.py` handles warehouse, category (case-insensitive), and status filtering
- `filter_by_month()` handles time filtering with quarter-to-month mapping
- Pydantic models are defined alongside their endpoints in `server/main.py`

## API Endpoints
- `GET /api/inventory` — Filters: warehouse, category
- `GET /api/orders` — Filters: warehouse, category, status, month
- `GET /api/dashboard/summary` — All filters; computes inventory value, low stock count, pending/backlog counts
- `GET /api/demand`, `/api/backlog` — No filters
- `GET /api/spending/summary|monthly|categories|transactions`
- `GET /api/reports/quarterly` — Fulfillment rate, avg order value, revenue per quarter
- `GET /api/reports/monthly-trends` — Month-over-month trend data

## Common Issues
1. Use unique keys in `v-for` (not `index`) — use `sku`, `month`, `id`, etc.
2. Validate dates before `.getMonth()` calls
3. Update Pydantic models in `server/main.py` when changing JSON data structure
4. Inventory filters don't support month (no time dimension)
5. Revenue goals: $800K/month single warehouse, $9.6M YTD across all months

## File Locations
- Views: `client/src/views/*.vue`
- Composables: `client/src/composables/`
- API Client: `client/src/api.js`
- i18n translations: `client/src/locales/en.js`, `client/src/locales/ja.js`
- Currency utils: `client/src/utils/currency.js`
- Backend: `server/main.py`, `server/mock_data.py`
- Data: `server/data/*.json`
- Tests: `tests/backend/`
- Global styles / CSS variables: `client/src/App.vue`

## Design System
- Colors: Slate/gray (#0f172a, #64748b, #e2e8f0)
- Status: green (delivered), blue (shipped), yellow (processing), red (backordered/low stock)
- Charts: Custom SVG, CSS Grid for layouts
- No emojis in UI
