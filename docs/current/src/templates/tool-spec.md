# Tool Template Specification

Tool templates are the most complex template type. Each tool is a self-contained Python package that agents invoke at runtime. This chapter is the authoritative reference for building conformant tool templates.

## Required Files

Every tool template directory **must** contain:

```
{tool_name}/
├── tool.py              # REQUIRED: Main entrypoint
└── requirements.txt     # REQUIRED: Python dependencies
```

Optional additional files are permitted:

```
{tool_name}/
├── tool.py
├── requirements.txt
├── README.md            # Documentation
├── helper_module.py     # Additional Python modules
└── data/                # Data files the tool uses
    └── config.json
```

## tool.py Specification

The entrypoint file must contain all of the following components. Agent Studio validates their presence via Python AST parsing.

### 1. Module Docstring

```python
"""
Description of what this tool does.
This text is extracted as the tool's description and shown to agents.
"""
```

The module-level docstring is extracted via `ast.get_docstring()` and used as the tool's description in the UI and in agent prompts.

### 2. UserParameters Class

```python
from pydantic import BaseModel
from typing import Optional

class UserParameters(BaseModel):
    """Configuration parameters set once when the tool is added to a workflow."""
    api_key: str                        # Required parameter
    endpoint: Optional[str] = None      # Optional parameter with default
```

- Must inherit from `pydantic.BaseModel`
- Must be named exactly `UserParameters`
- Defines configuration that is set once per tool instance (API keys, database URLs, etc.)
- Can be empty (`pass`) if no configuration is needed
- Agent Studio extracts parameter metadata via AST:
  - `Optional[T]` → marked as not required
  - Fields with default values → marked as not required
  - All other fields → marked as required

### 3. ToolParameters Class

```python
from pydantic import BaseModel, Field

class ToolParameters(BaseModel):
    """Arguments passed by an agent each time it invokes this tool."""
    query: str = Field(description="The search query to execute")
    max_results: int = Field(default=10, description="Maximum number of results")
```

- Must inherit from `pydantic.BaseModel`
- Must be named exactly `ToolParameters`
- `Field(description=...)` annotations are passed to the LLM agent as parameter descriptions — these help the agent decide what values to pass
- Supported types: `str`, `int`, `float`, `bool`, `List[T]`, `Optional[T]`, `Literal["val1", "val2"]`

### 4. run_tool Function

```python
from typing import Any

def run_tool(config: UserParameters, args: ToolParameters) -> Any:
    """Main tool logic. Return value is sent back to the calling agent."""
    result = do_work(config.api_key, args.query)
    return result
```

- Must be named exactly `run_tool`
- Receives validated `UserParameters` and `ToolParameters` instances
- Return value should be JSON-serializable for the agent to parse

### 5. OUTPUT_KEY Constant

```python
OUTPUT_KEY = "tool_output"
```

- Module-level string constant
- When present, only stdout content printed **after** this key is returned to the agent
- Allows debug/log output before the key without polluting the agent's response
- Recommended but not strictly required for import — however, without it, all stdout is returned to the agent

### 6. CLI Entrypoint

```python
import json
import argparse

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--user-params", required=True, help="JSON string of UserParameters")
    parser.add_argument("--tool-params", required=True, help="JSON string of ToolParameters")
    args = parser.parse_args()

    config = UserParameters(**json.loads(args.user_params))
    params = ToolParameters(**json.loads(args.tool_params))

    output = run_tool(config, params)
    print(OUTPUT_KEY, output)
```

Agent Studio invokes tools as subprocesses:
```bash
python tool.py --user-params '{"api_key": "..."}' --tool-params '{"query": "..."}'
```

## Complete Example

```python
"""
A simple JSON reader tool that reads JSON files from a given path
relative to the tool's directory.
"""

from pydantic import BaseModel, Field
from typing import Any
import json
import argparse
from pathlib import Path
import sys
import os

ROOT_DIR = Path(__file__).parent
sys.path.append(str(ROOT_DIR))
os.chdir(ROOT_DIR)


class UserParameters(BaseModel):
    pass


class ToolParameters(BaseModel):
    filepath: str = Field(
        description="The local path to the JSON file to read, relative to the tool's directory."
    )


def run_tool(config: UserParameters, args: ToolParameters) -> Any:
    filepath = args.filepath
    data = json.load(open(filepath))
    return json.dumps(data, indent=2)


OUTPUT_KEY = "tool_output"

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--user-params", required=True)
    parser.add_argument("--tool-params", required=True)
    args = parser.parse_args()

    config = UserParameters(**json.loads(args.user_params))
    params = ToolParameters(**json.loads(args.tool_params))

    output = run_tool(config, params)
    print(OUTPUT_KEY, output)
```

## requirements.txt

Standard pip format. `pydantic` is always required (it's the validation framework):

```
pydantic
requests>=2.28.0
numpy==1.24.0
```

## Name Validation

Tool template names must match the regex:

```
^[a-zA-Z0-9 ]+$
```

Only alphanumeric characters and spaces are allowed. No special characters, underscores, or hyphens.

Names must be unique within their scope (global or within a specific workflow template).

## Directory Naming Convention

When packaged in a ZIP, tool template directories follow this naming pattern:

```
{slugified_name}_{random_6_chars}
```

Where `slugified_name` is the tool name lowercased with spaces replaced by underscores, and the random suffix is a 6-character alphanumeric string. Examples:

```
json_reader_abc123/
my_api_tool_xk9f2m/
calculator_def456/
```

## Virtual Environment (Runtime)

At runtime, Agent Studio creates a `.venv/` inside each tool instance directory and installs dependencies from `requirements.txt`. The following paths are excluded from template exports:

- `.venv/`
- `__pycache__/`
- `.requirements_hash.txt`

## Testing Locally

Tools can be tested outside Agent Studio:

```bash
cd my_tool/
pip install -r requirements.txt
python tool.py \
  --user-params '{}' \
  --tool-params '{"filepath": "data/sample.json"}'
```

Expected output:
```
tool_output {"key": "value", ...}
```
