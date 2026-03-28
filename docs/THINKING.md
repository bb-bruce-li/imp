# imp — Thinking Process Dump

## How we got here: the full decision journey

This document captures the thinking process behind imp's design. It's meant to be read by future contributors, co-founders, or Claude Code sessions that need context on *why* decisions were made, not just *what* was decided.

---

## Phase 1: Problem Discovery

### Starting Point
Bruce (CTO, 2-person ad automation startup) wanted to build a long-term memory system that "clones" each user — capturing preferences, knowledge, understanding, characteristics, environment, everything — so when a user talks to their AI, it always knows them well.

### Key Insight That Changed Everything
The initial framing was about "memory" (storing facts). But through discussion, we realized the real problem isn't memory — it's **cognitive modeling**. The critical examples Bruce gave:

- A CEO agent doesn't care about implementation details — it cares about timeline and cost/profit
- An engineer doesn't care about profit — just wants to make a perfect product  
- A DBA cares about security and normalization
- **Every human engineer has different degrees of understanding** — some can only understand basic code, and for them we should NOT write code they can't read
- For a senior, based on what they perform, the agent can write advanced code and communicate at a higher level — **the communication becomes simpler but more meaningful**

This reframing from "memory" to "cognition" is the entire foundation of imp.

### The 2×2 → 2+2 Principle
Bruce crystallized the core idea in one sentence: "If you don't know 2×2, I just use 2+2."

Same answer (4). Different encoding. Matched to what the user can process. This became the product's core design principle.

---

## Phase 2: Related Work Research

### What We Found

We did extensive research across three searches covering:
1. Persistent memory systems for LLMs
2. Digital twin / AI clone products
3. Cognitive modeling and adaptive communication research

### Landscape Map

**Memory Infrastructure (the plumbing):**
- **Mem0** (YC-backed) — Hybrid datastore (graph + vector + KV). 26% better than OpenAI memory on LOCOMO benchmark. Cloud-first, open source version has degraded features (graph memory is cloud-only). $19-249/mo.
- **Letta/MemGPT** — LLM-as-OS paradigm. Agent self-manages memory like an OS manages RAM/disk. Most sophisticated architecture but no user cognitive modeling.
- **OpenMemory** (CaviraOSS) — Open source cognitive memory engine with HTTP API + MCP.

**Digital Twin Products:**
- **Sentience** — Launched March 26, 2026 (literally during our conversation). $6.5M seed from Bain Capital. Ingests emails, Slack, Apple Notes to create a chatbot that mimics your tone and opinions. Goal: BE you. Not SERVE you.
- **IgniteTech MyPersonas** — Enterprise employee replicas. CES 2026.
- **MindBank AI, Sensay** — Various digital twin products.

**Academic Research:**
- Westhäußer et al. (Oct 2025) — Framework for persistent memory + evolving user profiles
- PersonaLLM (MIT) — LLMs can express Big Five personality traits through prompt conditioning
- Adaptive XAI (Samsung SDS) — Adjusts explanation complexity based on user expertise + cognitive load theory
- ZPD + AI research — Proven that real-time adaptive instruction reduces cognitive load and increases learning

### The Gap We Identified

Nobody is doing:
1. **Implicit cognitive profiling from conversation** — inferring expertise, abstraction preference, decision heuristics automatically
2. **Adaptive output calibration** — same agent producing different code complexity / explanation depth per user
3. **Role-aware value function modeling** — understanding what the user optimizes for
4. **Evolving skill boundary tracking** — detecting "this user now understands X, stop explaining it"

This became imp's positioning: "Mem0 remembers what you said. imp understands how you think."

---

## Phase 3: Product Decisions

### Decision 1: Onboarding
**Options considered:**
- Zero setup (infer everything silently)
- Light onboarding (role + 2-3 questions, then infer)
- Rich profile (user fills out expertise upfront)
- Hybrid (import from existing data + light questions)

**Chosen: Hybrid.** Import from Slack/GitHub/etc for cold-start signal, ask 2-3 calibration questions, then infer everything else silently. Balances speed with accuracy.

### Decision 2: Visibility
**Options considered:**
- Invisible (user never sees the model)
- Transparent (user can view and correct)
- Prompted (system occasionally asks "did I get this right?")
- Dashboard (full visibility with overrides)

**Chosen: Invisible by default.** The adaptation should feel like magic, not like a settings page. The user just notices "this AI gets me." No cognitive overhead of managing a profile. Opt-in visibility is available via CLI (`imp status`, `imp export`) for users who want to inspect or debug their model.

### Decision 3: Target User
**Options considered:**
- Ad platform users only
- General B2B SaaS
- Developer tools
- Standalone product / API layer

**Chosen: Universal layer.** imp sits between human and AI — any tool can use it. This is infrastructure, not a feature. Like MCP itself — a protocol that any tool adopts.

### Decision 4: Deployment
**Initial decision: Fully local, self-hosted, privacy-first.**

