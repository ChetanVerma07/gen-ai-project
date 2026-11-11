# AgentForge — Autonomous AI Agent Platform (Phase 2: Tool Use & Web Research)

AgentForge is a web-based autonomous AI agent platform. You submit a natural-language goal, and the agent **plans**, **invokes real tools (live web search, page extraction, calculator)**, **tracks evidence & sources**, and **delivers citation-backed final reports** in a real-time dashboard.

---

## What Phase 2 Can Do

- ✅ **Goal Planning**: Converts any goal into a structured, bounded 3–6 task execution plan.
- ✅ **Agentic Tool Calling Loop**: The LLM autonomously determines whether to answer directly or invoke tools iteratively.
- ✅ **Tool 1: `web_search`**: Discovers live information via Tavily API with query normalization, deduplication, and snippet extraction.
- ✅ **Tool 2: `extract_webpage`**: Deep-extracts full page content with strict SSRF defense (blocking private IPs, loopback, and cloud metadata).
- ✅ **Tool 3: `calculator`**: Safe mathematical AST parser/evaluator with zero `eval()` and support for arithmetic, powers, and math functions.
- ✅ **Source Tracking & Evidence Quality**: Persists all retrieved sources, distinguishing between `Page Extracted` and `Search Snippet`.
- ✅ **Citation-Backed Reports**: Generates final reports linking claims to actual retrieved sources `[Title](url)` and concludes with a consolidated references section.
- ✅ **Execution Timeline**: Live dashboard tab tracking tool invocations, inputs, execution duration, and results.
- ✅ **Source Panel**: Inspect all retrieved sources, domains, evidence snippets, and direct links.
- ✅ **Report Export**: Copy Markdown or download the final report as `.md`.
- ✅ **Zero-Cost Mock Mode**: Runs completely offline without API keys for both Gemini and Tavily.
- ✅ **Persistent History**: All runs, tasks, tool calls, and sources persist across page refreshes in PostgreSQL.

## What Phase 2 Does NOT Do (Honest Boundaries)

- ❌ No multi-agent collaboration (belongs to a later phase).
- ❌ No authenticated user account actions (no logging into LinkedIn, GitHub, or emails).
- ❌ No automatic job submissions or email sending (tools are read-only).
- ❌ No arbitrary code execution or shell access.
- ❌ No vector database / RAG semantic memory (planned for Phase 3).

---

## Prerequisites

| Tool        | Version  | Purpose           |
|-------------|----------|-------------------|
| Python      | 3.11+    | Backend runtime   |
| Node.js     | 18+      | Frontend build    |
| PostgreSQL  | 14+      | Database          |
| npm         | 9+       | Package manager   |

---

## Quick Start

### 1. Configure Environment

Copy the template:
```bash
cp .env.example .env
```

#### Option A: Mock Mode (Zero Cost, No API Keys Required)
Ideal for testing, development, and offline demos:
```env
LLM_PROVIDER=mock
SEARCH_PROVIDER=mock
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/agentforge
```

#### Option B: Live AI & Live Web Search
```env
LLM_PROVIDER=gemini
GEMINI_API_KEY=your-gemini-api-key
GEMINI_MODEL=gemini-2.0-flash

SEARCH_PROVIDER=tavily
TAVILY_API_KEY=your-tavily-api-key
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/agentforge
```

- **Gemini API Key**: https://aistudio.google.com/app/apikey
- **Tavily API Key**: https://tavily.com

---

### 2. Database Setup

Create the PostgreSQL database:
```bash
# PostgreSQL CLI
psql -U postgres -c "CREATE DATABASE agentforge;"
```

---

### 3. Backend Setup

```bash
cd backend

# Create & activate virtual environment
# Windows:
python -m venv venv
venv\Scripts\activate
# Linux/macOS:
# python -m venv venv && source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run migrations (creates runs, tasks, tool_calls, and sources tables)
alembic upgrade head

# Start FastAPI server
uvicorn app.main:app --reload --port 8000
```

- API is live at `http://localhost:8000`
- Interactive OpenAPI docs at `http://localhost:8000/docs`

---

### 4. Frontend Setup

In a second terminal:
```bash
cd frontend

# Install dependencies
npm install

# Start Vite development server
npm run dev
```

- Dashboard opens at `http://localhost:5173`
- API calls to `/api/*` are automatically proxied to `http://localhost:8000`

---

### 5. Production Build

```bash
cd frontend
npm run build    # TypeScript type check + Vite production bundle
```

---

## Example Goal: Internship Research

Submit this goal in the AgentForge dashboard:

> "Research five software engineering internships suitable for a second-year BTech student with Python and web development skills. Find the eligibility requirements, application deadlines, company details, and application links. Compare the opportunities and create a report with sources."

