# Frontend Web Architecture

## 1. Tổng quan

Frontend của Dify là một **Next.js 15 application** sử dụng:
- **React 19** với functional components và hooks
- **TypeScript** với strict mode
- **Zustand** cho client state management
- **TanStack React Query** cho server state
- **ReactFlow** cho workflow editor DAG visualization
- **Tailwind CSS** cho styling

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         FRONTEND ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Next.js App Router                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │   │
│  │  │   layout    │  │   pages     │  │      middleware         │ │   │
│  │  │  (Providers)│  │  (Routes)   │  │   (Auth, i18n)         │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     State Management                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │   │
│  │  │   Zustand    │  │ React Query  │  │    React Context     │  │   │
│  │  │ (UI State)   │  │(Server State)│  │  (Global Config)     │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       Components                                 │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │   │
│  │  │    Base    │  │  Feature   │  │  Workflow  │  │   App     │ │   │
│  │  │ (Primitives│  │(Datasets,  │  │  (Editor)  │  │ (Config)  │ │   │
│  │  │  103 dirs) │  │ Tools,etc) │  │            │  │           │ │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └───────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Service Layer                                │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │                   React Query Hooks                         │ │   │
│  │  │  useWorkflow | useApps | useDatasets | useModels | ...      │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │                   HTTP Client (ky)                          │ │   │
│  │  │  get | post | patch | put | del | streaming                 │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. Next.js App Router Structure

### 3.1 Root Layout

**Reference**: `/web/app/layout.tsx` (lines 1-118)

```typescript
// Hierarchical provider pattern
export default async function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider>
          <BrowserInitializer>
            <SentryInitializer>
              <TanstackQueryInitializer>
                <I18nServer>
                  <GlobalPublicStoreProvider>
                    {children}
                  </GlobalPublicStoreProvider>
                </I18nServer>
              </TanstackQueryInitializer>
            </SentryInitializer>
          </BrowserInitializer>
        </ThemeProvider>
      </body>
    </html>
  )
}
```

### 3.2 Route Structure

```
/web/app/
├── layout.tsx                           # Root layout (providers)
├── page.tsx                             # Root page (redirects to /apps)
│
├── (commonLayout)/                      # Admin dashboard routes
│   ├── layout.tsx                       # Sidebar layout
│   ├── apps/page.tsx                    # App listing
│   ├── app/[appId]/                     # App detail routes
│   │   ├── overview/page.tsx           # App overview
│   │   ├── workflow/page.tsx           # Workflow editor
│   │   ├── configuration/page.tsx      # App settings
│   │   └── logs/page.tsx               # Run logs
│   ├── datasets/                        # Knowledge base
│   │   ├── page.tsx                    # Dataset listing
│   │   └── [datasetId]/page.tsx        # Dataset detail
│   ├── plugins/page.tsx                 # Plugin marketplace
│   └── tools/page.tsx                   # Tool management
│
├── (shareLayout)/                       # Public/shared routes
│   ├── layout.tsx                       # Minimal layout
│   ├── chat/[token]/page.tsx           # Shared chat interface
│   └── workflow/[token]/page.tsx       # Shared workflow runner
│
├── signin/page.tsx                      # Login
├── signup/page.tsx                      # Registration
├── account/                             # Account settings
│   └── settings/page.tsx
│
└── oauth-callback/page.tsx              # OAuth handling
```

### 3.3 Workflow Page Example

**Reference**: `/web/app/(commonLayout)/app/(appDetailLayout)/[appId]/workflow/page.tsx`

```typescript
'use client'

import WorkflowApp from '@/app/components/workflow-app'

export default function WorkflowPage() {
  // Simple wrapper delegating to WorkflowApp component
  return <WorkflowApp />
}
```

## 4. State Management

