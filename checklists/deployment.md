# Deployment Checklist

## OpenClaw

- [ ] Observability plugin installed or built-in diagnostics configured.
- [ ] `NODE_OPTIONS` preload applied to the real gateway process.
- [ ] `OPENCLAW_OTEL_CONTENT_POLICY` captures all OpenClaw-side raw content.
- [ ] `hooks.allowConversationAccess` enabled for path-loaded plugin.
- [ ] OpenClaw exports to OTel Collector, not directly to multiple backends.
- [ ] Each LLM call gets an `openclaw_llm_call_id`.
- [ ] LiteLLM request includes trace context and correlation metadata.

## LiteLLM

- [ ] `callbacks: ["otel"]`.
- [ ] `LITELLM_OTEL_V2=true`.
- [ ] `OTEL_IGNORE_CONTEXT_PROPAGATION=false`.
- [ ] `USE_OTEL_LITELLM_REQUEST_SPAN=true`.
- [ ] Raw request/response logging disabled.
- [ ] Prompt storage in spend logs disabled.
- [ ] GenAI message content capture disabled for LiteLLM process.
- [ ] `config/litellm.env.example` values applied to the LiteLLM process.
- [ ] `litellm.call_id` visible in traces/logs.

## vLLM

- [ ] `--otlp-traces-endpoint` configured.
- [ ] `traceparent` and `tracestate` propagated into vLLM.
- [ ] `--enable-request-id-headers` enabled if supported by deployed version.
- [ ] Prefix caching enabled.
- [ ] KV metrics enabled if overhead is acceptable.
- [ ] KV events enabled if block lineage is required.

## LMCache

- [ ] Metrics enabled.
- [ ] KV events enabled.
- [ ] Deterministic hash algorithm configured for multi-worker setups.
- [ ] MP observability enabled if tier-level trace is required.
- [ ] Storage-level trace recording enabled for simulation replay.

## Collector

- [ ] Receives OTLP HTTP and/or gRPC.
- [ ] Exports traces to Langfuse and Tempo.
- [ ] Metrics are routed to metrics backend, not Tempo.
- [ ] No secrets embedded in config files.
