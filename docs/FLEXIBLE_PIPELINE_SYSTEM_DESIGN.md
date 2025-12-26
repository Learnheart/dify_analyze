# Hướng dẫn Xây dựng Hệ thống Pipeline Linh hoạt

> Tài liệu thiết kế dựa trên phân tích kiến trúc Dify Workflow Engine

## Mục lục

1. [Tổng quan Kiến trúc](#1-tổng-quan-kiến-trúc)
2. [Thiết kế Graph-based Pipeline](#2-thiết-kế-graph-based-pipeline)
3. [Hệ thống Node Types](#3-hệ-thống-node-types)
4. [Xử lý Edge và Branching](#4-xử-lý-edge-và-branching)
5. [Execution Engine](#5-execution-engine)
6. [Variable Pool & State Management](#6-variable-pool--state-management)
7. [Condition Processing System](#7-condition-processing-system)
8. [File Config Schema](#8-file-config-schema)
9. [Nguyên tắc Thiết kế Cốt lõi](#9-nguyên-tắc-thiết-kế-cốt-lõi)
10. [Hướng dẫn Triển khai](#10-hướng-dẫn-triển-khai)

---

## 1. Tổng quan Kiến trúc

### 1.1 Mô hình Tổng thể

```
┌─────────────────────────────────────────────────────────────────┐
│                     PIPELINE SYSTEM                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │   Config    │───▶│    Graph    │───▶│  Execution Engine   │  │
│  │   Parser    │    │   Builder   │    │                     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│         │                  │                      │              │
│         ▼                  ▼                      ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │    Node     │    │    Edge     │    │   Variable Pool     │  │
│  │   Factory   │    │  Processor  │    │   State Manager     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│                            │                      │              │
│                            ▼                      ▼              │
│                     ┌─────────────┐    ┌─────────────────────┐  │
│                     │    Skip     │    │  Condition          │  │
│                     │ Propagator  │    │  Processor          │  │
│                     └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Các Thành phần Chính

| Thành phần | Chức năng |
|------------|-----------|
| **Graph** | Cấu trúc dữ liệu lưu trữ nodes và edges |
| **Node** | Đơn vị xử lý cơ bản trong pipeline |
| **Edge** | Kết nối giữa các nodes, định hướng luồng dữ liệu |
| **Execution Engine** | Điều phối thực thi các nodes |
| **Variable Pool** | Quản lý biến và trạng thái runtime |
| **Condition Processor** | Đánh giá điều kiện cho branching |
| **Skip Propagator** | Lan truyền skip qua các nhánh không được chọn |

---

## 2. Thiết kế Graph-based Pipeline

### 2.1 Cấu trúc Graph

Pipeline được mô hình hóa dưới dạng **Directed Graph** (không nhất thiết phải acyclic nếu cho phép loops):

```python
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from enum import StrEnum

class NodeState(StrEnum):
    UNKNOWN = "unknown"    # Chưa xử lý
    TAKEN = "taken"        # Đã/sẽ thực thi
    SKIPPED = "skipped"    # Bị bỏ qua

@dataclass
class Edge:
    id: str
    source_node_id: str      # Node bắt đầu (tail)
    target_node_id: str      # Node kết thúc (head)
    source_handle: str = "default"  # Handle cho branching
    state: NodeState = NodeState.UNKNOWN

@dataclass
class Graph:
    nodes: Dict[str, 'Node'] = field(default_factory=dict)
    edges: Dict[str, Edge] = field(default_factory=dict)

    # Mapping để traverse nhanh
    in_edges: Dict[str, List[str]] = field(default_factory=dict)   # node_id -> incoming edge_ids
    out_edges: Dict[str, List[str]] = field(default_factory=dict)  # node_id -> outgoing edge_ids

    root_node_id: Optional[str] = None

    def get_incoming_edges(self, node_id: str) -> List[Edge]:
        """Lấy tất cả edges đi vào một node"""
        edge_ids = self.in_edges.get(node_id, [])
        return [self.edges[eid] for eid in edge_ids]

    def get_outgoing_edges(self, node_id: str) -> List[Edge]:
        """Lấy tất cả edges đi ra từ một node"""
        edge_ids = self.out_edges.get(node_id, [])
        return [self.edges[eid] for eid in edge_ids]

    def get_downstream_nodes(self, node_id: str) -> List[str]:
        """Lấy tất cả nodes tiếp theo"""
        return [self.edges[eid].target_node_id
                for eid in self.out_edges.get(node_id, [])]
```

### 2.2 Tại sao dùng Graph thay vì List?

| Đặc điểm | Linear List | Graph |
|----------|-------------|-------|
| Branching (if-else) | ❌ Không hỗ trợ | ✅ Nhiều out_edges |
| Parallel paths | ❌ Không hỗ trợ | ✅ Fork/Join pattern |
| Loops | ❌ Không hỗ trợ | ✅ Edges quay lại |
| Skip propagation | ❌ Không cần | ✅ DFS/BFS traversal |
| Dependency tracking | ❌ Index-based | ✅ Edge-based |

---

## 3. Hệ thống Node Types

### 3.1 Phân loại Node theo Execution Type

```python
from enum import StrEnum
from abc import ABC, abstractmethod

class NodeExecutionType(StrEnum):
    ROOT = "root"           # Entry points (Start, Trigger)
    EXECUTABLE = "executable"  # Nodes thực thi logic
    BRANCH = "branch"       # Nodes phân nhánh (If-Else)
    CONTAINER = "container" # Nodes chứa subgraph (Loop, Iteration)
    RESPONSE = "response"   # Exit points (End, Answer)

class NodeType(StrEnum):
    # Root nodes
    START = "start"
    TRIGGER_WEBHOOK = "trigger-webhook"
    TRIGGER_SCHEDULE = "trigger-schedule"

    # Executable nodes
    HTTP_REQUEST = "http-request"
    CODE = "code"
    LLM = "llm"
    TRANSFORM = "transform"
    DATABASE = "database"

    # Branch nodes
    IF_ELSE = "if-else"
    SWITCH = "switch"
    QUESTION_CLASSIFIER = "question-classifier"

    # Container nodes
    LOOP = "loop"
    ITERATION = "iteration"
    PARALLEL = "parallel"

    # Response nodes
    END = "end"
    ANSWER = "answer"
```

### 3.2 Base Node Structure

```python
from dataclasses import dataclass
from typing import Generic, TypeVar, Generator, Any, ClassVar

NodeDataT = TypeVar('NodeDataT', bound='BaseNodeData')

@dataclass
class BaseNodeData:
    """Base data cho tất cả node types"""
    id: str
    title: str
    type: NodeType
    description: str = ""

    # Error handling
    error_strategy: str = "fail-branch"  # hoặc "default-value"
    retry_count: int = 0
    retry_delay: float = 1.0

@dataclass
class NodeRunResult:
    """Kết quả thực thi node"""
    outputs: Dict[str, Any] = field(default_factory=dict)
    edge_source_handle: Optional[str] = None  # Cho branch nodes
    error: Optional[str] = None
    metadata: Dict[str, Any] = field(default_factory=dict)

class Node(ABC, Generic[NodeDataT]):
    """Abstract base class cho tất cả nodes"""

    node_type: ClassVar[NodeType]
    execution_type: ClassVar[NodeExecutionType]

    def __init__(
        self,
        node_data: NodeDataT,
        runtime_state: 'RuntimeState'
    ):
        self.node_data = node_data
        self.runtime_state = runtime_state

    @abstractmethod
    def _run(self) -> NodeRunResult | Generator:
        """Logic thực thi chính của node"""
        pass

    def execute(self) -> NodeRunResult | Generator:
        """Wrapper với error handling và retry"""
        for attempt in range(self.node_data.retry_count + 1):
            try:
                return self._run()
            except Exception as e:
                if attempt == self.node_data.retry_count:
                    return self._handle_error(e)
                time.sleep(self.node_data.retry_delay)

    def _handle_error(self, error: Exception) -> NodeRunResult:
        if self.node_data.error_strategy == "fail-branch":
            return NodeRunResult(
                edge_source_handle="fail",
                error=str(error)
            )
        return NodeRunResult(outputs=self._get_default_outputs())
```

### 3.3 Các Node Types Cụ thể

#### A. If-Else Node (Branch Type)

```python
@dataclass
class Condition:
    variable_selector: List[str]  # [node_id, variable_name]
    operator: str                  # "equals", "contains", "greater_than", etc.
    value: Any

@dataclass
class Case:
    case_id: str
    conditions: List[Condition]
    logical_operator: str = "and"  # "and" hoặc "or"

@dataclass
class IfElseNodeData(BaseNodeData):
    cases: List[Case] = field(default_factory=list)

class IfElseNode(Node[IfElseNodeData]):
    node_type = NodeType.IF_ELSE
    execution_type = NodeExecutionType.BRANCH

    def _run(self) -> NodeRunResult:
        for case in self.node_data.cases:
            result = self._evaluate_case(case)
            if result:
                return NodeRunResult(
                    edge_source_handle=case.case_id,
                    outputs={"selected_case": case.case_id}
                )

        # Default/else case
        return NodeRunResult(
            edge_source_handle="else",
            outputs={"selected_case": "else"}
        )

    def _evaluate_case(self, case: Case) -> bool:
        results = [
            self._evaluate_condition(cond)
            for cond in case.conditions
        ]

        if case.logical_operator == "and":
            return all(results)
        return any(results)  # "or"
```

#### B. Loop Node (Container Type)

```python
@dataclass
class LoopNodeData(BaseNodeData):
    max_iterations: int = 100
    break_conditions: List[Condition] = field(default_factory=list)
    start_node_id: str = ""  # Entry point của subgraph
    loop_variables: List[str] = field(default_factory=list)

class LoopNode(Node[LoopNodeData]):
    node_type = NodeType.LOOP
    execution_type = NodeExecutionType.CONTAINER

    def _run(self) -> Generator:
        """Yield events cho mỗi iteration"""
        for iteration in range(self.node_data.max_iterations):
            # Check break conditions
            if self._should_break():
                yield LoopBreakEvent(iteration=iteration)
                break

            # Initialize loop variables
            self._init_iteration_variables(iteration)

            # Execute subgraph
            yield LoopIterationStartEvent(iteration=iteration)

            # Subgraph execution được handle bởi engine
            yield LoopIterationEndEvent(iteration=iteration)

        yield LoopCompletedEvent(
            total_iterations=iteration + 1
        )
```

#### C. Iteration Node (Parallel Container)

```python
@dataclass
class IterationNodeData(BaseNodeData):
    iterator_selector: List[str]  # Variable chứa list items
    start_node_id: str = ""
    parallel_mode: bool = False
    batch_size: int = 10

class IterationNode(Node[IterationNodeData]):
    node_type = NodeType.ITERATION
    execution_type = NodeExecutionType.CONTAINER

    def _run(self) -> Generator:
        items = self._get_iterator_items()
        results = []

        yield IterationStartEvent(total_items=len(items))

        if self.node_data.parallel_mode:
            # Process in batches
            for batch_start in range(0, len(items), self.node_data.batch_size):
                batch = items[batch_start:batch_start + self.node_data.batch_size]
                batch_results = yield ParallelBatchEvent(items=batch)
                results.extend(batch_results)
        else:
            # Sequential processing
            for index, item in enumerate(items):
                yield IterationNextEvent(index=index, item=item)
                # Result collected from subgraph

        yield IterationCompletedEvent(results=results)
```

---

## 4. Xử lý Edge và Branching

### 4.1 Edge Processor

```python
class EdgeProcessor:
    def __init__(self, graph: Graph, state_manager: 'StateManager'):
        self.graph = graph
        self.state_manager = state_manager

    def process_node_completion(
        self,
        node_id: str,
        result: NodeRunResult
    ) -> List[str]:
        """
        Xử lý sau khi node hoàn thành.
        Returns: List node_ids sẵn sàng thực thi tiếp.
        """
        node = self.graph.nodes[node_id]
        outgoing_edges = self.graph.get_outgoing_edges(node_id)

        if node.execution_type == NodeExecutionType.BRANCH:
            return self._process_branch_completion(
                outgoing_edges,
                result.edge_source_handle
            )
        else:
            return self._process_normal_completion(outgoing_edges)

    def _process_branch_completion(
        self,
        edges: List[Edge],
        selected_handle: str
    ) -> List[str]:
        """
        Với branch node, chỉ edge được chọn là TAKEN.
        Các edges khác là SKIPPED.
        """
        ready_nodes = []

        for edge in edges:
            if edge.source_handle == selected_handle:
                self.state_manager.mark_edge_taken(edge.id)
                target_node = edge.target_node_id
                if self._is_node_ready(target_node):
                    ready_nodes.append(target_node)
            else:
                self.state_manager.mark_edge_skipped(edge.id)
                # Trigger skip propagation
                self._propagate_skip(edge.id)

        return ready_nodes

    def _process_normal_completion(
        self,
        edges: List[Edge]
    ) -> List[str]:
        """Với node thường, tất cả outgoing edges đều TAKEN"""
        ready_nodes = []

        for edge in edges:
            self.state_manager.mark_edge_taken(edge.id)
            target_node = edge.target_node_id
            if self._is_node_ready(target_node):
                ready_nodes.append(target_node)

        return ready_nodes

    def _is_node_ready(self, node_id: str) -> bool:
        """
        Node sẵn sàng khi:
        - Không có incoming edges, HOẶC
        - Có ít nhất 1 incoming edge là TAKEN và không có edge nào UNKNOWN
        """
        incoming = self.graph.get_incoming_edges(node_id)

        if not incoming:
            return True

        has_unknown = any(e.state == NodeState.UNKNOWN for e in incoming)
        if has_unknown:
            return False

        return any(e.state == NodeState.TAKEN for e in incoming)
```

### 4.2 Skip Propagator

```python
class SkipPropagator:
    """
    Lan truyền SKIP qua các nhánh không được chọn.

    Quy tắc:
    1. Nếu node có edge UNKNOWN -> DỪNG (chưa đủ thông tin)
    2. Nếu node có edge TAKEN -> Node có thể execute (không skip)
    3. Nếu TẤT CẢ edges là SKIPPED -> Skip node và tiếp tục propagate
    """

    def __init__(self, graph: Graph, state_manager: 'StateManager'):
        self.graph = graph
        self.state_manager = state_manager

    def propagate_from_edge(self, edge_id: str) -> None:
        """Bắt đầu propagation từ một skipped edge"""
        edge = self.graph.edges[edge_id]
        self._propagate_to_node(edge.target_node_id)

    def _propagate_to_node(self, node_id: str) -> None:
        """Recursive propagation đến một node"""
        incoming = self.graph.get_incoming_edges(node_id)

        # Rule 1: Có edge UNKNOWN -> dừng
        if any(e.state == NodeState.UNKNOWN for e in incoming):
            return

        # Rule 2: Có edge TAKEN -> node sẽ execute
        if any(e.state == NodeState.TAKEN for e in incoming):
            return

        # Rule 3: Tất cả edges SKIPPED -> skip node
        self.state_manager.mark_node_skipped(node_id)

        # Propagate đến downstream nodes
        for outgoing_edge in self.graph.get_outgoing_edges(node_id):
            self.state_manager.mark_edge_skipped(outgoing_edge.id)
            self._propagate_to_node(outgoing_edge.target_node_id)

    def skip_branch_paths(self, unselected_edges: List[Edge]) -> None:
        """Skip toàn bộ nhánh không được chọn"""
        for edge in unselected_edges:
            self.propagate_from_edge(edge.id)
```

### 4.3 Visualization của Skip Propagation

```
Ví dụ: If-Else với 2 nhánh

        ┌─────┐
        │START│
        └──┬──┘
           │
      ┌────▼────┐
      │ IF-ELSE │
      └────┬────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
┌───────┐    ┌───────┐
│CASE A │    │CASE B │  ◄── Nếu chọn A, B bị SKIP
└───┬───┘    └───┬───┘
    │            │
    ▼            ▼
┌───────┐    ┌───────┐
│NODE A1│    │NODE B1│  ◄── B1 cũng bị SKIP (propagation)
└───┬───┘    └───┬───┘
    │            │
    └──────┬─────┘
           │
      ┌────▼────┐
      │  JOIN   │  ◄── Chờ đến khi biết A TAKEN, B SKIPPED
      └────┬────┘         rồi mới execute
           │
        ┌──▼──┐
        │ END │
        └─────┘
```

---

## 5. Execution Engine

### 5.1 Kiến trúc Engine

```python
from queue import Queue
from threading import Thread
from typing import Generator
from dataclasses import dataclass

@dataclass
class ExecutionEvent:
    """Base class cho tất cả execution events"""
    timestamp: float = field(default_factory=time.time)

@dataclass
class NodeStartEvent(ExecutionEvent):
    node_id: str
    node_type: NodeType

@dataclass
class NodeCompleteEvent(ExecutionEvent):
    node_id: str
    result: NodeRunResult

@dataclass
class NodeSkipEvent(ExecutionEvent):
    node_id: str
    reason: str

class GraphEngine:
    """
    Engine thực thi pipeline dựa trên queue.

    Luồng xử lý:
    1. Khởi tạo: đưa root node vào ready_queue
    2. Worker lấy node từ queue, thực thi
    3. Sau khi node hoàn thành, Edge Processor xác định nodes tiếp theo
    4. Nodes sẵn sàng được đưa vào ready_queue
    5. Lặp lại cho đến khi queue rỗng
    """

    def __init__(
        self,
        graph: Graph,
        variable_pool: 'VariablePool',
        node_factory: 'NodeFactory',
        max_workers: int = 4
    ):
        self.graph = graph
        self.variable_pool = variable_pool
        self.node_factory = node_factory

        # Queues
        self.ready_queue: Queue[str] = Queue()
        self.event_queue: Queue[ExecutionEvent] = Queue()

        # Processors
        self.state_manager = StateManager(graph)
        self.edge_processor = EdgeProcessor(graph, self.state_manager)
        self.skip_propagator = SkipPropagator(graph, self.state_manager)

        # Worker pool
        self.max_workers = max_workers
        self.workers: List[Thread] = []
        self._stop_flag = False

    def run(self) -> Generator[ExecutionEvent, None, None]:
        """Main execution loop - yields events"""
        # Initialize với root node
        self.ready_queue.put(self.graph.root_node_id)
        self.state_manager.mark_node_enqueued(self.graph.root_node_id)

        # Start workers
        self._start_workers()

        try:
            while not self._is_complete():
                # Yield events as they come
                try:
                    event = self.event_queue.get(timeout=0.1)
                    yield event

                    # Process completion events
                    if isinstance(event, NodeCompleteEvent):
                        self._handle_node_complete(event)

                except Empty:
                    continue
        finally:
            self._stop_workers()

    def _worker_loop(self):
        """Worker thread logic"""
        while not self._stop_flag:
            try:
                node_id = self.ready_queue.get(timeout=0.1)
            except Empty:
                continue

            self._execute_node(node_id)

    def _execute_node(self, node_id: str):
        """Thực thi một node"""
        node_data = self.graph.nodes[node_id]

        # Emit start event
        self.event_queue.put(NodeStartEvent(
            node_id=node_id,
            node_type=node_data.node_type
        ))

        # Create and run node
        runtime_state = RuntimeState(
            variable_pool=self.variable_pool,
            graph=self.graph
        )
        node = self.node_factory.create(node_data, runtime_state)
        result = node.execute()

        # Handle container nodes (generators)
        if isinstance(result, Generator):
            self._handle_container_node(node_id, result)
        else:
            # Store outputs
            self.variable_pool.add(node_id, result.outputs)

            # Emit complete event
            self.event_queue.put(NodeCompleteEvent(
                node_id=node_id,
                result=result
            ))

    def _handle_node_complete(self, event: NodeCompleteEvent):
        """Xử lý sau khi node hoàn thành"""
        ready_nodes = self.edge_processor.process_node_completion(
            event.node_id,
            event.result
        )

        for node_id in ready_nodes:
            self.ready_queue.put(node_id)
            self.state_manager.mark_node_enqueued(node_id)

    def _is_complete(self) -> bool:
        """Kiểm tra đã hoàn thành chưa"""
        return (
            self.ready_queue.empty() and
            self.state_manager.all_nodes_processed()
        )
```

### 5.2 State Manager

```python
class StateManager:
    """Quản lý trạng thái của nodes và edges"""

    def __init__(self, graph: Graph):
        self.graph = graph
        self.node_states: Dict[str, NodeState] = {
            nid: NodeState.UNKNOWN
            for nid in graph.nodes
        }
        self._lock = threading.Lock()

    def mark_node_enqueued(self, node_id: str):
        with self._lock:
            self.node_states[node_id] = NodeState.TAKEN

    def mark_node_skipped(self, node_id: str):
        with self._lock:
            self.node_states[node_id] = NodeState.SKIPPED

    def mark_edge_taken(self, edge_id: str):
        with self._lock:
            self.graph.edges[edge_id].state = NodeState.TAKEN

    def mark_edge_skipped(self, edge_id: str):
        with self._lock:
            self.graph.edges[edge_id].state = NodeState.SKIPPED

    def all_nodes_processed(self) -> bool:
        with self._lock:
            return all(
                state != NodeState.UNKNOWN
                for state in self.node_states.values()
            )

    def get_execution_summary(self) -> Dict:
        with self._lock:
            return {
                "taken": [nid for nid, s in self.node_states.items()
                         if s == NodeState.TAKEN],
                "skipped": [nid for nid, s in self.node_states.items()
                           if s == NodeState.SKIPPED],
                "unknown": [nid for nid, s in self.node_states.items()
                           if s == NodeState.UNKNOWN]
            }
```

---

## 6. Variable Pool & State Management

### 6.1 Variable Pool Design

```python
from typing import Any, Dict, List, Optional, Union

# Supported variable types
VariableValue = Union[str, int, float, bool, list, dict, None]

@dataclass
class Variable:
    name: str
    value: VariableValue
    type: str  # "string", "number", "boolean", "array", "object"

    @classmethod
    def from_value(cls, name: str, value: Any) -> 'Variable':
        type_map = {
            str: "string",
            int: "number",
            float: "number",
            bool: "boolean",
            list: "array",
            dict: "object",
            type(None): "null"
        }
        return cls(
            name=name,
            value=value,
            type=type_map.get(type(value), "unknown")
        )

class VariablePool:
    """
    Quản lý variables theo namespace (node_id).

    Cấu trúc: {node_id: {variable_name: Variable}}

    Selector format: [node_id, variable_name]
    Ví dụ: ["node_1", "response"] -> lấy biến "response" từ node_1
    """

    def __init__(self):
        self._variables: Dict[str, Dict[str, Variable]] = {}
        self._system_variables: Dict[str, Variable] = {}
        self._lock = threading.RLock()

    def add(
        self,
        node_id: str,
        outputs: Dict[str, Any]
    ) -> None:
        """Thêm outputs của một node vào pool"""
        with self._lock:
            if node_id not in self._variables:
                self._variables[node_id] = {}

            for name, value in outputs.items():
                self._variables[node_id][name] = Variable.from_value(name, value)

    def get(
        self,
        selector: List[str]
    ) -> Optional[VariableValue]:
        """
        Lấy giá trị biến theo selector.

        selector: [node_id, variable_name] hoặc [node_id, variable_name, nested_key, ...]
        """
        with self._lock:
            if len(selector) < 2:
                return None

            node_id, var_name = selector[0], selector[1]

            # System variables
            if node_id == "sys":
                var = self._system_variables.get(var_name)
                return var.value if var else None

            # Node variables
            node_vars = self._variables.get(node_id)
            if not node_vars:
                return None

            var = node_vars.get(var_name)
            if not var:
                return None

            value = var.value

            # Handle nested access
            for key in selector[2:]:
                if isinstance(value, dict):
                    value = value.get(key)
                elif isinstance(value, list) and isinstance(key, int):
                    value = value[key] if key < len(value) else None
                else:
                    return None

            return value

    def set_system_variable(self, name: str, value: Any) -> None:
        """Set system-level variables"""
        with self._lock:
            self._system_variables[name] = Variable.from_value(name, value)

    def get_node_outputs(self, node_id: str) -> Dict[str, Any]:
        """Lấy tất cả outputs của một node"""
        with self._lock:
            node_vars = self._variables.get(node_id, {})
            return {name: var.value for name, var in node_vars.items()}

    def create_scoped_pool(self, scope_id: str) -> 'ScopedVariablePool':
        """Tạo scoped pool cho loops/iterations"""
        return ScopedVariablePool(self, scope_id)


class ScopedVariablePool:
    """
    Variable pool với scope riêng cho loops/iterations.
    Có thể ghi đè biến của parent nhưng không ảnh hưởng parent.
    """

    def __init__(self, parent: VariablePool, scope_id: str):
        self._parent = parent
        self._scope_id = scope_id
        self._local_variables: Dict[str, Dict[str, Variable]] = {}

    def add(self, node_id: str, outputs: Dict[str, Any]) -> None:
        """Thêm vào local scope"""
        scoped_node_id = f"{self._scope_id}.{node_id}"
        if scoped_node_id not in self._local_variables:
            self._local_variables[scoped_node_id] = {}

        for name, value in outputs.items():
            self._local_variables[scoped_node_id][name] = Variable.from_value(name, value)

    def get(self, selector: List[str]) -> Optional[VariableValue]:
        """Tìm trong local trước, rồi đến parent"""
        # Try local first
        if len(selector) >= 2:
            scoped_node_id = f"{self._scope_id}.{selector[0]}"
            local_vars = self._local_variables.get(scoped_node_id)
            if local_vars and selector[1] in local_vars:
                return local_vars[selector[1]].value

        # Fallback to parent
        return self._parent.get(selector)
```

---

## 7. Condition Processing System

### 7.1 Condition Processor

```python
from typing import Literal, Sequence

COMPARISON_OPERATORS = Literal[
    # String/General
    "equals", "not_equals",
    "contains", "not_contains",
    "starts_with", "ends_with",
    "is_empty", "is_not_empty",
    "is_null", "is_not_null",

    # Numeric
    "greater_than", "greater_than_or_equal",
    "less_than", "less_than_or_equal",

    # Array
    "in", "not_in",
    "array_contains", "array_not_contains",

    # Regex
    "matches_regex",
]

@dataclass
class Condition:
    variable_selector: List[str]
    operator: COMPARISON_OPERATORS
    value: Any = None

@dataclass
class ConditionResult:
    condition: Condition
    actual_value: Any
    expected_value: Any
    result: bool

@dataclass
class ConditionGroupResult:
    conditions: List[ConditionResult]
    logical_operator: str
    final_result: bool

class ConditionProcessor:
    """
    Xử lý đánh giá conditions.
    Hỗ trợ short-circuit evaluation.
    """

    def __init__(self, variable_pool: VariablePool):
        self.variable_pool = variable_pool

    def evaluate_conditions(
        self,
        conditions: Sequence[Condition],
        logical_operator: Literal["and", "or"] = "and"
    ) -> ConditionGroupResult:
        """
        Đánh giá một nhóm conditions với logical operator.
        """
        results = []

        for condition in conditions:
            result = self._evaluate_single(condition)
            results.append(result)

            # Short-circuit evaluation
            if logical_operator == "and" and not result.result:
                break  # AND: một False -> kết thúc
            if logical_operator == "or" and result.result:
                break  # OR: một True -> kết thúc

        if logical_operator == "and":
            final = all(r.result for r in results)
        else:
            final = any(r.result for r in results)

        return ConditionGroupResult(
            conditions=results,
            logical_operator=logical_operator,
            final_result=final
        )

    def _evaluate_single(self, condition: Condition) -> ConditionResult:
        """Đánh giá một condition đơn"""
        actual = self.variable_pool.get(condition.variable_selector)
        expected = condition.value

        result = self._compare(actual, condition.operator, expected)

        return ConditionResult(
            condition=condition,
            actual_value=actual,
            expected_value=expected,
            result=result
        )

    def _compare(
        self,
        actual: Any,
        operator: str,
        expected: Any
    ) -> bool:
        """Thực hiện so sánh"""

        # Null checks
        if operator == "is_null":
            return actual is None
        if operator == "is_not_null":
            return actual is not None

        # Empty checks
        if operator == "is_empty":
            return actual is None or actual == "" or actual == [] or actual == {}
        if operator == "is_not_empty":
            return not (actual is None or actual == "" or actual == [] or actual == {})

        # Equality
        if operator == "equals":
            return actual == expected
        if operator == "not_equals":
            return actual != expected

        # String operations
        if operator == "contains":
            return expected in str(actual) if actual else False
        if operator == "not_contains":
            return expected not in str(actual) if actual else True
        if operator == "starts_with":
            return str(actual).startswith(str(expected)) if actual else False
        if operator == "ends_with":
            return str(actual).endswith(str(expected)) if actual else False

        # Numeric comparisons
        if operator == "greater_than":
            return float(actual) > float(expected)
        if operator == "greater_than_or_equal":
            return float(actual) >= float(expected)
        if operator == "less_than":
            return float(actual) < float(expected)
        if operator == "less_than_or_equal":
            return float(actual) <= float(expected)

        # Array operations
        if operator == "in":
            return actual in expected if isinstance(expected, (list, tuple)) else False
        if operator == "not_in":
            return actual not in expected if isinstance(expected, (list, tuple)) else True
        if operator == "array_contains":
            return expected in actual if isinstance(actual, (list, tuple)) else False
        if operator == "array_not_contains":
            return expected not in actual if isinstance(actual, (list, tuple)) else True

        # Regex
        if operator == "matches_regex":
            import re
            return bool(re.match(str(expected), str(actual))) if actual else False

        return False
```

---

## 8. File Config Schema

### 8.1 JSON Schema cho Pipeline Config

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Pipeline Configuration",
  "type": "object",
  "required": ["id", "name", "nodes", "edges"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique identifier cho pipeline"
    },
    "name": {
      "type": "string",
      "description": "Tên hiển thị của pipeline"
    },
    "description": {
      "type": "string"
    },
    "version": {
      "type": "string",
      "default": "1.0.0"
    },
    "nodes": {
      "type": "array",
      "items": { "$ref": "#/definitions/Node" }
    },
    "edges": {
      "type": "array",
      "items": { "$ref": "#/definitions/Edge" }
    },
    "variables": {
      "type": "object",
      "description": "Global variables/inputs",
      "additionalProperties": {
        "$ref": "#/definitions/VariableDefinition"
      }
    }
  },
  "definitions": {
    "Node": {
      "type": "object",
      "required": ["id", "type"],
      "properties": {
        "id": { "type": "string" },
        "type": {
          "type": "string",
          "enum": ["start", "end", "if-else", "loop", "iteration",
                   "http-request", "code", "transform", "llm"]
        },
        "title": { "type": "string" },
        "description": { "type": "string" },
        "position": {
          "type": "object",
          "properties": {
            "x": { "type": "number" },
            "y": { "type": "number" }
          }
        },
        "data": {
          "type": "object",
          "description": "Node-specific configuration"
        },
        "error_strategy": {
          "type": "string",
          "enum": ["fail-branch", "default-value"],
          "default": "fail-branch"
        }
      }
    },
    "Edge": {
      "type": "object",
      "required": ["id", "source", "target"],
      "properties": {
        "id": { "type": "string" },
        "source": { "type": "string" },
        "target": { "type": "string" },
        "sourceHandle": {
          "type": "string",
          "default": "default",
          "description": "Handle cho branching (e.g., 'true', 'false', 'case_1')"
        }
      }
    },
    "VariableDefinition": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["string", "number", "boolean", "array", "object"]
        },
        "default": {},
        "required": { "type": "boolean" },
        "description": { "type": "string" }
      }
    }
  }
}
```

### 8.2 Ví dụ Config File

```yaml
# pipeline-example.yaml
id: "order-processing-pipeline"
name: "Order Processing Pipeline"
description: "Pipeline xử lý đơn hàng với validation và branching"
version: "1.0.0"

# Input variables
variables:
  order_id:
    type: string
    required: true
    description: "ID đơn hàng cần xử lý"
  customer_type:
    type: string
    default: "regular"

# Nodes definition
nodes:
  - id: "start"
    type: "start"
    title: "Start"
    position: { x: 100, y: 100 }
    data:
      inputs:
        - name: "order_id"
          type: "string"
          required: true
        - name: "customer_type"
          type: "string"
          default: "regular"

  - id: "fetch_order"
    type: "http-request"
    title: "Fetch Order Details"
    position: { x: 100, y: 200 }
    data:
      method: "GET"
      url: "https://api.example.com/orders/{{order_id}}"
      headers:
        Authorization: "Bearer {{sys.api_token}}"
    error_strategy: "fail-branch"

  - id: "check_order_valid"
    type: "if-else"
    title: "Validate Order"
    position: { x: 100, y: 300 }
    data:
      cases:
        - case_id: "valid"
          logical_operator: "and"
          conditions:
            - variable_selector: ["fetch_order", "status_code"]
              operator: "equals"
              value: 200
            - variable_selector: ["fetch_order", "body", "status"]
              operator: "not_equals"
              value: "cancelled"
        - case_id: "invalid"
          logical_operator: "or"
          conditions:
            - variable_selector: ["fetch_order", "status_code"]
              operator: "not_equals"
              value: 200

  - id: "check_customer_type"
    type: "if-else"
    title: "Check Customer Type"
    position: { x: 200, y: 400 }
    data:
      cases:
        - case_id: "vip"
          conditions:
            - variable_selector: ["start", "customer_type"]
              operator: "equals"
              value: "vip"
        - case_id: "regular"
          conditions:
            - variable_selector: ["start", "customer_type"]
              operator: "equals"
              value: "regular"

  - id: "process_vip"
    type: "code"
    title: "Process VIP Order"
    position: { x: 100, y: 500 }
    data:
      language: "python"
      code: |
        discount = 0.2
        order_total = inputs["fetch_order"]["body"]["total"]
        final_total = order_total * (1 - discount)
        return {"final_total": final_total, "discount_applied": discount}

  - id: "process_regular"
    type: "code"
    title: "Process Regular Order"
    position: { x: 300, y: 500 }
    data:
      language: "python"
      code: |
        order_total = inputs["fetch_order"]["body"]["total"]
        return {"final_total": order_total, "discount_applied": 0}

  - id: "process_items"
    type: "iteration"
    title: "Process Order Items"
    position: { x: 200, y: 600 }
    data:
      iterator_selector: ["fetch_order", "body", "items"]
      parallel_mode: true
      batch_size: 5
      start_node_id: "process_single_item"

  - id: "process_single_item"
    type: "http-request"
    title: "Update Item Stock"
    position: { x: 200, y: 650 }
    data:
      method: "POST"
      url: "https://api.example.com/inventory/update"
      body:
        item_id: "{{item.id}}"
        quantity: "{{item.quantity}}"

  - id: "send_notification"
    type: "http-request"
    title: "Send Notification"
    position: { x: 200, y: 700 }
    data:
      method: "POST"
      url: "https://api.example.com/notifications"
      body:
        order_id: "{{start.order_id}}"
        status: "processed"

  - id: "handle_invalid"
    type: "code"
    title: "Handle Invalid Order"
    position: { x: 400, y: 400 }
    data:
      language: "python"
      code: |
        return {"error": "Invalid order", "order_id": inputs["start"]["order_id"]}

  - id: "end_success"
    type: "end"
    title: "Success"
    position: { x: 200, y: 800 }

  - id: "end_failure"
    type: "end"
    title: "Failure"
    position: { x: 400, y: 500 }

# Edges definition
edges:
  - id: "e1"
    source: "start"
    target: "fetch_order"

  - id: "e2"
    source: "fetch_order"
    target: "check_order_valid"

  - id: "e3"
    source: "check_order_valid"
    target: "check_customer_type"
    sourceHandle: "valid"

  - id: "e4"
    source: "check_order_valid"
    target: "handle_invalid"
    sourceHandle: "invalid"

  - id: "e5"
    source: "check_customer_type"
    target: "process_vip"
    sourceHandle: "vip"

  - id: "e6"
    source: "check_customer_type"
    target: "process_regular"
    sourceHandle: "regular"

  - id: "e7"
    source: "process_vip"
    target: "process_items"

  - id: "e8"
    source: "process_regular"
    target: "process_items"

  - id: "e9"
    source: "process_items"
    target: "send_notification"

  - id: "e10"
    source: "send_notification"
    target: "end_success"

  - id: "e11"
    source: "handle_invalid"
    target: "end_failure"

  # Error branch từ fetch_order
  - id: "e12"
    source: "fetch_order"
    target: "handle_invalid"
    sourceHandle: "fail"
```

### 8.3 Graph Visualization của Config trên

```
                    ┌─────────┐
                    │  START  │
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │  FETCH  │──────────────┐
                    │  ORDER  │              │ (fail)
                    └────┬────┘              │
                         │                   │
                    ┌────▼────┐              │
                    │ VALIDATE│              │
                    │  ORDER  │              │
                    └────┬────┘              │
              ┌──────────┴──────────┐        │
              │ (valid)      (invalid)       │
              ▼                     ▼        ▼
        ┌───────────┐         ┌────────────────┐
        │  CHECK    │         │    HANDLE      │
        │ CUSTOMER  │         │    INVALID     │
        └─────┬─────┘         └───────┬────────┘
       ┌──────┴──────┐                │
       │ (vip)  (regular)             │
       ▼             ▼                ▼
  ┌─────────┐  ┌─────────┐     ┌───────────┐
  │PROCESS  │  │PROCESS  │     │END FAILURE│
  │  VIP    │  │ REGULAR │     └───────────┘
  └────┬────┘  └────┬────┘
       │            │
       └──────┬─────┘
              │
        ┌─────▼─────┐
        │ ITERATION │ ◄── Process mỗi item
        │  (items)  │
        └─────┬─────┘
              │
        ┌─────▼─────┐
        │   SEND    │
        │   NOTIF   │
        └─────┬─────┘
              │
        ┌─────▼─────┐
        │END SUCCESS│
        └───────────┘
```

---

## 9. Nguyên tắc Thiết kế Cốt lõi

### 9.1 Separation of Concerns

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   REST API   │  │   GraphQL    │  │   Visual Editor  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                    APPLICATION LAYER                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Pipeline   │  │   Execution  │  │   Event          │   │
│  │   Service    │  │   Service    │  │   Publisher      │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                      DOMAIN LAYER                            │
│  ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌─────────────┐   │
│  │  Graph   │ │  Node    │ │  Variable  │ │  Condition  │   │
│  │  Engine  │ │  Factory │ │  Pool      │ │  Processor  │   │
│  └──────────┘ └──────────┘ └────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                  INFRASTRUCTURE LAYER                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Database   │  │   Message    │  │   External       │   │
│  │   Repository │  │   Queue      │  │   Services       │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Design Patterns Áp dụng

| Pattern | Áp dụng trong | Mục đích |
|---------|---------------|----------|
| **Factory** | NodeFactory | Tạo node instances từ config |
| **Strategy** | ConditionProcessor | Các phép so sánh khác nhau |
| **Observer** | Event System | Notify execution events |
| **Command** | CommandChannel | Pause/Resume/Abort execution |
| **Composite** | Container Nodes | Nodes chứa subgraph |
| **State** | StateManager | Track node/edge states |
| **Template Method** | Base Node | Common execution flow |

### 9.3 Nguyên tắc SOLID

```python
# 1. Single Responsibility
class EdgeProcessor:      # Chỉ xử lý edge routing
    pass

class SkipPropagator:     # Chỉ xử lý skip propagation
    pass

class StateManager:       # Chỉ quản lý state
    pass


# 2. Open/Closed - Mở rộng node types không cần sửa engine
class NodeFactory:
    def __init__(self):
        self._registry: Dict[NodeType, Type[Node]] = {}

    def register(self, node_type: NodeType, node_class: Type[Node]):
        """Đăng ký node type mới"""
        self._registry[node_type] = node_class

    def create(self, node_data: BaseNodeData, runtime: RuntimeState) -> Node:
        node_class = self._registry.get(node_data.type)
        if not node_class:
            raise ValueError(f"Unknown node type: {node_data.type}")
        return node_class(node_data, runtime)


# 3. Liskov Substitution - Tất cả nodes đều thay thế được
def execute_node(node: Node) -> NodeRunResult:
    return node.execute()  # Works với bất kỳ node type nào


# 4. Interface Segregation
class Executable(Protocol):
    def execute(self) -> NodeRunResult: ...

class Configurable(Protocol):
    def validate_config(self) -> List[str]: ...

class Describable(Protocol):
    def get_description(self) -> str: ...


# 5. Dependency Inversion
class GraphEngine:
    def __init__(
        self,
        graph: Graph,                        # Abstract dependency
        variable_pool: VariablePool,         # Abstract dependency
        node_factory: NodeFactory,           # Abstract dependency
    ):
        self.graph = graph
        self.variable_pool = variable_pool
        self.node_factory = node_factory
```

### 9.4 Error Handling Strategy

```python
class ErrorStrategy(StrEnum):
    FAIL_BRANCH = "fail-branch"      # Route đến fail branch
    DEFAULT_VALUE = "default-value"  # Dùng default output, tiếp tục
    RETRY = "retry"                  # Retry với backoff
    SKIP = "skip"                    # Skip node, tiếp tục

@dataclass
class RetryConfig:
    max_attempts: int = 3
    initial_delay: float = 1.0
    max_delay: float = 60.0
    backoff_multiplier: float = 2.0

class NodeExecutor:
    def execute_with_error_handling(
        self,
        node: Node,
        retry_config: Optional[RetryConfig] = None
    ) -> NodeRunResult:

        strategy = node.node_data.error_strategy

        for attempt in range(retry_config.max_attempts if retry_config else 1):
            try:
                return node._run()

            except Exception as e:
                if strategy == ErrorStrategy.SKIP:
                    return NodeRunResult(
                        outputs=node._get_default_outputs(),
                        metadata={"skipped": True, "error": str(e)}
                    )

                if strategy == ErrorStrategy.FAIL_BRANCH:
                    return NodeRunResult(
                        edge_source_handle="fail",
                        error=str(e)
                    )

                if strategy == ErrorStrategy.DEFAULT_VALUE:
                    return NodeRunResult(
                        outputs=node._get_default_outputs(),
                        error=str(e)
                    )

                # RETRY
                if attempt < retry_config.max_attempts - 1:
                    delay = min(
                        retry_config.initial_delay * (retry_config.backoff_multiplier ** attempt),
                        retry_config.max_delay
                    )
                    time.sleep(delay)
                    continue

                raise  # Final attempt failed
```

---

## 10. Hướng dẫn Triển khai

### 10.1 Phase 1: Core Foundation (MVP)

**Mục tiêu:** Pipeline tuyến tính cơ bản

```python
# Checklist Phase 1:
# [ ] Graph data structure (nodes, edges)
# [ ] Base Node class với execute()
# [ ] Simple node types: Start, End, Code, HTTP
# [ ] Variable Pool basic
# [ ] Linear execution engine
# [ ] Config parser (JSON/YAML)
```

**Deliverables:**
- Chạy được pipeline: Start → Code → HTTP → End
- Load từ config file
- Output execution log

### 10.2 Phase 2: Branching & Conditions

**Mục tiêu:** If-Else và conditional routing

```python
# Checklist Phase 2:
# [ ] If-Else node với multiple cases
# [ ] Condition Processor với operators
# [ ] Edge source_handle cho branching
# [ ] Skip Propagator
# [ ] State tracking (TAKEN/SKIPPED)
```

**Deliverables:**
- Pipeline với branching logic
- Skip unused branches correctly
- Join nodes hoạt động đúng

### 10.3 Phase 3: Loops & Containers

**Mục tiêu:** Loop, Iteration nodes

```python
# Checklist Phase 3:
# [ ] Loop node với break conditions
# [ ] Iteration node cho arrays
# [ ] Scoped Variable Pool
# [ ] Subgraph execution
# [ ] Loop events (start, next, end)
```

**Deliverables:**
- For-each iteration over arrays
- While-like loops với conditions
- Nested loops

### 10.4 Phase 4: Advanced Features

**Mục tiêu:** Production-ready

```python
# Checklist Phase 4:
# [ ] Parallel execution (worker pool)
# [ ] Command channel (pause/resume/abort)
# [ ] Event streaming
# [ ] Error handling với strategies
# [ ] Retry logic
# [ ] Execution timeouts
# [ ] State persistence
```

### 10.5 Project Structure Khuyến nghị

```
pipeline-engine/
├── src/
│   ├── core/
│   │   ├── __init__.py
│   │   ├── graph.py           # Graph, Node, Edge
│   │   ├── engine.py          # GraphEngine
│   │   ├── variable_pool.py   # VariablePool
│   │   └── state_manager.py   # StateManager
│   │
│   ├── nodes/
│   │   ├── __init__.py
│   │   ├── base.py            # Base Node class
│   │   ├── factory.py         # NodeFactory
│   │   ├── start.py
│   │   ├── end.py
│   │   ├── if_else.py
│   │   ├── loop.py
│   │   ├── iteration.py
│   │   ├── code.py
│   │   └── http_request.py
│   │
│   ├── conditions/
│   │   ├── __init__.py
│   │   ├── processor.py       # ConditionProcessor
│   │   └── operators.py       # Comparison operators
│   │
│   ├── traversal/
│   │   ├── __init__.py
│   │   ├── edge_processor.py
│   │   └── skip_propagator.py
│   │
│   ├── events/
│   │   ├── __init__.py
│   │   ├── types.py           # Event classes
│   │   └── publisher.py       # EventPublisher
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   ├── parser.py          # Config parser
│   │   ├── validator.py       # Schema validation
│   │   └── schema.json        # JSON Schema
│   │
│   └── api/
│       ├── __init__.py
│       └── service.py         # PipelineService
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│
├── examples/
│   ├── simple_pipeline.yaml
│   ├── branching_pipeline.yaml
│   └── loop_pipeline.yaml
│
└── docs/
    └── api.md
```

### 10.6 Quick Start Code

```python
# main.py - Quick start example

from src.config.parser import ConfigParser
from src.core.graph import Graph
from src.core.engine import GraphEngine
from src.core.variable_pool import VariablePool
from src.nodes.factory import NodeFactory

def run_pipeline(config_path: str, inputs: dict):
    # 1. Parse config
    parser = ConfigParser()
    config = parser.parse_file(config_path)

    # 2. Build graph
    factory = NodeFactory()
    factory.register_default_nodes()

    graph = Graph.from_config(config, factory)

    # 3. Initialize runtime
    variable_pool = VariablePool()
    variable_pool.add("start", inputs)

    # 4. Execute
    engine = GraphEngine(
        graph=graph,
        variable_pool=variable_pool,
        node_factory=factory
    )

    # 5. Stream events
    for event in engine.run():
        print(f"[{event.timestamp}] {event.__class__.__name__}: {event}")

    # 6. Get results
    return variable_pool.get_node_outputs("end")


if __name__ == "__main__":
    result = run_pipeline(
        "examples/order_processing.yaml",
        {"order_id": "ORD-12345", "customer_type": "vip"}
    )
    print(f"Pipeline result: {result}")
```

---

## Tổng kết

Tài liệu này mô tả kiến trúc và quy trình xây dựng hệ thống pipeline linh hoạt dựa trên phân tích Dify Workflow Engine. Các điểm chính:

1. **Graph-based Design**: Sử dụng directed graph cho phép branching, looping, parallel paths
2. **Typed Nodes**: Phân loại rõ ràng (Root, Executable, Branch, Container, Response)
3. **Smart Routing**: Edge Processor + Skip Propagator đảm bảo routing chính xác
4. **Scoped Variables**: Variable Pool với namespacing theo node
5. **Event-driven**: Execution events cho monitoring và debugging
6. **Extensible**: Factory pattern cho phép thêm node types dễ dàng

**Reference Implementation:** Xem chi tiết tại `/home/user/dify_analyze/api/core/workflow/`