### 4.1 State Management Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    STATE HIERARCHY                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Component Local State                  │   │
│  │                   (useState)                         │   │
│  │            Per-component UI state                    │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │               Client State (Zustand)                 │   │
│  │  ┌────────────────────────────────────────────────┐ │   │
│  │  │              Workflow Store                     │ │   │
│  │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐          │ │   │
│  │  │  │workflow │ │  node   │ │ panel   │          │ │   │
│  │  │  │ slice   │ │ slice   │ │ slice   │ ...      │ │   │
│  │  │  └─────────┘ └─────────┘ └─────────┘          │ │   │
│  │  │         (15 slices total)                      │ │   │
│  │  └────────────────────────────────────────────────┘ │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │             Server State (React Query)               │   │
│  │  ┌────────────────────────────────────────────────┐ │   │
│  │  │ useWorkflow | useApps | useModels | useDatasets│ │   │
│  │  │    (API caching, background refresh)           │ │   │
│  │  └────────────────────────────────────────────────┘ │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │             Global Config (React Context)            │   │
│  │  ┌────────────────────────────────────────────────┐ │   │
│  │  │ AppContext | ProviderContext | I18nContext     │ │   │
│  │  │    (User, workspace, providers, i18n)          │ │   │
│  │  └────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Zustand Store - Workflow Example

**Reference**: `/web/app/components/workflow/store/workflow/index.ts` (lines 40-97)

```typescript
import { createStore } from 'zustand'

// Slice types combined
export type Shape =
  & ChatVariableSliceShape
  & EnvVariableSliceShape
  & FormSliceShape
  & HelpLineSliceShape
  & HistorySliceShape
  & NodeSliceShape
  & PanelSliceShape
  & ToolSliceShape
  & VersionSliceShape
  & WorkflowDraftSliceShape
  & WorkflowSliceShape
  & InspectVarsSliceShape
  & LayoutSliceShape

export type CreateWorkflowStoreParams = {
  workflowId: string
  tenantId: string
  // ... other params
}

export const createWorkflowStore = (params: CreateWorkflowStoreParams) => {
  return createStore<Shape>((...args) => ({
    // Combine all slices
    ...createChatVariableSlice(...args),
    ...createEnvVariableSlice(...args),
    ...createFormSlice(...args),
    ...createHelpLineSlice(...args),
    ...createHistorySlice(...args),
    ...createNodeSlice(...args),
    ...createPanelSlice(...args),
    ...createToolSlice(...args),
    ...createVersionSlice(...args),
    ...createWorkflowDraftSlice(...args),
    ...createWorkflowSlice(params)(...args),
    ...createInspectVarsSlice(...args),
    ...createLayoutSlice(...args),
  }))
}
```

### 4.3 Individual Slice Example

**Reference**: `/web/app/components/workflow/store/workflow/node-slice.ts`

```typescript
import { StateCreator } from 'zustand'

export type NodeSliceShape = {
  // State
  showSingleRunPanel: boolean
  nodeAnimation: boolean
  candidateNode?: Node
  contextMenuNode?: Node

  // Actions
  setShowSingleRunPanel: (show: boolean) => void
  setNodeAnimation: (animate: boolean) => void
  setCandidateNode: (node?: Node) => void
  setContextMenuNode: (node?: Node) => void
}

export const createNodeSlice: StateCreator<NodeSliceShape> = (set) => ({
  // Initial state
  showSingleRunPanel: false,
  nodeAnimation: false,
  candidateNode: undefined,
  contextMenuNode: undefined,

  // Actions
  setShowSingleRunPanel: (show) =>
    set(() => ({ showSingleRunPanel: show })),

  setNodeAnimation: (animate) =>
    set(() => ({ nodeAnimation: animate })),

  setCandidateNode: (node) =>
    set(() => ({ candidateNode: node })),

  setContextMenuNode: (node) =>
    set(() => ({ contextMenuNode: node })),
})
```

### 4.4 React Query Setup

**Reference**: `/web/context/query-client.tsx` (lines 1-25)

```typescript
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

// 30 minute stale time
const STALE_TIME = 1000 * 60 * 30

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: STALE_TIME,
      refetchOnWindowFocus: false,
    },
  },
})

export function TanstackQueryInitializer({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  )
}
```

### 4.5 React Context Example

**Reference**: `/web/context/app-context.tsx` (lines 20-75)

