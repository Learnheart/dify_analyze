# Backend API Architecture

## 1. Tổng quan

Backend API của Dify là một Flask application được tổ chức theo Domain-Driven Design (DDD) và Clean Architecture principles. Hệ thống xử lý:
- REST API endpoints
- Async task processing (Celery)
- Real-time streaming (SSE)
- Multi-tenant data isolation

## 2. Flask Application Factory

### 2.1 Entry Point

**Reference**: `/api/app.py` và `/api/app_factory.py`

```python
# /api/app_factory.py (lines 16-51)
def create_flask_app_with_configs() -> DifyApp:
    """Creates Flask app with configurations."""
    dify_app = DifyApp(__name__)

    # Load configuration from .env via DifyConfig
    dify_app.config.from_mapping(dify_config.model_dump())

    # Request hooks for context management
    @dify_app.before_request
    def before_request():
        RecyclableContextVar.increment_thread_recycles()

    return dify_app
```

### 2.2 Extension Initialization

**Reference**: `/api/app_factory.py` (lines 64-134)

```python
def initialize_extensions(app: DifyApp):
    """Initialize 20+ extensions in specific order."""
    extensions = [
        ext_timezone,           # Timezone handling
        ext_logging,            # Logging setup
        ext_warnings,           # Warning filters
        ext_import_modules,     # Dynamic module imports
        ext_orjson,             # Fast JSON serialization
        ext_forward_refs,       # Forward reference resolution
        ext_set_secretkey,      # Flask secret key
        ext_compress,           # Response compression
        ext_code_based_extension, # Plugin extensions
        ext_database,           # SQLAlchemy setup
        ext_app_metrics,        # Prometheus metrics
        ext_migrate,            # Database migrations
        ext_redis,              # Redis client
        ext_storage,            # File storage (S3, local)
        ext_logstore,           # Log storage
        ext_celery,             # Task queue
        ext_login,              # Flask-Login
        ext_mail,               # Email sending
        ext_hosting_provider,   # Cloud hosting
        ext_sentry,             # Error tracking
        ext_proxy_fix,          # Proxy headers
        ext_blueprints,         # Route registration
        ext_commands,           # CLI commands
        ext_otel,               # OpenTelemetry
        ext_request_logging,    # Request logging
        ext_session_factory,    # Session management
    ]
```

## 3. Controller Layer (Presentation)

### 3.1 Blueprint Organization

**Reference**: `/api/extensions/ext_blueprints.py` (lines 24-82)

```
Controllers Structure:
/api/controllers/
├── console/                # Admin Console API (/console/api/)
│   ├── app/               # App CRUD operations
│   ├── workspace/         # Workspace management
│   ├── datasets/          # Dataset/knowledge base
│   ├── auth/              # Authentication
│   ├── billing/           # Billing & subscription
│   ├── explore/           # App exploration
│   └── __init__.py        # Blueprint definition
├── service_api/            # Public Service API (/v1/)
├── web/                    # Web App API (/web/api/)
├── files/                  # File Operations (/files/)
├── inner_api/              # Internal Services (/inner/api/)
├── mcp/                    # Model Context Protocol
└── trigger/                # Webhook Triggers
```

### 3.2 Controller Pattern

**Reference**: `/api/controllers/console/app/app.py`

```python
from flask_restx import Resource, Namespace

# Namespace definition
console_ns = Namespace("console", description="Console API operations")

@console_ns.route("/apps")
class AppListApi(Resource):
    """App listing endpoint."""

    # Decorators for authentication and authorization
    method_decorators = [
        account_initialization_required,
        login_required,
        setup_required,
    ]

    @console_ns.doc("list_apps")
    @console_ns.expect(app_list_parser)
    def get(self):
        """List all apps for current tenant."""
        # Parse request arguments
        args = app_list_parser.parse_args()

        # Call service layer
        app_service = AppService()
        apps = app_service.list_apps(
            tenant_id=current_user.current_tenant_id,
            page=args["page"],
            limit=args["limit"],
        )

        return {"data": apps}
```

