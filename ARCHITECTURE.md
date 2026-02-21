# Kendra — System Architecture

> C4 Model architecture documentation for the Kendra personal AI system.

---

## System Context (C4 Level 1)

```mermaid
C4Context
    title System Context — Kendra Personal AI System

    Person(user, "User", "Voice, CLI, GUI, Mobile via Tailscale")

    System(kendra, "Kendra", "Personal AI System — Python 3.13, Apple Silicon")

    System_Ext(claude, "Claude CLI", "Primary LLM backend via subprocess")
    System_Ext(ollama, "Ollama", "Local LLM fallback — 32B model, embedding model")
    System_Ext(knowledge, "Knowledge Engine", "Semantic search over personal documents")
    System_Ext(career, "Career System", "Experience data, skill analysis, job matching")
    System_Ext(crm, "Personal CRM", "Contacts, interactions, follow-ups")
    System_Ext(coordination, "Coordination Hub", "Multi-agent task dispatch and tracking")
    System_Ext(apple, "macOS Apps", "Mail, Calendar, Reminders, Notes, iMessage")
    System_Ext(messaging, "Messaging Platforms", "Telegram, Slack, Discord")

    Rel(user, kendra, "Voice, CLI, REST API, React GUI, iMessage, Telegram")
    Rel(kendra, claude, "Subprocess with session management")
    Rel(kendra, ollama, "HTTP API for LLM generation and embeddings")
    Rel(kendra, knowledge, "REST API / MCP for semantic search")
    Rel(kendra, career, "REST API / MCP for experience queries")
    Rel(kendra, crm, "REST API for contact intelligence")
    Rel(kendra, coordination, "File I/O / MCP for coordination")
    Rel(kendra, apple, "AppleScript for CRUD operations")
    Rel(kendra, messaging, "Bot APIs with content scanning")
```

---

## Container Architecture (C4 Level 2)

```mermaid
C4Container
    title Container Architecture — Kendra

    Person(user, "User")

    System_Boundary(kendra, "Kendra") {

        Container_Boundary(interface, "Interface Layer") {
            Container(cli, "CLI", "Python", "9 run modes")
            Container(menubar, "Menu Bar", "rumps", "macOS menu bar with voice modes")
            Container(api, "FastAPI Server", "FastAPI + Uvicorn", "REST API + WebSocket")
            Container(gui, "React Dashboard", "React 19 + Vite", "Real-time chat, settings, board room")
            Container(voice, "Voice Pipeline", "MLX Whisper + Kokoro", "STT/TTS with wake word detection")
            Container(msgbridges, "Messaging Bridges", "Python", "Telegram, Slack, Discord, iMessage")
        }

        Container_Boundary(brain, "Brain Layer") {
            Container(controller, "Controller", "Python", "5-state async state machine")
            Container(router, "Intent Router", "Python", "Table-driven from skill metadata, longest-match-first")
            Container(llm, "LLM Client", "Python", "Factory returns Claude or Ollama client")
            Container(mood, "Mood Manager", "Python", "Dynamic personality adjustment")
            Container(scheduler, "Proactive Scheduler", "APScheduler", "Calendar, goals, health, board checks")
        }

        Container_Boundary(body, "Body Layer") {
            Container(skills, "Skill Manager", "Python", "26 skills with sandboxed execution")
            Container(mem0, "Semantic Memory", "Mem0 + Qdrant", "Fact extraction + vector search")
            Container(totalrecall, "Experiential Memory", "SQLite + LanceDB", "7-phase hybrid retrieval")
            Container(rag, "Knowledge Search", "Python", "REST API + MCP fallback")
            Container(mcp, "MCP Server", "FastMCP", "Skills exposed as external tools")
        }

        Container_Boundary(security, "Security Layer") {
            Container(pathguard, "Path Guard", "Python", "Filesystem sandbox — read-only project dir")
            Container(codevalidator, "Code Validator", "Python", "AST analysis — blocks dangerous imports/calls")
            Container(ratelimiter, "Rate Limiter", "Python", "Per-app limits + spike detection")
            Container(contentscan, "Content Scanner", "Python", "Injection detection + sandboxing delimiters")
            Container(auditlog, "Audit Logger", "Python", "Append-only security event log")
        }
    }

    Rel(user, cli, "Terminal commands")
    Rel(user, menubar, "Menu bar clicks, wake word")
    Rel(user, gui, "Browser")
    Rel(user, msgbridges, "Telegram/Slack/Discord messages")
    Rel(api, controller, "WebSocket + REST")
    Rel(cli, controller, "Direct async call")
    Rel(voice, controller, "Transcribed text")
    Rel(controller, router, "Route intent")
    Rel(router, contentscan, "Security scan before routing")
    Rel(router, skills, "Matched skill execution")
    Rel(router, llm, "Chat fallback")
    Rel(llm, mem0, "Context injection")
    Rel(llm, totalrecall, "Experiential context")
    Rel(skills, pathguard, "Filesystem access check")
    Rel(skills, ratelimiter, "Rate limit check")
```

---

