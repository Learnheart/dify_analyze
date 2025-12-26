# Workflow Engine Architecture

## 1. Tổng quan

Workflow Engine là một **queue-based, distributed execution system** được thiết kế để thực thi các workflow phức tạp theo graph-based model. Engine hỗ trợ:
- 25+ node types (LLM, Code, HTTP, Tool, etc.)
- Real-time streaming qua SSE
- Multi-threaded execution với worker pool
- Pause/Resume/Stop capabilities
- Variable passing giữa các nodes

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         WORKFLOW ENGINE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      GraphEngine                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │   │
│  │  │ Ready Queue │  │ Event Queue │  │    Command Channel      │ │   │
│  │  │ (Node IDs)  │  │  (Events)   │  │   (Stop/Pause/Resume)   │ │   │
│  │  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘ │   │
│  │         │                │                     │               │   │
│  │         ▼                ▼                     ▼               │   │
│  │  ┌─────────────────────────────────────────────────────────┐  │   │
│  │  │                 Worker Pool (N threads)                  │  │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │  │   │
│  │  │  │ Worker1 │  │ Worker2 │  │ Worker3 │  │ WorkerN │    │  │   │
│  │  │  │ (node)  │  │ (node)  │  │ (node)  │  │ (node)  │    │  │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │  │   │
│  │  └─────────────────────────────────────────────────────────┘  │   │
│  │                          │                                     │   │
│  │                          ▼                                     │   │
│  │  ┌─────────────────────────────────────────────────────────┐  │   │
│  │  │                    Dispatcher                            │  │   │
│  │  │  (Event processing, state updates, edge traversal)       │  │   │
│  │  └─────────────────────────────────────────────────────────┘  │   │
│  │                          │                                     │   │
│  │                          ▼                                     │   │
│  │  ┌─────────────────────────────────────────────────────────┐  │   │
│  │  │                 State Manager                            │  │   │
│  │  │  (Node states, Edge states, Variable pool)               │  │   │
│  │  └─────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         Layers                                   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │   │
│  │  │Debug Logging │ │Execution     │ │Observability │             │   │
│  │  │    Layer     │ │Limits Layer  │ │   Layer      │             │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘             │   │
│  │  ┌──────────────┐                                               │   │
│  │  │ Persistence  │                                               │   │
│  │  │    Layer     │                                               │   │
│  │  └──────────────┘                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. Core Components

### 3.1 GraphEngine

**Reference**: `/api/core/workflow/graph_engine/graph_engine.py` (lines 53-364)

```python
class GraphEngine:
    """Main orchestrator for workflow execution."""

    def __init__(
        self,
        workflow_id: str,
        graph: Graph,
        graph_runtime_state: GraphRuntimeState,
        command_channel: CommandChannel,
    ):
        # Core components
        self._workflow_id = workflow_id
        self._graph = graph
        self._graph_runtime_state = graph_runtime_state

        # Queues
        self._ready_queue = InMemoryReadyQueue()
        self._event_queue: queue.Queue[GraphNodeEventBase] = queue.Queue()

        # State management
        self._state_manager = GraphStateManager(
            graph=graph,
            ready_queue=self._ready_queue,
            variable_pool=graph_runtime_state.variable_pool,
        )

        # Workers and dispatcher
        self._worker_pool = WorkerPool(
            ready_queue=self._ready_queue,
            event_queue=self._event_queue,
            graph=self._graph,
            state_manager=self._state_manager,
        )
        self._dispatcher = Dispatcher(
            event_queue=self._event_queue,
            state_manager=self._state_manager,
            command_channel=command_channel,
        )

        # Extensible layers
        self._layers: list[GraphEngineLayer] = []

    def layer(self, layer: GraphEngineLayer) -> "GraphEngine":
        """Add a layer for extensibility."""
        self._layers.append(layer)
        return self

    def run(self) -> Generator[GraphEngineEvent, None, None]:
        """Execute the workflow and yield events."""
        # Initialize layers
        for layer in self._layers:
            layer.initialize(
                self._graph_runtime_state,
                self._command_channel,
            )

        # Start execution
        yield GraphRunStartedEvent(...)

        # Enqueue root node
        self._state_manager.enqueue_node(self._graph.root_node.id)

        # Start worker pool and dispatcher
        self._worker_pool.start()
        self._dispatcher.start()

        # Yield events as they occur
        while not self._execution_complete:
            event = self._event_queue.get(timeout=0.1)
            yield event

            # Pass events through layers
            for layer in self._layers:
                layer.on_event(event)

        # Cleanup and final event
        self._worker_pool.stop()
        self._dispatcher.stop()
        yield GraphRunSucceededEvent(...)
```

