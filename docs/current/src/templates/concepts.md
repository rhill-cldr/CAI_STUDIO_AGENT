# Template System Concepts

## Templates vs Instances

Agent Studio separates **templates** (reusable blueprints) from **instances** (live copies within a workflow):

- A **tool template** is a code package in the template catalog. A **tool instance** is a copy of that template added to a specific workflow, with its own virtualenv and configuration.
- An **agent template** defines role/backstory/goal and tool references. An **agent instance** (just called "agent") is a copy within a workflow, bound to specific tool instances and an LLM model.
- **Task templates** and **task instances** follow the same pattern.

Templates are the unit of portability. When you export a workflow template as a ZIP, the export captures all nested agent, tool, task, and MCP templates. When you import that ZIP into another Agent Studio instance, new templates are created, and a workflow can be instantiated from them.

## Entity Graph

A workflow template aggregates all other template types through ID references:

```d2
WorkflowTemplate -> AgentTemplate: agent_template_ids
AgentTemplate -> ToolTemplate: tool_template_ids
AgentTemplate -> MCPTemplate: mcp_template_ids
WorkflowTemplate -> TaskTemplate: task_template_ids
TaskTemplate -> AgentTemplate: assigned_agent_template_id {style.stroke-dash: 3}
WorkflowTemplate -> AgentTemplate: manager_agent_template_id (optional) {style.stroke-dash: 3}
```

All references are by UUID. The `workflow_template.json` manifest in a ZIP must maintain referential integrity — every ID referenced must correspond to an entry in the appropriate array.

## ID Management

All IDs are UUIDv4 strings. Two critical rules:

1. **On export**: All IDs are regenerated. The exported ZIP contains fresh UUIDs that differ from the database.
2. **On import**: All IDs are regenerated again. The UUIDs in the ZIP are never used as-is in the target database.

This means IDs in a template ZIP are **ephemeral** — they exist only to establish cross-references between entities within the ZIP. An SDK validator must check that these cross-references are consistent, but the actual UUID values are arbitrary.

## Scoping

In the Agent Studio database, templates can be either:

- **Global**: `workflow_template_id` is null. Available to all workflows.
- **Scoped**: `workflow_template_id` is set. Tied to a specific workflow template.

In exported ZIPs, all templates are scoped — every tool, agent, task, and MCP template has its `workflow_template_id` set to the workflow template's ID. This ensures the import creates a self-contained set of templates.

## Process Modes

Workflows support two execution modes, reflected in the `process` field:

| Mode | Task Assignment | Manager Agent |
|---|---|---|
| `sequential` | Each task has an `assigned_agent_template_id`. Tasks execute in array order. | Not used. |
| `hierarchical` | Manager agent decides which worker agent handles each task. | Required — either `manager_agent_template_id` is set, or `use_default_manager` is true. |
