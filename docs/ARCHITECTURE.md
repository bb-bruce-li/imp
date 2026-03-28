# imp — Adaptive Cognitive User Model

## Project Spec v0.1

> "AI makes juniors productive. imp makes them competent."

---

## 1. What This Is

A self-hosted, privacy-first **invisible tutor** that turns every AI interaction into a learning opportunity — without slowing anyone down. imp models how users think, detects what they don't know they don't know, and makes AI teach them while it helps them.

Unlike memory systems (Mem0, Letta) that store facts, imp models *cognition*: what a user knows, how they think, what they optimize for, and where their learning edge is. It exposes this understanding to any AI tool via MCP protocol.

### The Problem

AI lets juniors produce working code without understanding it. They skip fundamentals because AI makes it possible — and they won't go back to learn basics. The only viable learning path is **learning while doing**. But users can't scaffold themselves — they don't know what they don't know, and they won't ask for help with things they don't realize they're missing.

### Core Insight ("2×2 → 2+2")

Same answer, different encoding, matched to what the user can process. imp delivers this through three mechanisms:

1. **Level matching** — don't write code the user can't read, don't explain concepts they already know
2. **Principle inference** — learn underlying engineering values (e.g., "minimal indirection") and generalize to new situations
3. **Pacing** — break solutions into digestible steps, sequence the teaching so users absorb before moving forward

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  AI Tools                        │
│  (Claude, Copilot, Cursor, LangChain agents)    │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │         MCP Client (built-in)            │   │
│  └──────────────┬───────────────────────────┘   │
└─────────────────┼───────────────────────────────┘
                  │ MCP Protocol (local)
                  │
┌─────────────────┼───────────────────────────────┐
│  imp Local Server                               │
│                                                  │
│  ┌──────────────┴───────────────────────────┐   │
│  │           MCP Server Layer                │   │
│  │  Tools: observe, get_contract, calibrate  │   │
│  └──────────────┬───────────────────────────┘   │
│                 │                                │
│  ┌──────────────┴───────────────────────────┐   │
│  │        Extraction Pipeline                │   │
│  │  (Claude API → cognitive signal parsing)  │   │
│  └──────────────┬───────────────────────────┘   │
│                 │                                │
│  ┌──────────────┴───────────────────────────┐   │
│  │       Cognitive Model Engine              │   │
│  │  (update, decay, promote, conflict       │   │
│  │   resolution, ZPD tracking)              │   │
│  └──────────────┬───────────────────────────┘   │
│                 │                                │
│  ┌──────────────┴───────────────────────────┐   │
│  │     Postgres + pgvector (local)           │   │
│  │  (cognitive profiles, evidence store,     │   │
│  │   conversation embeddings)               │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │       Data Ingestion Workers              │   │
│  │  (GitHub, Slack, email importers)         │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## 3. User Flow

### 3.1 Installation

```bash
# Option A: pip
pip install imp-ai
imp init  # creates local Postgres DB, generates config

# Option B: Docker
docker run -v imp-data:/data ghcr.io/imp/imp:latest
```

### 3.2 Onboarding (< 2 minutes active input)

**Step 1: Connect data sources (optional, 30 seconds)**

```bash
imp connect github --token ghp_xxx
imp connect slack --token xoxb_xxx
```

Background workers ingest and build initial profile:
- GitHub: analyze commit messages, PR reviews, code complexity, languages used
- Slack: analyze communication style, vocabulary level, topics discussed, role signals
- Email: analyze priorities, decision patterns, formality level

**Step 2: Light calibration (60-90 seconds)**

On first conversation through any connected AI tool, imp intercepts and asks:

```
Before we start, a few quick questions to help me adapt to you:

1. What's your role? (e.g., CTO, engineer, product manager)
2. When you work with AI, what do you care about most?
   → Speed / Correctness / Learning / Cost
3. How do you prefer explanations?
   → Just the answer / Brief context / Full reasoning
```

These three answers seed the value_function and communication_profile.

**Step 3: Silent adaptation (forever)**

From here, every conversation turn refines the model. The user never thinks about imp again.

### 3.3 Cross-Tool Usage

User adds imp MCP server to any tool:

```json
// Claude Desktop config
{
  "mcpServers": {
    "imp": {
      "command": "imp",
      "args": ["mcp-serve"]
    }
  }
}
```

```json
// Copilot / Cursor MCP config (same protocol)
{
  "mcpServers": {
    "imp": {
      "command": "imp",
      "args": ["mcp-serve"]
    }
  }
}
```