### 3.2 Ready Queue

**Reference**: `/api/core/workflow/graph_engine/ready_queue/in_memory.py` (lines 1-141)

```python
import queue
from typing import Protocol

class ReadyQueue(Protocol):
    """Protocol for node ready queue."""

    def put(self, item: str) -> None:
        """Add node_id to ready queue."""
        ...

    def get(self, timeout: float | None = None) -> str:
        """Get next node_id to execute."""
        ...

    def task_done(self) -> None:
        """Mark task as completed."""
        ...

class InMemoryReadyQueue(ReadyQueue):
    """Thread-safe in-memory implementation."""

    def __init__(self, maxsize: int = 0):
        self._queue: queue.Queue[str] = queue.Queue(maxsize=maxsize)

    def put(self, item: str) -> None:
        self._queue.put(item)

    def get(self, timeout: float | None = None) -> str:
        return self._queue.get(timeout=timeout)

    def task_done(self) -> None:
        self._queue.task_done()
```

### 3.3 Worker Pool và Worker

**Reference**: `/api/core/workflow/graph_engine/worker.py` (lines 29-178)

```python
import threading

class Worker(threading.Thread):
    """Worker thread for node execution."""

    def __init__(
        self,
        ready_queue: ReadyQueue,
        event_queue: queue.Queue[GraphNodeEventBase],
        graph: Graph,
        state_manager: GraphStateManager,
        flask_app: Flask | None = None,
        context_vars: dict | None = None,
    ):
        super().__init__(daemon=True)
        self._ready_queue = ready_queue
        self._event_queue = event_queue
        self._graph = graph
        self._state_manager = state_manager
        self._flask_app = flask_app
        self._context_vars = context_vars
        self._stop_event = threading.Event()

    def run(self) -> None:
        """Main worker loop."""
        while not self._stop_event.is_set():
            try:
                # Get next node from queue (blocks with timeout)
                node_id = self._ready_queue.get(timeout=0.1)
            except queue.Empty:
                continue

            # Execute node
            node = self._graph.nodes[node_id]
            try:
                self._execute_node(node)
            except Exception as e:
                error_event = NodeRunFailedEvent(
                    node_id=node_id,
                    error=str(e),
                )
                self._event_queue.put(error_event)
            finally:
                self._ready_queue.task_done()

    def _execute_node(self, node: Node) -> None:
        """Execute a single node."""
        node.ensure_execution_id()

        # Preserve Flask context if available
        if self._flask_app and self._context_vars:
            with preserve_flask_contexts(self._flask_app, self._context_vars):
                self._run_node(node)
        else:
            self._run_node(node)

    def _run_node(self, node: Node) -> None:
        """Run node and emit events."""
        # Invoke pre-hooks
        self._invoke_node_run_start_hooks(node)

        try:
            # Run node - returns generator of events
            node_events = node.run()
            for event in node_events:
                self._event_queue.put(event)
        except Exception as exc:
            raise
        finally:
            # Invoke post-hooks
            self._invoke_node_run_end_hooks(node, error=None)

    def stop(self) -> None:
        """Signal worker to stop."""
        self._stop_event.set()
```

### 3.4 Dispatcher

**Reference**: `/api/core/workflow/graph_engine/orchestration/dispatcher.py` (lines 28-126)