### 3.3 Authentication Decorators

**Reference**: `/api/libs/login.py` và `/api/controllers/console/setup.py`

```python
# Common decorator stack
method_decorators = [
    account_initialization_required,  # Account setup complete
    login_required,                   # User authenticated
    setup_required,                   # System setup complete
    cloud_edition_billing_resource_check("apps"),  # Cloud billing
]

# For editing operations
method_decorators = [
    ...
    edit_permission_required,         # User has edit permission
]
```

## 4. Service Layer (Application)

### 4.1 Service Organization

**Reference**: `/api/services/`

```
/api/services/
├── account_service.py          # User & account management (55KB)
├── app_service.py              # App CRUD operations
├── app_dsl_service.py          # DSL import/export
├── app_generate_service.py     # App generation routing
├── dataset_service.py          # Dataset operations (166KB)
├── workflow_service.py         # Workflow management (45KB)
├── workflow_run_service.py     # Workflow execution
├── model_provider_service.py   # LLM provider management
├── feature_service.py          # Feature flags
├── auth/                       # Authentication services
├── rag_pipeline/               # RAG services
├── plugin/                     # Plugin management
├── tools/                      # Tool services
├── enterprise/                 # Enterprise features
├── entities/                   # DTOs and value objects
└── errors/                     # Domain exceptions
```

### 4.2 Service Implementation Pattern

**Reference**: `/api/services/account_service.py` (lines 77-150)

```python
class AccountService:
    """Account and authentication service."""

    # Rate limiters for security
    reset_password_rate_limiter = RateLimiter(
        prefix="reset_password_rate_limit",
        max_attempts=1,
        time_window=60,
    )

    # Error limit constants
    LOGIN_MAX_ERROR_LIMITS = 5
    FORGOT_PASSWORD_MAX_ERROR_LIMITS = 5

    @staticmethod
    def _store_refresh_token(account_id: str, token: str, expiry: int):
        """Store refresh token in Redis."""
        key = f"refresh_token:{account_id}"
        redis_client.setex(key, expiry, token)

    @staticmethod
    def load_user(user_id: str) -> Account | None:
        """Load user by ID with permission checks."""
        account = db.session.query(Account).get(user_id)

        if not account:
            return None

        if account.status == AccountStatus.BANNED.value:
            raise AccountBannedError()

        # Verify tenant association
        if current_user.current_tenant_id:
            TenantService.validate_member(
                account.id,
                current_user.current_tenant_id
            )

        return account
```

### 4.3 Domain Error Handling

**Reference**: `/api/services/errors/`

```python
# /api/services/errors/base.py
class BaseServiceError(ValueError):
    """Base class for service layer errors."""
    def __init__(self, description: str | None = None):
        self.description = description

# /api/services/errors/account.py
class AccountNotFoundError(BaseServiceError):
    pass

class AccountRegisterError(BaseServiceError):
    pass

class TenantNotFoundError(BaseServiceError):
    pass

class NoPermissionError(BaseServiceError):
    pass

# /api/services/errors/app.py
class QuotaExceededError(ValueError):
    """Raised when resource quota is exceeded."""
    def __init__(self, feature: str, tenant_id: str, required: int):
        self.feature = feature
        self.tenant_id = tenant_id
        self.required = required
```

## 5. Repository Layer (Infrastructure)

### 5.1 Protocol-Based Design

**Reference**: `/api/core/workflow/repositories/workflow_execution_repository.py` (lines 6-30)

```python
from typing import Protocol

class WorkflowExecutionRepository(Protocol):
    """
    Repository interface for WorkflowExecution.

    This interface defines the contract for accessing and manipulating
    WorkflowExecution data, regardless of the underlying storage mechanism.
    """

    def save(self, execution: WorkflowExecution):
        """Save or update a WorkflowExecution instance."""
        ...

    def get_by_id(self, execution_id: str) -> WorkflowExecution | None:
        """Retrieve execution by ID."""
        ...

    def get_by_workflow_id(
        self,
        workflow_id: str,
        limit: int = 10,
    ) -> list[WorkflowExecution]:
        """Get executions for a workflow."""
        ...
```

