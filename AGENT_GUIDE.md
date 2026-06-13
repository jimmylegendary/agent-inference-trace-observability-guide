# AGENT GUIDE

## Mission

Configure an observability path that lets downstream analysis build input traces
for architecture/workload simulation.

Required correlation chain:

```text
OpenClaw agent turn
  -> LiteLLM request
    -> vLLM request
      -> LMCache KV/cache operations
```

## Hard Requirements

1. Propagate W3C trace context.
2. Generate a stable `openclaw_llm_call_id` before each LiteLLM call.
3. Use the same value as `x-request-id` where possible.
4. Capture raw prompt/completion/tool content in OpenClaw only.
5. Do not capture raw prompt/completion in LiteLLM logs.
6. Export LiteLLM/vLLM/LMCache timing, token, routing, and cache metadata.
7. Store request-level detail in traces/events/logs, not Prometheus labels.
8. Build final input traces by joining records on IDs below.

Read [docs/hash-collection-fact-check.md](docs/hash-collection-fact-check.md)
before adding any model-specific hash collection guide. That document separates
confirmed request/hash evidence from claims that still need deployment
validation or instrumentation.

## Identity Contract

Use these IDs exactly.

| Field | Owner | Purpose |
| --- | --- | --- |
| `trace_id` | OpenTelemetry | Same distributed request flow |
| `openclaw_turn_id` | OpenClaw | One agent turn containing multiple LLM/tool spans |
| `openclaw_llm_call_id` | OpenClaw | One LLM call inside an agent turn |
| `x_request_id` | HTTP/vLLM | Request ID visible at vLLM boundary |
| `litellm_call_id` | LiteLLM | LiteLLM internal request/spend-log ID |
| `vllm_request_id` | vLLM | vLLM serving request ID/span attribute |
| `block_hash` | vLLM/LMCache | KV cache block identity |
| `parent_block_hash` | vLLM/LMCache | KV block lineage |

## Expected Trace Shape

```text
trace_id = one distributed request flow

OpenClaw span: agent.turn
  attributes:
    openclaw.turn_id
    openclaw.llm_call_id
    openclaw.content.input_message
    openclaw.content.prompt
    openclaw.content.output_message

LiteLLM span: proxy/gateway call
  attributes:
    litellm.call_id
    gen_ai.request.model
    gen_ai.usage.input_tokens
    gen_ai.usage.output_tokens
    metadata.openclaw_llm_call_id
    metadata.openclaw_turn_id
  must_not_contain:
    raw prompt content
    raw completion content

vLLM span: request/model/worker spans
  attributes:
    gen_ai.request.id
    gen_ai.request.model
    gen_ai.latency.time_to_first_token
    gen_ai.latency.time_in_queue
    gen_ai.usage.prompt_tokens
    gen_ai.usage.completion_tokens

LMCache events/metrics/traces:
  keys:
    block_hash
    parent_block_hash
    tier_or_medium
    operation
    latency_ms
```

## Collection Surfaces

### OpenClaw

OpenClaw owns raw content.

Enable:

- raw input messages
- raw output messages
- raw system prompt
- tool inputs
- tool outputs
- agent turn sequence

Use [config/openclaw-gateway.env.example](config/openclaw-gateway.env.example)
and [config/openclaw-observability.example.json](config/openclaw-observability.example.json).

### LiteLLM

LiteLLM does not own raw content in this design.

Enable:

- OTel callback
- routing metadata
- usage/cost metadata
- `litellm.call_id`
- spend logs without prompt bodies

Disable:

- raw request/response logging
- prompt storage in spend logs
- GenAI message-content capture in the LiteLLM process

Use [config/litellm.config.example.yaml](config/litellm.config.example.yaml).
Apply process environment from [config/litellm.env.example](config/litellm.env.example).

### vLLM

vLLM owns serving behavior.

Enable:

- OTel trace export
- `X-Request-Id` propagation from upstream clients where possible
- request ID response headers during validation
- detailed traces only if overhead is acceptable
- prefix caching
- KV cache metrics
- KV events when block lineage is required

Do not use a generic vLLM template. First choose a deployment mode below, then
apply exactly one of:

- Non-MP validation: [config/non-mp-vllm.env.example](config/non-mp-vllm.env.example).
- MP production path: [config/mp-vllm.env.example](config/mp-vllm.env.example).

### LMCache

LMCache owns cache tier behavior.

Enable:

- metrics
- request hit/load logs where the deployed integration emits `Reqid`
- non-MP KV events only when validating `LMCacheConnectorV1`
- MP observability when using `LMCacheMPConnector`
- MP storage-level trace recording for simulation replay when tier-level truth is required
- chunk statistics `file_hash` only as offline chunk-reuse/hash evidence, not as
  a guaranteed per-request block lineage stream

Choose one deployment path before editing configs:

- Non-MP: [config/non-mp-lmcache.yaml](config/non-mp-lmcache.yaml) and
  [config/non-mp-vllm.env.example](config/non-mp-vllm.env.example).
- MP: [config/mp-lmcache-server.env.example](config/mp-lmcache-server.env.example) and
  [config/mp-vllm.env.example](config/mp-vllm.env.example).

Read [docs/lmcache-deployment-modes.md](docs/lmcache-deployment-modes.md) first.

## Join Algorithm

1. Load OpenClaw spans by `trace_id`.
2. Extract `openclaw_turn_id` and ordered `openclaw_llm_call_id` values.
3. Load LiteLLM spans/logs where:
   - `trace_id` matches, or
   - `metadata.openclaw_llm_call_id` matches.
4. Extract `litellm_call_id`, route, model, token/cost usage.
5. Load vLLM spans where:
   - `trace_id` matches, or
   - `gen_ai.request.id` / `x_request_id` matches.
6. Load LMCache request logs by `Reqid` when present.
7. If using non-MP with vLLM KV events, load KV events by time window and block lineage.
8. If using MP, load LMCache MP metrics/logging/tracing records by request window, tier, and operation.
9. If chunk statistics is enabled, attach file-hash artifacts as offline evidence.
10. Emit records following [schemas/input-trace.schema.json](schemas/input-trace.schema.json).

## Critical Limitations

1. vLLM OTel does not reliably expose request-level cached prefix tokens today.
2. Prometheus metrics are aggregate signals and cannot be the per-request source of truth.
3. KV events may not expose every CPU/SSD-tier event in every vLLM/LMCache version.
4. Use LMCache MP observability/storage trace or a custom subscriber when tier-level truth is required.
5. Do not put high-cardinality request IDs into Prometheus labels.
6. vLLM `--kv-events-config` ZMQ events are directly validated with `LMCacheConnectorV1`.
   Do not assume `LMCacheMPConnector` emits the same event stream unless verified in the deployed version.
7. Do not describe `LMCacheConnectorV1` as MP mode. MP mode requires
   `LMCacheMPConnector` plus a running `lmcache server`.
8. Do not describe LMCache chunk statistics `file_hash` as guaranteed
   per-request block-hash export.
9. If complete per-request block-hash lists are a hard simulation requirement,
   add version-pinned vLLM/LMCache instrumentation and treat stock observability
   as evidence, not as the whole source of truth.

## Success Definition

An AI agent has completed setup when it can produce one JSONL record per LLM call
containing:

- raw prompt/completion from OpenClaw
- LiteLLM route and call ID
- vLLM serving latency and token usage
- cache hit/tier signals from vLLM/LMCache
- KV block lineage if enabled
- enough IDs to replay and audit joins