Every connected tool reads from and writes to the same cognitive model.

### 3.4 Export / Portability

```bash
imp export --format json --output my-cognitive-model.json
imp export --format portable --output my-model.imp
imp import --file my-model.imp  # on a new machine
```

---

## 4. Postgres Schema

### 4.1 Core Tables

```sql
-- The user (supports multi-user for team/family setups)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id TEXT,                  -- identifier from MCP client or auth provider
    display_name TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    calibration_complete BOOLEAN DEFAULT false,

    UNIQUE(external_id)
);

-- Slow-changing identity and value function
CREATE TABLE cognitive_profile (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    
    -- Identity
    role TEXT,                          -- 'CTO', 'engineer', 'designer'
    domain TEXT,                        -- 'adtech', 'healthcare', 'fintech'
    
    -- Value function (what they optimize for)
    priorities JSONB DEFAULT '[]',     -- ['cost', 'time_to_market', 'reliability']
    tradeoff_style TEXT,               -- 'pragmatic', 'perfectionist', 'balanced'
    risk_tolerance FLOAT DEFAULT 0.5,  -- 0=conservative, 1=aggressive
    
    -- Communication profile
    abstraction_preference TEXT,        -- 'outcome', 'architecture', 'implementation'
    detail_tolerance TEXT,              -- 'minimal', 'moderate', 'exhaustive'
    format_preference TEXT,             -- 'prose', 'bullets', 'code_first', 'visual'
    explanation_style TEXT,             -- 'just_answer', 'brief_context', 'full_reasoning'
    
    -- Trust calibration
    autonomy_comfort FLOAT DEFAULT 0.5, -- how much they let AI decide
    verification_tendency TEXT,          -- 'trusts', 'spot_checks', 'verifies_everything'
    
    -- Metadata
    confidence FLOAT DEFAULT 0.0,      -- how confident we are in this profile
    last_updated TIMESTAMPTZ DEFAULT now(),
    evidence_count INT DEFAULT 0,
    
    UNIQUE(user_id)
);

-- Fast-changing expertise map (per domain/skill)
CREATE TABLE expertise_map (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    
    domain TEXT NOT NULL,               -- 'postgresql', 'react', 'kubernetes'
    level TEXT NOT NULL,                 -- 'novice', 'beginner', 'intermediate', 'advanced', 'expert'
    
    -- ZPD tracking
    zone TEXT DEFAULT 'unknown',        -- 'mastered', 'zpd_edge', 'beyond'
    zpd_concepts JSONB DEFAULT '[]',   -- concepts at their learning edge in this domain
    
    -- Evidence
    evidence JSONB DEFAULT '[]',       -- [{"signal": "used EXPLAIN ANALYZE", "date": "...", "confidence": 0.8}]
    positive_signals INT DEFAULT 0,     -- times they demonstrated competence
    negative_signals INT DEFAULT 0,     -- times they needed help
    
    -- Temporal
    first_observed TIMESTAMPTZ DEFAULT now(),
    last_observed TIMESTAMPTZ DEFAULT now(),
    last_promoted TIMESTAMPTZ,          -- when level was last upgraded
    last_demoted TIMESTAMPTZ,           -- when level was last downgraded
    
    UNIQUE(user_id, domain)
);

-- Individual evidence observations (the raw signals)
CREATE TABLE observations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    
    -- What was observed
    signal_type TEXT NOT NULL,          -- 'vocabulary_use', 'concept_reference', 'question_pattern',
                                       -- 'correction_behavior', 'abstraction_level', 'decision_pattern'
    signal_value JSONB NOT NULL,        -- the actual observation
    domain TEXT,                         -- which expertise domain this relates to
    
    -- Context
    source TEXT,                         -- 'conversation', 'github', 'slack'
    source_ref TEXT,                     -- conversation_id, commit_sha, etc.
    
    -- Confidence
    confidence FLOAT DEFAULT 0.5,      -- how sure we are about this signal
    
    -- Embedding for semantic search (dimension depends on chosen model)
    embedding vector(1024),             -- pgvector embedding of the observation
    
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Conversation contracts (cached, for fast retrieval)
CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    topic TEXT,                          -- topic/domain of the current conversation
    
    contract JSONB NOT NULL,            -- the full communication contract
    
    generated_at TIMESTAMPTZ DEFAULT now(),
    valid_until TIMESTAMPTZ             -- contracts expire and regenerate
    -- TTL: default 1 hour; invalidated early when new observations
    -- shift expertise level or communication preferences
);

-- Data source connections
CREATE TABLE data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    
    source_type TEXT NOT NULL,          -- 'github', 'slack', 'email'
    config JSONB NOT NULL,              -- encrypted connection config
    last_synced TIMESTAMPTZ,
    sync_status TEXT DEFAULT 'pending',
    
    UNIQUE(user_id, source_type)
);

-- Indexes
CREATE INDEX idx_expertise_user ON expertise_map(user_id);
CREATE INDEX idx_expertise_domain ON expertise_map(user_id, domain);
CREATE INDEX idx_observations_user ON observations(user_id);
CREATE INDEX idx_observations_type ON observations(user_id, signal_type);
CREATE INDEX idx_observations_embedding ON observations USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_contracts_user_topic ON contracts(user_id, topic);
```

