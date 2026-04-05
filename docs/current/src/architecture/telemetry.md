# Telemetry Pipeline (OTel + Phoenix)

Agent Studio uses [OpenTelemetry](https://opentelemetry.io/) with [OpenInference](https://github.com/Arize-ai/openinference) semantic conventions for workflow observability. Traces are collected by an embedded [Arize Phoenix](https://github.com/Arize-ai/phoenix) instance. An external harness should implement a compatible telemetry pipeline to enable visual execution monitoring.

## Architecture

```d2
runner: Workflow Engine Runner {
  otel: OTel TracerProvider {
    label: "OTel TracerProvider\n(spans: workflow, agent,\ntask, tool, LLM calls)"
  }
  events: Event Pipeline {
    label: "Event Pipeline\n(structured JSON events\nper CrewAI event bus)"
  }
}

phoenix: Phoenix (port 50052) {
  traces: /v1/traces
  event_endpoint: /events
}

frontend: Frontend {
  poll: "Poll /api/workflow/events\n?trace_id=..."
  canvas: XYFlow canvas overlay
  poll -> canvas
}

runner.otel -> phoenix.traces
runner.events -> phoenix.event_endpoint
phoenix.event_endpoint -> frontend.poll
```

## Trace Export

Each workflow execution creates a root span and exports traces to Phoenix:

```python
from phoenix.otel import register

tracer_provider = register(
    project_name=workflow_name,
    endpoint=f"{ops_endpoint}/v1/traces",
    headers={"Authorization": f"Bearer {api_key}"},
)
```

The trace ID is extracted from the root span as a 32-character hex string:

```python
tracer = tracer_provider.get_tracer("opentelemetry.agentstudio.workflow.model")
with tracer.start_as_current_span(f"Workflow Run: {time}") as span:
    trace_id = f"{span.get_span_context().trace_id:032x}"
```

## Framework Instrumentors

Agent Studio uses OpenInference instrumentors to automatically capture framework-level spans:

| Framework | Instrumentor | Captures |
|---|---|---|
| CrewAI | `openinference.instrumentation.crewai.CrewAIInstrumentor` | Crew, Agent, Task execution spans |
| LiteLLM | `openinference.instrumentation.litellm.LiteLLMInstrumentor` | LLM API call spans |
| LangChain | `openinference.instrumentation.langchain.LangChainInstrumentor` | LangGraph node execution spans |

Instrumentation is applied once per runner process:

```python
CrewAIInstrumentor().instrument(tracer_provider=tracer_provider)
LiteLLMInstrumentor().instrument(tracer_provider=tracer_provider)
```

## Structured Event Pipeline

In addition to OTel traces, Agent Studio captures a structured event stream from CrewAI's event bus. This powers the real-time canvas overlay.

### Event Registration

Global handlers are registered on the CrewAI event bus singleton:

```python
from crewai.utilities.events import crewai_event_bus

for event_cls in EVENT_PROCESSORS:
    crewai_event_bus.on(event_cls)(post_event)
```

### Event Types

The following events are captured and processed:

| Category | Events |
|---|---|
| **Crew lifecycle** | `CrewKickoffStarted`, `CrewKickoffCompleted`, `CrewKickoffFailed` |
| **Agent execution** | `AgentExecutionStarted`, `AgentExecutionCompleted`, `AgentExecutionError` |
| **Task execution** | `TaskStarted`, `TaskCompleted`, `TaskFailed` |
| **Tool usage** | `ToolUsageStarted`, `ToolUsageFinished`, `ToolUsageError` |
| **LLM calls** | `LLMCallStarted`, `LLMCallCompleted`, `LLMCallFailed` |

Each event is processed by a type-specific extractor that selects relevant fields (e.g., tool name, agent ID, error message) and enriched with:
- `timestamp` — event time
- `type` — event class name
- `agent_studio_id` — maps the event to a specific tool/agent instance in the canvas

### Event Posting

Events are POSTed as JSON to the Phoenix event broker:

```
POST {ops_endpoint}/events
Authorization: Bearer {api_key}
Content-Type: application/json

{
  "trace_id": "a1b2c3d4...",
  "event": {
    "timestamp": "2025-01-15T10:30:00",
    "type": "tool_usage_started",
    "agent_studio_id": "tool-instance-uuid",
    "tool_name": "json_reader",
    "tool_args": "{\"filepath\": \"data.json\"}"
  }
}
```

### Trace Context Propagation

Each async workflow task maintains its own trace context via Python `contextvars`. This allows the global event handlers (shared across all concurrent workflows on a single runner) to route events to the correct trace:

```python
from contextvars import ContextVar
_trace_id_var: ContextVar[str] = ContextVar("trace_id")

def get_trace_id() -> str:
    return _trace_id_var.get()
```

## Harness Implementation Guide

To replicate this telemetry stack in an external harness:

1. **Deploy Phoenix** (or any OTel-compatible collector) as your trace backend
2. **Register a TracerProvider** using `phoenix.otel.register()` pointed at your collector
3. **Instrument your framework** with the appropriate OpenInference instrumentor
4. **Implement an event stream** — POST structured events keyed by trace ID to your collector
5. **Poll events from the frontend** — the canvas subscribes to `GET /events?trace_id={id}` and overlays them onto nodes
6. **Propagate trace context** through async task boundaries using contextvars