### 5.2 SQLAlchemy Implementation

**Reference**: `/api/repositories/sqlalchemy_api_workflow_run_repository.py`

```python
from sqlalchemy import select
from sqlalchemy.orm import Session, sessionmaker

class SQLAlchemyAPIWorkflowRunRepository(APIWorkflowRunRepository):
    """SQLAlchemy implementation of workflow run repository."""

    def __init__(self, session_maker: sessionmaker[Session]):
        self._session_maker = session_maker

    def get_paginated_workflow_runs(
        self,
        tenant_id: str,
        app_id: str,
        triggered_from: WorkflowRunTriggeredFrom,
        limit: int = 20,
        last_id: str | None = None,
        status: str | None = None,
    ) -> InfiniteScrollPagination:
        """Get paginated workflow runs with cursor-based pagination."""
        with self._session_maker() as session:
            query = (
                select(WorkflowRun)
                .where(
                    WorkflowRun.tenant_id == tenant_id,
                    WorkflowRun.app_id == app_id,
                    WorkflowRun.triggered_from == triggered_from,
                )
                .order_by(WorkflowRun.created_at.desc())
            )

            if last_id:
                # Cursor-based pagination
                last_run = session.get(WorkflowRun, last_id)
                if last_run:
                    query = query.where(
                        WorkflowRun.created_at < last_run.created_at
                    )

            if status:
                query = query.where(WorkflowRun.status == status)

            results = session.scalars(query.limit(limit + 1)).all()

            has_more = len(results) > limit
            items = results[:limit]

            return InfiniteScrollPagination(
                items=items,
                has_more=has_more,
            )
```

### 5.3 Repository Factory

**Reference**: `/api/repositories/factory.py` (lines 17-82)

```python
from werkzeug.utils import import_string
from configs import dify_config

class DifyAPIRepositoryFactory(DifyCoreRepositoryFactory):
    """Factory for creating DifyAPI repository instances."""

    @classmethod
    def create_api_workflow_node_execution_repository(
        cls,
        session_maker: sessionmaker[Session],
    ) -> DifyAPIWorkflowNodeExecutionRepository:
        """Create repository with dependency injection."""
        # Configuration-driven class loading
        class_path = dify_config.API_WORKFLOW_NODE_EXECUTION_REPOSITORY

        try:
            repository_class = import_string(class_path)
            return repository_class(session_maker=session_maker)
        except (ImportError, Exception) as e:
            raise RepositoryImportError(
                f"Failed to create repository: {e}"
            ) from e
```

## 6. Model Layer (Entities)

### 6.1 Base Model Classes

**Reference**: `/api/models/base.py` (lines 1-50)

```python
from datetime import datetime
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    """Base class for all models."""
    metadata = metadata

class DefaultFieldsMixin:
    """Common fields for all entities."""

    id: Mapped[str] = mapped_column(
        StringUUID,
        primary_key=True,
        default=lambda: str(uuidv7()),
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        nullable=False,
        default=naive_utc_now,
        server_default=func.current_timestamp(),
    )

    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        nullable=False,
        default=naive_utc_now,
        server_default=func.current_timestamp(),
        onupdate=func.current_timestamp(),
    )

    def __repr__(self) -> str:
        return f"<{self.__class__.__name__}(id={self.id})>"
```

### 6.2 Domain Models

**Reference**: `/api/models/account.py` (lines 87-100)

