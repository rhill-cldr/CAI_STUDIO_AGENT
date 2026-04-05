# Workflow Template ZIP Format

This chapter specifies the exact structure of a workflow template ZIP file — the portable interchange format for Agent Studio templates. SDK builders must validate that generated ZIPs conform to this structure.

## ZIP Structure

```
workflow_template.zip
├── workflow_template.json                              # REQUIRED: manifest
└── studio-data/                                        # REQUIRED if tools or icons exist
    ├── tool_templates/                                 # Tool code packages
    │   ├── {slug}_{random}/                            # One directory per tool template
    │   │   ├── tool.py                                 # REQUIRED per tool
    │   │   ├── requirements.txt                        # REQUIRED per tool
    │   │   └── [additional files...]                   # Optional supporting files
    │   └── ...
    └── dynamic_assets/                                 # Icon images
        ├── tool_template_icons/
        │   └── {slug}_{random}_icon.{png|jpg|jpeg}
        ├── agent_template_icons/
        │   └── {uuid}_icon.{png|jpg|jpeg}
        └── mcp_template_icons/
            └── {slug}_{random}_icon.{png|jpg|jpeg}
```

## Path Requirements

### Root Manifest

The file `workflow_template.json` **must** be at the ZIP root (not inside a subdirectory).

### studio-data/ Directory

The path `studio-data/` is a hard-coded constant in Agent Studio. All file paths stored in the manifest JSON are relative to the ZIP root and rooted under `studio-data/`. During import, the entire `studio-data/` subtree is copied into the project's root `studio-data/` directory.

### Tool Template Directories

Each tool template directory must be located at:
```
studio-data/tool_templates/{directory_name}/
```

The `directory_name` follows the convention `{slugified_name}_{random_string}`:
- `slugified_name`: tool name lowercased, spaces replaced with underscores
- `random_string`: 6-character alphanumeric string for uniqueness

Examples: `json_reader_abc123`, `my_api_tool_xk9f2m`

The `source_folder_path` field in the tool template JSON must match the actual directory path within the ZIP.

### Icon Files

Icon paths in the manifest must point to files that actually exist within the ZIP:

| Template Type | Path Convention |
|---|---|
| Tool | `studio-data/dynamic_assets/tool_template_icons/{slug}_{random}_icon.{ext}` |
| Agent | `studio-data/dynamic_assets/agent_template_icons/{uuid}_icon.{ext}` |
| MCP | `studio-data/dynamic_assets/mcp_template_icons/{slug}_{random}_icon.{ext}` |

If a template has no icon, the corresponding path field should be an empty string `""`.

## Import Behavior

When Agent Studio imports a ZIP, the following transformations occur:

1. **UUID regeneration**: All IDs in the manifest are remapped to fresh UUIDs. Cross-references are updated to match.
2. **Directory renaming**: Tool template directories are renamed with new slugs and random suffixes.
3. **Icon renaming**: Icon files are renamed to match the new directory names or UUIDs.
4. **Path updates**: All `source_folder_path`, `tool_image_path`, `agent_image_path`, and `mcp_image_path` fields are updated to reflect the new names.
5. **File copy**: The `studio-data/` subtree is merged into the project's existing `studio-data/` directory.
6. **Database insert**: All templates are inserted into the SQLite database with new IDs.
7. **MCP validation**: Background jobs start MCP servers and populate their `tools` fields.

Because of step 2-4, the specific directory names and icon filenames in your ZIP are not preserved. They only need to be valid and internally consistent at the time of import.

## Generating ZIPs Programmatically

When building a ZIP outside of Agent Studio (e.g., in a CI pipeline):

1. Generate UUIDv4 strings for all template IDs
2. Ensure all cross-references are consistent
3. Place tool code under `studio-data/tool_templates/{slug}_{random}/`
4. Place icons under the appropriate `studio-data/dynamic_assets/` subdirectory
5. Write `workflow_template.json` at the ZIP root with correct path references
6. Create the ZIP archive with standard deflate compression

The ZIP should not contain:
- `.venv/` directories
- `__pycache__/` directories
- `.requirements_hash.txt` files
- Any files outside of `workflow_template.json` and `studio-data/`
