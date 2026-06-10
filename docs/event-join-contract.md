# Event Join Contract

## Required Inputs

| Stream | Required fields |
| --- | --- |
| OpenClaw spans | `trace_id`, `span_id`, `openclaw_turn_id`, `openclaw_llm_call_id`, raw content |
| LiteLLM spans/logs | `trace_id`, `span_id`, `litellm.call_id`, route, model, usage |
| vLLM spans | `trace_id`, `span_id`, `gen_ai.request.id`, latency, usage |
| vLLM response usage | request id, prompt tokens, completion tokens, cached tokens if present |
| vLLM KV events | block hashes, parent hashes, token ids, event type, medium if present |
| LMCache metrics | lookup/retrieve/store counts, hit tokens, latencies, L0/L1/L2 histograms |
| LMCache traces | storage operation, args, timestamp, trace file path |

## Join Order

1. `trace_id`
2. `openclaw_llm_call_id`
3. `x_request_id`
4. `litellm.call_id`
5. `gen_ai.request.id`
6. time window
7. `block_hash`
8. `parent_block_hash`

## Output Rules

- Preserve all evidence references.
- Do not infer a tier unless an event, metric label, or trace record supports it.
- Mark inferred fields with `inferred: true`.
- Mark aggregate-only values with `granularity: aggregate`.
- Mark request-level values with `granularity: request`.
- Never put request IDs into Prometheus labels.