---

## 5. Extraction Pipeline

**Note:** Each user message triggers a Claude API call for signal extraction. Consider batching low-signal messages (e.g., "thanks", "ok") and skipping extraction when no cognitive signal is likely. This controls both cost and rate limit exposure.

### 5.1 Per-Turn Extraction Prompt

This is the core intelligence — the Claude API call that processes each conversation turn and extracts cognitive signals.

```python
EXTRACTION_SYSTEM_PROMPT = """
You are a cognitive signal extractor. Given a user's message in a conversation,
analyze it for implicit signals about how they think, what they know, and how
they prefer to communicate.

You are NOT extracting facts or preferences (like "user likes Python").
You ARE extracting cognitive patterns:

1. VOCABULARY_LEVEL: What's the most advanced concept they use correctly?
   What concepts do they reference without needing explanation?
   
2. ABSTRACTION_LEVEL: Are they thinking at the outcome level ("make it faster"),
   architecture level ("we need a caching layer"), or implementation level
   ("add a Redis LRU with 512MB max")?
   
3. QUESTION_PATTERN: Do they ask "why" before "how"? Do they explore the problem
   space or jump to solutions? Do they ask clarifying questions or accept ambiguity?
   
4. CORRECTION_BEHAVIOR: If they correct the AI, how precise is the correction?
   Vague ("that's not right") vs precise ("the join order is wrong, you need to
   drive from the smaller table") — precision of correction = expertise proxy.
   
5. DECISION_PATTERN: When facing tradeoffs, what do they optimize for?
   Speed vs correctness? Cost vs reliability? Simplicity vs power?
   
6. TRUST_SIGNAL: Do they verify, question, or accept the AI's output?
   Do they ask for sources? Do they push back?

7. ZPD_SIGNAL: Is there a concept they're engaging with but not yet fluent in?
   They ask good questions about it, follow explanations, but don't use it
   independently. This is their learning edge.

Respond ONLY with a JSON array of signals. Each signal:
{
  "type": "vocabulary_use|abstraction_level|question_pattern|correction_behavior|decision_pattern|trust_signal|zpd_signal",
  "domain": "the technical/knowledge domain if applicable",
  "value": "description of the specific observation",
  "concepts": ["relevant", "concepts", "mentioned"],
  "level_indicator": "novice|beginner|intermediate|advanced|expert",
  "confidence": 0.0-1.0,
  "zpd_zone": "mastered|zpd_edge|beyond|null"
}

If the message contains no cognitive signals (e.g., "hello", "thanks"), return [].
Do not over-extract. Only log signals you are genuinely confident about.
"""

EXTRACTION_USER_PROMPT = """
Current user profile summary (for context):
{profile_summary}

Previous messages in this conversation (last 3 turns):
{recent_context}

New message from user:
{user_message}

Extract cognitive signals from the new message only.
"""
```

### 5.2 Profile Update Logic

