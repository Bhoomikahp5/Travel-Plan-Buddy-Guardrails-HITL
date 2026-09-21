# Travel Plan Buddy — Supervisor, Guardrails & HITL

A multi-agent travel planner where a **supervisor agent** decides which specialists to run, a **guardrail** screens the request before any work starts, and a **human approves the draft** before the final plan is written.

Ask for *"a 5 day Tokyo trip from New York under $2000"* and the supervisor runs flights, hotels, weather, and budget. Ask *"what's the weather in Rome next week"* and it runs only the weather agent. Nothing is hardcoded into a fixed sequence.

---

## Project Stages

This is **Part 3** of a three-stage build. The earlier stages live in a separate repo:

- **[Part 1 — Basic Multi-Agent Workflow](https://github.com/Bhoomikahp5/Travel-Plan-Buddy---LangGraph-Multi-Agent-Travel-Planner/tree/part-1-basic-workflow)** — sequential LangGraph pipeline, direct API calls, no MCP
- **[Part 2 — MCP Integration](https://github.com/Bhoomikahp5/Travel-Plan-Buddy---LangGraph-Multi-Agent-Travel-Planner/tree/part-2-MCP)** — Tavily, AviationStack, and a custom weather MCP server
- **Part 3 — Supervisor, Guardrails & HITL** — this repo

Part 3 got its own repo because the architecture changed substantially: the fixed four-step
pipeline was replaced by dynamic supervisor routing, input guardrails, and a pause-for-approval step.

---

## What changed from Part 2

| | Part 2 | Part 3 |
|---|---|---|
| Routing | Fixed: flight → hotel → itinerary → final | Supervisor picks agents per request |
| Agents | 4 | 9 |
| Off-topic requests | Planned anyway | Blocked by guardrail |
| Human input | None | Approve or revise the draft itinerary |
| Budget | LLM guess inside the itinerary | Dedicated budget agent |

## How the graph is wired

```text
                    ┌─ blocked ──────────────────────────→ END
START → supervisor ─┤
                    └─ flight / hotel / weather / budget (only what's needed)
                             ↓
                        itinerary → human_approval ⏸ → final → END
```

Nine nodes in [backend.py](backend.py):

| Node | What it does |
|---|---|
| `supervisor` | Runs the guardrail, extracts trip constraints, picks which specialists to run |
| `guardrail_blocked` | Explains the refusal and ends the run |
| `flight_agent` | Flights via the AviationStack MCP server |
| `hotel_agent` | Hotel search via the Tavily MCP server |
| `weather_agent` | Current conditions and forecast via a custom MCP server |
| `budget_agent` | Cost breakdown against the user's stated budget |
| `itinerary_agent` | Drafts the day-by-day plan |
| `human_approval` | **Pauses** and waits for a person |
| `final_agent` | Writes the polished answer, folding in any feedback |

Routing is done with `add_conditional_edges`. After each specialist finishes, `route_after_agent` checks what's left in `selected_agents` and jumps to the next one, or to the itinerary agent when the list is empty.

## The guardrail

Before any agent runs, the supervisor asks the LLM to classify the request as travel-related or not, returning strict JSON. Off-topic and harmful requests get routed to `guardrail_blocked` and never reach the specialists.

It **fails open**: if the model returns malformed JSON or errors out, the request is allowed through. A formatting hiccup shouldn't break travel planning for everyone.

## Human-in-the-loop

The interesting part. `human_approval_agent` calls LangGraph's `interrupt()`, which genuinely **suspends the graph mid-run** — the process stops, and the whole state is written to PostgreSQL.

The API returns `requires_approval: true` along with the draft itinerary. Nothing further happens until someone calls the approve endpoint with that `thread_id`. The graph then resumes from exactly where it paused.

Approve it and the final agent writes it up. Reject it with feedback and the final agent rewrites accordingly.

This is why the Postgres checkpointer isn't optional here — without it, a paused run would be lost.

## Stack

Python 3.11 · LangGraph · LangChain · MCP · Groq (`openai/gpt-oss-120b`) · FastAPI + Uvicorn · PostgreSQL · Jinja2 + vanilla JS frontend

Three MCP servers, wired in [mcp_client.py](mcp_client.py):

- **Tavily** — remote HTTP server, for hotels and general search
- **AviationStack** — launched locally through `uvx`, for flights
- **Weather** — [custom_weather_mcp_server.py](custom_weather_mcp_server.py), written from scratch on OpenWeather

## Setup

```bash
conda create -n travel python=3.11 -y
conda activate travel
pip install -r requirements.txt
```

Create a `.env` in the project root:

```env
GROQ_API_KEY=your_groq_key
TAVILY_API_KEY=your_tavily_key
AVIATIONSTACK_API_KEY=your_aviationstack_key
OPENWEATHER_API_KEY=your_openweather_key
DATABASE_URL=postgresql://user:password@host:5432/dbname
```

Any PostgreSQL works — local, Render, Supabase, Neon. `sslmode=require` is appended automatically if your URL doesn't specify it.

## Running it

```bash
python app.py          # http://127.0.0.1:8000
```

Docker:

```bash
docker build -t travel-plan-buddy .
docker run -p 8000:8000 --env-file .env travel-plan-buddy
```

The MCP servers are launched as child processes using `sys.executable`, so run the app from the same environment where you installed the requirements.

## API

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/` | Chat interface |
| `POST` | `/api/travel` | Start a plan |
| `POST` | `/api/travel/approve` | Approve or revise a paused plan |
| `GET` | `/health` | Health check |

**Start a plan:**

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d '{"message": "Plan a 5 day Tokyo trip from New York in March under $2000"}'
```

Comes back with `requires_approval: true`, the `draft_itinerary`, and a `thread_id`.

**Approve it:**

```bash
curl -X POST http://127.0.0.1:8000/api/travel/approve \
  -H "Content-Type: application/json" \
  -d '{"thread_id": "user_abc123", "approved": true}'
```

**Or send it back for changes:**

```bash
curl -X POST http://127.0.0.1:8000/api/travel/approve \
  -H "Content-Type: application/json" \
  -d '{"thread_id": "user_abc123", "approved": false, "feedback": "Too packed, make day 3 lighter"}'
```

Rejecting without feedback returns a 400 — the final agent needs to know what to change.

## Known limits

- AviationStack's free tier returns flight status, not fares. Budget figures are LLM estimates, not quotes.
- Hotel results are search snippets, not live availability or rates.
- The guardrail fails open by design, so a malformed LLM response lets an off-topic request through.
- A paused run stays parked in Postgres forever. There's no expiry on unapproved plans.
- No booking, no payments. It plans trips, it doesn't reserve anything.

## Project layout

```text
.
├── app.py                        # FastAPI routes, including the approve endpoint
├── backend.py                    # Supervisor, guardrail, 9-node graph, HITL
├── mcp_client.py                 # Connects to the three MCP servers
├── custom_weather_mcp_server.py  # Custom MCP server on OpenWeather
├── test.py                       # CLI harness
├── templates/index.html
├── static/                       # style.css, script.js
├── Dockerfile
└── requirements.txt
```

## License

MIT — see [LICENSE](LICENSE).
