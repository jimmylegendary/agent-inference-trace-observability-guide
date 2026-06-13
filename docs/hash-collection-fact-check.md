# Hash Collection Fact Check

This page records what is currently safe to claim about collecting request IDs,
KV block hashes, cache hits, and LMCache timing for input trace generation.

Last checked: 2026-06-14 KST.

## Bottom Line

Use `X-Request-Id` as the request-level join spine, but do not claim that public
vLLM plus LMCache MP automatically emits a complete per-request list of all block
hashes and tier movements.

The safe architecture is:

```text
OpenClaw openclaw_llm_call_id
  -> LiteLLM request metadata/header
    -> vLLM X-Request-Id / request_id
      -> vLLM Request.block_hashes where instrumented or exposed
      -> LMCache Reqid logs and request/tier metrics where exposed
      -> Trace Builder evidence merge
```

## Confirmed

### vLLM can use `X-Request-Id` as its request ID input

The vLLM OpenAI serving code has `_base_request_id()` which reads the
`X-Request-Id` request header and falls back to a generated UUID. vLLM docs also
document `X-Request-Id` as the supported extra HTTP header, enabled by
`--enable-request-id-headers` for response header behavior.

Operational rule:

- Send `X-Request-Id: <openclaw_llm_call_id>` from LiteLLM or the upstream
  client where possible.
- Keep `--enable-request-id-headers` on in validation so the response echoes the
  ID and the join can be tested.
- Be aware that vLLM documents high-QPS overhead for request ID headers.

### vLLM `Request` objects carry `block_hashes`

Current vLLM V1 `Request` initializes `self.block_hashes` and builds it from the
engine request. The KV cache manager uses `request.block_hashes` for longest
prefix-cache-hit lookup and allocates/free blocks by `request.request_id`.

This supports the claim that `request_id` and block hashes exist in the same
runtime object graph.

It does not by itself prove that stock vLLM exports the full list of block hashes
per request to logs, traces, or metrics.

### vLLM prefix-cache hashes should use deterministic serialization

For `vllm serve`, use:

```bash
--prefix-caching-hash-algo sha256_cbor
```

vLLM documents `sha256_cbor` as reproducible and cross-language compatible.
`sha256` is the default in newer vLLM versions, but it uses Python pickle
serialization and may not be reproducible across Python or vLLM versions.

### LMCache emits request-level hit/load logs with `Reqid`

LMCache docs show request-level log lines like:

```text
LMCache INFO: Reqid: <id>, Total tokens <n>, LMCache hit tokens: <m>, need to load: <k>
```

This is strong evidence for request-level cache hit/load summaries when the
target LMCache/vLLM integration logs this path.

### LMCache metrics include useful latency and movement timing

LMCache documents metrics such as:

- `time_to_retrieve`
- `time_to_store`
- `time_to_lookup`
- `retrieve_to_gpu_time`
- `store_from_gpu_time`
- `store_put_time`
- `remote_backend_batched_get_blocking_time`
- `p2p_time_to_transfer`
- storage event counts
- chunk statistics metrics

These are useful for simulation trace construction, but Prometheus metrics remain
aggregate/windowed unless the deployed version exposes request-scoped labels or
separate request logs/traces. Do not put high-cardinality request IDs into
Prometheus labels.

### LMCache chunk statistics `file_hash` records chunk reuse evidence

LMCache chunk statistics supports:

```yaml
enable_chunk_statistics: true
chunk_statistics_strategy: "file_hash"
chunk_statistics_auto_start_statistics: true
extra_config:
  chunk_statistics_file_output_dir: "/path/to/output"
  chunk_statistics_file_rotation_size: 104857600
  chunk_statistics_file_max_count: 100
```

This is valid as an offline chunk-hash evidence channel.

Do not describe it as a guaranteed per-request block-hash export. The documented
purpose is chunk reuse statistics and offline hash files, not a complete
request-to-block lineage stream.

## Not Confirmed

### Stock MP mode complete per-request block-hash export

The current repo already separates non-MP `LMCacheConnectorV1` from MP
`LMCacheMPConnector`. Keep that separation.

Safe statements:

- Non-MP with `LMCacheConnectorV1` is the direct validation path for vLLM KV
  event publication.
- MP with `LMCacheMPConnector` is the production-oriented shared-cache path.
- MP observability should be validated through LMCache server metrics, logs,
  tracing, and storage trace artifacts.

Unsafe statement:

- "MP mode automatically gives a complete per-request block_hash list from
  public metrics/logs alone."

### `LMCacheConnectorV1` command is not MP mode

If the guide says "MP mode" but the command uses:

```json
{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}
```

that is inconsistent. MP mode should use `LMCacheMPConnector` and a running
`lmcache server`.

### Controller `/lookup` location mapping is not part of this repo contract yet

Do not rely on an AIBrix/controller `/lookup` API as a core guide requirement
unless the exact deployed controller, endpoint schema, auth, and response fields
are validated in the company environment.

Keep it as an optional adapter candidate, not as a required path.

## Recommended Collection Plan

### Path 1: Production MP, evidence-based trace builder

Use for Kimi or other production serving once LMCache MP is required.

Collect:

- OpenClaw raw content and `openclaw_llm_call_id`
- LiteLLM routing/cost/usage metadata
- vLLM request spans, request logs, and usage
- LMCache MP logs with `Reqid`
- LMCache MP Prometheus metrics
- LMCache tracing/storage trace artifacts when enabled
- optional chunk statistics file-hash artifacts

Output:

- request-level hit/load summary when `Reqid` logs exist
- aggregate latency/tier metrics with `granularity: aggregate`
- tier movement only when supported by trace/log/storage evidence
- `inferred: true` whenever reconstructing from time windows rather than direct
  per-request events

### Path 2: Non-MP validation harness for block lineage

Use for proving block hash and lineage behavior before production.

Collect:

- vLLM KV events via `--kv-events-config`
- `LMCacheConnectorV1` connector KV events
- repeated prompts with controlled `X-Request-Id`
- deterministic hash configuration

Output:

- block hashes
- parent block hashes
- token IDs if present
- block stored/removed lifecycle

### Path 3: Minimal patch if complete per-request hashes are required

If simulation requires one complete per-request list of block hashes, add a
version-pinned instrumentation patch around the vLLM/LMCache boundary instead of
pretending existing aggregate observability is enough.

Patch target candidates:

- vLLM request/block-hash construction path
- vLLM KV cache manager prefix-hit path
- LMCache adapter path that receives `request_id`, token IDs, and block hashes

Patch output shape:

```json
{
  "x_request_id": "...",
  "vllm_request_id": "...",
  "model": "...",
  "block_hashes": ["..."],
  "parent_block_hashes": ["..."],
  "hit_tokens": 0,
  "need_to_load_tokens": 0,
  "evidence": {
    "source": "vllm_patch",
    "vllm_version": "..."
  }
}
```

## Claims To Avoid

- "LMCache MP gives all hash, location, and latency data externally with no
  patch."
- "Chunk statistics file_hash is a request-level block-hash log."
- "Prometheus metrics are enough to recover per-request tier movement."
- "A Kimi MP deployment can use `LMCacheConnectorV1` and still be called MP."
- "NIM API docs prove internal vLLM block-hash export behavior." They can help
  validate model/API behavior, but not the vLLM/LMCache internals in this guide.