```typescript
import { createContext } from 'use-context-selector'

export type AppContextValue = {
  // User info
  userProfile: UserProfileResponse
  currentWorkspace: ICurrentWorkspace

  // Permission checks
  isCurrentWorkspaceManager: boolean
  isCurrentWorkspaceOwner: boolean
  isCurrentWorkspaceEditor: boolean
  isCurrentWorkspaceDatasetOperator: boolean

  // System info
  langGeniusVersionInfo: LangGeniusVersionResponse
  isLoadingCurrentWorkspace: boolean
}

const AppContext = createContext<AppContextValue>({} as AppContextValue)

export const useAppContext = () => useContext(AppContext)

export function AppContextProvider({
  children,
  userProfile,
  workspace,
}: {
  children: React.ReactNode
  userProfile: UserProfileResponse
  workspace: ICurrentWorkspace
}) {
  const value: AppContextValue = useMemo(() => ({
    userProfile,
    currentWorkspace: workspace,
    isCurrentWorkspaceManager: workspace.role === 'owner' || workspace.role === 'admin',
    isCurrentWorkspaceOwner: workspace.role === 'owner',
    isCurrentWorkspaceEditor: ['owner', 'admin', 'editor'].includes(workspace.role),
    isCurrentWorkspaceDatasetOperator: workspace.role === 'dataset_operator',
    langGeniusVersionInfo: {},
    isLoadingCurrentWorkspace: false,
  }), [userProfile, workspace])

  return (
    <AppContext.Provider value={value}>
      {children}
    </AppContext.Provider>
  )
}
```

## 5. Component Architecture

### 5.1 Component Organization

```
/web/app/components/
├── base/                    # Atomic/Primitive UI (103 directories)
│   ├── button/             # Button variants
│   ├── input/              # Input components
│   ├── dialog/             # Modal dialogs
│   ├── toast/              # Notifications
│   ├── loading/            # Loading states
│   ├── badge/              # Status badges
│   ├── icons/              # Icon library
│   ├── select/             # Dropdowns
│   ├── checkbox/           # Checkbox/radio
│   ├── tabs/               # Tab navigation
│   ├── tooltip/            # Tooltips
│   ├── chat/               # Chat display
│   ├── markdown/           # Markdown renderer
│   ├── code-editor/        # Code editor (Monaco)
│   ├── prompt-editor/      # Prompt template editor
│   ├── file-uploader/      # File uploads
│   ├── tag-management/     # Tag CRUD
│   └── ...                 # 90+ more
│
├── workflow/                # Workflow editor
│   ├── index.tsx           # Main editor
│   ├── store/              # Zustand stores
│   ├── nodes/              # 31 node types
│   ├── panel/              # Config panels
│   ├── header/             # Toolbar
│   ├── run/                # Run UI
│   └── hooks/              # Custom hooks
│
├── workflow-app/            # Workflow app container
│   └── index.tsx
│
├── app/                     # App management
│   ├── store.ts            # App store
│   ├── overview/           # Overview page
│   ├── configuration/      # Config page
│   └── ...
│
├── datasets/                # Knowledge base
│   ├── create/             # Dataset creation
│   ├── documents/          # Document list
│   └── settings/           # Dataset settings
│
├── tools/                   # Tool management
│   ├── tool-list/          # Tool listing
│   ├── tool-detail/        # Tool details
│   └── ...
│
├── plugins/                 # Plugin system
│   ├── marketplace/        # Plugin store
│   ├── installed/          # Installed plugins
│   └── ...
│
├── header/                  # Top navigation
│   ├── account/            # Account menu
│   └── nav/                # Main nav
│
└── signin/                  # Auth components
    └── signup/
```

### 5.2 Workflow Node Components

**Reference**: `/web/app/components/workflow/nodes/`

