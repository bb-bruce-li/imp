# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

imp is a self-hosted, privacy-first **cognitive modeling layer** for AI tools. It models how users think — expertise, communication style, decision patterns, learning edge — and exposes that understanding via MCP protocol. Unlike memory systems (Mem0, Letta) that store facts, imp models cognition.

Core principle ("2×2 → 2+2"): same answer, different encoding, matched to what the user can process.

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

1. **Core Loop** — Postgres schema, extraction pipeline, engine, contract generator, MCP server, CLI (`init`, `mcp-serve`)
2. **Onboarding** — calibration flow, GitHub/Slack ingesters, `imp connect`
3. **Polish** — export/import, ZPD promotion thresholds, contradiction resolution, contract caching, Docker compose
4. **Distribution** — PyPI, Docker image, integration guides
