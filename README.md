# Support Ticket AI System

An AI-powered system for querying customer support ticket data in plain English and automatically detecting anomalies. Built for the DOTMappers AI Engineer assessment.

## Quick Start

```bash
git clone <your-repo-url>
cd support-ticket-ai
pip install -r requirements.txt
cp .env.example .env          # optional: add a free Groq API key
uvicorn app.main:app --reload --port 8000
```

Open `http://localhost:8000` for the UI, or use the REST API directly. No API key is required — see "LLM Strategy" below.

## Architecture

```
support_tickets.csv
        │
        ▼
  app/data_layer.py   ── loads CSV into DuckDB (in-memory), exposes safe SQL execution
        │
        ▼
  app/nl_query.py     ── question → SQL (LLM or rule engine) → execute → phrase answer
  app/anomaly.py       ── rule-based SLA checks + per-category IQR statistical outliers
        │
        ▼
  app/llm_client.py    ── Groq API wrapper, raises LLMUnavailable on any failure
  app/rule_engine.py   ── deterministic NL→SQL fallback (zero-dependency safety net)
        │
        ▼
  app/main.py           ── FastAPI: /query, /anomalies, /health, /stats, and a bundled HTML UI at "/"
```

**Why DuckDB instead of pandas filtering by hand or a vector DB:** the LLM generates plain SQL, which is easy to validate, log, and debug, and DuckDB runs entirely in-process with no server to manage. A vector store would be the wrong tool here — these are structured analytical questions ("average rating per category"), not semantic search over free text.

**Why a two-step generate-SQL-then-phrase-answer pipeline, not a single LLM call that "just answers":** if the LLM is asked to read 500 rows of CSV and answer directly, it can hallucinate numbers. By forcing it to emit SQL, executing that SQL for real, and then asking it to phrase *only* the returned rows into English, the actual numbers in any answer are always traceable to a real query result, never invented.

**Why a rule-based fallback exists in production code, not just as a test stub:** the assessment explicitly requires the system to run at zero cost with no paid APIs, and ideally with no setup friction at all. The rule engine (`app/rule_engine.py`) is a pattern-matching NL→SQL translator that covers counts, averages, top-N rankings, and time/threshold filters — it independently handles all five sample queries in the brief. If `GROQ_API_KEY` is unset, or any Groq call fails for any reason (rate limit, network, key revoked), the system transparently falls back to it for both SQL generation and answer phrasing. This makes the whole system resilient and gradeable offline.

## Model / Tools Used

| Component | Choice | Why |
|---|---|---|
| LLM | Groq free tier, `llama-3.1-8b-instant` | Free, fast (sub-second), OpenAI-style API, generous free quota |
| LLM fallback | Rule-based NL→SQL (`rule_engine.py`) | Zero-cost guarantee, zero network dependency, deterministic |
| Data engine | DuckDB (in-memory) | SQL on a dataframe with no server; safe to validate/sandbox |
| API | FastAPI | Async, typed, auto-docs at `/docs` |
| UI | Single bundled HTML page (no build step) | "At least a minimal UI" requirement, zero extra dependencies |

To use the LLM path: get a free key at https://console.groq.com/keys, put it in `.env` as `GROQ_API_KEY`. Without it, the system still satisfies every functional requirement via the rule engine.

## API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Health check: row count loaded, whether LLM is configured |
| GET | `/stats` | Dataset overview (counts by status/priority/category) |
| POST | `/query` | Body `{"question": "..."}` → natural language answer + generated SQL + result rows |
| GET | `/anomalies?window=all\|month\|week` | Detected anomalies + plain-language summary |
| GET | `/` | Minimal HTML UI (chat box + live anomaly panel) |
| GET | `/docs` | Auto-generated Swagger UI |

## Example Queries and Outputs

These are exact outputs from a live run with no `GROQ_API_KEY` set (rule-engine path), to demonstrate the zero-cost guarantee:

**"How many critical tickets are unresolved?"**
```json
{"answer": "Ticket count: 31", "sql": "SELECT COUNT(*) AS ticket_count FROM tickets WHERE priority = 'Critical' AND status != 'Resolved'"}
```

