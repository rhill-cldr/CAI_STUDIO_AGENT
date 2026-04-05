# Building a Validation SDK

This chapter provides practical guidance for building a validation library that checks workflow template ZIPs against the [Validation Rules](./rules.md). The SDK is intended to be used in CI/CD pipelines (e.g., as a GitHub Action) to catch errors before templates reach Agent Studio.

## Recommended Architecture

```python
from dataclasses import dataclass
from enum import Enum
from typing import List

class Severity(Enum):
    ERROR = "error"
    WARNING = "warning"

@dataclass
class ValidationIssue:
    rule: str          # Rule code (e.g., "T-004")
    severity: Severity
    message: str       # Human-readable description
    path: str          # File or field path within the ZIP

@dataclass
class ValidationResult:
    valid: bool                    # True if no ERROR-level issues
    issues: List[ValidationIssue]

def validate_template_zip(zip_path: str) -> ValidationResult:
    """Main entry point for SDK consumers."""
    ...
```

## Validation Pipeline

### Step 1: Open ZIP and Check Structure

```python
import zipfile
import json

def validate_structure(zf: zipfile.ZipFile) -> List[ValidationIssue]:
    issues = []
    names = zf.namelist()

    # S-001: workflow_template.json at root
    if "workflow_template.json" not in names:
        issues.append(ValidationIssue(
            rule="S-001", severity=Severity.ERROR,
            message="workflow_template.json not found at ZIP root",
            path="/"
        ))
    return issues
```

### Step 2: Parse Manifest and Validate Top-Level Keys

```python
def validate_manifest(data: dict) -> List[ValidationIssue]:
    issues = []
    required_keys = {
        "template_version": "M-001",
        "workflow_template": "M-002",
        "agent_templates": "M-003",
        "tool_templates": "M-004",
        "task_templates": "M-005",
    }
    for key, rule in required_keys.items():
        if key not in data:
            issues.append(ValidationIssue(
                rule=rule, severity=Severity.ERROR,
                message=f"Required key '{key}' missing from manifest",
                path="workflow_template.json"
            ))
    # M-006: mcp_templates (may be absent in older exports)
    if "mcp_templates" not in data:
        data["mcp_templates"] = []  # Normalize
    return issues
```

### Step 3: Build ID Index and Validate Cross-References

```python
def validate_cross_references(data: dict) -> List[ValidationIssue]:
    issues = []

    # Build ID sets
    agent_ids = {a["id"] for a in data.get("agent_templates", [])}
    tool_ids = {t["id"] for t in data.get("tool_templates", [])}
    mcp_ids = {m["id"] for m in data.get("mcp_templates", [])}
    task_ids = {t["id"] for t in data.get("task_templates", [])}

    wt = data.get("workflow_template", {})

    # X-001: agent_template_ids
    for aid in wt.get("agent_template_ids", []):
        if aid not in agent_ids:
            issues.append(ValidationIssue(
                rule="X-001", severity=Severity.ERROR,
                message=f"workflow_template.agent_template_ids references unknown agent '{aid}'",
                path="workflow_template.json"
            ))

    # X-002: task_template_ids
    for tid in wt.get("task_template_ids", []):
        if tid not in task_ids:
            issues.append(ValidationIssue(
                rule="X-002", severity=Severity.ERROR,
                message=f"workflow_template.task_template_ids references unknown task '{tid}'",
                path="workflow_template.json"
            ))

    # X-003: manager_agent_template_id
    mgr = wt.get("manager_agent_template_id")
    if mgr and mgr not in agent_ids:
        issues.append(ValidationIssue(
            rule="X-003", severity=Severity.ERROR,
            message=f"manager_agent_template_id references unknown agent '{mgr}'",
            path="workflow_template.json"
        ))

    # X-004, X-005: agent -> tool/mcp references
    for agent in data.get("agent_templates", []):
        for tid in agent.get("tool_template_ids", []):
            if tid not in tool_ids:
                issues.append(ValidationIssue(
                    rule="X-004", severity=Severity.ERROR,
                    message=f"Agent '{agent['id']}' references unknown tool '{tid}'",
                    path="workflow_template.json"
                ))
        for mid in agent.get("mcp_template_ids", []):
            if mid not in mcp_ids:
                issues.append(ValidationIssue(
                    rule="X-005", severity=Severity.ERROR,
                    message=f"Agent '{agent['id']}' references unknown MCP '{mid}'",
                    path="workflow_template.json"
                ))

    # X-006: task -> agent references
    for task in data.get("task_templates", []):
        assigned = task.get("assigned_agent_template_id")
        if assigned and assigned not in agent_ids:
            issues.append(ValidationIssue(
                rule="X-006", severity=Severity.ERROR,
                message=f"Task '{task['id']}' references unknown agent '{assigned}'",
                path="workflow_template.json"
            ))

    return issues
```

