# Architecture

## Principles

- **Routes stay thin.** A FastAPI route does three things only: validate
  the request, call one service, and return the response. A route never
  contains business logic.
- **Services own business logic.** Each service is a single-responsibility
  class — `SessionService`, `PipelineService`, `RiskService`,
  `AnalysisService`, `ChatService`, `AgentService`, and others. Routes
  call into these services; the services do the actual work.
- **Domain models represent pipeline state.** These models are separate
  from both the API request/response shapes and the ML/provider layer. A
  clause moves through four stages as it passes through the pipeline:
  `ExtractedClause` → `ClassifiedClause` → `RiskAssessedClause` →
  `RetrievedClauseContext`. Each stage wraps the previous one. No stage
  mutates the one before it.
- **Providers are swappable.** The LLM (Large Language Model) provider and
  the embedding-model provider each sit behind an interface —
  `LLMProvider` and `EmbeddingProvider`. A config value selects which
  vendor is active (`LLMProviderSource` and `EmbeddingProviderSource`).
  No provider name is hardcoded into a service. The agent layer uses one
  more factory, `LangChainModelFactory`, because LangGraph's tool-calling
  loop needs LangChain's own `BaseChatModel` type, not the app's own
  `LLMProvider` type.
- **Sessions are isolated workspaces.** Every document upload creates its
  own session. Each session gets its own disk directory, its own vector
  index namespace, and its own in-memory agent state. Two concurrent
  users — or two concurrent documents from the same user — never share
  state.

## System diagram

```mermaid
flowchart TB
    subgraph Client["Frontend (React + TypeScript)"]
        UI["Document workspace:<br/>PDF viewer · clause overlay · chat sidebar"]
    end

    subgraph API["FastAPI routes — app/api/routes/"]
        Sessions["/sessions"]
        Documents["/documents"]
        Chat["/chat"]
        Agent["/agent"]
    end

    subgraph Services["Service layer — app/services/"]
        SessionSvc["SessionService"]
        Pipeline["PipelineService<br/>(classify → assess risk → retrieve context)"]
        AnalysisSvc["AnalysisService<br/>(generates LLM reports)"]
        ChatSvc["ChatService"]
        AgentSvc["AgentService<br/>(LangGraph tool-calling loop)"]
    end

    subgraph Providers["Provider abstraction — app/providers/"]
        LLMProvider["LLMProvider<br/>(Groq / OpenAI / Anthropic / Ollama)"]
        EmbedProvider["EmbeddingProvider<br/>(local BGE model / OpenAI / Voyage)"]
        LCFactory["LangChainModelFactory<br/>(used by the agent only)"]
    end

    subgraph Data["Data"]
        Classifier[("Legal-BERT<br/>clause classifier")]
        Chroma[("ChromaDB<br/>vector index")]
        Workspace[("Session workspace<br/>disk storage")]
    end

    UI --> Sessions & Documents & Chat & Agent
    Sessions --> SessionSvc
    Documents --> Pipeline
    Documents --> AnalysisSvc
    Chat --> ChatSvc
    Agent --> AgentSvc

    Pipeline --> Classifier
    Pipeline --> Chroma
    AnalysisSvc --> LLMProvider
    ChatSvc --> LLMProvider
    ChatSvc --> Chroma
    AgentSvc --> LCFactory
    LCFactory --> LLMProvider
    AgentSvc --> Chroma

    SessionSvc --> Workspace
    Pipeline --> Workspace
    EmbedProvider --> Chroma
```

## Session lifecycle

A session is the unit of isolation in ClauseGuard. One session holds:
one uploaded document, one workspace directory, one vector-index
namespace, and one slice of in-memory agent state.

A session ends in one of two ways: the user ends it explicitly, or it
sits idle past a timeout. Both paths run the exact same cleanup.

```mermaid
sequenceDiagram
    participant U as User
    participant API as FastAPI
    participant SS as SessionService
    participant WS as Workspace (disk)
    participant VI as ChromaDB
    participant AG as In-memory agent state

    U->>API: POST /sessions
    API->>SS: create_session()
    SS->>WS: Create a workspace directory
    SS-->>U: Return session_id

    Note over U,AG: Upload → prepare → classify → assess risk → chat.<br/>Every step below is scoped to this session_id.

    alt User ends the session
        U->>API: DELETE /sessions/{id}
    else Session sits idle past the timeout
        SS->>SS: is_expired() check, on next access
    end

    API->>SS: delete_session()
    SS->>VI: Delete this session's vector data
    SS->>WS: Remove the workspace directory
    SS->>AG: Clear the in-memory agent state
```

## Directory layout (private repo)

```
backend/
  app/
    api/routes/       # thin FastAPI routes
    api/deps.py        # shared route dependencies (get_session, validate_document)
    core/               # config, enums, exceptions, risk rules, Chroma client
    domain/             # pipeline-stage dataclasses
    providers/           # LLM / embedding provider abstraction + factories
    schemas/             # Pydantic request/response models + mappers
    services/             # business logic
    prompts/               # Jinja2 prompt templates
  ml/
    training/           # dataset, tokenizer, model, metrics, train loop
    scripts/             # preprocess / train / evaluate / calibrate entry points
    inference/           # classifier loader + predictor used by the running app
    data/, models/, reports/   # gitignored, versioned by CLAUSE_CLASSIFIER_VERSION (v1/ v2/)
frontend/
  src/
    app/                 # shell, layout
    core/api/             # typed API client
    features/               # document, clause, chat, reports, session, pdf, settings
    components/               # shared/reusable UI components
    theme/, styles/             # single-source-of-truth design tokens (see design philosophy)
    lib/                     # hooks, shared utilities
```
