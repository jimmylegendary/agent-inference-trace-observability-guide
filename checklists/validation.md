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
- [ ] `litellm.call_id` present.
- [ ] Join works for one turn with at least three LLM calls.

## Cache Signals

- [ ] vLLM metrics expose prefix/KV cache signals.
- [ ] LMCache metrics expose retrieve/store/lookup counts and latencies.
- [ ] KV events expose block hashes.
- [ ] Tier-level source of truth exists for L0/L1/L2 movement.
- [ ] Storage trace recording produces replayable artifacts.

## Output

- [ ] Trace Builder emits JSONL records matching `schemas/input-trace.schema.json`.
- [ ] Each emitted record has evidence pointers to spans/events/metrics/traces.
- [ ] At least one record includes a cache hit and a cache miss example.
