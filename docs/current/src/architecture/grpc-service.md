# gRPC Service Design

Agent Studio exposes its backend via a gRPC service defined in Protocol Buffers. For external harnesses, we recommend starting from the [Open Inference Protocol](https://github.com/Arize-ai/openinference) (OIP) as your protobuf foundation and evolving gRPC interfaces to support your harness's specific needs. Agent Studio's own proto (`agent_studio.proto`) is an implementation detail — it is tightly coupled to the Cloudera AI platform and should not be adopted directly.

## Recommended Starting Point: Open Inference Protocol

OIP defines standard protobuf messages for AI inference traces, spans, and evaluations. It aligns with OpenTelemetry semantic conventions and is already used by Agent Studio's telemetry layer (via the `openinference-instrumentation-*` packages).

For a harness that needs to:
- Execute and observe agent workflows → extend OIP trace/span definitions
- Manage templates and deployments → define your own CRUD service RPCs
- Proxy between a frontend and backend → follow the REST-to-gRPC proxy pattern described below

## Agent Studio's Service Architecture (Reference Only)

Agent Studio's gRPC service is organized into domain-specific RPC groups:

| Domain | Key RPCs | Count |
|---|---|---|
| Models | ListModels, AddModel, TestModel, SetStudioDefaultModel | 8 |
| Tool Templates | ListToolTemplates, AddToolTemplate, UpdateToolTemplate | 5 |
| Tool Instances | ListToolInstances, CreateToolInstance, TestToolInstance | 5 |
| MCP Templates | ListMcpTemplates, AddMcpTemplate, UpdateMcpTemplate | 5 |
| MCP Instances | ListMcpInstances, CreateMcpInstance, UpdateMcpInstance | 5 |
| Agents | ListAgents, AddAgent, UpdateAgent | 5 |
| Tasks | ListTasks, AddTask, UpdateTask | 5 |
| Workflows | ListWorkflows, AddWorkflow, CloneWorkflow, TestWorkflow, DeployWorkflow | 12 |
| Workflow Templates | ListWorkflowTemplates, ExportWorkflowTemplate, ImportWorkflowTemplate | 6 |
| Agent/Task Templates | CRUD for reusable agent and task blueprints | 10+ |
| Deployed Workflows | ListDeployedWorkflows, UndeployWorkflow, Suspend/Resume | 5 |
| Utility | HealthCheck, FileUpload/Download, GetAssetData | 9 |

**Total**: 125+ RPCs.

### Service Pattern

The gRPC service (`studio/service.py`) acts as a thin router:

```python
class AgentStudioApp(AgentStudioServicer):
    def __init__(self):
        self.dao = AgentStudioDao()

    def ListWorkflows(self, request, context):
        return list_workflows(request, cml=self.cml, dao=self.dao)

    def ExportWorkflowTemplate(self, request, context):
        return export_workflow_template(request, cml=self.cml, dao=self.dao)
```

Each RPC delegates to a domain function that receives `(request, cml, dao)`. Business logic lives entirely in the domain modules (`studio/agents/`, `studio/tools/`, `studio/workflow/`, etc.).

### Key Programmatic Entry Points

For template management, the most important RPCs are:

- **`ExportWorkflowTemplate(id)`** → Returns a ZIP file path containing the complete template package
- **`ImportWorkflowTemplate(file_path)`** → Imports a ZIP, regenerates UUIDs, copies assets, creates DB records
- **`DeployWorkflow(workflow_id, config)`** → Packages and deploys a workflow to a target endpoint
- **`TestWorkflow(workflow_id, inputs)`** → Executes a workflow on an in-studio runner with full telemetry

## Frontend Proxy Pattern

The Next.js frontend does not speak gRPC directly. Instead, API routes translate HTTP requests:

```d2
direction: right

browser: Browser
nextjs: Next.js API Route {
  label: "Next.js API Route\napp/api/grpc/[slug]/route.ts"
}
backend: Backend Service

browser -> nextjs: HTTP
nextjs -> backend: gRPC
```

The proxy uses slug-based routing — the URL path segment maps to the gRPC method name. This pattern means any REST client can interact with the backend without a gRPC client library.

Additionally, Phoenix traffic is proxied via Next.js rewrites:
```
/api/ops/*  →  http://localhost:50052/*
```

## Recommendations for External Harnesses

1. **Define your own proto** starting from OIP trace definitions. Add CRUD RPCs as needed for your entity model.
2. **Keep the service layer thin** — route to domain modules. This makes the service testable and extensible.
3. **Expose a REST proxy** if your frontend doesn't support gRPC natively. The slug-based pattern is simple and effective.
4. **Use the Import/Export ZIP format** as your interchange format with Agent Studio. The [Template Specification](../templates/concepts.md) chapters document this format exhaustively.
5. **Don't depend on `agent_studio.proto`** — it is versioned for internal use and subject to change without notice.
