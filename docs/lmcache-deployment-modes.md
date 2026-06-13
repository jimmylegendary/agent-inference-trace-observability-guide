# LMCache Deployment Modes

LMCache currently has two practical deployment paths with vLLM.

## Decision

Use MP mode for production-oriented observability and shared cache behavior.
Use non-MP mode for quick single-process validation and vLLM KV event stream
testing.

| Mode | Connector | Best For | Do Not Assume |
| --- | --- | --- | --- |
| Non-MP / in-process | `LMCacheConnectorV1` | Quick validation, vLLM `--kv-events-config` ZMQ stream, block lifecycle events | Shared cache across multiple vLLM processes |
| MP / multiprocess | `LMCacheMPConnector` + `lmcache server` | Shared host-side L1 cache, management endpoints, MP observability, L1/L2 tier behavior | Same vLLM KV event stream as non-MP |

## Non-MP Mode

Use this when the agent must first prove:

- vLLM can connect to LMCache.
- `BlockStored` / `BlockRemoved` events appear on the vLLM KV events ZMQ topic.
- `block_hash` / `parent_block_hash` can be joined with request windows.

Config:

- [config/non-mp-lmcache.yaml](../config/non-mp-lmcache.yaml)
- [config/non-mp-vllm.env.example](../config/non-mp-vllm.env.example)

Important fact:

```text
LMCacheConnectorV1.take_events() emits connector KV events.
```

Therefore this is the direct path for vLLM `--kv-events-config`.

## MP Mode

Use this when the agent needs production-like cache behavior:

- standalone LMCache server
- shared host-side L1 cache
- multiple vLLM processes or DP ranks using one cache layer
- management/observability endpoints
- L1/L2 adapter behavior
- storage trace recording/replay

Config:

- [config/mp-lmcache-server.env.example](../config/mp-lmcache-server.env.example)
- [config/mp-vllm.env.example](../config/mp-vllm.env.example)

Important fact:

```text
LMCacheMPConnector is the connector for MP mode.
The LMCache MP server default ZMQ port is 5555.
This ZMQ channel is connector/server communication, not the vLLM kv-events topic.
```

Version-specific connector rule:

```text
For vLLM >= 0.20.0, prefer the LMCache-shipped connector implementation by
setting kv_connector_module_path="lmcache.integration.vllm.lmcache_mp_connector".
Without this field, LMCacheMPConnector may resolve to the implementation bundled
inside vLLM rather than the newer LMCache package implementation.
```

Current caveat:

```text
LMCacheMPConnector.take_events() returns an empty iterable in current vLLM source.
```

Do not rely on vLLM `--kv-events-config` to provide the same event stream in MP
mode until this is explicitly validated in the target version.

## Recommended Company Flow

1. Validate non-MP event stream first.
2. Validate MP cache behavior separately.
3. Build the Trace Builder with two adapters:
   - `vllm_kv_events_adapter` for non-MP ZMQ KV event stream.
   - `lmcache_mp_observability_adapter` for MP metrics/logging/tracing.
4. Add `lmcache_chunk_statistics_adapter` only as offline chunk-reuse/hash
   evidence, not as a per-request lineage source unless request IDs are present
   in the deployed artifact.
5. Use MP mode as the production path.
6. Keep non-MP as a narrow event-stream test harness.
7. If complete per-request block-hash lists are mandatory, add a version-pinned
   instrumentation patch at the vLLM/LMCache boundary and record its exact
   version in the validation report.

## Validation Signals

### Non-MP

- vLLM emits KV events on configured ZMQ topic.
- Events contain block hash lineage.
- Repeated prompt produces expected hit/store behavior.

### MP

- `lmcache server` is reachable.
- vLLM connects with `LMCacheMPConnector`.
- For vLLM >= 0.20.0, vLLM uses the LMCache-shipped connector module path.
- MP metrics expose lookup/retrieve/store and L1/L2 behavior.
- Storage trace recording produces replayable artifacts when enabled.
- Cache hit stats appear in `KVTransferParams`, response metadata, logs, traces,
  or storage trace artifacts only if the target version exposes them.
- Request logs include `Reqid` lines before the Trace Builder marks hit/load
  counts as request-level.
