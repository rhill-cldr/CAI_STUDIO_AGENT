# CollatedInput Schema

The `CollatedInput` is the universal runtime contract between Agent Studio and the workflow engine. It fully describes a workflow's execution environment — language models, agents, tasks, tools, and MCP servers.

## Top-Level Structure

```python
class CollatedInput(BaseModel):
    default_language_model_id: str
    language_models: List[Input__LanguageModel]
    tool_instances: List[Input__ToolInstance]
    mcp_instances: List[Input__MCPInstance]
    agents: List[Input__Agent]
    tasks: List[Input__Task]
    workflow: Input__Workflow
```

## Language Models

```json
{
  "default_language_model_id": "model-uuid",
  "language_models": [
    {
      "model_id": "model-uuid",
      "model_name": "gpt-4o",
      "generation_config": {
        "do_sample": true,
        "temperature": 0.1,
        "max_new_tokens": 4096,
        "top_p": 1,
        "top_k": 50,
        "num_beams": 1,
        "max_length": null
      }
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `default_language_model_id` | string | ID of the model to use when an agent doesn't specify one |
| `language_models[].model_id` | string | Unique identifier |
| `language_models[].model_name` | string | Human-readable name |
| `language_models[].generation_config` | object | LLM generation parameters |

## Tool Instances

```json
{
  "tool_instances": [
    {
      "id": "tool-uuid",
      "name": "JSON Reader",
      "python_code_file_name": "tool.py",
      "python_requirements_file_name": "requirements.txt",
      "source_folder_path": "studio-data/workflows/my_workflow/tools/json_reader_abc123",
      "tool_metadata": "{\"user_params\": [\"api_key\"], \"user_params_metadata\": {\"api_key\": {\"required\": true}}}",
      "tool_image_uri": "tool_template_icons/json_reader_abc123_icon.png",
      "is_venv_tool": true
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique tool instance identifier |
| `name` | string | Tool name |
| `python_code_file_name` | string | Always `"tool.py"` |
| `python_requirements_file_name` | string | Always `"requirements.txt"` |
| `source_folder_path` | string | Path to tool code directory within the artifact |
| `tool_metadata` | string | JSON-encoded metadata including user parameter definitions |
| `tool_image_uri` | string or null | Relative path to icon |
| `is_venv_tool` | boolean | Whether the tool uses a virtual environment |

## MCP Instances

```json
{
  "mcp_instances": [
    {
      "id": "mcp-uuid",
      "name": "Filesystem Server",
      "type": "NODE",
      "args": ["npx", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env_names": [],
      "tools": ["read_file", "write_file", "list_directory"],
      "mcp_image_uri": null
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique MCP instance identifier |
| `name` | string | Server name |
| `type` | string | `"PYTHON"` or `"NODE"` |
| `args` | array of strings | Server startup arguments |
| `env_names` | array of strings | Required environment variable names |
| `tools` | array of strings or null | Available tool names (populated at runtime) |
| `mcp_image_uri` | string or null | Relative path to icon |

## Agents

```json
{
  "agents": [
    {
      "id": "agent-uuid",
      "name": "Research Analyst",
      "llm_provider_model_id": "model-uuid",
      "crew_ai_role": "Senior Research Analyst",
      "crew_ai_backstory": "You are an experienced analyst...",
      "crew_ai_goal": "Produce accurate research reports",
      "crew_ai_allow_delegation": true,
      "crew_ai_verbose": true,
      "crew_ai_cache": true,
      "crew_ai_temperature": 0.7,
      "crew_ai_max_iter": 10,
      "tool_instance_ids": ["tool-uuid"],
      "mcp_instance_ids": [],
      "agent_image_uri": ""
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique agent identifier |
| `name` | string | Agent name |
| `llm_provider_model_id` | string or null | Override LLM for this agent (uses default if null) |
| `crew_ai_role` | string | Agent's role description |
| `crew_ai_backstory` | string | Agent's background context |
| `crew_ai_goal` | string | Agent's primary objective |
| `crew_ai_allow_delegation` | boolean | Can delegate to other agents |
| `crew_ai_verbose` | boolean | Detailed logging |
| `crew_ai_cache` | boolean | Cache LLM responses |
| `crew_ai_temperature` | float or null | Sampling temperature |
| `crew_ai_max_iter` | integer or null | Maximum reasoning iterations |
| `tool_instance_ids` | array of strings | IDs of tools this agent can use |
| `mcp_instance_ids` | array of strings | IDs of MCP servers this agent can use |
| `agent_image_uri` | string | Relative path to icon |

## Tasks

```json
{
  "tasks": [
    {
      "id": "task-uuid",
      "description": "Analyze the quarterly sales data and identify trends.",
      "expected_output": "A structured report with trends and recommendations.",
      "assigned_agent_id": "agent-uuid"
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique task identifier |
| `description` | string | Task instructions for the agent |
| `expected_output` | string | What the agent should produce |
| `assigned_agent_id` | string or null | Agent assigned to this task (sequential mode) |

## Workflow

```json
{
  "workflow": {
    "id": "workflow-uuid",
    "name": "Sales Analysis Workflow",
    "description": "Analyzes quarterly sales data",
    "crew_ai_process": "sequential",
    "agent_ids": ["agent-uuid"],
    "task_ids": ["task-uuid"],
    "manager_agent_id": null,
    "llm_provider_model_id": null,
    "is_conversational": false,
    "created_at": "2025-01-15T10:30:00"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique workflow identifier |
| `name` | string | Workflow name |
| `description` | string or null | Workflow description |
| `crew_ai_process` | string | `"sequential"` or `"hierarchical"` |
| `agent_ids` | array of strings | Participating agent IDs |
| `task_ids` | array of strings | Task IDs in execution order |
| `manager_agent_id` | string or null | Manager agent for hierarchical mode |
| `llm_provider_model_id` | string or null | Default LLM override for the workflow |
| `is_conversational` | boolean | Multi-turn conversation support |
| `created_at` | string (ISO 8601) or null | Creation timestamp |
