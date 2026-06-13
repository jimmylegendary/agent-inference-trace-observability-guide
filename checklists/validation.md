# Validation Checklist

Run these checks after deployment.

## Trace Continuity

- [ ] OpenClaw agent turn span exists.
- [ ] LiteLLM span exists in the same `trace_id`.
- [ ] vLLM span exists in the same `trace_id`.
- [ ] LiteLLM/vLLM spans are children or linked descendants of OpenClaw model-call span.

## Content Capture Boundary

- [ ] OpenClaw span contains raw input message.
- [ ] OpenClaw span contains raw completion.
- [ ] OpenClaw span contains tool input/output when tools run.
- [ ] LiteLLM span does not contain raw prompt body.
- [ ] LiteLLM spend log does not store raw prompt body.

## Request IDs

- [ ] `openclaw_turn_id` present.
- [ ] `openclaw_llm_call_id` present.
- [ ] `x_request_id` present at vLLM boundary when supported.
- [ ] Validation request response echoes `X-Request-Id` when `--enable-request-id-headers` is enabled.
- [ ] `litellm.call_id` present.
- [ ] Join works for one turn with at least three LLM calls.

## Cache Signals

- [ ] Deployment mode is recorded in the validation report.
- [ ] vLLM metrics expose prefix/KV cache signals.
- [ ] LMCache metrics expose retrieve/store/lookup counts and latencies.
- [ ] LMCache request logs include `Reqid` hit/load lines before request-level hit/load fields are emitted.
- [ ] Non-MP path: vLLM KV events expose block hashes.
- [ ] MP path: LMCache MP metrics/tracing expose tier movement or the output marks tier fields as inferred/aggregate.
- [ ] If chunk statistics `file_hash` is enabled, the artifact is treated as offline evidence unless request IDs are present.
- [ ] Tier-level source of truth exists for L0/L1/L2 movement.
- [ ] Storage trace recording produces replayable artifacts.
- [ ] Complete per-request block-hash list is either directly exported by the deployed version or captured by a version-pinned patch.

## Output

- [ ] Trace Builder emits JSONL records matching `schemas/input-trace.schema.json`.
- [ ] Each emitted record has evidence pointers to spans/events/metrics/traces.
- [ ] At least one record includes a cache hit and a cache miss example.