```python
class Dispatcher(threading.Thread):
    """Processes events and manages workflow state."""

    def __init__(
        self,
        event_queue: queue.Queue[GraphNodeEventBase],
        state_manager: GraphStateManager,
        command_channel: CommandChannel,
        execution_coordinator: ExecutionCoordinator,
    ):
        super().__init__(daemon=True)
        self._event_queue = event_queue
        self._state_manager = state_manager
        self._command_channel = command_channel
        self._execution_coordinator = execution_coordinator
        self._event_handler = EventHandler(state_manager)
        self._stop_event = threading.Event()

    def run(self) -> None:
        """Main dispatcher loop."""
        try:
            # Process any pending commands at start
            self._process_commands()

            while not self._stop_event.is_set():
                # Check termination conditions
                if (
                    self._execution_coordinator.aborted
                    or self._execution_coordinator.paused
                    or self._execution_coordinator.execution_complete
                ):
                    break

                # Check for worker scaling
                self._execution_coordinator.check_scaling()

                try:
                    # Get and dispatch event
                    event = self._event_queue.get(timeout=0.1)
                    self._event_handler.dispatch(event)
                    self._event_queue.task_done()

                    # Check for commands after event
                    self._process_commands(event)
                except queue.Empty:
                    time.sleep(0.1)

        except Exception as e:
            logger.exception("Dispatcher error")
            self._execution_coordinator.abort(str(e))

    def _process_commands(self, event=None) -> None:
        """Process any pending commands from channel."""
        while True:
            command = self._command_channel.recv_command()
            if command is None:
                break

            if isinstance(command, AbortCommand):
                self._execution_coordinator.abort(command.reason)
            elif isinstance(command, PauseCommand):
                self._execution_coordinator.pause(command.reason)
```

### 3.5 State Manager

**Reference**: `/api/core/workflow/graph_engine/graph_state_manager.py` (lines 24-100)

```python
import threading

class GraphStateManager:
    """Thread-safe state management for workflow execution."""

    def __init__(
        self,
        graph: Graph,
        ready_queue: ReadyQueue,
        variable_pool: VariablePool,
    ):
        self._graph = graph
        self._ready_queue = ready_queue
        self._variable_pool = variable_pool
        self._lock = threading.RLock()

    def enqueue_node(self, node_id: str) -> None:
        """Mark node as TAKEN and add to ready queue."""
        with self._lock:
            self._graph.nodes[node_id].state = NodeState.TAKEN
            self._ready_queue.put(node_id)

    def is_node_ready(self, node_id: str) -> bool:
        """Check if all incoming edges are resolved."""
        with self._lock:
            incoming_edges = self._graph.get_incoming_edges(node_id)

            if not incoming_edges:
                return True

            # At least one incoming edge must be TAKEN
            if any(edge.state == NodeState.UNKNOWN for edge in incoming_edges):
                return False

            return any(edge.state == NodeState.TAKEN for edge in incoming_edges)

    def mark_node_succeeded(
        self,
        node_id: str,
        outputs: dict,
        edge_source_handle: str | None = None,
    ) -> list[str]:
        """Mark node as succeeded and determine next nodes."""
        with self._lock:
            node = self._graph.nodes[node_id]
            node.state = NodeState.SUCCEEDED

            # Update variable pool with outputs
            for key, value in outputs.items():
                self._variable_pool.add([node_id, key], value)

            # Determine outgoing edges to traverse
            next_nodes = []
            for edge in self._graph.get_outgoing_edges(node_id):
                if edge_source_handle and edge.source_handle != edge_source_handle:
                    edge.state = NodeState.SKIPPED
                    continue

                edge.state = NodeState.TAKEN
                target_node_id = edge.target

                # Check if target node is ready
                if self.is_node_ready(target_node_id):
                    next_nodes.append(target_node_id)

            return next_nodes

    def mark_node_skipped(self, node_id: str) -> None:
        """Mark node as SKIPPED."""
        with self._lock:
            self._graph.nodes[node_id].state = NodeState.SKIPPED
```

## 4. Node System

### 4.1 Node Base Class

**Reference**: `/api/core/workflow/nodes/base/node.py` (lines 61-450)

