# Hướng dẫn triển khai Node trong Dify

## 1. Tổng quan Node System

Dify sử dụng một hệ thống node registration tự động dựa trên Python metaclass và TypeScript maps. Khi bạn tạo một node class mới kế thừa từ `Node[T]`, nó sẽ tự động được đăng ký vào registry.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         NODE REGISTRATION FLOW                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐   │
│  │  Define Node    │────►│ __init_subclass_│────►│  Auto-Register  │   │
│  │  class MyNode   │     │    (metaclass)  │     │   in Registry   │   │
│  │  (Node[T])      │     │                 │     │                 │   │
│  └─────────────────┘     └─────────────────┘     └─────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Node._registry                              │   │
│  │  {                                                               │   │
│  │    NodeType.LLM: {"1": LLMNode, "latest": LLMNode},             │   │
│  │    NodeType.CODE: {"1": CodeNode, "latest": CodeNode},          │   │
│  │    NodeType.IF_ELSE: {"1": IfElseNode, "latest": IfElseNode},   │   │
│  │    ...                                                           │   │
│  │  }                                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2. Backend Node Architecture

### 2.1 Node Base Class với Auto-Registration

**Reference**: `/api/core/workflow/nodes/base/node.py` (lines 61-168)

```python
from typing import Generic, TypeVar, ClassVar
from abc import abstractmethod

NodeDataT = TypeVar("NodeDataT", bound=BaseNodeData)

class Node(Generic[NodeDataT]):
    """Base class for all workflow nodes with auto-registration."""

    # Class-level attributes
    node_type: ClassVar["NodeType"]                    # Required: node type enum
    execution_type: NodeExecutionType = NodeExecutionType.EXECUTABLE
    _node_data_type: ClassVar[type[BaseNodeData]] = BaseNodeData

    # Global registry for all node types
    _registry: ClassVar[dict["NodeType", dict[str, type["Node"]]]] = {}

    def __init_subclass__(cls, **kwargs):
        """
        Auto-registration hook - called when a class inherits from Node.
        This is the magic that makes node registration automatic!
        """
        super().__init_subclass__(**kwargs)

        # 1. Extract generic type parameter T from Node[T]
        node_data_type = cls._extract_node_data_type_from_generic()
        cls._node_data_type = node_data_type

        # 2. Validate that it's a BaseNodeData subclass
        if not issubclass(node_data_type, BaseNodeData):
            raise TypeError(f"Node data type must be BaseNodeData subclass")

        # 3. Register in global registry if has node_type
        if hasattr(cls, "node_type") and cls.node_type:
            node_type = cls.node_type
            version = cls.version()

            # Create bucket for this node type if not exists
            bucket = Node._registry.setdefault(node_type, {})

            # Store by version
            bucket[version] = cls

            # Update "latest" pointer to highest version
            versions = [v for v in bucket.keys() if v != "latest"]
            if versions:
                latest_version = max(versions, key=lambda v: int(v) if v.isdigit() else 0)
                bucket["latest"] = bucket[latest_version]

    @classmethod
    def version(cls) -> str:
        """Override to specify node version. Default is "1"."""
        return "1"

    @classmethod
    def get_default_config(cls, filters: dict | None = None) -> dict:
        """Return default configuration for this node type."""
        return {
            "type": cls.node_type.value,
            "data": cls._node_data_type.model_json_schema(),
        }

    @abstractmethod
    def _run(self) -> NodeRunResult | Generator[NodeEventBase, None, None]:
        """
        Main execution logic - MUST be implemented by subclasses.

        Returns either:
        - NodeRunResult: For simple nodes
        - Generator[NodeEventBase]: For streaming nodes (LLM, etc.)
        """
        ...
```

### 2.2 Node Type Enum

**Reference**: `/api/core/workflow/enums.py` (lines 36-66)

