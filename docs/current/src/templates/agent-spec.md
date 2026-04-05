# Agent Template Specification

An agent template defines a reusable AI agent with a persona (role, backstory, goal) and references to the tools and MCP servers it can use. At runtime, agent templates map to [CrewAI Agent](https://docs.crewai.com/) instances.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string (UUID) | Yes | — | Unique identifier |
| `workflow_template_id` | string (UUID) | No | null | Scopes the template to a workflow template. Set in exported ZIPs. |
| `name` | string | Yes | — | Human-readable agent name |
| `description` | string | No | null | Description of the agent's purpose |
| `role` | string | No | null | CrewAI role — what the agent does (e.g., "Senior Data Analyst") |
| `backstory` | string | No | null | CrewAI backstory — context that shapes agent behavior |
| `goal` | string | No | null | CrewAI goal — what the agent is trying to achieve |
| `allow_delegation` | boolean | No | true | Whether this agent can delegate tasks to other agents |
| `verbose` | boolean | No | true | Whether to log detailed execution info |
| `cache` | boolean | No | true | Whether to cache LLM responses |
| `temperature` | float | No | 0.7 | LLM sampling temperature |
| `max_iter` | integer | No | 10 | Maximum reasoning iterations before the agent must produce output |
| `tool_template_ids` | array of UUIDs | No | [] | References to tool templates this agent can use |
| `mcp_template_ids` | array of UUIDs | No | [] | References to MCP templates this agent can use |
| `pre_packaged` | boolean | No | false | Whether shipped as part of the studio. Always `false` in exports. |
| `agent_image_path` | string | No | "" | Relative path to the agent's icon within the ZIP |

## CrewAI Mapping

When instantiated, agent template fields map to CrewAI's `Agent` constructor:

| Template Field | CrewAI Parameter |
|---|---|
| `role` | `role` |
| `backstory` | `backstory` |
| `goal` | `goal` |
| `allow_delegation` | `allow_delegation` |
| `verbose` | `verbose` |
| `cache` | `cache` |
| `temperature` | `temperature` |
| `max_iter` | `max_iter` |

## Cross-References

- Every UUID in `tool_template_ids` must correspond to an entry in the ZIP's `tool_templates` array.
- Every UUID in `mcp_template_ids` must correspond to an entry in the ZIP's `mcp_templates` array.
- The agent's own `id` may be referenced by:
  - `workflow_template.agent_template_ids`
  - `workflow_template.manager_agent_template_id`
  - `task_template.assigned_agent_template_id`

## JSON Example

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "workflow_template_id": "w9x8y7z6-5432-1098-fedc-ba0987654321",
  "name": "Research Analyst",
  "description": "Analyzes data and produces research reports",
  "role": "Senior Research Analyst",
  "backstory": "You are an experienced analyst with expertise in data interpretation.",
  "goal": "Produce thorough, accurate research reports based on available data.",
  "allow_delegation": true,
  "verbose": true,
  "cache": true,
  "temperature": 0.7,
  "max_iter": 10,
  "tool_template_ids": ["t1234567-..."],
  "mcp_template_ids": [],
  "pre_packaged": false,
  "agent_image_path": "studio-data/dynamic_assets/agent_template_icons/a1b2c3d4-e5f6-7890-abcd-ef1234567890_icon.png"
}
```