```python
from typing import Generic, TypeVar, ClassVar

NodeDataT = TypeVar("NodeDataT", bound=NodeData)

class Node(Generic[NodeDataT]):
    """Base class for all workflow nodes."""

    # Class registry for node types
    _registry: ClassVar[dict["NodeType", dict[str, type["Node"]]]] = {}

    # Node type identifier (set in subclasses)
    node_type: ClassVar[NodeType]

    # Execution type
    execution_type: ClassVar[NodeExecutionType] = NodeExecutionType.NORMAL

    def __init_subclass__(cls, **kwargs):
        """Auto-register node types when subclassed."""
        super().__init_subclass__(**kwargs)

        # Extract node data type from generic parameter
        node_data_type = cls._extract_node_data_type_from_generic()
        cls._node_data_type = node_data_type

        # Register in global registry
        if hasattr(cls, "node_type"):
            node_type = cls.node_type
            version = cls.version()
            bucket = Node._registry.setdefault(node_type, {})
            bucket[version] = cls
            bucket["latest"] = bucket[max(bucket.keys())]

    def __init__(
        self,
        node_id: str,
        config: NodeConfig,
        graph_runtime_state: GraphRuntimeState,
    ):
        self._node_id = node_id
        self._config = config
        self._graph_runtime_state = graph_runtime_state

        # Parse and validate node data
        self._node_data: NodeDataT = self._parse_node_data()

    @property
    def node_data(self) -> NodeDataT:
        """Get typed node configuration."""
        return self._node_data

    @property
    def graph_runtime_state(self) -> GraphRuntimeState:
        """Access runtime state including variable pool."""
        return self._graph_runtime_state

    def run(self) -> Generator[GraphNodeEventBase, None, None]:
        """Main node execution method."""
        execution_id = self.ensure_execution_id()
        self._start_at = naive_utc_now()

        # Emit start event
        yield NodeRunStartedEvent(
            id=execution_id,
            node_id=self._node_id,
            node_type=self.node_type,
            node_title=self.title,
            start_at=self._start_at,
        )

        try:
            # Execute node implementation
            result = self._run()

            if isinstance(result, NodeRunResult):
                yield self._convert_to_event(result)
                return

            # Handle generator results
            for event in result:
                if isinstance(event, NodeEventBase):
                    yield self._dispatch(event)
                else:
                    yield event

        except Exception as e:
            logger.exception("Node %s failed", self._node_id)
            yield NodeRunFailedEvent(
                node_id=self._node_id,
                error=str(e),
            )

    @abstractmethod
    def _run(self) -> NodeRunResult | Generator[NodeEventBase, None, None]:
        """Implement node logic in subclasses."""
        ...
```

### 4.2 Node Types (25+)

**Reference**: `/api/core/workflow/enums.py` (lines 36-86)

```python
from enum import StrEnum

class NodeType(StrEnum):
    """All available workflow node types."""

    # Entry/Exit
    START = "start"
    END = "end"
    ANSWER = "answer"

    # AI/LLM
    LLM = "llm"
    AGENT = "agent"

    # Logic
    IF_ELSE = "if-else"
    QUESTION_CLASSIFIER = "question-classifier"

    # Operations
    CODE = "code"
    HTTP_REQUEST = "http-request"
    TOOL = "tool"
    KNOWLEDGE_RETRIEVAL = "knowledge-retrieval"
    KNOWLEDGE_INDEX = "knowledge-index"

    # Control Flow
    ITERATION = "iteration"
    LOOP = "loop"
    LOOP_START = "loop-start"
    LOOP_END = "loop-end"
    ITERATION_START = "iteration-start"

    # Data Operations
    VARIABLE_AGGREGATOR = "variable-aggregator"
    VARIABLE_ASSIGNER = "variable-assigner"
    TEMPLATE_TRANSFORM = "template-transform"
    PARAMETER_EXTRACTOR = "parameter-extractor"
    DOCUMENT_EXTRACTOR = "document-extractor"
    LIST_OPERATOR = "list-operator"

    # Triggers
    TRIGGER_WEBHOOK = "trigger-webhook"
    TRIGGER_SCHEDULE = "trigger-schedule"
    TRIGGER_PLUGIN = "trigger-plugin"

    # Input
    HUMAN_INPUT = "human-input"
    DATASOURCE = "datasource"
```

### 4.3 Example Node Implementations

#### Start Node (Simplest)

**Reference**: `/api/core/workflow/nodes/start/start_node.py` (lines 14-68)

```python
class StartNode(Node[StartNodeData]):
    """Workflow entry point node."""

    node_type = NodeType.START
    execution_type = NodeExecutionType.ROOT

    def _run(self) -> NodeRunResult:
        """Validate and pass through user inputs."""
        # Get user inputs from variable pool
        node_inputs = dict(
            self.graph_runtime_state.variable_pool.user_inputs
        )

        # Validate JSON inputs
        self._validate_and_normalize_json_object_inputs(node_inputs)

        # Add system variables
        system_inputs = self.graph_runtime_state.variable_pool.system_variables.to_dict()
        for var in system_inputs:
            node_inputs[SYSTEM_VARIABLE_NODE_ID + "." + var] = system_inputs[var]

        # Return success
        outputs = dict(node_inputs)
        return NodeRunResult(
            status=WorkflowNodeExecutionStatus.SUCCEEDED,
            inputs=node_inputs,
            outputs=outputs,
        )
```

