# MCP Template Specification

An MCP (Model Context Protocol) template defines a configuration for an MCP server that agents can use to access external tools and resources. Agent Studio supports running MCP servers as child processes during workflow execution.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string (UUID) | Yes | — | Unique identifier |
| `workflow_template_id` | string (UUID) | No | null | Scopes the template to a workflow template |
| `name` | string | Yes | — | Human-readable server name |
| `type` | string | Yes | — | Server type: `"PYTHON"` or `"NODE"` |
| `args` | array of strings | Yes | — | Command-line arguments for starting the server |
| `env_names` | array of strings | Yes | — | Environment variable names the server requires |
| `tools` | object or null | No | null | MCP-exposed tool definitions. **Set to null on export.** Populated during import validation. |
| `status` | string | No | "" | Validation status. **Set to empty string on export.** |
| `mcp_image_path` | string | No | "" | Relative path to the server's icon within the ZIP |

## Server Types

| Type | Runtime | Example |
|---|---|---|
| `PYTHON` | Executed via Python | `python -m my_mcp_server` |
| `NODE` | Executed via Node.js | `npx @modelcontextprotocol/server-filesystem` |

## Export/Import Behavior

Two fields receive special treatment during the export/import cycle:

- **`tools`**: Set to `null` on export. During import, Agent Studio starts the MCP server, queries it for available tools, and populates this field. SDK validators should accept `null` here.
- **`status`**: Set to empty string `""` on export. During import, it transitions through `"VALIDATING"` → `"VALID"` or `"VALIDATION_FAILED"`. SDK validators should accept empty string or any status value.

## Cross-References

- Every MCP template `id` referenced in an agent template's `mcp_template_ids` must correspond to an entry in the ZIP's `mcp_templates` array
- The `mcp_templates` array in the manifest may be empty or absent in older exports

## JSON Example

```json
{
  "id": "m1234567-89ab-cdef-0123-456789abcdef",
  "workflow_template_id": "w9x8y7z6-5432-1098-fedc-ba0987654321",
  "name": "Filesystem Server",
  "type": "NODE",
  "args": ["npx", "@modelcontextprotocol/server-filesystem", "/workspace"],
  "env_names": [],
  "tools": null,
  "status": "",
  "mcp_image_path": "studio-data/dynamic_assets/mcp_template_icons/filesystem_server_abc123_icon.png"
}
```