```python
from enum import StrEnum

class NodeType(StrEnum):
    """All available workflow node types."""

    # Entry/Exit nodes
    START = "start"
    END = "end"
    ANSWER = "answer"

    # AI nodes
    LLM = "llm"
    AGENT = "agent"

    # Logic nodes
    IF_ELSE = "if-else"
    QUESTION_CLASSIFIER = "question-classifier"

    # Operation nodes
    CODE = "code"
    HTTP_REQUEST = "http-request"
    TOOL = "tool"
    KNOWLEDGE_RETRIEVAL = "knowledge-retrieval"
    KNOWLEDGE_INDEX = "knowledge-index"

    # Control flow nodes
    ITERATION = "iteration"
    LOOP = "loop"

    # Data nodes
    VARIABLE_AGGREGATOR = "variable-aggregator"
    VARIABLE_ASSIGNER = "assigner"
    TEMPLATE_TRANSFORM = "template-transform"
    PARAMETER_EXTRACTOR = "parameter-extractor"
    DOCUMENT_EXTRACTOR = "document-extractor"
    LIST_OPERATOR = "list-operator"

    # Trigger nodes
    TRIGGER_WEBHOOK = "trigger-webhook"
    TRIGGER_SCHEDULE = "trigger-schedule"
    TRIGGER_PLUGIN = "trigger-plugin"

    # Input nodes
    HUMAN_INPUT = "human-input"
    DATASOURCE = "datasource"


class NodeExecutionType(StrEnum):
    """How node execution is handled."""
    EXECUTABLE = "executable"   # Normal execution
    RESPONSE = "response"       # Answer, End - terminal nodes
    BRANCH = "branch"           # If-else, Question classifier
    CONTAINER = "container"     # Iteration, Loop
    ROOT = "root"               # Start, Triggers
```

### 2.3 Node Data Model

**Reference**: `/api/core/workflow/nodes/base/entities.py` (lines 168-200)

```python
from abc import ABC
from pydantic import BaseModel

class BaseNodeData(ABC, BaseModel):
    """Base class for all node configuration data."""

    title: str                                    # Display title
    desc: str | None = None                       # Description
    version: str = "1"                            # Node version
    error_strategy: ErrorStrategy | None = None   # Error handling
    default_value: list[DefaultValue] | None = None
    retry_config: RetryConfig = RetryConfig()     # Retry settings

    @property
    def default_value_dict(self) -> dict[str, Any]:
        """Convert default values to dict."""
        if self.default_value:
            return {item.key: item.value for item in self.default_value}
        return {}


class ErrorStrategy(StrEnum):
    """How to handle node execution errors."""
    FAIL_BRANCH = "fail-branch"       # Stop affected branch
    DEFAULT_VALUE = "default-value"   # Use default, continue


class RetryConfig(BaseModel):
    """Retry configuration for failed nodes."""
    max_retries: int = 0
    retry_interval: int = 0           # milliseconds
```

### 2.4 Node Factory

**Reference**: `/api/core/workflow/nodes/node_factory.py`

```python
from typing import final
from core.workflow.nodes.base.node import Node
from core.workflow.nodes.node_mapping import NODE_TYPE_CLASSES_MAPPING, LATEST_VERSION

@final
class DifyNodeFactory(NodeFactory):
    """Factory for creating node instances from config."""

    def __init__(
        self,
        graph_init_params: GraphInitParams,
        graph_runtime_state: GraphRuntimeState,
    ):
        self.graph_init_params = graph_init_params
        self.graph_runtime_state = graph_runtime_state

    def create_node(self, node_config: dict[str, object]) -> Node:
        """
        Create a node instance from configuration.

        Args:
            node_config: Node configuration from workflow graph

        Returns:
            Instantiated node ready for execution
        """
        node_data = node_config.get("data", {})
        node_id = node_config.get("id")

        # 1. Get node type
        node_type_str = node_data.get("type")  # e.g., "llm"
        node_type = NodeType(node_type_str)

        # 2. Look up in registry
        node_mapping = NODE_TYPE_CLASSES_MAPPING.get(node_type)
        if not node_mapping:
            raise ValueError(f"Unknown node type: {node_type}")

        # 3. Get specific version or latest
        node_version = str(node_data.get("version", "1"))
        node_class = node_mapping.get(node_version) or node_mapping.get(LATEST_VERSION)

        # 4. Instantiate
        return node_class(
            id=node_id,
            config=node_config,
            graph_init_params=self.graph_init_params,
            graph_runtime_state=self.graph_runtime_state,
        )
```