```
nodes/
├── start/                   # Entry node
│   ├── node.tsx            # Compact display
│   ├── panel.tsx           # Config panel
│   └── types.ts            # TypeScript types
│
├── end/                     # Exit node
├── answer/                  # Answer output
│
├── llm/                     # LLM invocation
│   ├── node.tsx            # Model selector display
│   ├── panel.tsx           # Full LLM config
│   ├── types.ts
│   ├── hooks/              # LLM-specific hooks
│   └── components/         # Sub-components
│       ├── prompt-editor/
│       ├── memory-config/
│       └── output-vars/
│
├── code/                    # Code execution
├── http/                    # HTTP request
├── tool/                    # Tool invocation
├── agent/                   # Agent node
│
├── knowledge-base/          # Knowledge retrieval
├── if-else/                 # Conditional branching
├── loop/                    # Loop iteration
├── iteration/               # Iteration control
│
├── variable-assigner/       # Variable assignment
├── template-transform/      # String templates
├── parameter-extractor/     # Data extraction
├── question-classifier/     # Input classification
│
├── trigger-webhook/         # Webhook trigger
├── trigger-schedule/        # Schedule trigger
├── trigger-plugin/          # Plugin trigger
│
└── _base/                   # Shared node utilities
    ├── node.tsx            # Base node wrapper
    ├── panel.tsx           # Base panel wrapper
    └── hooks/              # Shared hooks
```

### 5.3 LLM Node Example

**Reference**: `/web/app/components/workflow/nodes/llm/node.tsx` (lines 1-36)

```typescript
import { FC } from 'react'
import { NodeProps } from 'reactflow'
import ModelSelector from '@/app/components/base/model-selector'
import { useTextGenerationCurrentProviderAndModelAndModelList } from '@/app/components/workflow/hooks'

type LLMNodeType = {
  model?: {
    provider: string
    name: string
  }
  // ... other props
}

const Node: FC<NodeProps<LLMNodeType>> = ({ data }) => {
  const { provider, name: modelId } = data.model || {}
  const { textGenerationModelList } = useTextGenerationCurrentProviderAndModelAndModelList()

  const hasSetModel = provider && modelId

  if (!hasSetModel) return null

  return (
    <div className="mb-1 px-3 py-1">
      <ModelSelector
        defaultModel={{ provider, modelId }}
        modelList={textGenerationModelList}
        triggerClassName="!h-6 !rounded-md"
        readonly
      />
    </div>
  )
}

export default Node
```

## 6. Service Layer

### 6.1 HTTP Client

**Reference**: `/web/service/base.ts` (lines 1-150)

```typescript
import ky from 'ky'

export type IOtherOptions = {
  isPublicAPI?: boolean
  isMarketplaceAPI?: boolean
  bodyStringify?: boolean
  needAllResponseContent?: boolean
  deleteContentType?: boolean
  silent?: boolean

  // Streaming callbacks
  onData?: IOnData
  onThought?: IOnThought
  onFile?: IOnFile
  onMessageEnd?: IOnMessageEnd
  onMessageReplace?: IOnMessageReplace
  onError?: IOnError
  onCompleted?: IOnCompleted

  // Workflow stream handlers
  onWorkflowStarted?: IOnWorkflowStarted
  onWorkflowFinished?: IOnWorkflowFinished
  onNodeStarted?: IOnNodeStarted
  onNodeFinished?: IOnNodeFinished
  onIterationStart?: IOnIterationStarted
  onIterationNext?: IOnIterationNext
  onIterationFinished?: IOnIterationFinished
  // ... more callbacks
}

// Base HTTP client
const client = ky.create({
  prefixUrl: '/api',
  timeout: 60000,
  hooks: {
    beforeRequest: [
      (request) => {
        // Add auth token
        const token = getToken()
        if (token) {
          request.headers.set('Authorization', `Bearer ${token}`)
        }
      },
    ],
    afterResponse: [
      async (request, options, response) => {
        // Handle 401 - refresh token
        if (response.status === 401) {
          await refreshToken()
          // Retry request
        }
      },
    ],
  },
})

export async function get<T>(
  url: string,
  params?: Record<string, any>,
  options?: IOtherOptions,
): Promise<T> {
  const response = await client.get(url, {
    searchParams: params,
    ...options,
  })
  return response.json()
}

export async function post<T>(
  url: string,
  { body, ...options }: { body?: any } & IOtherOptions,
): Promise<T> {
  const response = await client.post(url, {
    json: body,
    ...options,
  })
  return response.json()
}

// Similar for patch, put, del...
```