**Later pivot: Cloud-hosted + open source self-hosted.**

Reasoning: Bruce initially wanted local-only for privacy. But then realized "storing data in our service is way more convenient." The convenience argument wins for 90% of users. The 10% who care deeply can self-host. This follows the Supabase/PostHog model — open core, cloud default.

Key principle: **Don't lock features behind the cloud.** Cloud and self-hosted have identical capabilities. The only difference is operational convenience.

### Decision 5: Storage
**Chosen: Postgres + pgvector.** One database handles both structured data (cognitive profiles, expertise maps) and vector search (observation embeddings). Bruce already runs Postgres for the ad platform. No additional infrastructure needed.

### Decision 6: Language
**Options considered:**
- Python (safest bet, best LLM ecosystem)
- Go (single binary, easy distribution)
- Rust (maximum performance)

**Chosen: Python.** Bruce knows it best, and the heavy lifting (cognitive inference) happens in Claude's API, not in the service itself. The service is mostly an API relay with persistence logic. Python's library ecosystem matters more than raw performance here.

### Decision 7: Distribution
**Chosen: MCP server as primary distribution.** MCP is becoming the universal plugin protocol for AI tools. imp as an MCP server works with Claude Code, Claude Desktop, Copilot, Cursor, and any future MCP-compatible tool. One integration, every tool.

### Decision 8: Open Source
**Status: Undecided.** We explored multiple options:

- **Fully open source** — build community, imp becomes the standard
- **Proprietary** — keep as secret weapon inside the ad platform
- **Open protocol, closed engine** — open the format/MCP interface, keep extraction logic proprietary

The argument for open source: Mem0 is Apache 2.0 and that's a big reason for its traction. Infrastructure layers win by adoption.

The argument against: competitors can use the same tool. But the counter-argument: the tool isn't the moat, the data is. A competitor who installs imp starts with zero cognitive models.

**Decision deferred.** Architecture works either way. Only repo visibility changes.

### Decision 9: Naming
**Journey:** ACUM (too acronym-y) → Attune (namespace collisions) → imp

**Why imp:**
- Short for imprint (cognitive imprint concept)
- An imp is a small, clever creature that follows you and learns
- 3 letters, easy to type, memorable
- `imp init` / `imp mcp-serve` / `imp export` reads clean as CLI
- Package name: `imp-ai` (avoids conflict with deprecated Python `imp` module)

---

## Phase 4: Architecture Design

### The Cognitive Model (the novel data structure)

Not a flat key-value store. A multi-dimensional model with different update frequencies:

**Slow-changing (monthly):**
- Identity: role, domain
- Value function: priorities, tradeoff style, risk tolerance

**Medium-changing (weekly):**
- Communication profile: abstraction preference, detail tolerance, format preference
- Trust calibration: autonomy comfort, verification tendency

**Fast-changing (daily):**
- Expertise map: per-domain skill level, evidence, ZPD concepts
- Observations: raw signals from conversations

### ZPD (Zone of Proximal Development) — The Core Innovation

Three zones per concept per user:
1. **Mastered** — reference freely, no explanation needed
2. **ZPD Edge** — use with light scaffolding (this is where learning happens)
3. **Beyond** — avoid or translate to simpler terms

The ZPD edge moves over time as the user learns. The system detects promotion ("this user now uses EXPLAIN ANALYZE independently") and graduation ("move this concept from ZPD to mastered").

This is what makes imp fundamentally different from Mem0. Mem0 stores "user knows PostgreSQL." imp tracks "user knows basic SQL, is learning query optimization, and isn't ready for partitioning strategies yet."

### The Communication Contract

Before every AI response, imp generates a contract:
```json
{
  "abstraction_level": "architecture",
  "known_concepts": ["async/await", "CQRS"],
  "zpd_edge": ["causal inference"],
  "avoid_concepts": ["formal verification"],
  "code_level": "advanced",
  "framing": "systems-and-tradeoffs",
  "detail_depth": "moderate",
  "autonomy_level": 0.8
}
```

Any AI tool that reads this contract instantly becomes a better communicator for that specific human. The tool doesn't need to be modified — it just receives a briefing note.

### Signal Extraction Pipeline

The novel part — using Claude API to analyze each user message for implicit cognitive signals:

1. **Vocabulary level** — most advanced concept used correctly
2. **Abstraction level** — outcome / architecture / implementation thinking
3. **Question patterns** — "why" before "how"? Explores or jumps to solutions?
4. **Correction behavior** — vague ("that's wrong") vs precise ("the join order is wrong") — precision = expertise proxy
5. **Decision patterns** — what they optimize for in tradeoffs
6. **Trust signals** — verify, question, or accept AI output?
7. **ZPD signals** — concepts they engage with but don't use independently

---

## Phase 5: Business Model (Speculative)

### Revenue Logic
- Open source builds trust and community (Supabase/PostHog model)
- Cloud hosting provides convenience — this is what most users choose
- Team tier and API access are primary revenue drivers
- No feature locking — cloud and self-hosted are functionally identical

