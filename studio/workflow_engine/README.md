# Workflow Engine

This package contains all code necessary for Agent Studio's Workflow Engine, which is the container/orchestrator responsible for running deployed workflows in Cloudera AI.

For architecture details, see the [Developer's Guide](https://rhill-cldr.github.io/CAI_STUDIO_AGENT/):
- [System Overview](https://rhill-cldr.github.io/CAI_STUDIO_AGENT/architecture/overview.html) — component topology and data flow
- [CollatedInput Schema](https://rhill-cldr.github.io/CAI_STUDIO_AGENT/deployment/collated-input.html) — the runtime contract between studio and engine
- [Telemetry Pipeline](https://rhill-cldr.github.io/CAI_STUDIO_AGENT/architecture/telemetry.html) — OTel traces and structured event stream