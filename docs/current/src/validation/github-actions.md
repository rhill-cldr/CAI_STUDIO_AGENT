# GitHub Actions Integration

This chapter shows how to integrate template validation into a CI/CD pipeline using GitHub Actions.

## Example Workflow

```yaml
name: Validate Agent Studio Templates

on:
  pull_request:
    paths:
      - 'templates/**'
  push:
    branches: [main]
    paths:
      - 'templates/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install validation SDK
        run: pip install as-template-validator  # Your published SDK package

      - name: Build template ZIP
        run: |
          cd templates/my-workflow
          zip -r ../../my-workflow.zip workflow_template.json studio-data/

      - name: Validate template
        run: |
          as-validate my-workflow.zip
          # Exit code 0 = valid, non-zero = errors found

      - name: Post results to PR
        if: failure() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const { execSync } = require('child_process');
            const output = execSync('as-validate my-workflow.zip 2>&1 || true').toString();
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Template Validation Results\n\`\`\`\n${output}\n\`\`\``
            });
```

## Repository Structure

A recommended repository layout for template development:

```
my-agent-templates/
├── .github/
│   └── workflows/
│       └── validate.yml           # CI workflow above
├── templates/
│   └── my-workflow/
│       ├── workflow_template.json  # Manifest
│       └── studio-data/
│           ├── tool_templates/
│           │   └── my_tool_abc123/
│           │       ├── tool.py
│           │       └── requirements.txt
│           └── dynamic_assets/
│               └── tool_template_icons/
│                   └── my_tool_abc123_icon.png
├── scripts/
│   └── build-zip.sh               # ZIP packaging script
└── README.md
```

## Build Script

A simple script to package a template directory into a ZIP:

```bash
#!/bin/bash
# scripts/build-zip.sh
# Usage: ./scripts/build-zip.sh templates/my-workflow output.zip

set -euo pipefail

TEMPLATE_DIR="$1"
OUTPUT_ZIP="$2"

if [ ! -f "$TEMPLATE_DIR/workflow_template.json" ]; then
  echo "ERROR: workflow_template.json not found in $TEMPLATE_DIR"
  exit 1
fi

cd "$TEMPLATE_DIR"
zip -r "$(realpath "$OUTPUT_ZIP")" \
  workflow_template.json \
  studio-data/ \
  -x "**/.venv/*" \
  -x "**/__pycache__/*" \
  -x "**/.requirements_hash.txt"

echo "Created $OUTPUT_ZIP"
```

## Optional: Deploy to Agent Studio

If your CI pipeline has access to an Agent Studio instance, you can deploy validated templates automatically.

Agent Studio supports deployment directly from GitHub URLs using the `github` workflow source type. The platform clones the repository, packages the workflow, and deploys it.

For programmatic deployment, use the gRPC API:

```bash
# Upload the ZIP via the Agent Studio API
# (requires CDSW_APIV2_KEY and CDSW_DOMAIN as GitHub secrets)
curl -X POST "https://${CDSW_DOMAIN}/api/v1/workflow-templates/import" \
  -H "Authorization: Bearer ${CDSW_APIV2_KEY}" \
  -F "file=@my-workflow.zip"
```

> **Note**: The exact HTTP endpoint depends on your Agent Studio deployment configuration. The canonical interface is the `ImportWorkflowTemplate` gRPC RPC, proxied through the Next.js API layer.

## Multi-Template Validation

For repositories containing multiple templates:

```yaml
      - name: Validate all templates
        run: |
          EXIT_CODE=0
          for dir in templates/*/; do
            ZIP="/tmp/$(basename $dir).zip"
            (cd "$dir" && zip -r "$ZIP" workflow_template.json studio-data/)
            echo "Validating $ZIP..."
            if ! as-validate "$ZIP"; then
              EXIT_CODE=1
            fi
          done
          exit $EXIT_CODE
```