### 6.2 Service Hooks

**Reference**: `/web/service/use-workflow.ts` (lines 21-127)

```typescript
import { useQuery, useMutation, useInfiniteQuery } from '@tanstack/react-query'
import { get, post, patch } from './base'

const NAME_SPACE = 'workflow'

export const workflowQueryKeys = {
  workflow: (appId: string) => [NAME_SPACE, 'workflow', appId] as const,
  runHistory: (url?: string) => [NAME_SPACE, 'runHistory', url] as const,
  versionHistory: (appId: string) => [NAME_SPACE, 'versions', appId] as const,
  lastRun: (appId: string, nodeId: string) =>
    [NAME_SPACE, 'lastRun', appId, nodeId] as const,
}

// Fetch published workflow
export const useAppWorkflow = (appId: string) => {
  return useQuery({
    queryKey: workflowQueryKeys.workflow(appId),
    queryFn: () => get<WorkflowResponse>(`/apps/${appId}/workflows/published`),
    enabled: !!appId,
  })
}

// Fetch run history with pagination
export const useWorkflowRunHistory = (url?: string) => {
  return useQuery({
    queryKey: workflowQueryKeys.runHistory(url),
    queryFn: () => get<WorkflowRunHistoryResponse>(url!),
    enabled: !!url,
  })
}

// Infinite query for version history
export const useWorkflowVersionHistory = (appId: string) => {
  return useInfiniteQuery({
    queryKey: workflowQueryKeys.versionHistory(appId),
    queryFn: ({ pageParam = 1 }) =>
      get<WorkflowVersionsResponse>(`/apps/${appId}/workflows/versions`, {
        page: pageParam,
        limit: 20,
      }),
    getNextPageParam: (lastPage) =>
      lastPage.has_more ? lastPage.page + 1 : undefined,
    initialPageParam: 1,
  })
}

// Update workflow mutation
export const useUpdateWorkflow = (appId: string) => {
  return useMutation({
    mutationFn: (data: UpdateWorkflowRequest) =>
      patch<WorkflowResponse>(`/apps/${appId}/workflows/draft`, { body: data }),
    onSuccess: () => {
      // Invalidate related queries
      queryClient.invalidateQueries({
        queryKey: workflowQueryKeys.workflow(appId),
      })
    },
  })
}

// Publish workflow mutation
export const usePublishWorkflow = (appId: string) => {
  return useMutation({
    mutationFn: () =>
      post<WorkflowResponse>(`/apps/${appId}/workflows/publish`, {}),
  })
}

// Fetch last run for node tracing
export const useLastRun = (appId: string, nodeId: string) => {
  return useQuery({
    queryKey: workflowQueryKeys.lastRun(appId, nodeId),
    queryFn: () =>
      get<NodeExecutionResponse>(`/apps/${appId}/workflows/nodes/${nodeId}/last-run`),
    enabled: !!appId && !!nodeId,
  })
}
```

### 6.3 Service Organization

```
/web/service/
├── base.ts                  # Core HTTP client (ky wrapper)
├── fetch.ts                 # Fetch utilities
│
├── use-common.ts            # Common queries (user, workspace, providers)
├── use-workflow.ts          # Workflow queries and mutations
├── use-apps.ts              # App queries
├── use-models.ts            # Model provider queries
├── use-tools.ts             # Tool queries
├── use-plugins.ts           # Plugin queries
├── use-pipeline.ts          # RAG pipeline queries
│
├── share.ts                 # Public share API
├── universal-chat.ts        # Chat API
├── annotations.ts           # Annotation API
├── billing.ts               # Billing API
│
└── knowledge/               # Knowledge base services
    ├── index.ts             # Dataset queries
    ├── documents.ts         # Document queries
    └── hit-testing.ts       # Retrieval testing
```

## 7. Workflow Editor

### 7.1 Main Editor Component

**Reference**: `/web/app/components/workflow/index.tsx` (lines 1-100)