```python
def update_cognitive_model(user_id: str, signals: list[dict]):
    """
    Takes extracted signals and updates the cognitive model.
    Handles:
    - Evidence accumulation (multiple signals strengthen a conclusion)
    - Contradiction resolution (new signal conflicts with existing model)
    - ZPD promotion (concept moves from zpd_edge to mastered)
    - Confidence decay (old signals lose weight over time)
    """
    
    for signal in signals:
        # 1. Store the raw observation
        store_observation(user_id, signal)
        
        # 2. Update expertise map if domain-specific
        if signal.get('domain'):
            update_expertise(
                user_id=user_id,
                domain=signal['domain'],
                level_indicator=signal['level_indicator'],
                zpd_zone=signal.get('zpd_zone'),
                concepts=signal.get('concepts', []),
                confidence=signal['confidence']
            )
        
        # 3. Update cognitive profile if it's a meta-signal
        if signal['type'] in ('abstraction_level', 'decision_pattern', 'trust_signal'):
            update_profile(user_id, signal)


def update_expertise(user_id, domain, level_indicator, zpd_zone, concepts, confidence):
    """
    Expertise update rules:
    
    PROMOTION: If positive_signals > 5 and last 3 signals all indicate higher level
               → promote level, move current zpd_concepts to mastered
               
    DEMOTION:  If negative_signals > 3 in a row for a domain
               → soft demote (lower confidence, don't change level immediately)
               → hard demote only after 5+ negative signals
               
    ZPD UPDATE: If a concept appears in zpd_edge and user demonstrates fluency
                → graduate it from zpd_edge
                → add newly detected edge concepts
                
    CONFLICT:   User claims expertise but demonstrates gaps
                → lower confidence score, don't change level
                → track as "claimed_vs_demonstrated" gap
    """
    pass  # Implementation follows


def generate_contract(user_id: str, topic: str = None) -> dict:
    """
    Generates a communication contract for the current interaction.
    This gets injected into the AI's system prompt.
    """
    profile = get_cognitive_profile(user_id)
    expertise = get_relevant_expertise(user_id, topic)
    
    contract = {
        # How to communicate
        "abstraction_level": profile.abstraction_preference,
        "detail_depth": profile.detail_tolerance,
        "format": profile.format_preference,
        "explanation_style": profile.explanation_style,
        
        # What they already know (don't explain these)
        "known_concepts": get_mastered_concepts(user_id, topic),
        
        # What they're learning (scaffold these)
        "zpd_edge": get_zpd_concepts(user_id, topic),
        
        # What's beyond them (avoid or translate)
        "avoid_concepts": get_beyond_concepts(user_id, topic),
        
        # How to frame information
        "framing": derive_framing(profile.priorities, profile.role),
        
        # Code generation settings
        "code_level": derive_code_level(expertise),
        "comment_density": derive_comment_density(expertise),
        
        # Autonomy
        "autonomy_level": profile.autonomy_comfort,
        "needs_confirmation": profile.verification_tendency == 'verifies_everything'
    }
    
    # Cache it
    cache_contract(user_id, topic, contract)
    
    return contract
```

---

## 6. MCP Server Interface

### 6.1 Tools Exposed

User identity is resolved from the `IMP_USER_ID` environment variable (set in MCP server config). Defaults to `"default"` for single-user setups.

```python
# Tool 1: observe — called after every user message
@mcp_tool(name="imp_observe")
def observe(user_message: str, conversation_context: str = "", topic: str = ""):
    """
    Process a user message and update their cognitive model.
    Call this after every user message in the conversation.
    """
    user_id = get_user_id()  # resolved from IMP_USER_ID env var
    signals = extract_signals(user_message, conversation_context)
    update_cognitive_model(user_id, signals)
    return {"status": "ok", "signals_detected": len(signals)}


# Tool 2: get_contract — called before generating any response
@mcp_tool(name="imp_contract")  
def get_contract(topic: str = ""):
    """
    Get the communication contract for the current user.
    Returns instructions on how to adapt the response to this specific user.
    """
    contract = generate_contract(get_user_id(), topic)
    return contract


# Tool 3: calibrate — called during onboarding
@mcp_tool(name="imp_calibrate")
def calibrate(role: str, priority: str, explanation_style: str):
    """
    Set initial calibration from onboarding questions.
    """
    update_calibration(get_user_id(), role, priority, explanation_style)
    return {"status": "calibrated"}


# Tool 4: export — user requests their model
@mcp_tool(name="imp_export")
def export():
    """
    Export the user's complete cognitive model as portable JSON.
    """
    return export_cognitive_model(get_user_id())


# Tool 5: status — diagnostic
@mcp_tool(name="imp_status")
def status():
    """
    Return a summary of what imp knows about the current user.
    Confidence levels, expertise domains, and model completeness.
    """
    return get_model_summary(get_user_id())
```

### 6.2 System Prompt Injection

When an AI tool fetches the contract, it should prepend this to its system prompt:

```python
CONTRACT_PROMPT_TEMPLATE = """
[imp Communication Contract for this user]

Adapt your response according to these parameters:
- Abstraction level: {abstraction_level}
- Detail depth: {detail_depth}  
- Explanation style: {explanation_style}
- Response format preference: {format}

Concepts this user has mastered (reference freely, no explanation needed):
{known_concepts}

Concepts at this user's learning edge (use with light scaffolding):
{zpd_edge}

Concepts beyond this user right now (avoid or translate to simpler terms):
{avoid_concepts}

Frame information in terms of: {framing}
Code complexity level: {code_level}
Comment density: {comment_density}

{autonomy_instructions}
"""
```

