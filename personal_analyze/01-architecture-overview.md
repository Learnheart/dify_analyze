# Kiến trúc tổng quan - Dify Platform

## 1. Cấu trúc dự án

Dify được chia thành 3 phần chính:

```
dify/
├── api/                    # Backend API (Python Flask)
├── web/                    # Frontend Web (Next.js)
└── docker/                 # Docker deployment configs
```

## 2. Tech Stack Chi tiết

### 2.1 Backend Stack

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Web Framework | Flask | 3.1.2 | HTTP request handling |
| ORM | SQLAlchemy | 2.0.29 | Database abstraction |
| Task Queue | Celery | 5.5.2 | Async task processing |
| Message Broker | Redis | 6.1.0 | Celery broker, caching |
| Database | PostgreSQL | - | Primary data storage |
| Validation | Pydantic | 2.11.4 | Data validation |
| WSGI Server | Gunicorn | - | Production server |
| Observability | OpenTelemetry | 1.27.0 | Distributed tracing |
| LLM Tracking | Langfuse | 2.51.3 | LLM observability |

**Reference**: `/api/pyproject.toml` (lines 1-100)

### 2.2 Frontend Stack

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Framework | Next.js | 15.5.9 | SSR/SSG React framework |
| UI Library | React | 19.2.3 | Component-based UI |
| Language | TypeScript | 5.9.3 | Type safety |
| State (Client) | Zustand | 5.0.9 | Client state management |
| State (Server) | React Query | 5.90.5 | Server state caching |
| Graph Viz | ReactFlow | 11.11.4 | Workflow DAG editor |
| Styling | Tailwind CSS | 3.4.18 | Utility-first CSS |
| HTTP Client | ky | 1.12.0 | Fetch wrapper |
| i18n | i18next | 23.16.8 | Internationalization |
| Rich Text | Lexical | 0.38.2 | Text editor |
| Testing | Vitest | 4.0.16 | Unit testing |

**Reference**: `/web/package.json` (lines 1-100)

## 3. Kiến trúc Multi-Layer

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              PRESENTATION LAYER                          │
│  ┌──────────────────────────────┐    ┌───────────────────────────────┐ │
│  │     Next.js Frontend         │    │       Flask Controllers       │ │
│  │  (React Components, Hooks)   │◄──►│    (REST API Endpoints)       │ │
│  └──────────────────────────────┘    └───────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                              APPLICATION LAYER                           │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                        Service Layer                                │ │
│  │   AccountService | AppService | WorkflowService | DatasetService   │ │
│  │                        (Business Logic)                             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                               DOMAIN LAYER                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │  Workflow Engine │  │   RAG Pipeline  │  │     Agent System        │ │
│  │  (Graph Exec)    │  │  (Retrieval)    │  │   (Tool Integration)    │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                           INFRASTRUCTURE LAYER                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         Repository Layer                          │  │
│  │  WorkflowExecutionRepository | DatasetRepository | UserRepository │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  PostgreSQL  │  │    Redis     │  │  Vector DB   │  │  Storage  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## 4. Backend Architecture (DDD + Clean Architecture)

### 4.1 Layer Structure

```
/api/
├── controllers/           # Presentation Layer (HTTP Handlers)
│   ├── console/          # Console API (Admin UI backend)
│   ├── service_api/      # Public API for integrations
│   ├── web/              # Web app public endpoints
│   ├── files/            # File upload/download
│   ├── inner_api/        # Internal service communication
│   ├── mcp/              # Model Context Protocol
│   └── trigger/          # Webhook triggers
│
├── services/              # Application Layer (Business Logic)
│   ├── account_service.py
│   ├── app_service.py
│   ├── workflow_service.py
│   ├── dataset_service.py
│   └── ...60+ services
│
├── core/                  # Domain Layer (Core Business Logic)
│   ├── workflow/         # Workflow execution engine
│   ├── rag/              # RAG pipeline
│   ├── agent/            # Agent system
│   ├── tools/            # Tool management
│   ├── model_runtime/    # LLM provider abstraction
│   └── plugin/           # Plugin system
│
├── repositories/          # Infrastructure Layer (Data Access)
│   ├── api_workflow_run_repository.py
│   ├── sqlalchemy_api_workflow_run_repository.py
│   └── factory.py
│
└── models/                # ORM Models (Entity Definitions)
    ├── account.py
    ├── app.py
    ├── workflow.py
    ├── dataset.py
    └── ...100+ models
```