#### If-Else Node (Branching)

**Reference**: `/api/core/workflow/nodes/if_else/if_else_node.py` (lines 15-92)

```python
class IfElseNode(Node[IfElseNodeData]):
    """Conditional branching node."""

    node_type = NodeType.IF_ELSE
    execution_type = NodeExecutionType.BRANCH

    def _run(self) -> NodeRunResult:
        """Evaluate conditions and select branch."""
        condition_processor = ConditionProcessor()
        selected_case_id = "false"

        # Evaluate each case in order
        for case in self.node_data.cases:
            input_conditions, group_result, final_result = (
                condition_processor.process_conditions(
                    variable_pool=self.graph_runtime_state.variable_pool,
                    conditions=case.conditions,
                    operator=case.logical_operator,
                )
            )

            if final_result:
                selected_case_id = case.case_id
                break

        # Return with edge_source_handle for routing
        outputs = {
            "result": final_result,
            "selected_case_id": selected_case_id,
        }
        return NodeRunResult(
            status=WorkflowNodeExecutionStatus.SUCCEEDED,
            inputs=node_inputs,
            process_data=process_data,
            edge_source_handle=selected_case_id,  # Routes to specific output
            outputs=outputs,
        )
```

#### LLM Node (Complex with Streaming)

**Reference**: `/api/core/workflow/nodes/llm/node.py` (lines 108-500)

```python
class LLMNode(Node[LLMNodeData]):
    """Node for invoking LLM models."""

    node_type = NodeType.LLM
    execution_type = NodeExecutionType.NORMAL

    def _run(self) -> Generator[NodeEventBase, None, None]:
        """Execute LLM call with streaming support."""
        # Prepare prompt from template
        prompt = self._prepare_prompt()

        # Get model credentials
        model_instance = self._get_model_instance()

        # Invoke LLM
        try:
            response = model_instance.invoke(
                prompt=prompt,
                stream=True,
                temperature=self.node_data.temperature,
                max_tokens=self.node_data.max_tokens,
            )

            # Stream response chunks
            full_text = ""
            for chunk in response:
                full_text += chunk.text
                yield NodeRunStreamChunkEvent(
                    node_id=self._node_id,
                    chunk=chunk.text,
                )

            # Final result
            yield NodeRunSucceededEvent(
                node_id=self._node_id,
                outputs={"text": full_text},
            )

        except Exception as e:
            yield NodeRunFailedEvent(
                node_id=self._node_id,
                error=str(e),
            )
```

## 5. Variable System

### 5.1 Variable Pool

**Reference**: `/api/core/workflow/runtime/variable_pool.py` (lines 28-80)

```python
from collections import defaultdict
from pydantic import BaseModel

class VariablePool(BaseModel):
    """Container for all workflow variables."""

    # Nested dict for fast lookups: {node_id: {var_name: value}}
    variable_dictionary: defaultdict[str, dict[str, VariableUnion]] = Field(
        default_factory=lambda: defaultdict(dict)
    )

    # Input sources
    user_inputs: Mapping[str, Any]
    system_variables: SystemVariable
    environment_variables: Sequence[VariableUnion]
    conversation_variables: Sequence[VariableUnion]
    rag_pipeline_variables: list[RAGPipelineVariableInput]

    def add(self, selector: Sequence[str], value: Any) -> None:
        """
        Add variable with path-like selector.

        Args:
            selector: ["node_id", "variable_name"]
            value: Variable value
        """
        if len(selector) < 2:
            raise ValueError("Selector must have at least 2 elements")

        node_id = selector[0]
        var_name = ".".join(selector[1:])
        self.variable_dictionary[node_id][var_name] = value

    def get(self, selector: Sequence[str]) -> VariableValue | None:
        """
        Retrieve variable by selector.

        Args:
            selector: ["node_id", "variable_name", ...]

        Returns:
            Variable value or None
        """
        if len(selector) < 2:
            return None

        node_id = selector[0]
        var_name = ".".join(selector[1:])

        # Check variable dictionary
        if node_id in self.variable_dictionary:
            if var_name in self.variable_dictionary[node_id]:
                return self.variable_dictionary[node_id][var_name]

        # Check system variables
        if node_id == "sys":
            return self.system_variables.get(var_name)

        # Check environment variables
        if node_id == "env":
            for env_var in self.environment_variables:
                if env_var.name == var_name:
                    return env_var.value

        return None
```

