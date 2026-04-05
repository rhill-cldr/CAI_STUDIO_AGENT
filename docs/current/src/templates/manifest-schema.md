# workflow_template.json Schema

The manifest file is the heart of a workflow template ZIP. It contains the complete definition of every template entity and their cross-references. This chapter documents every field.

## Top-Level Structure

```json
{
  "template_version": "0.0.1",
  "workflow_template": { ... },
  "agent_templates": [ ... ],
  "tool_templates": [ ... ],
  "mcp_templates": [ ... ],
  "task_templates": [ ... ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `template_version` | string | Yes | Schema version. Currently `"0.0.1"`. |
| `workflow_template` | object | Yes | The workflow template definition. |
| `agent_templates` | array | Yes | All agent templates referenced by the workflow. |
| `tool_templates` | array | Yes | All tool templates referenced by agents. |
| `mcp_templates` | array | Yes | All MCP templates referenced by agents. May be empty. |
| `task_templates` | array | Yes | All task templates referenced by the workflow. |

## workflow_template Object

```json
{
  "id": "uuid",
  "name": "My Workflow",
  "description": "A workflow that does X",
  "process": "sequential",
  "agent_template_ids": ["uuid-1", "uuid-2"],
  "task_template_ids": ["uuid-3", "uuid-4"],
  "manager_agent_template_id": null,
  "use_default_manager": false,
  "is_conversational": false,
  "pre_packaged": false
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string (UUID) | Yes | — | Unique identifier |
| `name` | string | Yes | — | Workflow name |
| `description` | string | No | null | Workflow description |
| `process` | string | No | null | `"sequential"` or `"hierarchical"` |
| `agent_template_ids` | array of UUIDs | No | null | Ordered list of agent template IDs |
| `task_template_ids` | array of UUIDs | No | null | Ordered list of task template IDs (defines execution order) |
| `manager_agent_template_id` | string (UUID) | No | null | Manager agent for hierarchical mode |
| `use_default_manager` | boolean | No | false | Use a default manager instead of a custom one |
| `is_conversational` | boolean | No | false | Whether the workflow supports multi-turn conversation |
| `pre_packaged` | boolean | No | false | Always `false` in exports |

## agent_templates Array

Each element:

```json
{
  "id": "uuid",
  "workflow_template_id": "uuid",
  "name": "Agent Name",
  "description": "What this agent does",
  "role": "Senior Analyst",
  "backstory": "Background context...",
  "goal": "Produce accurate analysis",
  "allow_delegation": true,
  "verbose": true,
  "cache": true,
  "temperature": 0.7,
  "max_iter": 10,
  "tool_template_ids": ["uuid-5"],
  "mcp_template_ids": [],
  "pre_packaged": false,
  "agent_image_path": "studio-data/dynamic_assets/agent_template_icons/uuid_icon.png"
}
```

See [Agent Template Specification](./agent-spec.md) for full field documentation.

## tool_templates Array

Each element:

```json
{
  "id": "uuid",
  "workflow_template_id": "uuid",
  "name": "JSON Reader",
  "python_code_file_name": "tool.py",
  "python_requirements_file_name": "requirements.txt",
  "source_folder_path": "studio-data/tool_templates/json_reader_abc123",
  "pre_built": false,
  "tool_image_path": "studio-data/dynamic_assets/tool_template_icons/json_reader_abc123_icon.png",
  "is_venv_tool": true
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string (UUID) | Yes | — | Unique identifier |
| `workflow_template_id` | string (UUID) | No | null | Scoping |
| `name` | string | Yes | — | Tool name (must match `^[a-zA-Z0-9 ]+$`) |
| `python_code_file_name` | string | Yes | — | Always `"tool.py"` |
| `python_requirements_file_name` | string | Yes | — | Always `"requirements.txt"` |
| `source_folder_path` | string | Yes | — | Path to tool directory within the ZIP |
| `pre_built` | boolean | No | false | Always `false` in exports |
| `tool_image_path` | string | No | "" | Path to icon within the ZIP |
| `is_venv_tool` | boolean | No | true | Whether the tool uses a virtual environment. Always `true` for new tools. |

## mcp_templates Array

Each element:

```json
{
  "id": "uuid",
  "workflow_template_id": "uuid",
  "name": "Filesystem Server",
  "type": "NODE",
  "args": ["npx", "@modelcontextprotocol/server-filesystem", "/workspace"],
  "env_names": [],
  "tools": null,
  "status": "",
  "mcp_image_path": ""
}
```

See [MCP Template Specification](./mcp-spec.md) for full field documentation.

## task_templates Array

Each element:

```json
{
  "id": "uuid",
  "workflow_template_id": "uuid",
  "name": "Analyze Data",
  "description": "Review the dataset and identify key trends.",
  "expected_output": "A structured report with findings.",
  "assigned_agent_template_id": "uuid"
}
```

See [Task Template Specification](./task-spec.md) for full field documentation.

## Cross-Reference Integrity Rules

An SDK validator **must** enforce these rules:

1. Every UUID in `workflow_template.agent_template_ids` must exist in `agent_templates[].id`
2. Every UUID in `workflow_template.task_template_ids` must exist in `task_templates[].id`
3. If `workflow_template.manager_agent_template_id` is set, it must exist in `agent_templates[].id`
4. Every UUID in each `agent_templates[].tool_template_ids` must exist in `tool_templates[].id`
5. Every UUID in each `agent_templates[].mcp_template_ids` must exist in `mcp_templates[].id`
6. If `task_templates[].assigned_agent_template_id` is set, it must exist in `agent_templates[].id`
7. Every `tool_templates[].source_folder_path` must reference a directory that exists in the ZIP
8. Every non-empty `*_image_path` field must reference a file that exists in the ZIP