## 3. Example: Analyzing the Code Node

### 3.1 Node Data Model

**Reference**: `/api/core/workflow/nodes/code/entities.py`

```python
from pydantic import BaseModel
from core.workflow.nodes.base.entities import BaseNodeData

class CodeLanguage(StrEnum):
    PYTHON3 = "python3"
    JAVASCRIPT = "javascript"


class CodeNodeData(BaseNodeData):
    """Configuration for Code node."""

    # Code content
    code: str                           # User's code
    code_language: CodeLanguage         # python3 or javascript

    # Variables
    variables: list[Variable] = []      # Input variables
    outputs: dict[str, Any] = {}        # Output schema

    # Execution settings
    timeout: int = 60                   # Timeout in seconds

    class Variable(BaseModel):
        variable: str                   # Variable name
        value_selector: list[str]       # Path to value in variable pool
```

### 3.2 Node Implementation

**Reference**: `/api/core/workflow/nodes/code/code_node.py`

```python
from core.workflow.nodes.base.node import Node
from core.workflow.enums import NodeType
from core.workflow.nodes.code.entities import CodeNodeData

class CodeNode(Node[CodeNodeData]):
    """Execute custom Python or JavaScript code."""

    node_type = NodeType.CODE

    @classmethod
    def version(cls) -> str:
        return "1"

    @classmethod
    def get_default_config(cls, filters: dict | None = None) -> dict:
        return {
            "type": cls.node_type.value,
            "config": {
                "code_language": "python3",
                "code": "def main(inputs):\n    return {}\n",
                "variables": [],
                "outputs": {},
            }
        }

    def _run(self) -> NodeRunResult:
        """Execute the code and return result."""
        # 1. Get input variables from variable pool
        inputs = {}
        for var in self.node_data.variables:
            value = self.graph_runtime_state.variable_pool.get(
                var.value_selector
            )
            inputs[var.variable] = value

        # 2. Execute code in sandbox
        code_executor = CodeExecutor(
            language=self.node_data.code_language,
            code=self.node_data.code,
            timeout=self.node_data.timeout,
        )

        try:
            outputs = code_executor.execute(inputs)

            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.SUCCEEDED,
                inputs=inputs,
                outputs=outputs,
            )
        except CodeExecutionError as e:
            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.FAILED,
                error=str(e),
            )
```

## 4. Frontend Node Architecture

### 4.1 Node Type Definition

**Reference**: `/web/app/components/workflow/types.ts` (lines 22-52)

```typescript
export enum BlockEnum {
  Start = 'start',
  End = 'end',
  Answer = 'answer',
  LLM = 'llm',
  KnowledgeRetrieval = 'knowledge-retrieval',
  QuestionClassifier = 'question-classifier',
  IfElse = 'if-else',
  Code = 'code',
  TemplateTransform = 'template-transform',
  HttpRequest = 'http-request',
  Tool = 'tool',
  VariableAssigner = 'variable-assigner',
  VariableAggregator = 'variable-aggregator',
  Iteration = 'iteration',
  Loop = 'loop',
  ParameterExtractor = 'parameter-extractor',
  DocumentExtractor = 'document-extractor',
  ListOperator = 'list-operator',
  Agent = 'agent',
  // Triggers
  TriggerWebhook = 'trigger-webhook',
  TriggerSchedule = 'trigger-schedule',
  TriggerPlugin = 'trigger-plugin',
  HumanInput = 'human-input',
  Datasource = 'datasource',
}
```

### 4.2 Node Metadata Structure

**Reference**: `/web/app/components/workflow/types.ts` (lines 322-360)