### 4.2 Flask Application Factory

**Reference**: `/api/app_factory.py` (lines 16-134)

```python
# Simplified structure
def create_app() -> DifyApp:
    """Main factory function."""
    dify_app = create_flask_app_with_configs()
    initialize_extensions(dify_app)
    return dify_app

def initialize_extensions(app: DifyApp):
    """Initialize 20+ extensions in order."""
    extensions = [
        ext_timezone,      # Timezone handling
        ext_logging,       # Logging configuration
        ext_database,      # SQLAlchemy setup
        ext_redis,         # Redis client
        ext_celery,        # Celery task queue
        ext_storage,       # File storage (S3, local)
        ext_blueprints,    # Route registration
        ext_otel,          # OpenTelemetry tracing
        # ... more extensions
    ]
    for ext in extensions:
        ext.init_app(app)
```

### 4.3 Blueprint Organization

**Reference**: `/api/extensions/ext_blueprints.py` (lines 24-82)

```python
# Route structure
/console/api/         # Admin console endpoints
/v1/                  # Public service API
/web/api/             # Web app endpoints
/files/               # File operations
/inner/api/           # Internal services
/mcp/                 # Model Context Protocol
/trigger/             # Webhooks
```

## 5. Frontend Architecture

### 5.1 Next.js App Router Structure

**Reference**: `/web/app/layout.tsx`

```
/web/app/
├── layout.tsx                    # Root layout (providers)
├── (commonLayout)/              # Admin dashboard routes
│   ├── layout.tsx               # Common sidebar layout
│   ├── apps/                    # App listing
│   ├── app/[appId]/             # App details
│   │   └── workflow/            # Workflow editor
│   ├── datasets/                # Knowledge base management
│   └── plugins/                 # Plugin marketplace
├── (shareLayout)/               # Public share routes
│   ├── chat/                    # Chat interface
│   └── workflow/                # Workflow runner
├── signin/                      # Authentication
├── signup/                      # Registration
└── account/                     # Account settings
```

### 5.2 State Management Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    STATE MANAGEMENT                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐                                        │
│  │  Local State    │  useState, useReducer                  │
│  │  (Component)    │  Per-component UI state                │
│  └────────┬────────┘                                        │
│           │                                                  │
│  ┌────────▼────────┐                                        │
│  │  Client State   │  Zustand Stores (15 slices)            │
│  │  (Zustand)      │  UI state, form state, selections     │
│  └────────┬────────┘                                        │
│           │                                                  │
│  ┌────────▼────────┐                                        │
│  │  Server State   │  TanStack Query                        │
│  │  (React Query)  │  API data, caching, background refresh │
│  └────────┬────────┘                                        │
│           │                                                  │
│  ┌────────▼────────┐                                        │
│  │  Global State   │  React Context                         │
│  │  (Context)      │  User, workspace, i18n, providers     │
│  └─────────────────┘                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Zustand Store Slices** (Workflow):
- `workflow-slice.ts` - Running data, selections
- `node-slice.ts` - Node interactions, animations
- `panel-slice.ts` - Side panel state
- `layout-slice.ts` - Canvas layout
- `history-slice.ts` - Undo/redo history
- `chat-variable-slice.ts` - Chat variables
- `env-variable-slice.ts` - Environment variables
- `tool-slice.ts` - Tool configuration
- ... (15 total slices)

**Reference**: `/web/app/components/workflow/store/workflow/index.ts` (lines 40-97)

## 6. Sequence Diagram - Request Flow

```
┌──────────┐     ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Browser │     │  Next.js    │     │  Flask API   │     │  Database   │
│  (React) │     │  (SSR/CSR)  │     │  (Backend)   │     │ (PostgreSQL)│
└────┬─────┘     └──────┬──────┘     └──────┬───────┘     └──────┬──────┘
     │                  │                    │                    │
     │  User Action     │                    │                    │
     │─────────────────►│                    │                    │
     │                  │                    │                    │
     │                  │  API Request       │                    │
     │                  │───────────────────►│                    │
     │                  │  (ky HTTP client)  │                    │
     │                  │                    │                    │
     │                  │                    │  SQL Query         │
     │                  │                    │───────────────────►│
     │                  │                    │                    │
     │                  │                    │  Result Set        │
     │                  │                    │◄───────────────────│
     │                  │                    │                    │
     │                  │  JSON Response     │                    │
     │                  │◄───────────────────│                    │
     │                  │                    │                    │
     │  React Query     │                    │                    │
     │  Cache Update    │                    │                    │
     │◄─────────────────│                    │                    │
     │                  │                    │                    │
     │  UI Re-render    │                    │                    │
     │◄─────────────────│                    │                    │
     │                  │                    │                    │
```

