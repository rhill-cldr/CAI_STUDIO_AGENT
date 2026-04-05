# Validation Rules Reference

This chapter is an exhaustive checklist of rules that a validation SDK must enforce when checking a workflow template ZIP. Rules are categorized by severity.

**ERROR** rules block import — Agent Studio will reject the ZIP if these fail.

**WARNING** rules indicate issues that won't prevent import but may cause problems at runtime.

## Structural Rules (ERROR)

| Rule | Description |
|---|---|
| `S-001` | ZIP must contain `workflow_template.json` at the root level (not nested in a subdirectory) |
| `S-002` | `workflow_template.json` must be valid JSON |
| `S-003` | If any tool templates exist, the `studio-data/tool_templates/` directory must exist in the ZIP |
| `S-004` | If any icons are referenced, the `studio-data/dynamic_assets/` directory must exist in the ZIP |

## Manifest Rules (ERROR)

| Rule | Description |
|---|---|
| `M-001` | `template_version` field must be present (currently `"0.0.1"`) |
| `M-002` | `workflow_template` field must be present and be an object |
| `M-003` | `agent_templates` field must be present and be an array |
| `M-004` | `tool_templates` field must be present and be an array |
| `M-005` | `task_templates` field must be present and be an array |
| `M-006` | `mcp_templates` field must be present and be an array (may be empty) |
| `M-007` | `workflow_template.id` must be a non-empty string |
| `M-008` | `workflow_template.name` must be a non-empty string |
| `M-009` | Every element in `agent_templates`, `tool_templates`, `mcp_templates`, `task_templates` must have an `id` field |

## Cross-Reference Integrity Rules (ERROR)

| Rule | Description |
|---|---|
| `X-001` | Every UUID in `workflow_template.agent_template_ids` must match an `agent_templates[].id` |
| `X-002` | Every UUID in `workflow_template.task_template_ids` must match a `task_templates[].id` |
| `X-003` | If `workflow_template.manager_agent_template_id` is set, it must match an `agent_templates[].id` |
| `X-004` | Every UUID in each `agent_templates[].tool_template_ids` must match a `tool_templates[].id` |
| `X-005` | Every UUID in each `agent_templates[].mcp_template_ids` must match a `mcp_templates[].id` |
| `X-006` | If `task_templates[].assigned_agent_template_id` is set, it must match an `agent_templates[].id` |
| `X-007` | No duplicate IDs across all entity arrays (all IDs must be unique within the manifest) |

## Tool Validation Rules (ERROR)

| Rule | Description |
|---|---|
| `T-001` | Every `tool_templates[].source_folder_path` must reference a directory that exists in the ZIP |
| `T-002` | Each tool directory must contain a file matching `python_code_file_name` (typically `tool.py`) |
| `T-003` | Each tool directory must contain a file matching `python_requirements_file_name` (typically `requirements.txt`) |
| `T-004` | `tool.py` must parse as valid Python (no syntax errors) |
| `T-005` | `tool.py` must contain a class named `UserParameters` |
| `T-006` | `tool.py` must contain a class named `ToolParameters` |
| `T-007` | `tool.py` must contain a function named `run_tool` |

## Tool Validation Rules (WARNING)

| Rule | Description |
|---|---|
| `T-W01` | `tool.py` should define `OUTPUT_KEY` at module level |
| `T-W02` | `tool.py` should contain an `if __name__ == "__main__":` block |
| `T-W03` | `requirements.txt` should list `pydantic` as a dependency |
| `T-W04` | `UserParameters` should inherit from `BaseModel` |
| `T-W05` | `ToolParameters` should inherit from `BaseModel` |

## Name Validation Rules (ERROR)

| Rule | Description |
|---|---|
| `N-001` | Every `tool_templates[].name` must match the regex `^[a-zA-Z0-9 ]+$` (alphanumeric and spaces only) |
| `N-002` | Tool template names must be unique within the manifest |

## Icon Validation Rules (ERROR)

| Rule | Description |
|---|---|
| `I-001` | If `tool_image_path` is non-empty, the file must exist in the ZIP |
| `I-002` | If `agent_image_path` is non-empty, the file must exist in the ZIP |
| `I-003` | If `mcp_image_path` is non-empty, the file must exist in the ZIP |
| `I-004` | Icon file extensions (lowercased) must be one of: `.png`, `.jpg`, `.jpeg` |

## Process Mode Rules (WARNING)

| Rule | Description |
|---|---|
| `P-W01` | If `process` is `"hierarchical"`, either `manager_agent_template_id` should be set or `use_default_manager` should be `true` |
| `P-W02` | If `process` is `"sequential"`, all task templates should have `assigned_agent_template_id` set |

## ID Format Rules (WARNING)

| Rule | Description |
|---|---|
| `F-W01` | All `id` fields should be valid UUID format (8-4-4-4-12 hex pattern) |