## Key Design Decisions

| Decision | Choice | Why | Alternatives Considered |
|----------|--------|-----|------------------------|
| Primary LLM | Claude CLI (subprocess) | Session management, MCP tool integration, cost tracking built-in | Direct API (no tool integration), local-only (limited reasoning) |
| LLM Fallback | Ollama (32B, M4 Max optimized) | Full local privacy, zero API cost, ~40 tok/s on Apple Silicon | No fallback (fragile), cloud API (privacy concern) |
| Intent Routing | Table-driven from skill metadata | Auto-discovery, no code changes to add skills, longest-match specificity | LLM-based classification (non-deterministic), hardcoded if-elif (inflexible) |
| Semantic Memory | Mem0 + Qdrant (embedded, no server) | Production-grade fact extraction, semantic dedup, contradiction resolution | Hand-rolled extraction (brittle), other vector DBs (less mature dedup) |
| Experiential Memory | SQLite + FTS5 + LanceDB | Hybrid BM25+vector with RRF fusion, bitemporal knowledge graph, consolidation | Single-vector-only (loses lexical precision), PostgreSQL (overkill for local) |
| Security Sandbox | Executor wrapping builtins at runtime | Thread-safe, audit trail, no container overhead, allowlisted safe dirs | Docker containers (heavy for single-user), chroot (complex), trust model (unsafe) |
| Voice STT | MLX Whisper (large-v3) | Apple Silicon native Metal acceleration, local-first, no cloud | Whisper.cpp (less Python integration), cloud STT (privacy concern) |
| GUI | React 19 + Vite + Zustand + WebSocket | Real-time streaming, modern tooling, runs alongside menu bar | PyQt (desktop-only, thread conflicts with menu bar), vanilla HTML (limited reactivity) |
| Skill System | Subdirectory pattern (metadata + implementation) | Auto-discovery via directory scan, self-documenting triggers, isolated and testable | Plugin registry (complex wiring), monolithic router (fragile, hard to extend) |
| State Machine | 5-state enum (IDLE/LISTENING/THINKING/SPEAKING/ERROR) | Deterministic, debuggable, prevents concurrent processing conflicts | Free-form (unpredictable), FSM library (unnecessary dependency) |

---

## Data Flow

### Primary Request Flow

```mermaid
flowchart TD
    A[User Input] --> B{Input Validation}
    B -->|>50K chars| C[Reject]
    B -->|Valid| D[Content Scanner]
    D -->|Injection detected| E[Blocked Response]
    D -->|Clean| F[Intent Router]

    F -->|Skill keyword match| J[Table Match]
    J -->|Matched| K{Rate Limit Check}
    K -->|Exceeded| L[Rate Limited Response]
    K -->|OK| M{Sensitive Action?}
    M -->|Yes| N[Require Confirmation]
    M -->|No| O[Execute Skill]
    N -->|Confirmed| O

    F -->|Search pattern| P[Knowledge Base Search]
    F -->|No match| Q[LLM Chat]

    O --> R[Response to User]
    P --> R
    Q --> R

    R --> S[Memory Pipeline]
    S --> T[Log to Experiential Memory]
    S --> U[Extract Facts via Semantic Memory]
    S --> V[Extract Goals]
```

### Skill Execution Flow

```mermaid
flowchart LR
    A[Skill Manager] --> B[Path Guard Check]
    B --> C[Sandboxed Executor]
    C --> D[Rate Limit Check]
    D --> E{Sensitive Action?}
    E -->|email, install, delete, wipe| F[Require Confirmation]
    E -->|Normal| G[Execute in Sandbox]
    F -->|Confirmed| G
    G --> H[Audit Log]
    H --> I[Return Result]
```

---

## Memory Architecture

```mermaid
flowchart TD
    subgraph T1["Tier 1: Core Identity"]
        profile["User Profile"]
        personality["Personality Definition"]
    end

    subgraph T2["Tier 2: Learned Preferences"]
        mem0["Semantic Memory Engine"]
        qdrant["Vector Store"]
        mem0 --> qdrant
    end

    subgraph T3["Tier 3: Experiential Memory"]
        store["Event Store (SQLite + FTS5)"]
        vector["Vector Search (LanceDB)"]
        kg["Knowledge Graph (SQLite triples)"]
        consolidation["Consolidation Engine"]
        reranker["Reranker (cross-encoder, HyDE)"]

        store --> vector
        store --> kg
        consolidation --> store
        consolidation --> kg
        vector --> reranker
    end

    subgraph T4["Tier 4: Scale Knowledge"]
        knowledge["Knowledge Engine (REST API)"]
        mcp_fallback["MCP Fallback"]
        knowledge -.-> mcp_fallback
    end

    input[User Interaction] --> T1
    input --> T2
    input --> T3
    T1 --> prompt["System Prompt Assembly"]
    T2 --> prompt
    T3 --> prompt
    T4 --> prompt
    prompt --> llm["LLM Generation"]
    llm --> response[Response]

    response --> episodic["Log Interaction"]
    episodic --> T3
    response --> extract["Extract Facts (fire-and-forget)"]
    extract --> T2
    response --> goals["Extract Goals"]
```