### Step 4: Validate Tool Templates via AST

This is the most critical validation — it ensures that `tool.py` files contain the required components. The logic mirrors Agent Studio's own AST-based extraction.

```python
import ast

def validate_tool_py(source_code: str, tool_path: str) -> List[ValidationIssue]:
    issues = []

    # T-004: Must parse as valid Python
    try:
        tree = ast.parse(source_code)
    except SyntaxError as e:
        issues.append(ValidationIssue(
            rule="T-004", severity=Severity.ERROR,
            message=f"Python syntax error: {e}",
            path=tool_path
        ))
        return issues  # Can't validate further

    # Collect top-level names
    class_names = set()
    function_names = set()
    assignments = set()
    has_main_block = False

    for node in ast.walk(tree):
        if isinstance(node, ast.ClassDef):
            class_names.add(node.name)
        elif isinstance(node, ast.FunctionDef):
            function_names.add(node.name)
        elif isinstance(node, ast.Assign):
            for target in node.targets:
                if isinstance(target, ast.Name):
                    assignments.add(target.id)

    # Check for if __name__ == "__main__"
    for node in ast.walk(tree):
        if isinstance(node, ast.If):
            test = node.test
            if (isinstance(test, ast.Compare) and
                isinstance(test.left, ast.Name) and
                test.left.id == "__name__"):
                has_main_block = True

    # T-005: UserParameters class
    if "UserParameters" not in class_names:
        issues.append(ValidationIssue(
            rule="T-005", severity=Severity.ERROR,
            message="Missing 'class UserParameters(BaseModel)'",
            path=tool_path
        ))

    # T-006: ToolParameters class
    if "ToolParameters" not in class_names:
        issues.append(ValidationIssue(
            rule="T-006", severity=Severity.ERROR,
            message="Missing 'class ToolParameters(BaseModel)'",
            path=tool_path
        ))

    # T-007: run_tool function
    if "run_tool" not in function_names:
        issues.append(ValidationIssue(
            rule="T-007", severity=Severity.ERROR,
            message="Missing 'def run_tool(...)' function",
            path=tool_path
        ))

    # T-W01: OUTPUT_KEY (warning)
    if "OUTPUT_KEY" not in assignments:
        issues.append(ValidationIssue(
            rule="T-W01", severity=Severity.WARNING,
            message="Missing 'OUTPUT_KEY' module-level assignment",
            path=tool_path
        ))

    # T-W02: __main__ block (warning)
    if not has_main_block:
        issues.append(ValidationIssue(
            rule="T-W02", severity=Severity.WARNING,
            message="Missing 'if __name__ == \"__main__\":' block",
            path=tool_path
        ))

    return issues
```

### Step 5: Validate Icons