### What AgentForge Does:
1. **Plans Tasks**: Creates a logical sequence (Search internships -> Extract eligibility & deadlines -> Compare opportunities & synthesize).
2. **Executes `web_search`**: Discovers live opportunities matching 2nd-year BTech eligibility.
3. **Executes `extract_webpage`**: Reads full page content from official career portals to verify exact deadlines and criteria.
4. **Tracks Sources**: Stores verified URLs, domains, snippets, and marks extraction status (`[Extracted]` or `[Search Snippet]`).
5. **Synthesizes Report**: Creates an Executive Summary, Key Findings, Comparative Evaluation, Limitations, and a `## Sources & Citations` section with clickable markdown links.

---

## Architecture Overview

```
User Goal
   │
   ▼
Planner (LLM) ──────► Execution Plan [Task 1, Task 2, ...]
                           │
                           ▼
                  Executor (Agentic Tool Loop)
                     │
                     ├─► LLM Request: web_search / extract_webpage / calculator
                     ├─► ToolRegistry (Allowlist & Pydantic Validation)
                     ├─► Backend Execution (Tavily Search / Safe AST Evaluator)
                     ├─► Persist ToolCall & Source Records to PostgreSQL
                     └─► Return Tool Output to LLM ──► Task Result
                           │
                           ▼
                  Reporter (LLM) ──► Citation-Backed Report + Sources Section
```

### Registered Tools

| Tool Name | Input Schema | Purpose | Security Controls |
|-----------|--------------|---------|-------------------|
| `web_search` | `query: str`, `max_results: int` | Live web search | Query validation, result deduplication, size limits |
| `extract_webpage` | `url: str` | Deep page content extraction | SSRF blocker (rejects loopback, private IPs, metadata endpoints), timeout, max 8000 chars |
| `calculator` | `expression: str` | Deterministic arithmetic | Safe AST evaluator — strictly no `eval()`, zero arbitrary code execution |

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | Health check probe |
| POST | `/api/runs` | Submit goal & start asynchronous execution |
| GET | `/api/runs` | List recent runs (with tool & source counts) |
| GET | `/api/runs/{run_id}` | Full run record (tasks, tool calls, sources, report) |
| GET | `/api/runs/{run_id}/tools` | Chronological tool call history for a run |
| GET | `/api/runs/{run_id}/sources` | All verified sources retrieved for a run |

---

## Automated Testing

All tests use in-memory SQLite and mock providers — **no paid API keys or network access required**:

```bash
cd backend

# Run all test suites
python -m pytest tests/ -v

# Run specific suites:
python -m pytest tests/test_calculator.py -v       # Safe AST calculator tests
python -m pytest tests/test_tools.py -v            # Tool registry allowlist & validation
python -m pytest tests/test_search.py -v           # Search normalization, SSRF blockers
python -m pytest tests/test_tool_orchestration.py -v # Multi-step agentic loop & citations
python -m pytest tests/test_api.py -v              # API endpoints, tools & sources routes
python -m pytest tests/test_planner.py -v          # Planning engine & validation
python -m pytest tests/test_executor.py -v         # Executor failure recovery
python -m pytest tests/test_reporter.py -v         # Report synthesis & fallback
```

---

## Configuration Reference (`.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `postgresql+asyncpg://...` | PostgreSQL async connection string |
| `LLM_PROVIDER` | `gemini` | `gemini` for real AI or `mock` for zero-cost offline mode |
| `GEMINI_API_KEY` | — | Google Gemini API key |
| `GEMINI_MODEL` | `gemini-2.0-flash` | Gemini model identifier |
| `SEARCH_PROVIDER` | `tavily` | `tavily` for live web search or `mock` for offline mode |
| `TAVILY_API_KEY` | — | Tavily search API key |
| `AGENT_ENABLE_WEB_SEARCH` | `true` | Enable/disable web search tool |
| `AGENT_ENABLE_CALCULATOR` | `true` | Enable/disable calculator tool |
| `MAX_TOOL_CALLS_PER_TASK` | `5` | Maximum tool iterations per task |
| `MAX_TOOL_CALLS_PER_RUN` | `15` | Maximum tool calls across an entire run |
| `TOOL_TIMEOUT_SECONDS` | `30` | Execution timeout per individual tool call |
| `MAX_EXTRACTED_CHARS` | `8000` | Content character limit for webpage extraction |

---

## Troubleshooting

### Tavily API error or missing key
- If you don't have a Tavily key yet, set `SEARCH_PROVIDER=mock` in `.env`.
- If `TAVILY_API_KEY` is empty while `SEARCH_PROVIDER=tavily`, the agent clearly logs that live search is unavailable and continues without crashing.

### Gemini API errors
- Verify your key at https://aistudio.google.com/app/apikey.
- Set `LLM_PROVIDER=mock` to run offline without any Gemini quota.

### SSRF error on page extraction
- The extractor strictly blocks `localhost`, `127.0.0.1`, private subnets (`192.168.x.x`, `10.x.x.x`), and non-HTTP schemes for security. Ensure targets are public `http://` or `https://` URLs.

---

## License

MIT
#   g e n - a i - p r o j e c t  
 #   g e n - a i - p r o j e c t  
 