```typescript
export type NodeDefault<T = {}> = {
  // Metadata for block selector UI
  metaData: {
    classification: BlockClassificationEnum  // Category in selector
    sort: number                             // Sort order
    type: BlockEnum                          // Node type
    title: string                            // Display title
    author: string                           // Author info
    description?: string                     // Help text
    helpLinkUri?: string                     // Documentation URL
    isRequired?: boolean                     // Cannot be removed
    isUndeletable?: boolean                  // Cannot be deleted
    isStart?: boolean                        // Is start node
    isSingleton?: boolean                    // Only one allowed
    isTypeFixed?: boolean                    // Cannot change type
  }

  // Default configuration when node is created
  defaultValue: Partial<T>

  // Default input for test runs
  defaultRunInputData?: Record<string, any>

  // Validation function
  checkValid: (
    payload: T,
    t: any,  // i18n translator
    moreDataForCheckValid?: any
  ) => {
    isValid: boolean
    errorMessage?: string
  }

  // Output variable schema
  getOutputVars?: (
    payload: T,
    allPluginInfoList: Record<string, ToolWithProvider[]>
  ) => Var[]
}
```

### 4.3 Block Classification

```typescript
export enum BlockClassificationEnum {
  Logic = 'logic',           // If-else, Question classifier
  Transform = 'transform',   // Code, Template, Extract
  Utilities = 'utilities',   // HTTP, Tool
  QuestionUnderstand = 'question-understand',  // LLM, Agent
  Default = 'default',       // Start, End
}
```

### 4.4 Node Default Configuration Example

**Reference**: `/web/app/components/workflow/nodes/code/default.ts`

```typescript
import type { NodeDefault } from '../../types'
import { BlockEnum, BlockClassificationEnum } from '../../types'

// TypeScript type for Code node data
export type CodeNodeType = {
  code_language: string
  code: string
  variables: Array<{
    variable: string
    value_selector: string[]
  }>
  outputs: Record<string, any>
}

// Metadata for block selector
const metaData = {
  classification: BlockClassificationEnum.Transform,
  sort: 3,
  type: BlockEnum.Code,
  title: 'Code',
  author: 'Dify',
  description: 'Execute custom Python or JavaScript code',
  helpLinkUri: 'https://docs.dify.ai/guides/workflow/nodes/code',
}

const nodeDefault: NodeDefault<CodeNodeType> = {
  metaData,

  // Default values when node is added
  defaultValue: {
    code_language: 'python3',
    code: `def main(inputs: dict) -> dict:
    """
    Process inputs and return outputs.

    Args:
        inputs: Dictionary of input variables

    Returns:
        Dictionary of output values
    """
    return {}
`,
    variables: [],
    outputs: {},
  },

  // Validation
  checkValid(payload: CodeNodeType, t: any) {
    if (!payload.code) {
      return {
        isValid: false,
        errorMessage: t('workflow.nodes.code.codeRequired'),
      }
    }

    if (!payload.code_language) {
      return {
        isValid: false,
        errorMessage: t('workflow.nodes.code.languageRequired'),
      }
    }

    return { isValid: true }
  },

  // Define output variables
  getOutputVars(payload: CodeNodeType) {
    return Object.keys(payload.outputs || {}).map(key => ({
      variable: key,
      type: payload.outputs[key].type,
    }))
  },
}

export default nodeDefault
```

### 4.5 Node Component

**Reference**: `/web/app/components/workflow/nodes/code/node.tsx`

```typescript
import { memo } from 'react'
import type { NodeProps } from 'reactflow'
import { CodeBracketIcon } from '@heroicons/react/24/outline'
import type { CodeNodeType } from './types'

// Compact node display on canvas
const CodeNode = memo(({ data }: NodeProps<CodeNodeType>) => {
  const { code_language } = data

  return (
    <div className="flex items-center gap-1 px-3 py-1">
      <CodeBracketIcon className="w-4 h-4 text-gray-500" />
      <span className="text-xs text-gray-500">
        {code_language === 'python3' ? 'Python' : 'JavaScript'}
      </span>
    </div>
  )
})

CodeNode.displayName = 'CodeNode'
export default CodeNode
```

### 4.6 Node Panel (Configuration UI)

**Reference**: `/web/app/components/workflow/nodes/code/panel.tsx`