```typescript
'use client'

import ReactFlow, {
  Background,
  ReactFlowProvider,
  SelectionMode,
  useEdgesState,
  useNodesState,
  useOnViewportChange,
  useReactFlow,
  useStoreApi,
} from 'reactflow'

import CustomNode from './nodes'
import CustomEdge from './custom-edge'
import CustomConnectionLine from './custom-connection-line'
import { useStore, useWorkflowStore } from './store'
import { HooksStoreContextProvider, useHooksStore } from './hooks-store'
import { WorkflowHistoryProvider } from './workflow-history-store'
import {
  useEdgesInteractions,
  useNodesInteractions,
  useNodesReadOnly,
  useNodesSyncDraft,
  usePanelInteractions,
  useSelectionInteractions,
  useShortcuts,
  useWorkflow,
  useWorkflowReadOnly,
  useWorkflowRefreshDraft,
} from './hooks'

export default function Workflow() {
  // Initialize workflow state
  const { nodes, edges, onNodesChange, onEdgesChange } = useWorkflow()

  // Interactions
  const { onConnect, onEdgeClick } = useEdgesInteractions()
  const { onNodeClick, onNodeDragStop } = useNodesInteractions()
  const { onPaneClick } = usePanelInteractions()
  const { onSelectionChange } = useSelectionInteractions()

  // Auto-save
  useNodesSyncDraft()
  useWorkflowRefreshDraft()

  // Keyboard shortcuts
  useShortcuts()

  // Custom node/edge types
  const nodeTypes = useMemo(() => ({
    custom: CustomNode,
  }), [])

  const edgeTypes = useMemo(() => ({
    custom: CustomEdge,
  }), [])

  return (
    <ReactFlowProvider>
      <div className="h-full w-full">
        <ReactFlow
          nodes={nodes}
          edges={edges}
          onNodesChange={onNodesChange}
          onEdgesChange={onEdgesChange}
          onConnect={onConnect}
          onNodeClick={onNodeClick}
          onNodeDragStop={onNodeDragStop}
          onEdgeClick={onEdgeClick}
          onPaneClick={onPaneClick}
          onSelectionChange={onSelectionChange}
          nodeTypes={nodeTypes}
          edgeTypes={edgeTypes}
          connectionLineComponent={CustomConnectionLine}
          selectionMode={SelectionMode.Partial}
          minZoom={0.1}
          maxZoom={2}
          defaultViewport={{ x: 0, y: 0, zoom: 1 }}
        >
          <Background />
        </ReactFlow>
      </div>
    </ReactFlowProvider>
  )
}
```

### 7.2 Workflow App Container

**Reference**: `/web/app/components/workflow-app/index.tsx` (lines 1-80)

```typescript
'use client'

import { useEffect } from 'react'
import Workflow from '@/app/components/workflow'
import { useWorkflowStore, WorkflowContextProvider } from '@/app/components/workflow/store'
import { useAppTriggers } from '@/service/use-tools'
import { useTriggerStatusStore } from '@/app/components/workflow/trigger-status-store'
import { useWorkflowInit } from './hooks'

export default function WorkflowApp() {
  return (
    <WorkflowContextProvider>
      <WorkflowAppWithContext />
    </WorkflowContextProvider>
  )
}

function WorkflowAppWithContext() {
  const { data, isLoading, fileUploadConfigResponse } = useWorkflowInit()
  const workflowStore = useWorkflowStore()
  const { setTriggerStatuses } = useTriggerStatusStore()

  // Get app ID from route
  const appId = useParams().appId as string
  const isWorkflowMode = true

  // Fetch trigger statuses
  const { data: triggersResponse } = useAppTriggers(
    isWorkflowMode ? appId : undefined,
    {
      staleTime: 5 * 60 * 1000,
      refetchOnWindowFocus: false,
    }
  )

  // Sync trigger statuses to store
  useEffect(() => {
    if (triggersResponse?.data) {
      const statusMap = triggersResponse.data.reduce((acc, trigger) => {
        acc[trigger.node_id] = trigger.status === 'enabled' ? 'enabled' : 'disabled'
        return acc
      }, {} as Record<string, 'enabled' | 'disabled'>)
      setTriggerStatuses(statusMap)
    }
  }, [triggersResponse?.data, setTriggerStatuses])

  if (isLoading) {
    return <Loading />
  }

  return (
    <div className="h-full flex flex-col">
      <WorkflowHeader />
      <div className="flex-1 overflow-hidden">
        <Workflow />
      </div>
      <WorkflowPanel />
    </div>
  )
}
```