**"Which agent has the lowest average customer rating?"**
```json
{"answer": "agent_id=AGT-08, avg_rating=3.48, n_rated=25; agent_id=AGT-11, avg_rating=3.48, n_rated=29; ...",
 "sql": "SELECT agent_id, ROUND(AVG(customer_rating), 2) AS avg_rating, COUNT(*) AS n_rated FROM tickets WHERE customer_rating IS NOT NULL GROUP BY agent_id ORDER BY avg_rating ASC LIMIT 5"}
```

**"What is the average customer rating for Technical category tickets?"**
```json
{"answer": "avg_rating=3.74, n=104", "sql": "SELECT ROUND(AVG(customer_rating), 2) AS avg_rating, COUNT(*) AS n FROM tickets WHERE category = 'Technical' AND customer_rating IS NOT NULL"}
```

**GET `/anomalies?window=all`** (abbreviated):
```json
{
  "tickets_considered": 500,
  "sla_breaches": {"count": 80},
  "resolution_time_outliers": {"count": 22},
  "response_time_outliers": {"count": 0},
  "summary": "Out of 500 tickets considered, found 80 unresolved High/Critical tickets breaching the 24h SLA, 22 resolution-time outliers, and 0 response-time outliers."
}
```

## Anomaly Detection Logic

Two complementary, independently-explainable techniques (no ML black box):

1. **SLA breach (rule-based):** any ticket with priority High or Critical, status != Resolved, and age > 24 hours. Direct translation of a real support-ops SLA policy.
2. **Statistical outliers (IQR method):** for `resolution_time_hrs` and `response_time_hrs`, computed **per category** (Billing/Technical/General have different normal ranges), flagging anything above `Q3 + 1.5×IQR`. IQR was chosen over z-score/mean+stdev because resolution times are right-skewed (a handful of very long-tail tickets), and IQR is robust to that skew, whereas a z-score threshold would be dragged around by the same outliers it's trying to detect.

Since the dataset is historical (Jan–Mar 2024), "now" for age/SLA/"this week" calculations is anchored to the dataset's own max `created_at`, not the real wall-clock date — otherwise every ticket would appear infinitely old.

## Known Limitations

- The rule-based fallback covers the query patterns demonstrated in the assessment brief (counts, averages, top-N by agent, threshold/age filters) but is not a general SQL generator — sufficiently novel phrasing will fall through to "I couldn't confidently translate that question" rather than guessing. The LLM path is materially more flexible.
- Anomaly thresholds (24h SLA, 1.5× IQR fence) are reasonable defaults, not tuned against ground-truth labeled anomalies, since none were provided.
- No persistence layer / auth — this is a read-only analytics prototype over a static CSV snapshot, not a live ticketing system integration.
- The bundled UI is intentionally minimal (single HTML file, no build step) to keep the "one command to run" constraint; it is not meant to replace a proper frontend.
- SQL safety is enforced by keyword/structure validation (single SELECT/WITH statement, no DDL/DML keywords), not a full SQL parser — sufficient for this trusted, single-table use case but not a substitute for a real permissions layer in a multi-tenant production system.

## What I'd Improve With More Time

- Add conversation memory so follow-up questions ("...and what about last month?") resolve against prior context.
- Replace the IQR fences with a model that accounts for ticket category, agent, and priority jointly (e.g. quantile regression) rather than one variable at a time.
- Add a confidence/explanation field to `/query` responses so the UI can visually flag "this answer came from the rule-based fallback, not the LLM."
- Containerize with `docker-compose up` as the literal single command (currently `uvicorn` directly satisfies "single command" but Docker would be more portable across evaluator machines).
- Add caching for repeated identical questions, and a small eval set of Q&A pairs to regression-test prompt changes.

## How I'd Scale This

- Swap DuckDB-on-CSV for a real warehouse (Postgres/BigQuery) behind the same `DataLayer` interface — the SQL-generation and safety-validation logic doesn't change.
- Move anomaly detection to a scheduled batch job writing to a `flags` table, instead of recomputing on every request.
- Put the LLM calls behind a queue with retries/backoff and add response caching for repeated questions.
- Add authentication and per-tenant data isolation if multiple support teams share the system.

## Testing

```bash
pip install pytest
python -m pytest tests/ -v
```

13 tests covering data loading, SQL injection/safety guards, all 5+ sample queries from the assessment brief (forced through the rule-engine path to verify offline operation), and anomaly detection correctness (including a unit test directly verifying SLA breach logic against synthetic edge cases).