```python
from flask_login import UserMixin
from sqlalchemy import String, Index

class Account(UserMixin, TypeBase):
    """User account model."""
    __tablename__ = "accounts"
    __table_args__ = (
        sa.PrimaryKeyConstraint("id", name="account_pkey"),
        sa.Index("account_email_idx", "email"),
    )

    id: Mapped[str] = mapped_column(
        StringUUID,
        insert_default=lambda: str(uuid4()),
        default_factory=lambda: str(uuid4()),
        init=False,
    )
    name: Mapped[str] = mapped_column(String(255))
    email: Mapped[str] = mapped_column(String(255))
    password: Mapped[str | None] = mapped_column(String(255), default=None)
    password_salt: Mapped[str | None] = mapped_column(String(255), default=None)
    avatar: Mapped[str | None] = mapped_column(String(255), default=None)
    interface_language: Mapped[str | None] = mapped_column(String(255))
    interface_theme: Mapped[str] = mapped_column(String(255), default="light")
    timezone: Mapped[str | None] = mapped_column(String(255))
    last_login_at: Mapped[datetime | None] = mapped_column(DateTime)
    last_login_ip: Mapped[str | None] = mapped_column(String(255))
    last_active_at: Mapped[datetime] = mapped_column(
        DateTime,
        nullable=False,
        server_default=func.current_timestamp(),
    )
    status: Mapped[str] = mapped_column(
        String(16),
        nullable=False,
        server_default=AccountStatus.ACTIVE.value,
    )
```

### 6.3 Enum Models

**Reference**: `/api/models/account.py` (lines 19-77)

```python
import enum

class TenantAccountRole(enum.StrEnum):
    """Roles within a tenant."""
    OWNER = "owner"
    ADMIN = "admin"
    EDITOR = "editor"
    NORMAL = "normal"
    DATASET_OPERATOR = "dataset_operator"

    @staticmethod
    def is_valid_role(role: str) -> bool:
        """Check if role string is valid."""
        return role in {
            TenantAccountRole.OWNER,
            TenantAccountRole.ADMIN,
            TenantAccountRole.EDITOR,
            TenantAccountRole.NORMAL,
            TenantAccountRole.DATASET_OPERATOR,
        }

    @staticmethod
    def is_privileged_role(role: Optional["TenantAccountRole"]) -> bool:
        """Check if role has elevated privileges."""
        return role in {TenantAccountRole.OWNER, TenantAccountRole.ADMIN}
```

## 7. Extension System

### 7.1 Database Extension

**Reference**: `/api/extensions/ext_database.py` (lines 53-55)

```python
from models.engine import db

def init_app(app: DifyApp):
    """Initialize SQLAlchemy with Flask app."""
    db.init_app(app)
    _setup_gevent_compatibility()
```

### 7.2 Redis Extension

**Reference**: `/api/extensions/ext_redis.py` (lines 229-245)

```python
class RedisClientWrapper:
    """Wrapper for Redis client supporting dynamic re-initialization."""

    _client: Union[redis.Redis, RedisCluster, None]

    def __init__(self) -> None:
        self._client = None

    def initialize(self, client: Union[redis.Redis, RedisCluster]) -> None:
        if self._client is None:
            self._client = client

    def __getattr__(self, item: str) -> Any:
        if self._client is None:
            raise RuntimeError(
                "Redis client is not initialized. Call init_app first."
            )
        return getattr(self._client, item)

redis_client: RedisClientWrapper = RedisClientWrapper()

def init_app(app: DifyApp):
    global redis_client

    # Determine Redis mode
    if dify_config.REDIS_USE_SENTINEL:
        client = _create_sentinel_client(...)
    elif dify_config.REDIS_USE_CLUSTERS:
        client = _create_cluster_client()
    else:
        client = _create_standalone_client(...)

    redis_client.initialize(client)
    app.extensions["redis"] = redis_client
```

### 7.3 Celery Extension

**Reference**: `/api/extensions/ext_celery.py` (lines 47-97)

```python
from celery import Celery, Task

def init_app(app: DifyApp) -> Celery:
    """Initialize Celery with Flask app context."""

    class FlaskTask(Task):
        """Task with Flask application context."""
        def __call__(self, *args: object, **kwargs: object) -> object:
            with app.app_context():
                return self.run(*args, **kwargs)

    celery_app = Celery(
        app.name,
        task_cls=FlaskTask,
        broker=dify_config.CELERY_BROKER_URL,
        backend=dify_config.CELERY_BACKEND,
    )

    # Configure Celery
    celery_app.conf.update(
        task_serializer="json",
        result_serializer="json",
        accept_content=["json"],
        timezone="UTC",
        enable_utc=True,
        # Queue routing
        task_routes={
            "tasks.workflow_*": {"queue": "workflow"},
            "tasks.rag_*": {"queue": "rag"},
            # ...
        },
    )

    celery_app.set_default()
    app.extensions["celery"] = celery_app
    return celery_app
```