## 7. Docker Architecture

**Reference**: `/docker/docker-compose.yaml`

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DOCKER COMPOSE SERVICES                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         NGINX (Reverse Proxy)                    │   │
│  │                         Port 80, 443                             │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│         ┌─────────────────────┼─────────────────────┐                  │
│         │                     │                     │                  │
│         ▼                     ▼                     ▼                  │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐          │
│  │  dify-web   │       │  dify-api   │       │ dify-worker │          │
│  │  (Next.js)  │       │   (Flask)   │       │  (Celery)   │          │
│  │  Port 3000  │       │  Port 5001  │       │   Workers   │          │
│  └─────────────┘       └──────┬──────┘       └──────┬──────┘          │
│                               │                     │                  │
│         ┌─────────────────────┴─────────────────────┘                  │
│         │                     │                     │                  │
│         ▼                     ▼                     ▼                  │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐          │
│  │ PostgreSQL  │       │    Redis    │       │  Weaviate   │          │
│  │  (Primary)  │       │ (Cache/MQ)  │       │ (Vector DB) │          │
│  └─────────────┘       └─────────────┘       └─────────────┘          │
│                                                                          │
│  ┌─────────────┐       ┌─────────────┐                                 │
│  │ SSRF Proxy  │       │  Sandbox    │                                 │
│  │  (Security) │       │ (Code Exec) │                                 │
│  └─────────────┘       └─────────────┘                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 8. Key Design Decisions

### 8.1 Domain-Driven Design (DDD)

Dify áp dụng DDD với các layers rõ ràng:
- **Controllers**: Chỉ xử lý HTTP request/response
- **Services**: Chứa business logic
- **Core**: Domain logic phức tạp (workflow, RAG, agent)
- **Repositories**: Abstract data access
- **Models**: Entity definitions

### 8.2 Dependency Injection

Backend sử dụng DI thông qua:
1. **Flask Extensions**: `app.extensions["redis"]`
2. **Factory Pattern**: Repository factories
3. **Context Variables**: Thread-safe request context

**Reference**: `/api/contexts/wrapper.py` (lines 14-65)

### 8.3 Event-Driven Architecture

Workflow engine sử dụng event-driven pattern:
- **Event Queue**: Thread-safe queue cho node events
- **Dispatcher**: Xử lý events theo thứ tự
- **Workers**: Parallel node execution

### 8.4 Repository Pattern

Data access được abstract qua Protocol-based repositories:

```python
# Protocol definition
class WorkflowExecutionRepository(Protocol):
    def save(self, execution: WorkflowExecution): ...
    def get_by_id(self, id: str) -> WorkflowExecution | None: ...

# SQLAlchemy implementation
class SQLAlchemyWorkflowExecutionRepository(WorkflowExecutionRepository):
    def __init__(self, session_maker: sessionmaker[Session]):
        self._session_maker = session_maker
```

**Reference**: `/api/core/workflow/repositories/workflow_execution_repository.py`

## 9. Configuration Management

**Reference**: `/api/configs/app_config.py` (lines 58-76)

```python
class DifyConfig(
    PackagingInfo,
    DeploymentConfig,
    FeatureConfig,
    MiddlewareConfig,
    ExtraServiceConfig,
    ObservabilityConfig,
    RemoteSettingsSourceConfig,
    EnterpriseFeatureConfig,
):
    """Hierarchical configuration from multiple sources."""
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )
```

Configuration sources (theo thứ tự ưu tiên):
1. Environment variables
2. `.env` file
3. Remote sources (Apollo, Nacos)
4. Default values

## 10. Security Considerations

### 10.1 Authentication
- JWT-based authentication
- Session management với Redis
- Rate limiting per user/IP

### 10.2 Authorization
- Role-based access control (RBAC)
- Tenant isolation
- Resource-level permissions

### 10.3 Infrastructure Security
- SSRF protection proxy
- Sandboxed code execution
- Input validation với Pydantic

---

*Tiếp theo: [Backend API Architecture](./02-backend-api.md)*