---

## 7. Data Ingestion Workers

### 7.1 GitHub Ingester

```python
def ingest_github(user_id: str, token: str):
    """
    Analyzes:
    - Commit messages → communication style, technical vocabulary
    - PR reviews given → expertise level, how they explain things
    - Code authored → languages, complexity level, patterns used
    - Issues filed → abstraction level, problem framing style
    """
    pass


def analyze_code_complexity(code: str, language: str) -> dict:
    """
    Returns signals about the author's coding level:
    - Uses advanced language features?
    - Error handling patterns?
    - Naming conventions?
    - Architectural patterns?
    """
    pass
```

### 7.2 Slack Ingester

```python
def ingest_slack(user_id: str, token: str):
    """
    Analyzes:
    - Message vocabulary → technical depth, domain expertise
    - Question/answer ratio → are they the expert or the learner?
    - Response length → communication style (terse vs verbose)
    - Channel participation → domain interests, role signals
    - Emoji/reaction patterns → communication formality
    """
    pass
```

---

## 8. Example: Same Agent, Two Users

### User A: CEO (Sarah)
```json
{
  "abstraction_level": "outcome",
  "detail_depth": "minimal",
  "known_concepts": ["ROI", "CAC", "LTV", "unit economics"],
  "zpd_edge": ["attribution modeling", "incrementality testing"],
  "avoid_concepts": ["SQL joins", "API rate limits", "caching strategies"],
  "framing": "cost-and-revenue",
  "code_level": "none",
  "explanation_style": "just_answer"
}
```

**Sarah asks:** "How are our Meta campaigns performing?"

**Agent response with imp:**
> Meta spend is up 12% this month but CPA dropped 8%, so we're getting more efficient. The top performer is the retargeting lookalike audience — it's driving 40% of conversions at half the cost of prospecting. One flag: the attribution window change Meta made last week might be inflating these numbers. I'd recommend running an incrementality test before scaling further.

---

### User B: Engineer (Bruce)
```json
{
  "abstraction_level": "architecture",
  "detail_depth": "moderate",
  "known_concepts": ["PostgreSQL", "ZFS", "async/await", "CQRS", "causal inference"],
  "zpd_edge": ["multi-agent orchestration", "LangGraph state management"],
  "avoid_concepts": [],
  "framing": "systems-and-tradeoffs",
  "code_level": "advanced",
  "explanation_style": "brief_context"
}
```

**Bruce asks:** "How are our Meta campaigns performing?"

**Agent response with imp:**
> Meta CPA is down 8% on 12% higher spend — the synthetic control attribution model confirms this is real lift, not just Meta's inflated reporting. The retargeting lookalike audience is the main driver. One thing to watch: Meta changed their attribution API response schema last week, our ingestion pipeline is handling it but I'd verify the conversion event mapping in the `meta_events` table hasn't drifted. The aggregation query for this report is hitting the slow path — might be worth checking if the materialized view refresh is still on schedule.

---

Same question. Same data. Completely different communication, shaped by each user's cognitive model.

---

## 9. Project Structure

```
imp/
├── pyproject.toml
├── README.md
├── docker-compose.yml          # Postgres + pgvector + imp
│
├── imp/
│   ├── __init__.py
│   ├── cli.py                  # CLI: init, connect, export, mcp-serve
│   ├── config.py               # Local config management
│   │
│   ├── server/
│   │   ├── mcp.py              # MCP server implementation
│   │   └── api.py              # Optional REST API (for non-MCP tools)
│   │
│   ├── core/
│   │   ├── models.py           # SQLAlchemy models
│   │   ├── extraction.py       # Signal extraction pipeline
│   │   ├── engine.py           # Cognitive model update engine
│   │   ├── contract.py         # Communication contract generator
│   │   └── zpd.py              # ZPD tracking and promotion logic
│   │
│   ├── ingestion/
│   │   ├── github.py           # GitHub data ingester
│   │   ├── slack.py            # Slack data ingester
│   │   └── email.py            # Email data ingester
│   │
│   └── export/
│       ├── portable.py         # Export/import cognitive models
│       └── schemas.py          # Portable format definitions
│
├── migrations/                 # Alembic migrations
│   └── versions/
│
└── tests/
    ├── test_extraction.py
    ├── test_engine.py
    ├── test_contract.py
    └── test_zpd.py
```