## 8. Custom Hooks

### 8.1 Workflow Hooks

**Reference**: `/web/app/components/workflow/hooks/`

```typescript
// useWorkflow - Main workflow logic
export const useWorkflow = () => {
  const store = useWorkflowStore()
  const { data } = useAppWorkflow(appId)

  // Initialize nodes and edges from API data
  useEffect(() => {
    if (data) {
      store.setNodes(data.graph.nodes)
      store.setEdges(data.graph.edges)
    }
  }, [data])

  return {
    nodes: store.nodes,
    edges: store.edges,
    onNodesChange: store.onNodesChange,
    onEdgesChange: store.onEdgesChange,
  }
}

// useNodesInteractions - Node click/drag handlers
export const useNodesInteractions = () => {
  const store = useWorkflowStore()

  const onNodeClick = useCallback((event: React.MouseEvent, node: Node) => {
    store.setSelectedNode(node.id)
    store.setShowPanel(true)
  }, [store])

  const onNodeDragStop = useCallback((event: React.MouseEvent, node: Node) => {
    store.updateNodePosition(node.id, node.position)
  }, [store])

  return { onNodeClick, onNodeDragStop }
}

// useNodesSyncDraft - Auto-save draft changes
export const useNodesSyncDraft = () => {
  const store = useWorkflowStore()
  const updateWorkflow = useUpdateWorkflow(appId)

  // Debounced save
  const debouncedSave = useDebouncedCallback(
    async (nodes, edges) => {
      await updateWorkflow.mutateAsync({
        graph: { nodes, edges },
      })
    },
    1000,
  )

  useEffect(() => {
    debouncedSave(store.nodes, store.edges)
  }, [store.nodes, store.edges])
}

// useShortcuts - Keyboard shortcuts
export const useShortcuts = () => {
  const store = useWorkflowStore()

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      // Ctrl/Cmd + Z - Undo
      if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
        e.preventDefault()
        store.undo()
      }

      // Ctrl/Cmd + Shift + Z - Redo
      if ((e.ctrlKey || e.metaKey) && e.shiftKey && e.key === 'z') {
        e.preventDefault()
        store.redo()
      }

      // Delete - Remove selected
      if (e.key === 'Delete' || e.key === 'Backspace') {
        store.deleteSelected()
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => window.removeEventListener('keydown', handleKeyDown)
  }, [store])
}
```

## 9. Data Flow

### 9.1 Loading Workflow

```
User visits /app/[appId]/workflow
              │
              ▼
┌─────────────────────────────────┐
│   WorkflowApp Component         │
│   useWorkflowInit() hook        │
└───────────────┬─────────────────┘
                │
                ├─► useAppWorkflow(appId)
                │   React Query [workflow, publish, appId]
                │
                ├─► useAppContext()
                │   AppContext (user & workspace)
                │
                └─► useAppTriggers(appId)
                    React Query [triggers, appId]
                │
                ▼
┌─────────────────────────────────┐
│   WorkflowContextProvider       │
│   createWorkflowStore(params)   │
│   15 slices combined            │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│   Workflow Component            │
│   ReactFlow with custom nodes   │
│   31 node types                 │
└─────────────────────────────────┘
```

### 9.2 Workflow Execution

