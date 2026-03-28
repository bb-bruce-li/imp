# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

imp is a self-hosted, privacy-first **invisible tutor** that turns every AI interaction into a learning opportunity — without slowing anyone down. It models how users think, detects what they don't know they don't know, and makes AI teach them while it helps them. Unlike memory systems (Mem0, Letta) that store facts, imp models cognition.

Tagline: "AI makes juniors productive. imp makes them competent."

Core principle ("2×2 → 2+2"): same answer, different encoding, matched to what the user can process. Delivered through three mechanisms: **level matching** (don't write code they can't read), **principle inference** (learn engineering values, generalize to new situations), and **pacing** (sequence teaching so users absorb before moving forward).

## Architecture

- **MCP Server** — exposes tools: `imp_observe`, `imp_contract`, `imp_calibrate`, `imp_export`, `imp_status`
- **Extraction Pipeline** — Claude API calls to extract cognitive signals (vocabulary level, abstraction level, question patterns, correction behavior, decision patterns, trust signals, ZPD signals)
- **Cognitive Model Engine** — updates profiles with evidence accumulation, contradiction resolution, ZPD promotion, confidence decay
- **Communication Contract Generator** — produces per-interaction instruction sets injected into AI system prompts
- **Data Ingestion Workers** — GitHub, Slack, email analyzers for cold-start profiles
- **Storage** — PostgreSQL + pgvector (cognitive profiles, observations with embeddings, expertise maps, cached contracts)

## Tech Stack

- Python
- PostgreSQL + pgvector
- Claude API (signal extraction)
- MCP protocol (tool interface)
- SQLAlchemy + Alembic (ORM + migrations)
- Docker Compose (local dev)
- PyPI package name: `imp-ai` (import as `from imp_ai import ...`)

## Key Concepts

- **Cognitive Profile** — slow-changing: role, domain, priorities, tradeoff style, communication preferences, trust calibration
- **Expertise Map** — fast-changing: per-domain skill level with ZPD zone tracking (mastered/zpd_edge/beyond)
- **ZPD (Zone of Proximal Development)** — tracks what users have mastered, what they're learning, and what's beyond them; drives promotion/demotion logic
- **Communication Contract** — generated before each AI response; contains abstraction level, known concepts, ZPD edge, avoid concepts, code complexity, autonomy level
- **Observations** — raw cognitive signals stored with vector embeddings for semantic search

## Project Structure (Target)

```
imp/
├── imp/
│   ├── cli.py                  # CLI: init, connect, export, mcp-serve
│   ├── config.py
│   ├── server/
│   │   ├── mcp.py              # MCP server
│   │   └── api.py              # Optional REST API
│   ├── core/
│   │   ├── models.py           # SQLAlchemy models
│   │   ├── extraction.py       # Signal extraction pipeline
│   │   ├── engine.py           # Cognitive model update engine
│   │   ├── contract.py         # Contract generator
│   │   └── zpd.py              # ZPD tracking/promotion
│   ├── ingestion/
│   │   ├── github.py
│   │   ├── slack.py
│   │   └── email.py
│   └── export/
│       ├── portable.py
│       └── schemas.py
├── migrations/                 # Alembic
└── tests/
```

## Design Documents

- `docs/ARCHITECTURE.md` — full spec: schema, extraction prompts, MCP interface, project phases
- `docs/THINKING.md` — design rationale, competitive landscape, key decisions and why
- `docs/計劃書.md` — Chinese version of the planning document

## Implementation Phases

1. **Core Loop + ZPD** — Postgres schema, extraction pipeline, engine with ZPD promotion/demotion, contract generator with pacing, MCP server, CLI (`init`, `mcp-serve`), calibration flow
2. **Cold-Start Acceleration** — GitHub/Slack ingesters, `imp connect`
3. **Polish** — export/import, contradiction resolution, contract caching, Docker compose
4. **Distribution** — PyPI, Docker image, integration guides