```typescript
import { memo, useCallback } from 'react'
import { useTranslation } from 'react-i18next'
import type { NodePanelProps } from '../../types'
import type { CodeNodeType } from './types'
import CodeEditor from '@/app/components/base/code-editor'
import Select from '@/app/components/base/select'

const CodePanel = memo(({
  id,
  data,
  onChange
}: NodePanelProps<CodeNodeType>) => {
  const { t } = useTranslation()

  // Handle language change
  const handleLanguageChange = useCallback((language: string) => {
    onChange({
      ...data,
      code_language: language,
    })
  }, [data, onChange])

  // Handle code change
  const handleCodeChange = useCallback((code: string) => {
    onChange({
      ...data,
      code,
    })
  }, [data, onChange])

  return (
    <div className="space-y-4 p-4">
      {/* Language Selector */}
      <div>
        <label className="block text-sm font-medium mb-1">
          {t('workflow.nodes.code.language')}
        </label>
        <Select
          value={data.code_language}
          onChange={handleLanguageChange}
          options={[
            { value: 'python3', label: 'Python 3' },
            { value: 'javascript', label: 'JavaScript' },
          ]}
        />
      </div>

      {/* Code Editor */}
      <div>
        <label className="block text-sm font-medium mb-1">
          {t('workflow.nodes.code.code')}
        </label>
        <CodeEditor
          language={data.code_language}
          value={data.code}
          onChange={handleCodeChange}
          height="300px"
        />
      </div>

      {/* Variables Section */}
      <VariablesConfig
        variables={data.variables}
        onChange={(variables) => onChange({ ...data, variables })}
      />

      {/* Outputs Section */}
      <OutputsConfig
        outputs={data.outputs}
        onChange={(outputs) => onChange({ ...data, outputs })}
      />
    </div>
  )
})

CodePanel.displayName = 'CodePanel'
export default CodePanel
```

### 4.7 Component Registry

**Reference**: `/web/app/components/workflow/nodes/components.ts`

```typescript
import { ComponentType } from 'react'
import { BlockEnum } from '../types'

// Node display components (on canvas)
import StartNode from './start/node'
import EndNode from './end/node'
import AnswerNode from './answer/node'
import LLMNode from './llm/node'
import CodeNode from './code/node'
import IfElseNode from './if-else/node'
// ... more imports

// Panel components (configuration UI)
import StartPanel from './start/panel'
import EndPanel from './end/panel'
import AnswerPanel from './answer/panel'
import LLMPanel from './llm/panel'
import CodePanel from './code/panel'
import IfElsePanel from './if-else/panel'
// ... more imports

// Node component mapping
export const NodeComponentMap: Record<string, ComponentType<any>> = {
  [BlockEnum.Start]: StartNode,
  [BlockEnum.End]: EndNode,
  [BlockEnum.Answer]: AnswerNode,
  [BlockEnum.LLM]: LLMNode,
  [BlockEnum.Code]: CodeNode,
  [BlockEnum.IfElse]: IfElseNode,
  [BlockEnum.KnowledgeRetrieval]: KnowledgeRetrievalNode,
  [BlockEnum.HttpRequest]: HttpRequestNode,
  [BlockEnum.Tool]: ToolNode,
  [BlockEnum.Iteration]: IterationNode,
  [BlockEnum.Loop]: LoopNode,
  // ... more nodes
}

// Panel component mapping
export const PanelComponentMap: Record<string, ComponentType<any>> = {
  [BlockEnum.Start]: StartPanel,
  [BlockEnum.End]: EndPanel,
  [BlockEnum.Answer]: AnswerPanel,
  [BlockEnum.LLM]: LLMPanel,
  [BlockEnum.Code]: CodePanel,
  [BlockEnum.IfElse]: IfElsePanel,
  [BlockEnum.KnowledgeRetrieval]: KnowledgeRetrievalPanel,
  [BlockEnum.HttpRequest]: HttpRequestPanel,
  [BlockEnum.Tool]: ToolPanel,
  [BlockEnum.Iteration]: IterationPanel,
  [BlockEnum.Loop]: LoopPanel,
  // ... more panels
}
```