```
User clicks "Run" button
              │
              ▼
┌─────────────────────────────────┐
│   handleRun() from HooksStore   │
│   POST /apps/{appId}/workflows/ │
│        draft/run                │
└───────────────┬─────────────────┘
                │
                │ Opens SSE connection
                │
                ▼
┌─────────────────────────────────┐
│   SSE Event Callbacks:          │
│   ├─ onWorkflowStarted         │
│   ├─ onNodeStarted             │
│   ├─ onNodeFinished            │
│   ├─ onTextChunk (LLM)         │
│   ├─ onIterationStart          │
│   ├─ onIterationFinished       │
│   ├─ onAgentLog                │
│   └─ onWorkflowFinished        │
└───────────────┬─────────────────┘
                │
                │ Updates store in real-time
                │
                ▼
┌─────────────────────────────────┐
│   Store Updates:                │
│   ├─ workflowRunningData       │
│   ├─ nodeAnimation             │
│   └─ Node output values        │
└───────────────┬─────────────────┘
                │
                │ UI reacts via selectors
                │
                ▼
┌─────────────────────────────────┐
│   UI Updates:                   │
│   ├─ Execution timeline         │
│   ├─ Node outputs              │
│   ├─ Final result              │
│   └─ Error messages            │
└─────────────────────────────────┘
```

## 10. Internationalization (i18n)

### 10.1 i18n Setup

**Reference**: `/web/context/i18n.ts` (lines 1-48)

```typescript
import { createContext, useContext } from 'react'
import i18n from 'i18next'
import { initReactI18next } from 'react-i18next'

export type Locale = 'en-US' | 'zh-Hans' | 'ja-JP' | 'de-DE' | 'fr-FR' // ...

export type II18NContext = {
  locale: Locale
  i18n: Record<string, any>
  setLocaleOnClient: (lang: Locale, reloadPage?: boolean) => Promise<void>
}

const I18NContext = createContext<II18NContext>({} as II18NContext)

export const useI18N = () => useContext(I18NContext)

// Hook for localized documentation links
export const useDocLink = (baseUrl?: string) => {
  const { locale } = useI18N()

  return useMemo(() => {
    const docLocale = locale === 'zh-Hans' ? 'zh-Hans' : 'en'
    return `${baseUrl || 'https://docs.dify.ai'}/${docLocale}`
  }, [locale, baseUrl])
}
```

### 10.2 Translation Files

```
/web/i18n/
├── en-US/                   # English (default)
│   ├── common.ts           # Common strings
│   ├── app.ts              # App-related
│   ├── workflow.ts         # Workflow strings
│   ├── dataset.ts          # Dataset strings
│   ├── tools.ts            # Tool strings
│   ├── billing.ts          # Billing strings
│   └── ...                 # 30+ files
│
├── zh-Hans/                 # Simplified Chinese
├── zh-Hant/                 # Traditional Chinese
├── ja-JP/                   # Japanese
├── de-DE/                   # German
├── fr-FR/                   # French
├── es-ES/                   # Spanish
├── ko-KR/                   # Korean
├── pt-BR/                   # Portuguese (Brazil)
├── ru-RU/                   # Russian
├── it-IT/                   # Italian
├── th-TH/                   # Thai
├── vi-VN/                   # Vietnamese
└── ...                      # 20+ languages
```

## 11. Key Patterns Summary

| Pattern | Location | Purpose |
|---------|----------|---------|
| **Slice Composition** | Zustand stores | Modular state management |
| **Provider Pattern** | React Context | Dependency injection |
| **Query Keys** | React Query | Cache management |
| **Hook Composition** | Custom hooks | Logic reuse |
| **Component Hierarchy** | Base → Feature → Page | Reusability |
| **Service Layer** | `/service/` | API abstraction |
| **SSE Streaming** | Workflow execution | Real-time updates |
| **Lazy Loading** | Dynamic imports | Performance |
| **Memoization** | useMemo, React.memo | Render optimization |

## 12. Build & Deployment

### 12.1 Build Commands

```bash
# Development
pnpm dev               # Start dev server
pnpm lint:fix          # Fix linting issues
pnpm type-check:tsgo   # Type checking

# Production
pnpm build             # Next.js production build
pnpm start             # Start production server

# Testing
pnpm test              # Run Vitest tests
pnpm test:coverage     # Coverage report
```

### 12.2 Docker Deployment

**Reference**: `/web/docker/`

```dockerfile
# Multi-stage build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "server.js"]
```

---

*Quay lại: [README](./README.md)*
