# Dynamic Assets and Icons

Icons are optional visual assets for tool, agent, and MCP templates. They are displayed in the Agent Studio UI and on the XYFlow visual canvas.

## Supported Formats

| Format | Extensions |
|---|---|
| PNG | `.png` |
| JPEG | `.jpg`, `.jpeg` |

Extensions are validated case-insensitively (e.g., `.PNG` and `.png` are both accepted, but stored lowercased).

No other image formats are supported. SVG, GIF, WebP, and BMP will be rejected.

## Storage Paths

Icons are stored under `studio-data/dynamic_assets/` with type-specific subdirectories:

```
studio-data/dynamic_assets/
├── tool_template_icons/
│   └── {slug}_{random}_icon.{ext}         e.g., json_reader_abc123_icon.png
├── agent_template_icons/
│   └── {uuid}_icon.{ext}                  e.g., a1b2c3d4-..._icon.png
└── mcp_template_icons/
    └── {slug}_{random}_icon.{ext}         e.g., filesystem_server_xyz789_icon.jpg
```

### Naming Conventions

| Template Type | Icon Filename Pattern |
|---|---|
| Tool | `{tool_directory_basename}_icon.{ext}` — matches the tool directory name |
| Agent | `{agent_template_uuid}_icon.{ext}` — uses the agent template ID |
| MCP | `{unique_slug}_{random}_icon.{ext}` — generated slug similar to tools |

## Manifest References

Each template type has a path field that references its icon:

| Template Type | JSON Field | Example Value |
|---|---|---|
| Tool | `tool_image_path` | `"studio-data/dynamic_assets/tool_template_icons/json_reader_abc123_icon.png"` |
| Agent | `agent_image_path` | `"studio-data/dynamic_assets/agent_template_icons/a1b2c3d4-..._icon.png"` |
| MCP | `mcp_image_path` | `"studio-data/dynamic_assets/mcp_template_icons/fs_server_xyz789_icon.png"` |

If a template has no icon, the path field should be an empty string `""`. Do not use `null` — use `""`.

## Validation Rules

For SDK validators:

1. If the path field is non-empty, the file **must** exist at that path within the ZIP
2. The file extension (lowercased) must be one of: `.png`, `.jpg`, `.jpeg`
3. The path must be under the appropriate `studio-data/dynamic_assets/` subdirectory
4. File content is not validated beyond existence — no dimension, size, or format header checks
