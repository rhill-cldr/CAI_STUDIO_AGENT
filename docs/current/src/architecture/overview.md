# System Overview

Agent Studio is composed of four primary subsystems that communicate over well-defined boundaries. Understanding this topology is essential for building an external harness that mirrors the same development experience.

## Component Topology

```d2
frontend: Next.js Frontend (port 8090) {
  editor: Workflow Editor UI
  redux: Redux Store
  canvas: XYFlow Canvas

  api: Next.js API Routes {
    label: "Next.js API Routes\napp/api/grpc/[slug]/"
  }

  editor -> api
}

backend: gRPC Backend (port 50051) {
  service: service.py {
    label: "service.py — thin router"
  }
  agents: studio/agents/
  tools: studio/tools/
  workflow: studio/workflow/
  models: studio/models/
  mcp: studio/as_mcp/
  deployments: studio/deployments/

  service -> agents
  service -> tools
  service -> workflow
  service -> models
  service -> mcp
  service -> deployments

  db: SQLite (SQLAlchemy ORM) {
    shape: cylinder
    label: "SQLite\n.app/state.db"
  }

  service -> db
}

phoenix: Phoenix Observability (port 50052) {
  label: "Phoenix Observability (port 50052)\nOTel trace collector + event stream broker\nEndpoints: /v1/traces, /events"
}

runners: Workflow Engine Runners (ports 51000+) {
  label: "Workflow Engine Runners (ports 51000+)\nFastAPI/Uvicorn sidecars\nCrewAI and LangGraph execution"
}

frontend.api -> backend.service: gRPC
runners -> phoenix: OTel traces + events
frontend.api -> phoenix: /api/ops/* proxy
```

## Entity Hierarchy

The core data model has two parallel hierarchies — **instances** (live workflow state) and **templates** (portable blueprints):

```d2
direction: right

instances: Instances (within a Workflow) {
  Workflow -> Agent
  Agent -> ToolInstance
  Agent -> MCPInstance
  Workflow -> Task
}

templates: Templates (portable) {
  WorkflowTemplate -> AgentTemplate
  AgentTemplate -> ToolTemplate
  AgentTemplate -> MCPTemplate
  WorkflowTemplate -> TaskTemplate
}

instances -> templates: export / import {style.stroke-dash: 3}
```

Templates are the packaging unit. When a workflow template is imported, Agent Studio creates instances from the templates. When exported, instances are snapshot back into templates.

## Data Flow

```d2
direction: right

zip: Template ZIP {shape: document}
db_templates: DB (templates) {shape: cylinder}
db_instances: DB (instances) {shape: cylinder}
artifact: Deployment Artifact {
  shape: document
  label: "Deployment Artifact\n(tar.gz)"
}
engine: Workflow Engine {
  label: "Workflow Engine\n(CollatedInput)"
}

zip -> db_templates: import
db_templates -> db_instances: instantiate
db_instances -> artifact: test / deploy
artifact -> engine
```

## File-Based Storage: `studio-data/`

Alongside the SQLite database, Agent Studio uses a filesystem tree for code and assets:

```text
studio-data/
├── tool_templates/              Tool code packages (tool.py + requirements.txt)
│   ├── sample_tool/
│   ├── json_reader_abc123/
│   └── calculator_def456/
├── dynamic_assets/              Icons for templates and instances
│   ├── tool_template_icons/
│   ├── tool_instance_icons/
│   ├── agent_template_icons/
│   ├── agent_icons/
│   ├── mcp_template_icons/
│   └── mcp_instance_icons/
├── workflows/                   Per-workflow working directories
├── deployable_workflows/        Staged deployment artifacts
├── deployable_applications/     Staged application artifacts
└── temp_files/                  Transient upload/export scratch space
```

The `studio-data/` path is a hard-coded constant (`consts.ALL_STUDIO_DATA_LOCATION = "studio-data"`). All file references stored in the database (e.g., `source_folder_path`, `tool_image_path`) are relative to the project root and rooted under this directory.

## Design Principles for Harness Builders

1. **Proto-first API**: All backend operations are defined in protobuf. The gRPC service is the single source of truth for capabilities.
2. **Thin service layer**: `service.py` is a router — business logic lives in domain modules. This pattern is easy to replicate.
3. **Dual storage**: SQLite for metadata, filesystem for code artifacts. Template ZIPs bridge both by bundling the manifest JSON and the `studio-data/` subtree.
4. **Isolated execution**: The workflow engine is a separate package with its own virtualenv. This ensures tool dependencies don't conflict with the studio itself.
5. **Observable by default**: Every workflow execution produces OTel traces and a structured event stream, enabling the visual canvas to show real-time progress.