### 4.8 Block Selector Registration

**Reference**: `/web/app/components/workflow/constants/node.ts`

```typescript
import startDefault from '../nodes/start/default'
import endDefault from '../nodes/end/default'
import answerDefault from '../nodes/answer/default'
import llmDefault from '../nodes/llm/default'
import codeDefault from '../nodes/code/default'
import ifElseDefault from '../nodes/if-else/default'
// ... more imports

// Nodes available in block selector
export const WORKFLOW_COMMON_NODES = [
  llmDefault,
  knowledgeRetrievalDefault,
  agentDefault,
  questionClassifierDefault,
  ifElseDefault,
  iterationDefault,
  loopDefault,
  codeDefault,
  templateTransformDefault,
  variableAggregatorDefault,
  documentExtractorDefault,
  assignerDefault,
  parameterExtractorDefault,
  httpRequestDefault,
  listOperatorDefault,
  toolDefault,
]

// Entry nodes (only one allowed)
export const WORKFLOW_ENTRY_NODES = [
  startDefault,
]

// Response nodes
export const WORKFLOW_RESPONSE_NODES = [
  answerDefault,
  endDefault,
]
```

## 5. Complete Flow: Adding a New Node

### Step 1: Backend - Define Node Type

**File**: `/api/core/workflow/enums.py`

```python
class NodeType(StrEnum):
    # ... existing types
    MY_CUSTOM_NODE = "my-custom-node"  # Add new type
```

### Step 2: Backend - Create Node Data Model

**File**: `/api/core/workflow/nodes/my_custom_node/entities.py`

```python
from pydantic import BaseModel, Field
from core.workflow.nodes.base.entities import BaseNodeData

class MyCustomNodeData(BaseNodeData):
    """Configuration for My Custom Node."""

    # Custom configuration fields
    input_variable: list[str] = Field(
        default=[],
        description="Variable selector for input"
    )
    custom_setting: str = Field(
        default="default_value",
        description="A custom setting"
    )
    enable_feature: bool = Field(
        default=False,
        description="Toggle a feature"
    )
```

### Step 3: Backend - Implement Node Class

**File**: `/api/core/workflow/nodes/my_custom_node/my_custom_node.py`

```python
from typing import Generator
from core.workflow.nodes.base.node import Node
from core.workflow.nodes.base.entities import NodeRunResult
from core.workflow.enums import NodeType, WorkflowNodeExecutionStatus
from .entities import MyCustomNodeData

class MyCustomNode(Node[MyCustomNodeData]):
    """My Custom Node implementation."""

    node_type = NodeType.MY_CUSTOM_NODE

    @classmethod
    def version(cls) -> str:
        return "1"

    @classmethod
    def get_default_config(cls, filters: dict | None = None) -> dict:
        """Return default configuration."""
        return {
            "type": cls.node_type.value,
            "config": {
                "input_variable": [],
                "custom_setting": "default_value",
                "enable_feature": False,
            }
        }

    def _run(self) -> NodeRunResult:
        """Execute the node."""
        # 1. Get inputs from variable pool
        input_value = None
        if self.node_data.input_variable:
            input_value = self.graph_runtime_state.variable_pool.get(
                self.node_data.input_variable
            )

        # 2. Process logic
        try:
            result = self._process(input_value)

            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.SUCCEEDED,
                inputs={"input": input_value},
                outputs={"result": result},
            )

        except Exception as e:
            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.FAILED,
                error=str(e),
            )

    def _process(self, input_value: Any) -> Any:
        """Custom processing logic."""
        # Your implementation here
        if self.node_data.enable_feature:
            return f"Processed: {input_value} with {self.node_data.custom_setting}"
        return input_value
```

### Step 4: Backend - Export Node

**File**: `/api/core/workflow/nodes/my_custom_node/__init__.py`

```python
from .my_custom_node import MyCustomNode
from .entities import MyCustomNodeData

__all__ = ["MyCustomNode", "MyCustomNodeData"]
```

### Step 5: Frontend - Define Types

