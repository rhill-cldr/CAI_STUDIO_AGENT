# Task Template Specification

A task template defines a unit of work to be executed by an agent. Tasks are the fundamental building blocks of workflow execution — they define what needs to be done and what output is expected.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string (UUID) | Yes | — | Unique identifier |
| `workflow_template_id` | string (UUID) | No | null | Scopes the template to a workflow template |
| `name` | string | No | null | Human-readable task name |
| `description` | string | No | null | What the agent should do. This is the primary instruction given to the assigned agent. |
| `expected_output` | string | No | null | Description of the expected result format. Guides the agent on what to produce. |
| `assigned_agent_template_id` | string (UUID) | No | null | The agent template responsible for executing this task |

## Execution Semantics

### Sequential Mode

When `workflow_template.process` is `"sequential"`:

- Tasks execute in the order they appear in `workflow_template.task_template_ids`
- Each task **should** have `assigned_agent_template_id` set, pointing to the agent that will execute it
- Output from one task may be available as context for subsequent tasks

### Hierarchical Mode

When `workflow_template.process` is `"hierarchical"`:

- A manager agent (specified by `manager_agent_template_id` or `use_default_manager`) orchestrates execution
- The manager decides which worker agent handles each task
- `assigned_agent_template_id` is optional — the manager may override assignments

## Cross-References

- `assigned_agent_template_id`, if set, must correspond to an entry in the ZIP's `agent_templates` array
- The task's `id` must be listed in `workflow_template.task_template_ids`
- Task order in `task_template_ids` defines execution order for sequential workflows

## JSON Example

```json
{
  "id": "t1234567-89ab-cdef-0123-456789abcdef",
  "workflow_template_id": "w9x8y7z6-5432-1098-fedc-ba0987654321",
  "name": "Analyze Sales Data",
  "description": "Review the quarterly sales data and identify trends, anomalies, and key insights.",
  "expected_output": "A structured report with sections for trends, anomalies, and actionable recommendations.",
  "assigned_agent_template_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```