## 8. Context Management

### 8.1 RecyclableContextVar

**Reference**: `/api/contexts/wrapper.py` (lines 14-65)

```python
from contextvars import ContextVar
from typing import Generic, TypeVar

T = TypeVar("T")

class RecyclableContextVar(Generic[T]):
    """
    Wrapper around ContextVar safe for Gunicorn with thread recycling.

    NOTE: Must call `increment_thread_recycles` before requests.
    """

    _thread_recycles: ContextVar[int] = ContextVar("thread_recycles")

    @classmethod
    def increment_thread_recycles(cls):
        """Increment thread recycle counter at request start."""
        try:
            recycles = cls._thread_recycles.get()
            cls._thread_recycles.set(recycles + 1)
        except LookupError:
            cls._thread_recycles.set(0)

    def __init__(self, context_var: ContextVar[T]):
        self._context_var = context_var
        self._updates = ContextVar[int](
            context_var.name + "_updates",
            default=0,
        )

    def get(self, default: T | HiddenValue = _default) -> T:
        """Get value, handling thread recycling."""
        thread_recycles = self._thread_recycles.get(0)
        self_updates = self._updates.get()

        if thread_recycles > self_updates:
            self._updates.set(thread_recycles)

        if thread_recycles < self_updates:
            return self._context_var.get()
        else:
            # Thread recycled - return default or raise
            if isinstance(default, HiddenValue):
                raise LookupError
            return default
```

## 9. Async Task Processing (Celery)

### 9.1 Task Definition

**Reference**: `/api/tasks/workflow_execution_tasks.py` (lines 24-82)

```python
from celery import shared_task

@shared_task(
    queue="workflow_storage",
    bind=True,
    max_retries=3,
)
def save_workflow_execution_task(
    self,
    execution_data: dict,
    tenant_id: str,
    app_id: str,
    triggered_from: str,
    creator_user_id: str,
    creator_user_role: str,
) -> bool:
    """Asynchronously save workflow run to database."""
    try:
        execution = WorkflowExecution.model_validate(execution_data)

        with Session(db.engine) as session:
            existing_run = session.scalar(
                select(WorkflowRun).where(
                    WorkflowRun.id == execution.id
                )
            )

            if existing_run:
                _update_workflow_run_from_execution(existing_run, execution)
            else:
                workflow_run = _create_workflow_run_from_execution(
                    execution,
                    tenant_id,
                    app_id,
                    triggered_from,
                    creator_user_id,
                    creator_user_role,
                )
                session.add(workflow_run)

            session.commit()
            return True

    except Exception as e:
        logger.exception("Failed to save workflow execution")
        raise self.retry(exc=e)
```

### 9.2 Queue Configuration

```python
# Queue routing in Celery config
task_routes = {
    "tasks.workflow_execution_tasks.*": {"queue": "workflow_storage"},
    "tasks.trigger_processing_tasks.*": {"queue": "trigger"},
    "tasks.rag_pipeline.*": {"queue": "rag"},
    "tasks.annotation.*": {"queue": "annotation"},
}

# Queue definitions
task_queues = [
    Queue("default", routing_key="default"),
    Queue("workflow", routing_key="workflow"),
    Queue("workflow_storage", routing_key="workflow_storage"),
    Queue("rag", routing_key="rag"),
    Queue("trigger", routing_key="trigger"),
]
```

## 10. Configuration Hierarchy

### 10.1 DifyConfig Class