```python
import os

VALID_EXTENSIONS = {".png", ".jpg", ".jpeg"}

def validate_icon(path: str, zip_names: set, rule: str) -> List[ValidationIssue]:
    issues = []
    if not path:  # Empty string = no icon
        return issues
    if path not in zip_names:
        issues.append(ValidationIssue(
            rule=rule, severity=Severity.ERROR,
            message=f"Icon file not found in ZIP: {path}",
            path=path
        ))
    else:
        ext = os.path.splitext(path)[1].lower()
        if ext not in VALID_EXTENSIONS:
            issues.append(ValidationIssue(
                rule="I-004", severity=Severity.ERROR,
                message=f"Invalid icon extension '{ext}' (must be .png, .jpg, or .jpeg)",
                path=path
            ))
    return issues
```

### Step 6: Assemble Results

```python
def validate_template_zip(zip_path: str) -> ValidationResult:
    issues = []

    with zipfile.ZipFile(zip_path, "r") as zf:
        names = set(zf.namelist())

        # Step 1: Structure
        issues.extend(validate_structure(zf))
        if any(i.rule == "S-001" for i in issues):
            return ValidationResult(valid=False, issues=issues)

        # Step 2: Parse manifest
        data = json.loads(zf.read("workflow_template.json"))
        issues.extend(validate_manifest(data))
        if any(i.severity == Severity.ERROR for i in issues):
            return ValidationResult(valid=False, issues=issues)

        # Step 3: Cross-references
        issues.extend(validate_cross_references(data))

        # Step 4: Tool templates
        for tool in data.get("tool_templates", []):
            src_path = tool.get("source_folder_path", "")
            tool_py_path = f"{src_path}/tool.py"
            req_path = f"{src_path}/requirements.txt"

            # T-001: Directory exists
            dir_entries = [n for n in names if n.startswith(src_path + "/")]
            if not dir_entries:
                issues.append(ValidationIssue(
                    rule="T-001", severity=Severity.ERROR,
                    message=f"Tool directory not found: {src_path}",
                    path=src_path
                ))
                continue

            # T-002: tool.py exists
            if tool_py_path not in names:
                issues.append(ValidationIssue(
                    rule="T-002", severity=Severity.ERROR,
                    message=f"tool.py not found in {src_path}",
                    path=tool_py_path
                ))
            else:
                source = zf.read(tool_py_path).decode("utf-8")
                issues.extend(validate_tool_py(source, tool_py_path))

            # T-003: requirements.txt exists
            if req_path not in names:
                issues.append(ValidationIssue(
                    rule="T-003", severity=Severity.ERROR,
                    message=f"requirements.txt not found in {src_path}",
                    path=req_path
                ))

            # N-001: Name validation
            import re
            name = tool.get("name", "")
            if not re.match(r"^[a-zA-Z0-9 ]+$", name):
                issues.append(ValidationIssue(
                    rule="N-001", severity=Severity.ERROR,
                    message=f"Tool name '{name}' contains invalid characters",
                    path="workflow_template.json"
                ))

            # Icons
            issues.extend(validate_icon(
                tool.get("tool_image_path", ""), names, "I-001"))

        # Step 5: Agent and MCP icons
        for agent in data.get("agent_templates", []):
            issues.extend(validate_icon(
                agent.get("agent_image_path", ""), names, "I-002"))
        for mcp in data.get("mcp_templates", []):
            issues.extend(validate_icon(
                mcp.get("mcp_image_path", ""), names, "I-003"))

    errors = [i for i in issues if i.severity == Severity.ERROR]
    return ValidationResult(valid=len(errors) == 0, issues=issues)
```

## CLI Interface

A simple CLI wrapper for use in scripts and CI:

```python
import sys

if __name__ == "__main__":
    result = validate_template_zip(sys.argv[1])
    for issue in result.issues:
        prefix = "ERROR" if issue.severity == Severity.ERROR else "WARN"
        print(f"[{prefix}] {issue.rule}: {issue.message} ({issue.path})")
    sys.exit(0 if result.valid else 1)
```

Usage:
```bash
python validate.py my_template.zip
# Exit code 0 = valid, 1 = errors found
```
