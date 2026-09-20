# Travel Plan Buddy

A multi-agent travel planner. You type something like *"Plan a 7 day Boston trip from New York under $1500"*, and four LangGraph agents run in sequence — flights, hotels, itinerary, final write-up — and hand back a single formatted plan.

Built as a hands-on exercise in LangGraph state machines, tool calling, and persistent conversation state. The interesting part isn't the travel domain, it's how little glue code a graph-based agent pipeline actually needs.

---

## Project Stages

Built in three stages. Each finished stage is frozen on its own branch:

- **[Part 1 — Basic Multi-Agent Workflow](../../tree/part-1-basic-workflow)** — sequential LangGraph pipeline, no MCP
- **Part 2 — MCP Integration** *(in progress)*
- **Part 3 — Supervisor, Guardrails & HITL** *(planned)*

`main` always holds the latest stage.

---

## What it does

Given one free-text request, the app:

1. Parses origin and destination out of plain English — no dropdowns, no IATA codes required
2. Pulls live flight data from AviationStack
3. Searches the web for hotel options via Tavily
4. Drafts a day-by-day itinerary with a Groq-hosted LLM
5. Reformats everything into a trip summary, budget estimate, and recommendations

Every run is checkpointed to PostgreSQL under a `thread_id`, so a conversation can be resumed instead of restarted.

## How the graph is wired

```text
START → flight_agent → hotel_agent → itinerary_agent → final_agent → END
```

Strictly sequential, on purpose — the itinerary agent needs both the flight and hotel results in state before it can write anything useful.

| Node | What it does | LLM? |
|------|--------------|------|
| `flight_agent` | Resolves the route, calls AviationStack | No — pure tool call |
| `hotel_agent` | Tavily search, trimmed to 5 results | No — pure tool call |
| `itinerary_agent` | Builds the day-by-day plan from state | Yes |
| `final_agent` | Formats the six-section user-facing answer | Yes |

Shared state (`TravelState` in [backend.py](backend.py)) carries `user_query`, `flight_results`, `hotel_results`, `itinerary`, and the running message list. Persistence is handled by `PostgresSaver` — `checkpointer.setup()` creates its own tables on first run, so there's no migration step.

## The location-parsing piece

Flight APIs want IATA codes; people don't write IATA codes. [tools/flight_tool.py](tools/flight_tool.py) handles that with a layered fallback:

- Regex patterns for `from X to Y`, `X → Y`, and destination-only phrasing
- A hand-maintained city → airport map (`new york → JFK`, `tokyo → NRT`, …)
- Country aliases (`usa`, `uk`, `uae`) resolved through `pycountry`
- A full `airportsdata` lookup as the last resort, picking the busiest airport for the country

If only a destination is mentioned, the origin falls back to `DEFAULT_ORIGIN_IATA`.

## Stack

Python 3.11 · LangGraph · LangChain · Groq (`openai/gpt-oss-120b`) · FastAPI + Uvicorn · PostgreSQL · Tavily · AviationStack · Jinja2 + vanilla JS frontend

## Setup

```bash
conda create -n travel python=3.11 -y
conda activate travel
pip install -r requirements.txt
```

Then create a `.env` in the project root:

```env
GROQ_API_KEY=your_groq_key
TAVILY_API_KEY=your_tavily_key
AVIATIONSTACK_API_KEY=your_aviationstack_key
DATABASE_URL=postgresql://user:password@host:5432/dbname
DEFAULT_ORIGIN_IATA=JFK
```

Notes on the database: any PostgreSQL instance works — local, Render, Supabase, Neon. `sslmode=require` is appended automatically if your URL doesn't already specify it, so hosted providers work as-is.

## Running it

Web UI:

```bash
python app.py          # http://127.0.0.1:8000
```

Terminal, if you want to skip the frontend while debugging the graph:

```bash
python test.py
```

Docker:

```bash
docker build -t travel-plan-buddy .
docker run -p 8000:8000 --env-file .env travel-plan-buddy
```

## API

| Method | Route | Purpose |
|--------|-------|---------|
| `GET` | `/` | Chat interface |
| `POST` | `/api/travel` | Run the planner |
| `GET` | `/health` | Health check |

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d '{"message": "Plan a 5 day Tokyo trip from New York in March"}'
```

Pass a `thread_id` to continue an existing conversation; omit it and one is generated for you. The response returns the formatted `answer` plus the raw `flight_results`, `hotel_results`, and `itinerary` from state, which is handy when you want to see what each agent actually contributed.

## Known limits

- AviationStack's free tier returns live flight status, not fares — budget figures in the output are the LLM's estimates, not quoted prices
- Hotel results are web search snippets, not availability or rates
- The city/country airport maps cover major hubs; smaller regional cities fall back to the nearest country-level airport
- No booking, no payments — this plans trips, it doesn't reserve anything

## Project layout

```text
.
├── app.py                  # FastAPI routes and server
├── backend.py              # LangGraph agents, state, checkpointer
├── test.py                 # CLI harness for the graph
├── tools/
│   ├── flight_tool.py      # AviationStack + IATA resolution
│   └── tavily_tool.py      # Web search for hotels
├── templates/index.html
├── static/                 # style.css, script.js
├── Dockerfile
└── requirements.txt
```

## License

MIT — see [LICENSE](LICENSE).