### 5.2 Variable Selector Format

```python
# Output variable from node
["llm_node_1", "text"]          # LLM node's text output
["code_node_1", "result"]       # Code node's result

# Nested field access
["http_node_1", "response", "data", "items"]

# System variables
["sys", "workflow_id"]
["sys", "user_id"]
["sys", "files"]
["sys", "conversation_id"]

# Environment variables
["env", "api_key"]
["env", "database_url"]

# Conversation variables
["conversation", "message_history"]
```

## 6. Graph Structure

### 6.1 Graph Class

**Reference**: `/api/core/workflow/graph/graph.py` (lines 35-125)

```python
from dataclasses import dataclass

@dataclass
class Graph:
    """Workflow graph representation."""

    nodes: dict[str, Node]              # node_id -> Node instance
    edges: dict[str, Edge]              # edge_id -> Edge
    in_edges: dict[str, list[str]]      # node_id -> incoming edge_ids
    out_edges: dict[str, list[str]]     # node_id -> outgoing edge_ids
    root_node: Node                     # Entry point

    @classmethod
    def build(
        cls,
        graph_config: dict,
        node_factory: NodeFactory,
        root_node_id: str | None = None,
    ) -> "Graph":
        """Build graph from JSON config."""
        nodes = {}
        edges = {}
        in_edges = defaultdict(list)
        out_edges = defaultdict(list)

        # Parse nodes
        for node_config in graph_config["nodes"]:
            node = node_factory.create_node(node_config)
            nodes[node.id] = node

        # Parse edges
        for edge_config in graph_config["edges"]:
            edge = Edge(
                id=edge_config["id"],
                source=edge_config["source"],
                target=edge_config["target"],
                source_handle=edge_config.get("sourceHandle"),
            )
            edges[edge.id] = edge
            in_edges[edge.target].append(edge.id)
            out_edges[edge.source].append(edge.id)

        # Find root node
        if root_node_id:
            root_node = nodes[root_node_id]
        else:
            # Find node with no incoming edges
            root_node = next(
                node for node in nodes.values()
                if node.id not in in_edges
            )

        return cls(
            nodes=nodes,
            edges=edges,
            in_edges=dict(in_edges),
            out_edges=dict(out_edges),
            root_node=root_node,
        )

    def get_incoming_edges(self, node_id: str) -> list[Edge]:
        """Get all incoming edges for a node."""
        edge_ids = self.in_edges.get(node_id, [])
        return [self.edges[eid] for eid in edge_ids]

    def get_outgoing_edges(self, node_id: str) -> list[Edge]:
        """Get all outgoing edges for a node."""
        edge_ids = self.out_edges.get(node_id, [])
        return [self.edges[eid] for eid in edge_ids]
```

### 6.2 Node State Machine

```
┌───────────────────────────────────────────────────────────────┐
│                    NODE STATE MACHINE                          │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│                        ┌─────────┐                            │
│                        │ UNKNOWN │ (Initial state)            │
│                        └────┬────┘                            │
│                             │                                  │
│              ┌──────────────┼──────────────┐                  │
│              │              │              │                  │
│              ▼              ▼              ▼                  │
│       ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│       │  TAKEN   │   │ SKIPPED  │   │  FAILED  │             │
│       │(Enqueued)│   │(Condition│   │  (Error) │             │
│       └────┬─────┘   │  false)  │   └──────────┘             │
│            │         └──────────┘                             │
│            │                                                   │
│            ▼                                                   │
│     ┌──────────┐                                              │
│     │SUCCEEDED │                                              │
│     │(Complete)│                                              │
│     └──────────┘                                              │
│                                                                │
└───────────────────────────────────────────────────────────────┘

Edge states follow the same pattern.
```

