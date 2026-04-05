# Deployment Artifact Format (tar.gz)

When a workflow is deployed, Agent Studio packages it into a tar.gz archive that the workflow engine can execute independently. This format differs from the template ZIP — it contains runtime-resolved data rather than template blueprints.

## Archive Structure

```
artifact.tar.gz
├── workflow.yaml                           # REQUIRED: artifact type descriptor
├── collated_input.json                     # REQUIRED: complete runtime definition
└── studio-data/                            # Tool code and workflow files
    └── workflows/
        └── {workflow_directory_name}/
            └── tools/
                └── {tool_slug}/
                    ├── tool.py
                    ├── requirements.txt
                    └── [additional files]
```

## workflow.yaml

A simple descriptor that tells the workflow engine how to load the artifact:

```yaml
type: collated_input
input: collated_input.json
```

Currently, `collated_input` is the only supported artifact type. The `input` field points to the JSON file containing the full workflow definition.

## collated_input.json

The complete runtime representation of the workflow. See [CollatedInput Schema](./collated-input.md) for the full specification.

## studio-data/ Subtree

The `studio-data/` directory is a filtered copy of the project's `studio-data/`:

- Only the specific workflow's directory under `studio-data/workflows/` is included
- `tool_templates/`, `temp_files/`, and `deployable_workflows/` are excluded
- Within tool directories, the following are excluded:
  - `.venv/`
  - `.next/`
  - `node_modules/`
  - `.nvm/`
  - `.requirements_hash.txt`

## How the Engine Consumes Artifacts

1. The engine extracts the tar.gz to a temporary directory
2. Reads `workflow.yaml` to determine the artifact type
3. Loads `collated_input.json` and parses it into the `CollatedInput` Pydantic model
4. Resolves tool paths relative to the extracted directory
5. Creates virtual environments for each tool from their `requirements.txt`
6. Executes the workflow using CrewAI or LangGraph based on the artifact contents

## LangGraph Artifact Alternative

LangGraph workflows use a different artifact structure, detected by the presence of `langgraph.json`:

```
langgraph_artifact/
├── langgraph.json                          # LangGraph graph definition
├── pyproject.toml                          # Dependencies (installed via pip)
├── .env                                    # Optional environment variables
└── src/
    └── graph.py                            # Graph implementation
```

The `langgraph.json` schema:
```json
{
  "graphs": {
    "graph_name": "src/graph.py:graph_symbol"
  }
}
```

Currently limited to a single graph per artifact. Dependencies are installed via `pip install .` from the artifact root.
