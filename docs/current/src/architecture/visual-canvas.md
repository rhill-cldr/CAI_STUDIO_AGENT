# Visual Canvas (XYFlow)

Agent Studio renders workflows as interactive directed graphs using [XYFlow](https://reactflow.dev/) (`@xyflow/react`, formerly ReactFlow). An external harness that aims to provide a similar visual development experience should implement a compatible node/edge model.

## Node Types

The canvas uses four node types, each with distinct visual styling and behavior:

| Node Type | Visual | Purpose |
|---|---|---|
| `task` | Light green | A CrewAI task — displays truncated description. Connected left-to-right in execution order. |
| `agent` | White/light blue | A CrewAI agent — displays name and optional icon. Pulsing animation when active during execution. |
| `tool` | Dark gray | A tool instance attached to an agent — displays name and optional icon. |
| `mcp` | Dark gray | An MCP server instance attached to an agent — displays name, icon, and tool list. |

## Data Model

The diagram state is defined as:

```typescript
interface DiagramState {
  nodes: Node[];     // XYFlow Node objects with type, position, data
  edges: Edge[];     // XYFlow Edge objects with source, target, handles
  hasCustomPositions?: boolean;  // Preserves user drag-to-reposition
}
```

Each node's `data` field carries entity-specific metadata:

```typescript
// Task node data
{ label, name, taskId, taskData, isConversational }

// Agent node data
{ label, name, iconData, agentId, agentData, manager?, isDefaultManager? }

// Tool node data
{ label, name, iconData, workflowId, toolInstanceId, agentId, agentTools }

// MCP node data
{ name, iconData, active, toolList, activeTool, mcpInstanceId, agentId, workflowId }
```

## Layout Algorithm

The layout is deterministic and derived from workflow structure:

```d2
direction: down

tasks: y=0 Task Row {
  direction: right
  task1: Task 1
  task2: Task 2
  task3: Task 3
  task1 -> task2
  task2 -> task3
}

manager: Manager Agent {
  label: "Manager Agent\n(hierarchical mode only)"
  style.stroke-dash: 3
}

agents: y=150 Agent Row {
  direction: right
  agentA: Agent A
  agentB: Agent B
}

toolsA: Agent A Tools {
  direction: right
  tool1: Tool 1
  tool2: Tool 2
}

toolsB: Agent B Tools {
  mcp1: MCP Server 1
}

tasks -> manager
manager -> agents
agents.agentA -> toolsA
agents.agentB -> toolsB
```

**Sequential mode**: Tasks connect directly down to their assigned agents.

**Hierarchical mode**: Tasks connect to a manager agent node, which connects down to worker agents.

Agent x-positions are calculated to center the group:
1. Sum total width: each agent contributes `220 * max(0, num_tools + num_mcps - 1) + 220`
2. Start offset: `-0.5 * totalWidth + 110`
3. Each tool/MCP shifts the offset by `+220`

## Edge Types

All edges use `MarkerType.Arrow` with 20x20 size.

| Edge | Source Handle | Target Handle | Condition |
|---|---|---|---|
| Task → Task | `right` | `left` | Sequential (adjacent tasks) |
| Task → Agent | `bottom` | `top` | Sequential mode |
| Task → Manager | `bottom` | (default) | Hierarchical mode |
| Manager → Agent | (default) | (default) | Hierarchical mode |
| Agent → Tool | (default) | (default) | Always |
| Agent → MCP | (default) | (default) | Always |

## Live Event Overlay

During workflow execution, the canvas receives events from the telemetry pipeline and overlays them onto nodes:

1. Events fetched from `/api/workflow/events?trace_id={id}`
2. `processEvents()` extracts active node IDs and event metadata
3. Active nodes receive: `active: true` (triggers CSS pulse animation), `info` (event text), `infoType` (event category)

Event info types displayed on nodes:
- `TaskStart`, `LLMCall`, `ToolInput`, `ToolOutput`
- `Delegate`, `EndDelegate`, `AskCoworker`, `EndAskCoworker`
- `Completion`, `FailedCompletion`

## Key Takeaway for Harness Builders

To build an equivalent canvas:
1. Parse the `workflow_template.json` or `CollatedInput` to extract the entity graph
2. Apply the layout algorithm above (or use XYFlow's auto-layout plugins)
3. Subscribe to the event stream to overlay live execution state
4. Use the `DiagramStateInput` interface as your data contract — it requires: workflow state, icon data map, tasks, tool instances, MCP instances, tool templates, and agents