## 7. Command Channel

### 7.1 Redis Channel Implementation

**Reference**: `/api/core/workflow/graph_engine/command_channels/redis_channel.py`

```python
import json
from redis import Redis

class RedisChannel(CommandChannel):
    """Redis-based command channel for external control."""

    def __init__(self, redis_client: Redis, key: str):
        self._redis = redis_client
        self._key = key  # "workflow:{task_id}:commands"

    def send_command(self, command: GraphEngineCommand) -> None:
        """Send stop/pause command to workflow."""
        self._redis.rpush(
            self._key,
            json.dumps(command.model_dump()),
        )

    def recv_command(self) -> GraphEngineCommand | None:
        """Receive command (non-blocking)."""
        data = self._redis.lpop(self._key)
        if data is None:
            return None
        return GraphEngineCommand.model_validate_json(data)
```

### 7.2 Stop Mechanism

**Reference**: `/api/core/workflow/graph_engine/manager.py` (lines 19-64)

```python
class GraphEngineManager:
    """API for controlling workflow execution."""

    @staticmethod
    def send_stop_command(
        task_id: str,
        reason: str | None = None,
    ) -> None:
        """Stop a running workflow."""
        abort_command = AbortCommand(
            reason=reason or "User requested stop"
        )
        channel = RedisChannel(
            redis_client,
            f"workflow:{task_id}:commands",
        )
        channel.send_command(abort_command)

    @staticmethod
    def send_pause_command(
        task_id: str,
        reason: str | None = None,
    ) -> None:
        """Pause a running workflow."""
        pause_command = PauseCommand(
            reason=reason or "User requested pause"
        )
        channel = RedisChannel(
            redis_client,
            f"workflow:{task_id}:commands",
        )
        channel.send_command(pause_command)
```

## 8. Extensible Layer System

### 8.1 Layer Base Class

**Reference**: `/api/core/workflow/graph_engine/layers/base.py`

```python
from abc import ABC

class GraphEngineLayer(ABC):
    """Base class for engine layers."""

    def initialize(
        self,
        runtime_state: GraphRuntimeState,
        command_channel: CommandChannel,
    ) -> None:
        """Called when engine starts."""
        pass

    def on_graph_start(self) -> None:
        """Called when graph execution begins."""
        pass

    def on_event(self, event: GraphEngineEvent) -> None:
        """Called for each event."""
        pass

    def on_graph_end(self, error: Exception | None = None) -> None:
        """Called when graph execution ends."""
        pass

    def on_node_run_start(self, node: Node) -> None:
        """Called before node execution."""
        pass

    def on_node_run_end(
        self,
        node: Node,
        error: Exception | None = None,
    ) -> None:
        """Called after node execution."""
        pass
```

### 8.2 Built-in Layers

```python
# Debug Logging Layer
class DebugLoggingLayer(GraphEngineLayer):
    """Detailed execution logs for debugging."""

    def on_event(self, event: GraphEngineEvent) -> None:
        logger.debug(f"Event: {event}")

# Execution Limits Layer
class ExecutionLimitsLayer(GraphEngineLayer):
    """Enforces execution limits."""

    def __init__(self, max_steps: int, max_time: int):
        self._max_steps = max_steps
        self._max_time = max_time
        self._step_count = 0
        self._start_time = None

    def on_graph_start(self) -> None:
        self._start_time = time.time()

    def on_event(self, event: GraphEngineEvent) -> None:
        if isinstance(event, NodeRunSucceededEvent):
            self._step_count += 1
            if self._step_count > self._max_steps:
                raise ExecutionLimitExceededError("Max steps exceeded")

            elapsed = time.time() - self._start_time
            if elapsed > self._max_time:
                raise ExecutionLimitExceededError("Max time exceeded")

# Observability Layer
class ObservabilityLayer(GraphEngineLayer):
    """OpenTelemetry instrumentation."""

    def on_node_run_start(self, node: Node) -> None:
        self._span = tracer.start_span(f"node.{node.node_type}")

    def on_node_run_end(self, node: Node, error=None) -> None:
        if error:
            self._span.set_status(StatusCode.ERROR)
        self._span.end()

# Persistence Layer
class WorkflowPersistenceLayer(GraphEngineLayer):
    """Saves execution to database via Celery."""

    def on_graph_end(self, error=None) -> None:
        save_workflow_execution_task.delay(
            execution_data=self._execution.model_dump(),
            tenant_id=self._tenant_id,
            app_id=self._app_id,
        )
```