---

## 10. MVP Scope (What to Build First)

### Phase 1: Core Loop + ZPD
- [ ] Postgres schema + migrations (including ZPD tracking)
- [ ] Extraction pipeline (Claude API call) — level, principle, and ZPD signals
- [ ] Cognitive model update engine with ZPD promotion/demotion
- [ ] Contract generator with pacing/sequencing output
- [ ] MCP server with `observe` and `get_contract` tools
- [ ] CLI: `imp init` and `imp mcp-serve`
- [ ] Calibration flow via MCP (cold-start from 2-3 questions)

### Phase 2: Cold-Start Acceleration
- [ ] GitHub ingester (infer expertise from code/PRs/reviews)
- [ ] Slack ingester (communication style, vocabulary, role signals)
- [ ] CLI: `imp connect`

### Phase 3: Polish
- [ ] Export/import
- [ ] Contradiction resolution
- [ ] Contract caching and invalidation
- [ ] Docker compose for one-command setup

### Phase 4: Distribution
- [ ] PyPI package
- [ ] Docker image on ghcr.io
- [ ] Claude Desktop integration guide
- [ ] Copilot MCP integration guide
- [ ] README with the Sarah vs Bruce example

---

## 11. Claude Code Integration

imp is a first-class plugin for Claude Code. Setup:

```bash
# Add imp as an MCP server in Claude Code
claude mcp add imp -- imp mcp-serve

# Or if using the Python module directly
claude mcp add imp -- python -m imp_ai.server.mcp
```

Once connected, Claude Code automatically:

1. Calls `imp_contract` at session start to load the user's cognitive model
2. Calls `imp_observe` after each user message to refine the model
3. Adapts all output — code complexity, explanation depth, framing — based on the contract

### Integration with Claude Code Agent Roles

If you use specialized agent roles (backend, database, frontend, etc.), imp enhances each one:

```
# The agent role defines WHAT it knows (database expertise)
# imp defines HOW it communicates with THIS user
# 
# Without imp: generic SQL explanations for everyone
# With imp: knows Bruce understands ZFS tuning, skips basics,
#           scaffolds multi-agent orchestration concepts
```

### Claude Desktop / Copilot / Cursor

```json
// Same MCP config works everywhere
{
  "mcpServers": {
    "imp": {
      "command": "imp",
      "args": ["mcp-serve"],
      "env": {
        "IMP_DB_URL": "postgresql://localhost:5432/imp",
        "IMP_USER_ID": "default"
      }
    }
  }
}
```

---

## 12. Open Source

**Deployment:** Self-hosted (local) or cloud-hosted. Both have identical features — cloud provides only operational convenience. See `docs/THINKING.md` Decision 4 for rationale.

**License**: Apache 2.0 (same as Mem0 — maximizes adoption, allows commercial use)

**PyPI**: `pip install imp-ai`
**CLI**: `imp`
**Import**: `from imp_ai import ImpClient`

### Community Contribution Points

Ingestion plugins are the easiest way for contributors to add value.
Each implements a simple interface:

```python
class BaseIngester:
    async def connect(self, config: dict) -> bool: ...
    async def ingest(self, user_id: str) -> list[Observation]: ...
    async def sync(self, user_id: str, since: datetime) -> list[Observation]: ...
```

Wanted ingesters: Notion, Linear, Discord, Google Docs, Calendar, browser history.

### Public Roadmap

- **v0.1** — Core loop: extraction + cognitive model + ZPD tracking + contract (with pacing) + MCP server + CLI + calibration
- **v0.2** — Cold-start: GitHub/Slack ingesters + Docker setup
- **v0.3** — Polish: contradiction resolution + contract caching + export/import
- **v0.4** — Ecosystem: Copilot/Cursor guides + LangChain SDK + plugin system
- **v1.0** — Teams: multi-user + admin dashboards + model versioning

---

## 13. Naming

**Brand**: imp
**Package**: imp-ai
**CLI**: imp
**Tagline**: "AI makes juniors productive. imp makes them competent."
**One-liner**: "Mem0 remembers what you said. imp understands how you think."

The name works because:
- Short for **imp**rint — your cognitive imprint
- An imp is a small, clever creature that follows you around and learns
- 3 letters, easy to type, memorable
- `imp init` / `imp mcp-serve` / `imp export` — clean CLI

---

*This document is the build spec. Take it into Claude Code and start with Phase 1.*
