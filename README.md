# Agent Inference Trace Observability Guide

Machine-oriented setup guide for collecting simulation-ready input traces from an
agent serving stack:

```text
OpenClaw -> LiteLLM -> vLLM -> LMCache
```

The target reader is another AI agent running inside a company environment. This
repo intentionally favors explicit contracts, config templates, and validation
checklists over prose.

## Goal

Create per-request input traces that connect:

- OpenClaw agent turn, tool calls, prompts, completions
- LiteLLM gateway routing and call IDs
- vLLM request spans, token usage, latency, prefix-cache signals
- LMCache KV block movement, cache tier behavior, and storage timing

## Non-Goal

Do not collect raw prompt/completion content from LiteLLM. Raw content capture is
owned by OpenClaw. LiteLLM should provide routing, IDs, usage, latency, and cost
metadata only.

## Start Here

1. Read [AGENT_GUIDE.md](AGENT_GUIDE.md).
2. Apply templates in [config/](config/).
3. Validate with [checklists/validation.md](checklists/validation.md).
4. Emit records matching [schemas/input-trace.schema.json](schemas/input-trace.schema.json).

## Public Sources

See [docs/sources.md](docs/sources.md).
