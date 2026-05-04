# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Layout

All project files live inside the `inventory-management-main/` subdirectory. The git root is the parent folder. Always operate from within `inventory-management-main/` when running commands.

```
inventory-management-main/   ← project root
├── client/                  ← Vue 3 + Vite frontend (port 3000)
├── server/                  ← Python FastAPI backend (port 8001)
├── tests/backend/           ← pytest test suite
└── scripts/                 ← start/stop shell scripts (macOS/Linux only)
```

## Commands

**Backend** (from `inventory-management-main/server/`):
```bash
uv venv && uv sync      # first-time setup
uv run python main.py   # start server → http://localhost:8001
                        # API docs → http://localhost:8001/docs
```

**Frontend** (from `inventory-management-main/client/`):
```bash
npm install     # first-time setup
npm run dev     # start dev server → http://localhost:3000
npm run build   # production build → client/dist/
```

**Tests** (from `inventory-management-main/tests/`):
```bash
uv run pytest              # all 51 tests
uv run pytest backend/test_inventory.py -v   # single file
uv run pytest --cov        # with coverage
```

> **Windows note:** `scripts/start.sh` and `scripts/stop.sh` are macOS/Linux only. Start each server manually in separate terminals.

## Stack

- **Frontend**: Vue 3 Composition API, Vue Router, Axios, Vite
- **Backend**: Python ≥3.11, FastAPI, Pydantic v2, Uvicorn
- **Package manager**: `uv` (Python), `npm` (Node)
- **Data**: JSON files in `server/data/` loaded into memory at startup — no database

## Architecture & Data Flow

```
FilterBar (useFilters composable)
  → api.js (axios + query params)
  → FastAPI main.py (apply_filters / filter_by_month)
  → In-memory JSON data (mock_data.py)
  → Pydantic response model
  → Vue computed properties → template
```

**4 global filters**: Time Period, Warehouse, Category, Order Status — passed as query params on every API call.

Filter scope limitations:
- `/api/inventory` — no month/time filter
- `/api/demand`, `/api/backlog` — no filters at all

## Key Patterns

**Backend filtering** — always check for `'all'` before filtering; use case-insensitive comparison; never mutate the global list:
```python
if category and category != 'all':
    results = [r for r in results if r['category'].lower() == category.lower()]
```

**Frontend state** — raw data in `ref()`, derived data in `computed()`; loading/error state on every fetch:
```javascript
const items = ref([])
const filteredItems = computed(() => items.value.filter(...))
```

**v-for keys** — always use a stable unique ID (`sku`, `id`, `month`), never array index.

**Date handling** — validate before parsing: `const d = new Date(s); if (!isNaN(d)) { ... }`

## Subagents & Skills

- **vue-expert** subagent — delegate any `.vue` file creation or significant modification
- **code-reviewer** subagent — use after writing significant code
- **backend-api-test** skill — use when writing or modifying tests in `tests/backend/`
- **GitHub MCP tools** (`mcp__github__*`) — use for all GitHub operations (exception: local-only branches use `git checkout -b`)
- **Playwright MCP tools** (`mcp__playwright__*`) — use for browser testing against `http://localhost:3000`

## Design Conventions

- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`) + status colors (green/blue/yellow/red)
- Charts: custom SVG, CSS Grid for layout
- No emojis in UI
- Currency: locale-aware formatting (`toLocaleString`); revenue goal $800K/month, $9.6M YTD
- i18n: English and Japanese via `client/src/locales/`

## Code Comments

Always document non-obvious logic changes with comments.

## Pydantic & Data Consistency

When modifying `server/data/*.json`, update the corresponding Pydantic model in `server/main.py` and restart the server. SKUs in orders must reference valid inventory items; category names must be consistent across data files.
