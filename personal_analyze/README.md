# Dify Core Components Analysis Report

## Tổng quan

Báo cáo này phân tích chi tiết kiến trúc và các core component của Dify - một nền tảng open-source để phát triển ứng dụng LLM với giao diện trực quan, kết hợp agentic AI workflows, RAG pipelines, agent capabilities và model management.

## Mục lục

1. [Kiến trúc tổng quan](./01-architecture-overview.md) - Cấu trúc dự án và tech stack
2. [Backend API](./02-backend-api.md) - Flask, SQLAlchemy, DDD patterns
3. [Workflow Engine](./03-workflow-engine.md) - Graph-based workflow execution
4. [RAG Pipeline](./04-rag-pipeline.md) - Document ingestion và retrieval
5. [Frontend Web](./05-frontend-web.md) - Next.js, React, State management

## Tech Stack Overview

### Backend (Python)
- **Framework**: Flask 3.1.2
- **ORM**: SQLAlchemy 2.0.29
- **Task Queue**: Celery 5.5.2
- **Cache**: Redis 6.1.0
- **Database**: PostgreSQL
- **Type Checking**: BasedPyright

### Frontend (TypeScript)
- **Framework**: Next.js 15.5.9
- **UI Library**: React 19.2.3
- **State Management**: Zustand 5.0.9
- **Server State**: TanStack React Query 5.90.5
- **Styling**: Tailwind CSS 3.4.18
- **Graph Visualization**: ReactFlow 11.11.4

### Infrastructure
- **Containerization**: Docker & Docker Compose
- **Observability**: OpenTelemetry, Sentry
- **Vector Databases**: 30+ integrations (Weaviate, Pinecone, PGVector, etc.)

## Kiến trúc tổng quát

```
┌─────────────────────────────────────────────────────────────────────┐
│                          DIFY PLATFORM                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐      ┌─────────────────┐     ┌──────────────┐ │
│  │   Frontend      │      │   Backend API   │     │   Workers    │ │
│  │   (Next.js)     │◄────►│   (Flask)       │◄───►│   (Celery)   │ │
│  │                 │      │                 │     │              │ │
│  │  - Workflow     │      │  - Controllers  │     │  - Async     │ │
│  │    Editor       │      │  - Services     │     │    Tasks     │ │
│  │  - Chat UI      │      │  - Repositories │     │  - RAG       │ │
│  │  - Dataset      │      │  - Core Logic   │     │    Indexing  │ │
│  │    Manager      │      │                 │     │              │ │
│  └────────┬────────┘      └────────┬────────┘     └──────┬───────┘ │
│           │                        │                      │         │
│           │              ┌─────────┴─────────┐           │         │
│           │              │                   │           │         │
│           ▼              ▼                   ▼           ▼         │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    Data Layer                                │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │  │
│  │  │  PostgreSQL  │  │    Redis     │  │  Vector DB   │      │  │
│  │  │  (Primary)   │  │  (Cache/MQ)  │  │  (Embeddings)│      │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    External Integrations                     │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │  │
│  │  │ LLM Providers│  │    Tools     │  │   Plugins    │      │  │
│  │  │ (OpenAI,etc) │  │  (Built-in)  │  │  (Extension) │      │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Workflow Engine
- **Graph-based execution**: Nodes và edges tạo thành DAG
- **25+ node types**: LLM, Code, HTTP, Tool, Knowledge Retrieval, etc.
- **Real-time streaming**: Server-Sent Events cho live updates
- **Multi-threaded execution**: Worker pool cho parallel node execution

### 2. RAG Pipeline
- **Document Ingestion**: 15+ file format extractors
- **Chunking Strategies**: Fixed, Recursive, Parent-Child, QA
- **Vector Storage**: 30+ vector database integrations
- **Retrieval Methods**: Semantic, Full-text, Hybrid search
- **Reranking**: Weight-based và model-based reranking

### 3. Agent System
- **Tool Integration**: Built-in, Custom API, Plugin-based, MCP tools
- **Agent Patterns**: ReAct, Function Calling
- **Memory Management**: Conversation history và context

### 4. Model Runtime
- **Provider Abstraction**: Unified interface cho 100+ models
- **Load Balancing**: Multi-model distribution
- **Credentials Management**: Secure provider authentication

## Directory Structure

```
dify_analyze/
├── api/                    # Backend API (Python Flask)
│   ├── app.py             # Flask app entry
│   ├── controllers/       # HTTP request handlers
│   ├── services/          # Business logic layer
│   ├── repositories/      # Data access layer
│   ├── models/            # SQLAlchemy ORM models
│   ├── core/              # Core business logic
│   │   ├── workflow/      # Workflow execution engine
│   │   ├── rag/           # RAG pipeline
│   │   ├── agent/         # Agent system
│   │   ├── tools/         # Tool management
│   │   └── model_runtime/ # LLM provider integration
│   ├── tasks/             # Celery async tasks
│   └── configs/           # Configuration management
│
├── web/                    # Frontend (Next.js)
│   ├── app/               # App Router pages
│   ├── components/        # React components
│   ├── context/           # React Context providers
│   ├── service/           # API client layer
│   ├── hooks/             # Custom React hooks
│   └── i18n/              # Internationalization
│
├── docker/                 # Docker deployment
│   ├── docker-compose.yaml
│   └── volumes/           # Persistent storage
│
└── personal_analyze/       # This analysis folder
    ├── README.md          # Overview (this file)
    ├── 01-architecture-overview.md
    ├── 02-backend-api.md
    ├── 03-workflow-engine.md
    ├── 04-rag-pipeline.md
    └── 05-frontend-web.md
```

## Key Design Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| **DDD (Domain-Driven Design)** | Backend | Clean separation of concerns |
| **Repository Pattern** | `/api/repositories/` | Data access abstraction |
| **Factory Pattern** | Extensions, Services | Object creation |
| **Observer Pattern** | Events, Workflows | Event-driven architecture |
| **Slice Pattern** | Frontend Stores | State management composition |
| **Provider Pattern** | React Contexts | Dependency injection |

## Làm thế nào để đọc báo cáo này

1. Bắt đầu với [Kiến trúc tổng quan](./01-architecture-overview.md) để hiểu bức tranh toàn cảnh
2. Đọc [Backend API](./02-backend-api.md) nếu bạn quan tâm đến server-side
3. Đọc [Workflow Engine](./03-workflow-engine.md) để hiểu core business logic
4. Đọc [RAG Pipeline](./04-rag-pipeline.md) để hiểu hệ thống knowledge retrieval
5. Đọc [Frontend Web](./05-frontend-web.md) để hiểu user interface

## Code References

Mỗi phần phân tích đều bao gồm:
- **File paths**: Đường dẫn chính xác đến source code
- **Line numbers**: Số dòng cụ thể cho các implementations quan trọng
- **Sequence diagrams**: Mô tả luồng xử lý
- **Code snippets**: Ví dụ code từ repository

---

*Báo cáo được tạo tự động bằng cách phân tích source code của Dify repository.*