### Pricing (preliminary)
- Free: 1 cognitive model, basic extraction
- Pro ($9-19/mo): full ZPD tracking, unlimited tools, export
- Team ($29-49/person/mo): shared cognitive models, cross-member knowledge graph
- API: usage-based for third-party integration
- Self-hosted: free, full features

### Moat
- **Data network effect** — longer use = more accurate model = harder to leave
- **Cross-tool stickiness** — once connected to Claude + Copilot + Cursor, switching cost is very high
- **Open source community** — contributor-built ingesters make the platform richer

---

## Key Quotes From the Conversation

> "CEO agent doesn't care about the implementation detail, it cares more about timeline and cost profit."

> "Every human engineer has different degree and understanding, some can only understand basic code, for him we should not write the code he can't read."

> "While for a senior, based in what he performs, agent can decide to write advanced code and discuss with him in a high level co-agreement — means the communication will becomes simple but more meaningful."

> "If you don't know 2×2, I just use 2+2."

> "It is just a layer between human and AI, so any tool can use it."

---

## Phase 6: Idea Review (2026-03-28)

### Reframing: Invisible Tutor, Not Adaptation Layer

Through critical review, we reframed imp from an "adaptation layer that adjusts communication style" to an **invisible tutor that makes AI teach while it helps**. The core problem: AI lets juniors produce without understanding. They skip fundamentals, can't debug what AI wrote, and either become dependent or reject AI entirely.

**New tagline:** "AI makes juniors productive. imp makes them competent."

### Three Mechanisms (B+C)

imp's value comes from three things working together:
1. **Level matching (C)** — don't write code the user can't read
2. **Principle inference (B)** — learn engineering values (e.g., "minimal indirection") and generalize to new situations
3. **Pacing** — break solutions into digestible steps, sequence the teaching

Preference detection (A) — "use asyncpg not ORM" — is Mem0 territory. Not imp's differentiator.

### ZPD Is Core, Not Phase 3

ZPD tracking was originally Phase 3 (Polish). But ZPD is the product — it's what detects unknown unknowns and drives scaffolded teaching. Moved to Phase 1.

### Cold-Start: Calibration Is Enough

Debated whether GitHub/Slack ingestion needs to be Phase 1 for time-to-value. Conclusion: no.
- Individual users who find imp are self-aware learners — they have patience
- Team/org deployment is invisible — there's no "first impression" to fail
- 2-3 calibration questions provide enough signal to generate a useful first contract

### Feedback Loop: Evidence Accumulation

No special correction mechanism needed. One bad signal doesn't move the needle if there are 50 prior signals. This is a sampling and weighting problem — with more samples, accuracy improves naturally. Default: err toward more explanation, not less. "Dumbing down" is mild annoyance; talking over heads is harmful.

### Buyer Tiers (Sequenced)

1. **Individual** — self-aware learners install it themselves (launch)
2. **Team** — engineering managers deploy it for their team (revenue)
3. **Platform** — AI tools embed imp natively (scale via partnerships)

### Contract Compliance

Trusting system prompt injection to steer AI behavior. Style/level/pacing adjustments are well within what system prompts reliably control. Not a concern.

### Competitive Moat

- Level matching (C) alone will get eaten by platforms (Claude already has memory)
- Principle inference (B) + cross-tool data is the real moat — platforms optimize for single-conversation quality, not long-term user modeling across tools

---

## Build Order

### Phase 1: Core Loop + ZPD
The minimum viable product is 5 files + MCP server:
1. `extraction.py` — Claude API call that extracts level, principle, and ZPD signals
2. `engine.py` — updates the cognitive model with ZPD promotion/demotion
3. `zpd.py` — ZPD tracking and promotion logic
4. `contract.py` — generates communication contracts with pacing/sequencing
5. `mcp.py` — MCP server exposing observe + get_contract + calibrate

### Phase 2: Cold-Start Acceleration
GitHub + Slack ingesters to bootstrap profiles faster.

### Phase 3: Polish
Contradiction resolution, contract caching, export/import.

### Phase 4: Distribution
PyPI package, cloud service, integration guides, product website.

---

## Open Questions (For Future Sessions)

1. **Extraction prompt tuning** — The signal extraction prompt is the secret sauce. Will need significant iteration against real conversations.
2. **ZPD promotion thresholds** — When exactly does "asked about it" become "knows it"? Needs real user testing.
3. **Contradiction handling UX** — User says "I know K8s" then asks what a pod is. How to downgrade without making them feel judged?
4. **Multi-tool signal merging** — If user talks to Claude Code about Python and Copilot about TypeScript, how to merge signals coherently?
5. **Team dynamics** — How does a shared team cognitive model work? Does the team model inherit from individual models?
6. **Open source vs proprietary** — Final decision still pending. Architecture supports either.

---

*This document should be included in the repo as docs/THINKING.md for anyone who wants to understand the design rationale.*
