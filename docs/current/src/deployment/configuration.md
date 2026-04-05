# Deployment Configuration

When deploying a workflow, Agent Studio applies a `DeploymentConfig` that controls runtime behavior — LLM generation parameters, tool configuration, MCP server secrets, and environment variables.

## DeploymentConfig

```python
class DeploymentConfig(BaseModel):
    generation_config: Dict = {}
    tool_config: Dict[str, Dict[str, str]] = {}
    mcp_config: Dict[str, Dict[str, str]] = {}
    llm_config: Dict = {}
    environment: Dict = {}
```

| Field | Type | Description |
|---|---|---|
| `generation_config` | object | Default LLM generation parameters applied to all language models |
| `tool_config` | object | Tool-specific configuration keyed by tool instance ID |
| `mcp_config` | object | MCP server secrets keyed by MCP instance ID |
| `llm_config` | object | LLM API keys and endpoint overrides |
| `environment` | object | Additional environment variables passed to the deployment |

## Default Generation Config

When no overrides are provided, Agent Studio uses these defaults:

```json
{
  "do_sample": true,
  "temperature": 0.1,
  "max_new_tokens": 4096,
  "top_p": 1,
  "top_k": 50,
  "num_beams": 1,
  "max_length": null
}
```

## Language Model Configuration

At deployment time, each language model receives connection details:

```python
class Input__LanguageModelConfig(BaseModel):
    provider_model: str              # Provider-specific model identifier (e.g., "gpt-4o")
    model_type: SupportedModelTypes  # Provider type enum
    api_base: Optional[str]          # Custom API endpoint
    api_key: Optional[str]           # Authentication key
    extra_headers: Optional[Dict[str, str]]  # Additional HTTP headers

    # AWS Bedrock specific
    aws_region_name: Optional[str]
    aws_access_key_id: Optional[str]
    aws_secret_access_key: Optional[str]
    aws_session_token: Optional[str]
```

## Supported Model Types

| Type | Description | Required Fields |
|---|---|---|
| `OPENAI` | OpenAI API | `api_key` |
| `OPENAI_COMPATIBLE` | OpenAI-compatible endpoints | `api_base`, `api_key` |
| `AZURE_OPENAI` | Azure OpenAI Service | `api_base`, `api_key` |
| `GEMINI` | Google Gemini | `api_key` |
| `ANTHROPIC` | Anthropic Claude | `api_key` |
| `CAII` | Cloudera AI Inference | `api_base`, `api_key` |
| `BEDROCK` | AWS Bedrock | `aws_region_name`, `aws_access_key_id`, `aws_secret_access_key` |

## Deployment Targets

Agent Studio supports multiple deployment target types:

| Target | Description |
|---|---|
| `workbench_model` | CML Model endpoint — the primary deployment target |
| `langgraph_server` | LangGraph Server deployment |
| `ai_inference` | Cloudera AI Inference (planned) |
| `model_registry` | Model Registry deployment (planned) |

## Workflow Source Types

Workflows can be deployed from multiple sources:

| Source | Description |
|---|---|
| `workflow` | An existing workflow in the current Agent Studio instance |
| `workflow_template` | A workflow template (instantiates first, then deploys) |
| `workflow_artifact` | A pre-packaged tar.gz artifact |
| `github` | A GitHub repository URL (cloned and packaged by Agent Studio) |

## Environment Variables at Runtime

Deployed workflows receive these environment variables:

| Variable | Description |
|---|---|
| `AGENT_STUDIO_OPS_ENDPOINT` | Phoenix observability endpoint for trace export |
| `AGENT_STUDIO_WORKFLOW_ARTIFACT` | Path to the extracted deployment artifact |
| `AGENT_STUDIO_WORKFLOW_DEPLOYMENT_CONFIG` | JSON-encoded `DeploymentConfig` |
| `AGENT_STUDIO_MODEL_EXECUTION_DIR` | Working directory for model execution |
| `CDSW_APIV2_KEY` | CML API v2 authentication key |
| `CDSW_PROJECT_ID` | CML project identifier |
| `CREWAI_DISABLE_TELEMETRY` | Set to disable CrewAI's built-in telemetry |
