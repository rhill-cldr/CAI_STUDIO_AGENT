# Introduction

Agent Studio is a full-stack platform for building, testing, and deploying AI agent workflows. It provides a visual workflow builder backed by a gRPC API, a workflow engine supporting CrewAI and LangGraph frameworks, and an embedded observability layer powered by Arize Phoenix. Workflows are composed of agents, tasks, tools, and MCP servers — packaged as portable templates that can be imported, exported, and deployed as production endpoints.

This guide serves two audiences:

| If you are... | Start here |
|---|---|
| **Building an agent harness** for rapid template development (visual canvas, telemetry, gRPC) | [Architecture Reference](./architecture/overview.md) |
| **Building a validation SDK** to ensure template ZIPs conform to the expected format | [Template Specification](./templates/concepts.md) and [Validation Rules](./validation/rules.md) |

## Terminology

| Term | Definition |
|---|---|
| **Workflow Template** | A portable, reusable blueprint for an entire workflow — aggregates agent, task, tool, and MCP templates. Exported/imported as ZIP files. |
| **Tool Template** | A Python package (tool.py + requirements.txt) that agents can invoke at runtime. Defines configuration parameters and invocation arguments via Pydantic models. |
| **Agent Template** | A reusable agent definition with role, backstory, goal, and references to its tools and MCP servers. Maps to a CrewAI Agent. |
| **Task Template** | A unit of work assigned to an agent, with a description and expected output. Execution order is defined by position in the workflow's task list. |
| **MCP Template** | A Model Context Protocol server configuration — type (Python or Node), startup arguments, and required environment variables. |
| **CollatedInput** | The runtime data structure that fully describes a workflow for execution — language models, agents, tasks, tools, MCP servers, and the workflow itself. |
| **Deployment Artifact** | A tar.gz archive containing a `workflow.yaml`, `collated_input.json`, and all required tool code. Consumed by the workflow engine. |
| **Workflow Engine** | A separate Python package (FastAPI/Uvicorn) that executes workflows. Runs as sidecar processes on ports 51000+. |

## Template Lifecycle

```d2
direction: right

author: Author Template {
  shape: document
  label: "Author Template\n(ZIP file)"
}
validate: Validate in CI {
  label: "Validate in CI\n(SDK / GitHub Action)"
}
import: Import to Studio {
  label: "Import to Studio\n(ImportWorkflowTemplate RPC)"
}
instantiate: Instantiate Workflow
test: Test in Studio {
  label: "Test in Studio\n(visual canvas + telemetry)"
}
deploy: Deploy to Production {
  label: "Deploy to Production\n(tar.gz artifact → engine runner)"
}

author -> validate
validate -> import
import -> instantiate
instantiate -> test
test -> deploy
```

The upstream SDK's responsibility ends at validation — ensuring the ZIP conforms to the structure documented in the [Template Specification](./templates/concepts.md) chapters. Agent Studio handles instantiation, testing, and deployment.
