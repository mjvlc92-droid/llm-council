# LLM Council

![llmcouncil](header.jpg)

**LLM Council** is a local web app that replaces single-model queries with a structured 3-stage deliberation process: multiple LLMs answer independently, anonymously critique each other, and a Chairman synthesizes the final response. Built on OpenRouter to access GPT, Gemini, Claude, Grok, and others through a single unified API.

> **Origin:** Forked and extensively reimagined from [karpathy/llm-council](https://github.com/karpathy/llm-council) — a Saturday vibe-code hack by Andrej Karpathy for side-by-side LLM evaluation while reading books with AI. This fork adds extended documentation and Mermaid diagrams for architectural clarity.

---

## How It Works — 3-Stage Council Protocol

```mermaid
flowchart LR
    subgraph STAGE1["Stage 1: First Opinions"]
        U([User Query]) --> M1[GPT-5.1]
        U --> M2[Gemini 3 Pro]
        U --> M3[Claude Sonnet]
        U --> M4[Grok 4]
        M1 --> R1[Response A]
        M2 --> R2[Response B]
        M3 --> R3[Response C]
        M4 --> R4[Response D]
    end

    subgraph STAGE2["Stage 2: Blind Peer Review"]
        R1 & R2 & R3 & R4 --> ANON[Anonymizer\nA/B/C/D labels]
        ANON --> MR1[GPT ranks others]
        ANON --> MR2[Gemini ranks others]
        ANON --> MR3[Claude ranks others]
        ANON --> MR4[Grok ranks others]
        MR1 & MR2 & MR3 & MR4 --> RANKS[Aggregated Rankings]
    end

    subgraph STAGE3["Stage 3: Chairman Synthesis"]
        STAGE1 & RANKS --> CHAIR[Chairman LLM\ngemini-3-pro]
        CHAIR --> FINAL[Final Response\nto User]
    end
```

**Stage 1 — First Opinions:** All configured LLMs receive the query simultaneously via `asyncio.gather`. Each answers independently without seeing the others' responses.

**Stage 2 — Blind Peer Review:** Each LLM receives all responses with model identities anonymized (`Model A`, `Model B`, etc.) to prevent favoritism. Each ranks by accuracy and insight.

**Stage 3 — Chairman Synthesis:** The designated Chairman LLM receives all original responses plus the peer rankings and compiles a single definitive answer.

---

## Complete Information Flow

```mermaid
flowchart TD
    USER[User types query\nin browser] -->|HTTP POST /api/query| API[FastAPI /api/query\nbackend/routes/council.py]

    API --> GATHER["asyncio.gather(\n  llm_call_1,\n  llm_call_2,\n  llm_call_n\n)"]

    GATHER -->|parallel HTTP| OR1[OpenRouter\ngpt-5.1]
    GATHER -->|parallel HTTP| OR2[OpenRouter\ngemini-3-pro]
    GATHER -->|parallel HTTP| OR3[OpenRouter\nclaude-sonnet]
    GATHER -->|parallel HTTP| OR4[OpenRouter\ngrok-4]

    OR1 & OR2 & OR3 & OR4 --> COLLECT[Collect all responses\nDict model_id → text]

    COLLECT --> ANON2[Anonymize:\nmodel_id → A/B/C/D map]

    ANON2 --> REVIEW_GATHER["asyncio.gather(\n  review_call_1..n\n)"]

    REVIEW_GATHER -->|parallel HTTP| ORR1[OpenRouter\ngpt-5.1 reviews A/B/C/D]
    REVIEW_GATHER -->|parallel HTTP| ORR2[OpenRouter\ngemini reviews A/B/C/D]
    REVIEW_GATHER -->|parallel HTTP| ORR3[OpenRouter\nclaude reviews A/B/C/D]
    REVIEW_GATHER -->|parallel HTTP| ORR4[OpenRouter\ngrok reviews A/B/C/D]

    ORR1 & ORR2 & ORR3 & ORR4 --> AGG[Aggregate rankings\ncompute score per model]

    AGG --> CHAIR_CALL[Call Chairman LLM\nwith all responses + rankings]
    CHAIR_CALL -->|SSE stream| STREAM[Streaming token response]

    STREAM -->|EventSource| BROWSER[Browser renders\nfinal answer in real-time]

    COLLECT --> SAVE[Save to JSON\ndata/conversations/UUID.json]
    AGG --> SAVE
    STREAM --> SAVE
```

---

## SSE Streaming Technical Flow

```mermaid
sequenceDiagram
    participant B as Browser (React)
    participant F as FastAPI Backend
    participant OR as OpenRouter API

    B->>F: POST /api/query {query, models, chairman}
    Note over F: Stage 1 — asyncio.gather all LLM calls
    F->>OR: POST /chat/completions (gpt-5.1)
    F->>OR: POST /chat/completions (gemini-3-pro)
    F->>OR: POST /chat/completions (claude-sonnet)
    F->>OR: POST /chat/completions (grok-4)
    OR-->>F: 4x responses (parallel)
    Note over F: Stage 2 — anonymize + peer review
    F->>OR: 4x review calls (anonymized A/B/C/D)
    OR-->>F: 4x ranking responses
    Note over F: Stage 3 — Chairman synthesis (streaming)
    F->>OR: POST /chat/completions (chairman, stream=true)
    OR-->>F: token stream (SSE)
    F-->>B: text/event-stream
    loop Each token chunk
        F-->>B: data: {"delta": "...", "stage": "chairman"}
    end
    F-->>B: data: {"done": true, "conversation_id": "UUID"}
    Note over B: EventSource reads stream,\nupdates React state token by token
```

---

## System Architecture

```mermaid
graph TB
    subgraph FRONTEND["Frontend — React + Vite (localhost:5173)"]
        CHAT[ChatInterface\nconversation history]
        TABS[TabView\none tab per model]
        STREAM_UI[StreamingText\nrealtime chairman]
        RANKS_UI[RankingTable\npeer scores]
    end

    subgraph BACKEND["Backend — FastAPI (localhost:8000)"]
        ROUTES[routes/council.py\nPOST /api/query\nGET /api/conversations]
        COUNCIL[council.py\norchestrator logic]
        CONFIG[config.py\nCOUNCIL_MODELS\nCHAIRMAN_MODEL]
        STORAGE[storage.py\nJSON persistence]
    end

    subgraph OPENROUTER["OpenRouter — Unified LLM Gateway"]
        GPT[openai/gpt-5.1]
        GEM[google/gemini-3-pro-preview]
        CLA[anthropic/claude-sonnet-4-5]
        GRK[x-ai/grok-4]
    end

    subgraph DATA["Storage"]
        JSON[data/conversations/\n*.json files]
    end

    CHAT -->|fetch| ROUTES
    TABS -->|reads state| COUNCIL
    STREAM_UI -->|EventSource| ROUTES

    ROUTES --> COUNCIL
    COUNCIL --> CONFIG
    COUNCIL -->|httpx async| OPENROUTER
    COUNCIL --> STORAGE
    STORAGE --> JSON
```

---

## Quick Start

### 1. Install Dependencies

The project uses [uv](https://docs.astral.sh/uv/) for Python package management.

**Backend:**
```bash
uv sync
```

**Frontend:**
```bash
cd frontend
npm install
cd ..
```

### 2. Configure API Key

Create a `.env` file in the project root:

```bash
OPENROUTER_API_KEY=sk-or-v1-...
```

Get your key at [openrouter.ai](https://openrouter.ai/).

### 3. Configure the Council (Optional)

Edit `backend/config.py`:

```python
COUNCIL_MODELS = [
    "openai/gpt-5.1",
    "google/gemini-3-pro-preview",
    "anthropic/claude-sonnet-4.5",
    "x-ai/grok-4",
]

CHAIRMAN_MODEL = "google/gemini-3-pro-preview"
```

Any model available on [OpenRouter](https://openrouter.ai/models) can be added.

### 4. Run

**One-command start:**
```bash
./start.sh
```

**Manual:**
```bash
# Terminal 1 — Backend
uv run python -m backend.main

# Terminal 2 — Frontend
cd frontend && npm run dev
```

Open **http://localhost:5173** in your browser.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | React + Vite | Chat UI, tab view per model |
| Frontend | `EventSource` API | SSE streaming of Chairman response |
| Frontend | `react-markdown` | Render LLM markdown output |
| Backend | FastAPI (Python 3.10+) | REST + SSE endpoint |
| Backend | `asyncio.gather` | Parallel LLM calls (Stage 1 + 2) |
| Backend | `httpx` async | HTTP client for OpenRouter |
| LLM Gateway | OpenRouter | Unified API for GPT/Gemini/Claude/Grok |
| Storage | JSON files | Conversation history (`data/conversations/*.json`) |
| Packaging | `uv` + `npm` | Python + JS dependency management |

---

## Project Structure

```
llm-council/
├── backend/
│   ├── main.py           # FastAPI app entry point
│   ├── config.py         # COUNCIL_MODELS + CHAIRMAN_MODEL
│   ├── council.py        # 3-stage orchestration logic
│   ├── routes/
│   │   └── council.py    # POST /api/query, GET /api/conversations
│   └── storage.py        # JSON conversation persistence
├── frontend/
│   ├── src/
│   │   ├── App.jsx       # Root + routing
│   │   ├── ChatInterface.jsx  # Main chat view
│   │   ├── TabView.jsx   # Per-model response tabs
│   │   └── StreamingText.jsx  # SSE Chairman streaming
│   ├── package.json
│   └── vite.config.js
├── data/
│   └── conversations/    # UUID.json per conversation
├── pyproject.toml        # uv project config
├── start.sh              # One-command launcher
└── .env                  # OPENROUTER_API_KEY (not committed)
```

---

## API Reference

| Method | Endpoint | Description |
|--------|---------|-------------|
| `POST` | `/api/query` | Submit query → runs full 3-stage council, streams Chairman via SSE |
| `GET` | `/api/conversations` | List saved conversations |
| `GET` | `/api/conversations/{id}` | Retrieve a specific conversation with all stages |

**POST `/api/query` body:**
```json
{
  "query": "What is the halting problem?",
  "models": ["openai/gpt-5.1", "google/gemini-3-pro-preview"],
  "chairman": "google/gemini-3-pro-preview"
}
```

**SSE events emitted during Chairman streaming:**
```
data: {"stage": "stage1_complete", "responses": {...}}
data: {"stage": "stage2_complete", "rankings": {...}}
data: {"delta": "The ", "stage": "chairman"}
data: {"delta": "halting ", "stage": "chairman"}
data: {"done": true, "conversation_id": "uuid-here"}
```

---

## Why This Pattern Is Useful

- **Reduces single-model blind spots:** each LLM has training biases; parallel responses surface disagreements
- **Anonymous peer review eliminates favoritism:** models can't favor themselves when IDs are hidden
- **Chairman synthesis produces higher-quality answers** than any single model alone on complex reasoning tasks
- **Side-by-side tab view** lets you inspect raw model reasoning before the synthesis

---

*Forked from [karpathy/llm-council](https://github.com/karpathy/llm-council) · Extended with Mermaid architecture diagrams*