**Reference**: `/api/configs/app_config.py` (lines 58-76)

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class DifyConfig(
    PackagingInfo,           # Version, build info
    DeploymentConfig,        # Deployment settings
    FeatureConfig,           # Feature flags
    MiddlewareConfig,        # Middleware settings
    ExtraServiceConfig,      # External services
    ObservabilityConfig,     # Monitoring settings
    RemoteSettingsSourceConfig,  # Remote config
    EnterpriseFeatureConfig,     # Enterprise features
):
    """Hierarchical configuration."""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    @classmethod
    def settings_customise_sources(cls, settings_cls, ...):
        """Custom configuration sources in priority order."""
        return (
            init_settings,       # Explicit init values
            env_settings,        # Environment variables
            dotenv_settings,     # .env file
            remote_settings,     # Remote sources (Apollo, Nacos)
            file_secret_settings,  # Secret files
        )
```

### 10.2 Configuration Categories

```python
# /api/configs/deploy/__init__.py
class DeploymentConfig:
    CONSOLE_WEB_URL: str
    SERVICE_API_URL: str
    APP_WEB_URL: str
    FILES_URL: str

# /api/configs/feature/__init__.py
class FeatureConfig:
    BILLING_ENABLED: bool = False
    CODE_EXECUTION_ENABLED: bool = True
    WORKFLOW_MAX_EXECUTION_STEPS: int = 500
    WORKFLOW_MAX_EXECUTION_TIME: int = 1200  # seconds

# /api/configs/middleware/__init__.py
class MiddlewareConfig:
    REDIS_HOST: str = "localhost"
    REDIS_PORT: int = 6379
    CELERY_BROKER_URL: str
    CELERY_BACKEND: str
```

## 11. Sequence Diagram - API Request Flow

```
┌──────────┐     ┌────────────┐     ┌───────────┐     ┌────────────┐     ┌──────────┐
│  Client  │     │ Controller │     │  Service  │     │ Repository │     │ Database │
└────┬─────┘     └─────┬──────┘     └─────┬─────┘     └─────┬──────┘     └────┬─────┘
     │                 │                  │                 │                 │
     │  HTTP Request   │                  │                 │                 │
     │────────────────►│                  │                 │                 │
     │                 │                  │                 │                 │
     │                 │  Validate Auth   │                 │                 │
     │                 │──────────────────│                 │                 │
     │                 │                  │                 │                 │
     │                 │  Parse Args      │                 │                 │
     │                 │──────────────────│                 │                 │
     │                 │                  │                 │                 │
     │                 │  Call Service    │                 │                 │
     │                 │─────────────────►│                 │                 │
     │                 │                  │                 │                 │
     │                 │                  │  Business Logic │                 │
     │                 │                  │─────────────────│                 │
     │                 │                  │                 │                 │
     │                 │                  │  Query Data     │                 │
     │                 │                  │────────────────►│                 │
     │                 │                  │                 │                 │
     │                 │                  │                 │  SQL Query      │
     │                 │                  │                 │────────────────►│
     │                 │                  │                 │                 │
     │                 │                  │                 │  Result Set     │
     │                 │                  │                 │◄────────────────│
     │                 │                  │                 │                 │
     │                 │                  │  Domain Objects │                 │
     │                 │                  │◄────────────────│                 │
     │                 │                  │                 │                 │
     │                 │  Result/DTO      │                 │                 │
     │                 │◄─────────────────│                 │                 │
     │                 │                  │                 │                 │
     │  JSON Response  │                  │                 │                 │
     │◄────────────────│                  │                 │                 │
     │                 │                  │                 │                 │
```

## 12. Key Design Patterns Summary

| Pattern | Location | Purpose |
|---------|----------|---------|
| **Factory** | `app_factory.py` | App creation with DI |
| **Repository** | `/repositories/` | Data access abstraction |
| **Protocol** | Core repositories | Interface definitions |
| **Service Layer** | `/services/` | Business logic encapsulation |
| **Decorator** | Controller methods | Cross-cutting concerns |
| **Extension** | `/extensions/` | Framework integration |
| **Context Variable** | `/contexts/` | Thread-safe request state |
| **Task Queue** | Celery tasks | Async processing |

---

*Tiếp theo: [Workflow Engine](./03-workflow-engine.md)*