### Retrieval Pipeline (Experiential Memory)

1. **Query** arrives from controller
2. **Lexical search** — BM25 via FTS5 with porter stemming, scope/time/type filters
3. **Vector search** — LanceDB with 1024-dim embeddings, multi-resolution
4. **Fusion** — Reciprocal Rank Fusion (k=60, vector weight=0.7, FTS weight=0.3)
5. **Reranking** — Cross-encoder reranking of fused results
6. **Advanced** — HyDE (hypothetical document expansion), multi-query decomposition
7. **Context Assembly** — Token-budget-aware formatting for LLM prompt

### Consolidation Engine ("Sleep Cycle")

- Watermark-based processing (only new events since last run)
- Conversation grouping and summarization
- Knowledge graph entity extraction (triples with bitemporal tracking)
- Temporal decay (older facts lose weight, contradictions resolved by recency)

---

## Security Posture

| Concern | Approach |
|---------|----------|
| Authentication | Bearer token for API (skip for local access), passphrase for messaging, rate limiting |
| Path Protection | Sandbox executor: project dir read-only, designated staging area writable, symlink resolution |
| Code Safety | AST validator blocks 41+ dangerous imports, 10+ dangerous calls, Bandit static analysis |
| Rate Limiting | Per-app limits (Mail 3/min, Calendar 10/min, etc.), spike detection >10/min |
| Content Safety | Prompt injection detection, content sandboxing with delimiters, PII redaction |
| Input Validation | 50K character limit, content scanning on every input, file validation |
| Source Authentication | Source parameter threaded through controller → router → skill manager; meta-skills blocked from non-local |
| Audit Trail | Append-only security event log (sensitive actions, rate limit events, auth attempts) |
| Data at Rest | SQLite files excluded from version control, full disk encryption, no secrets in committed code |
| Pre-commit | 12-phase security scanner (secrets, YAML values, .env, PII, code patterns, compliance) |

---

## Technology Stack

| Layer | Technology | Role |
|-------|-----------|------|
| Language | Python 3.13 | Core implementation, async/await throughout |
| Primary LLM | Claude CLI | Session management, MCP tools, cost tracking |
| Local LLM | Ollama (32B model) | Privacy-first fallback, Apple Silicon optimized (~40 tok/s) |
| Embeddings | Ollama (1024-dim model) | Semantic similarity for memory and retrieval |
| STT | MLX Whisper (large-v3) | Apple Silicon native speech-to-text via Metal |
| TTS | Kokoro ONNX | Local text-to-speech |
| Wake Word | Porcupine / OpenWakeWord | Sub-50ms activation |
| Frontend | React 19, TypeScript, Vite, Zustand, Tailwind CSS | Real-time dashboard with WebSocket streaming |
| API | FastAPI + Uvicorn | REST API + WebSocket server |
| Semantic Memory | Mem0 + Qdrant (embedded) | LLM-driven extraction, dedup, contradiction resolution |
| Experiential Memory | SQLite (WAL) + FTS5 + LanceDB | Event store, BM25, vector search, knowledge graph |
| macOS Integration | AppleScript, rumps, pynput | Menu bar, hotkey, Apple app integration |
| MCP | FastMCP | Bidirectional tool integration |
| Browser | Playwright | Headless Chromium automation |
| Messaging | python-telegram-bot, slack-sdk, discord.py | Multi-channel bridges |
| Testing | pytest (backend), vitest + RTL (frontend) | 3,863+ tests across 107 files |
| Security | Bandit, custom AST validator, 12-phase scanner | Static analysis + runtime sandboxing |

---

## Component Interaction

### Chat Request Lifecycle

```mermaid
sequenceDiagram
    participant U as User
    participant API as Interface Layer
    participant C as Controller
    participant S as Security Scanner
    participant R as Router
    participant LLM as LLM Client
    participant M0 as Semantic Memory
    participant TR as Experiential Memory
    participant SK as Skill Manager
    participant MEM as Memory Logger

    U->>API: Send message
    API->>C: process_input(text, source)
    C->>C: Set state → THINKING
    C->>S: Scan for injection
    S-->>C: Clean / Blocked

    C->>R: route(text)
    R->>R: Table match (longest substring first)

    alt Skill Match
        R-->>C: skill_name, params
        C->>SK: execute_skill(name, params)
        SK->>SK: Rate limit + sandbox check
        SK-->>C: Skill result
    else Chat (no match)
        R-->>C: intent=chat
        C->>M0: search(query) → semantic context
        C->>TR: retrieve(query) → experiential context
        C->>LLM: generate(prompt + contexts)
        LLM-->>C: Response text
    end

    C->>C: Set state → SPEAKING
    C->>API: Stream response to user
    C->>MEM: log_interaction (fire-and-forget)
    MEM->>M0: extract_facts (async)
    MEM->>TR: store_event (async)
    C->>C: Set state → IDLE
```

---

*Architecture documentation for the Kendra personal AI system.*
