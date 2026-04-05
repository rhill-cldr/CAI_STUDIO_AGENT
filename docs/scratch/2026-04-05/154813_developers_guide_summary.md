# Developer's Guide - Work Summary

## What was done

Created a complete mdbook developer's guide at `docs/current/` with 19 content chapters across 4 sections, targeting two audiences:

1. **Harness builders** — engineers building a claude-code-like agent development harness
2. **SDK builders** — engineers creating a validation SDK for CI/CD template packaging

## Structure

### Architecture Reference (4 chapters)
- System overview with component topology and entity hierarchy
- XYFlow visual canvas — node types, layout algorithm, live event overlay
- OTel + Phoenix telemetry — trace export, event pipeline, framework instrumentors
- gRPC service design — starting from Open Inference Protocol, not the existing proto

### Template Specification (8 chapters) — Critical Section
- Template system concepts (templates vs instances, ID management, scoping)
- Tool template spec (tool.py contract: UserParameters, ToolParameters, run_tool, OUTPUT_KEY)
- Agent, task, MCP template specs (all fields documented)
- Workflow template ZIP format (exact directory structure, path conventions)
- workflow_template.json manifest schema (all fields, cross-reference integrity rules)
- Dynamic assets and icons (formats, naming conventions)

### Deployment Artifacts (3 chapters)
- tar.gz artifact format (workflow.yaml + collated_input.json)
- CollatedInput schema (full runtime contract)
- Deployment configuration (model types, generation config, environment variables)

### Validation & CI/CD (3 chapters)
- Exhaustive validation rules reference (30+ rules with codes and severity)
- Complete SDK implementation guide with working Python code
- GitHub Actions integration with example workflow YAML

## Key source files consulted
- `studio/workflow/workflow_templates.py` (export/import logic)
- `studio/workflow_engine/src/engine/types.py` (CollatedInput)
- `studio/tools/utils.py` (AST validation)
- `studio/db/model.py` (template DB models)
- `studio/consts.py` (path constants)
- `app/workflows/diagrams.ts` (XYFlow canvas)
- `studio/workflow_engine/src/engine/crewai/events.py` (telemetry events)

## Build verification
`mdbook build docs/current` — clean build, no errors or warnings, all 19 chapters rendered.