**File**: `/web/app/components/workflow/types.ts`

```typescript
export enum BlockEnum {
  // ... existing types
  MyCustomNode = 'my-custom-node',  // Add new type
}
```

**File**: `/web/app/components/workflow/nodes/my-custom-node/types.ts`

```typescript
export type MyCustomNodeType = {
  input_variable: string[]
  custom_setting: string
  enable_feature: boolean
}
```

### Step 6: Frontend - Create Default Config

**File**: `/web/app/components/workflow/nodes/my-custom-node/default.ts`

```typescript
import type { NodeDefault } from '../../types'
import { BlockEnum, BlockClassificationEnum } from '../../types'
import type { MyCustomNodeType } from './types'

const metaData = {
  classification: BlockClassificationEnum.Transform,
  sort: 20,  // Position in block selector
  type: BlockEnum.MyCustomNode,
  title: 'My Custom Node',
  author: 'Your Name',
  description: 'Description of what this node does',
}

const nodeDefault: NodeDefault<MyCustomNodeType> = {
  metaData,

  defaultValue: {
    input_variable: [],
    custom_setting: 'default_value',
    enable_feature: false,
  },

  checkValid(payload: MyCustomNodeType, t: any) {
    // Add validation logic
    return { isValid: true }
  },

  getOutputVars(payload: MyCustomNodeType) {
    return [
      { variable: 'result', type: 'string' }
    ]
  },
}

export default nodeDefault
```

### Step 7: Frontend - Create Node Component

**File**: `/web/app/components/workflow/nodes/my-custom-node/node.tsx`

```typescript
import { memo } from 'react'
import type { NodeProps } from 'reactflow'
import { SparklesIcon } from '@heroicons/react/24/outline'
import type { MyCustomNodeType } from './types'

const MyCustomNode = memo(({ data }: NodeProps<MyCustomNodeType>) => {
  return (
    <div className="flex items-center gap-2 px-3 py-1">
      <SparklesIcon className="w-4 h-4 text-blue-500" />
      <span className="text-xs text-gray-600">
        {data.enable_feature ? 'Feature Enabled' : 'Standard'}
      </span>
    </div>
  )
})

MyCustomNode.displayName = 'MyCustomNode'
export default MyCustomNode
```

### Step 8: Frontend - Create Panel Component

**File**: `/web/app/components/workflow/nodes/my-custom-node/panel.tsx`

```typescript
import { memo, useCallback } from 'react'
import { useTranslation } from 'react-i18next'
import type { NodePanelProps } from '../../types'
import type { MyCustomNodeType } from './types'
import VarSelect from '@/app/components/workflow/nodes/_base/components/var-select'
import Input from '@/app/components/base/input'
import Switch from '@/app/components/base/switch'

const MyCustomNodePanel = memo(({
  id,
  data,
  onChange
}: NodePanelProps<MyCustomNodeType>) => {
  const { t } = useTranslation()

  return (
    <div className="space-y-4 p-4">
      {/* Input Variable Selector */}
      <div>
        <label className="block text-sm font-medium mb-1">
          Input Variable
        </label>
        <VarSelect
          value={data.input_variable}
          onChange={(value) => onChange({ ...data, input_variable: value })}
        />
      </div>

      {/* Custom Setting */}
      <div>
        <label className="block text-sm font-medium mb-1">
          Custom Setting
        </label>
        <Input
          value={data.custom_setting}
          onChange={(e) => onChange({
            ...data,
            custom_setting: e.target.value
          })}
        />
      </div>

      {/* Feature Toggle */}
      <div className="flex items-center justify-between">
        <label className="text-sm font-medium">
          Enable Feature
        </label>
        <Switch
          checked={data.enable_feature}
          onChange={(checked) => onChange({
            ...data,
            enable_feature: checked
          })}
        />
      </div>
    </div>
  )
})

MyCustomNodePanel.displayName = 'MyCustomNodePanel'
export default MyCustomNodePanel
```

### Step 9: Frontend - Register Components

**File**: `/web/app/components/workflow/nodes/components.ts`