## 9. Complete Execution Flow Diagram

```
┌──────────────┐
│ HTTP Request │  POST /workflows/run
│  (Client)    │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Controller Layer                                                  │
│ WorkflowRunApi.post() (/api/controllers/web/workflow.py:53)      │
└──────┬───────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Service Layer                                                     │
│ AppGenerateService.generate() (/api/services/app_generate_service.py:24)
│   ├─ Validates request                                           │
│   ├─ Gets workflow from database                                 │
│   └─ Routes to WorkflowAppGenerator                              │
└──────┬───────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│ App Generator                                                     │
│ WorkflowAppGenerator.generate()                                   │
│ (/api/core/app/apps/workflow/app_generator.py:100)               │
│   ├─ Creates background thread                                   │
│   ├─ Returns SSE stream immediately                              │
│   └─ Background thread calls WorkflowAppRunner                   │
└──────┬───────────────────────────────────────────────────────────┘
       │
       │ (Background Thread)
       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Workflow Runner                                                   │
│ WorkflowAppRunner.run() (/api/core/app/apps/workflow/app_runner.py:61)
│   ├─ Creates VariablePool (user inputs + system vars)            │
│   ├─ Builds Graph from config                                    │
│   ├─ Creates GraphRuntimeState                                   │
│   ├─ Creates WorkflowEntry                                       │
│   └─ Calls WorkflowEntry.run()                                   │
└──────┬───────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Workflow Entry                                                    │
│ WorkflowEntry.run() (/api/core/workflow/workflow_entry.py:106)   │
│   ├─ Creates GraphEngine with layers                             │
│   ├─ Adds DebugLoggingLayer (if DEBUG)                           │
│   ├─ Adds ExecutionLimitsLayer                                   │
│   ├─ Adds ObservabilityLayer (if OTEL enabled)                   │
│   └─ Calls GraphEngine.run()                                     │
└──────┬───────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Graph Engine Execution                                            │
│ GraphEngine.run() (/api/core/workflow/graph_engine/graph_engine.py:220)
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ 1. Initialize layers                                     │    │
│   │ 2. Emit GraphRunStartedEvent                            │    │
│   │ 3. Enqueue root node to ready_queue                     │    │
│   │ 4. Start WorkerPool (N threads)                         │    │
│   │ 5. Start Dispatcher (1 thread)                          │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ Worker Pool Loop:                                        │    │
│   │   while not stopped:                                     │    │
│   │     node_id = ready_queue.get()                         │    │
│   │     node = graph.nodes[node_id]                         │    │
│   │     for event in node.run():                            │    │
│   │       event_queue.put(event)                            │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ Dispatcher Loop:                                         │    │
│   │   while not stopped:                                     │    │
│   │     event = event_queue.get()                           │    │
│   │     event_handler.dispatch(event)                        │    │
│   │     if NodeRunSucceededEvent:                           │    │
│   │       - Update variable pool                            │    │
│   │       - Traverse edges                                  │    │
│   │       - Enqueue ready nodes                             │    │
│   │     check_commands()                                    │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ 6. yield events to caller (SSE stream)                  │    │
│   │ 7. On completion: emit GraphRunSucceededEvent           │    │
│   │ 8. Persistence layer saves to database                  │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│ SSE Stream   │  Events streamed to client in real-time
│  (Client)    │
└──────────────┘
```

## 10. Key Design Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| **Queue-Based** | Ready Queue, Event Queue | Decouple producers/consumers |
| **Worker Pool** | WorkerPool | Parallel node execution |
| **State Machine** | GraphStateManager | Track node/edge states |
| **Observer** | Dispatcher, Events | Event-driven updates |
| **Layer/Middleware** | GraphEngineLayer | Extensibility |
| **Factory** | NodeFactory | Dynamic node creation |
| **Generic Node** | Node[T] | Type-safe node data |
| **Protocol** | Repository interfaces | Abstraction |
| **Command** | CommandChannel | External control |

---

*Tiếp theo: [RAG Pipeline](./04-rag-pipeline.md)*