```typescript
// Add imports
import MyCustomNode from './my-custom-node/node'
import MyCustomNodePanel from './my-custom-node/panel'

// Add to NodeComponentMap
export const NodeComponentMap: Record<string, ComponentType<any>> = {
  // ... existing entries
  [BlockEnum.MyCustomNode]: MyCustomNode,
}

// Add to PanelComponentMap
export const PanelComponentMap: Record<string, ComponentType<any>> = {
  // ... existing entries
  [BlockEnum.MyCustomNode]: MyCustomNodePanel,
}
```

**File**: `/web/app/components/workflow/constants/node.ts`

```typescript
import myCustomNodeDefault from '../nodes/my-custom-node/default'

export const WORKFLOW_COMMON_NODES = [
  // ... existing nodes
  myCustomNodeDefault,
]
```

## 6. Sequence Diagram: Node Execution Flow

```
┌──────────┐     ┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│  User    │     │  Frontend   │     │   Backend     │     │  GraphEngine │
│          │     │  (React)    │     │   (Flask)     │     │              │
└────┬─────┘     └──────┬──────┘     └───────┬───────┘     └──────┬───────┘
     │                  │                    │                    │
     │ 1. Add Node      │                    │                    │
     │─────────────────►│                    │                    │
     │                  │                    │                    │
     │                  │ 2. Load default    │                    │
     │                  │    config          │                    │
     │                  │ (from default.ts)  │                    │
     │                  │                    │                    │
     │ 3. Configure     │                    │                    │
     │─────────────────►│                    │                    │
     │                  │                    │                    │
     │                  │ 4. Update store    │                    │
     │                  │ (Zustand)          │                    │
     │                  │                    │                    │
     │ 5. Run Workflow  │                    │                    │
     │─────────────────►│                    │                    │
     │                  │                    │                    │
     │                  │ 6. POST /workflows │                    │
     │                  │    /draft/run      │                    │
     │                  │───────────────────►│                    │
     │                  │                    │                    │
     │                  │                    │ 7. Build Graph     │
     │                  │                    │───────────────────►│
     │                  │                    │                    │
     │                  │                    │ 8. NodeFactory     │
     │                  │                    │    .create_node()  │
     │                  │                    │───────────────────►│
     │                  │                    │                    │
     │                  │                    │ 9. Lookup in       │
     │                  │                    │    _registry       │
     │                  │                    │    Instantiate     │
     │                  │                    │───────────────────►│
     │                  │                    │                    │
     │                  │                    │ 10. Execute        │
     │                  │                    │     node._run()    │
     │                  │                    │◄───────────────────│
     │                  │                    │                    │
     │                  │ 11. SSE Events     │                    │
     │                  │◄───────────────────│                    │
     │                  │                    │                    │
     │ 12. UI Updates   │                    │                    │
     │◄─────────────────│                    │                    │
     │                  │                    │                    │
```

## 7. Key Design Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| **Auto-Registration** | `Node.__init_subclass__` | Automatic node discovery |
| **Generic Types** | `Node[T]` | Type-safe node data |
| **Factory** | `DifyNodeFactory` | Node instantiation |
| **Registry** | `Node._registry` | Node type lookup |
| **Version Management** | `version()` method | Multiple node versions |
| **Component Map** | Frontend maps | Dynamic component loading |
| **Default Config** | `default.ts` files | Initial node configuration |

## 8. Best Practices

### Backend

1. **Always inherit from `Node[YourDataType]`** - This enables auto-registration
2. **Define `node_type` class variable** - Required for registry lookup
3. **Implement `version()` method** - Enables version management
4. **Use Pydantic for data models** - Automatic validation
5. **Handle errors gracefully** - Return `NodeRunResult` with error status

### Frontend

1. **Keep node component minimal** - Only essential info on canvas
2. **Full configuration in panel** - All settings in panel component
3. **Implement `checkValid`** - Prevent invalid configurations
4. **Define `getOutputVars`** - Enable variable connections
5. **Use i18n for all strings** - Support internationalization

---

*Quay lại: [Workflow Engine](./03-workflow-engine.md) | [README](./README.md)*
