# RFC-0008: YAMO Wire Protocol v1

**Status:** Draft
**Authors:** Soverane Labs
**Created:** 2026-02-20
**Implemented by:** `yamo-bridge/` (Elixir), `lib/bridge/bridge-client.ts` (Node.js)

---

## Summary

This RFC defines the **YAMO Wire Protocol v1** — the canonical network interface between YAMO OS kernels, the Elixir coordination plane (`yamo-bridge`), and external clients (dashboard, CLI, third-party integrations). It specifies message formats, channel topics, device authentication, and streaming semantics.

All YAMO network interfaces (gateway HTTP, WebSocket channels, bridge sync) MUST conform to this protocol.

---

## Motivation

YAMO OS components communicate over multiple transports today with no unified contract:
- `ChannelManager` uses ad-hoc WebSocket payloads
- `api.ts` uses untyped HTTP JSON
- The Elixir bridge (planned) needs a formal transport contract before implementation

RFC-0008 establishes that contract, enabling interoperability between Node.js kernels, the BEAM coordination plane, yamo-explorer, and third-party agents.

---

## Specification

### 1. Transport

**Primary:** WebSocket (Phoenix Channels on the BEAM side, raw WebSocket on the Node.js gateway side)
**Secondary:** HTTP/REST for health checks, one-off RPC, and metrics endpoints
**Serialization:** MessagePack (binary, primary) with JSON fallback for human-readable debugging

> **RFC-0005 §1.1 compliance:** MessagePack and JSON used at the network protocol layer are explicitly exempt from the Zero-JSON Mandate.

---

### 2. Device Authentication

All connections to YAMO network interfaces MUST authenticate using **Ed25519 key pairs**.

#### 2.1 Key Registration

```
POST /auth/register
Body: { public_key: base64(ed25519_pubkey), device_id: string, label: string }
Response: { device_token: string, expires_at: ISO-8601 }
```

The `device_token` is a signed JWT (`RS256`) issued by the gateway, embedding `device_id` and `public_key` fingerprint. TTL: 24 hours.

#### 2.2 Challenge-Response Handshake

On WebSocket/Channel connect:

```
Client → Server: { type: "auth", device_token: string, challenge_sig: base64 }
```

Where `challenge_sig = Ed25519.sign(private_key, challenge_bytes)` and `challenge_bytes` are issued by the server on the initial HTTP upgrade response header `X-YAMO-Challenge`.

```
Server → Client: { type: "auth_ok", kernel_id: string, session_id: string }
             OR: { type: "auth_fail", reason: string }
```

#### 2.3 Token Refresh

Clients MUST refresh the `device_token` before expiry using:
```
POST /auth/refresh
Header: Authorization: Bearer <current_device_token>
Response: { device_token: string, expires_at: ISO-8601 }
```

---

### 3. Phoenix Channel Topics

All real-time communication uses the following named topics. Clients subscribe to relevant topics after authentication.

| Topic | Direction | Description |
|---|---|---|
| `agent:lobby` | Server → Client | Agent registry broadcasts (register, update, kill) |
| `agent:{id}` | Bidirectional | Per-agent events scoped to a single agent |
| `kernel:{id}` | Bidirectional | Per-kernel heartbeat, drift, and evolution events |
| `cluster:consensus` | Server → Client | Raft leadership changes, quorum events, cluster health |
| `skill:evolution` | Server → Client | Skill promotion/demotion broadcast to all kernels |
| `memory:query` | Bidirectional | Routed memory queries (Phase 3 memory routing) |

#### 3.1 Topic Join

```json
{ "topic": "agent:lobby", "event": "phx_join", "payload": { "session_id": "..." }, "ref": "1" }
```

Server acknowledges:
```json
{ "topic": "agent:lobby", "event": "phx_reply", "payload": { "status": "ok" }, "ref": "1" }
```

---

### 4. Message Formats (MessagePack schema)

All payloads are MessagePack-encoded maps. Field names are strings. Timestamps are ISO-8601 strings.

#### 4.1 Agent Registry Events (`agent:lobby`)

**register_agent**
```
{
  type:       "register_agent",
  id:         string,          // Agent UUID
  kernel_id:  string,          // Owning kernel
  skill:      string,          // Active skill name
  status:     "idle",
  meta:       map,             // Arbitrary key-value metadata
  ts:         ISO-8601
}
```

**update_agent**
```
{
  type:           "update_agent",
  id:             string,
  status:         "idle" | "running" | "paused" | "failed",
  context_patch:  map,         // Incremental diff of execution context
  raft_log_index: integer,     // Monotonically increasing; LWW conflict resolution
  ts:             ISO-8601
}
```

**kill_agent**
```
{
  type:   "kill_agent",
  id:     string,
  reason: string,
  ts:     ISO-8601
}
```

#### 4.2 Kernel Events (`kernel:{id}`)

**heartbeat**
```
{
  type:           "heartbeat",
  kernel_id:      string,
  agent_count:    integer,
  entropy:        float,       // Current Shannon entropy from EntropyMeter
  drift_score:    float,       // Current drift from EvolutionGovernor
  memory_count:   integer,     // MemoryMesh total memories
  ts:             ISO-8601
}
```

**bridge_connected / bridge_disconnected**
```
{ type: "bridge_connected" | "bridge_disconnected", kernel_id: string, ts: ISO-8601 }
```

#### 4.3 Consensus Events (`cluster:consensus`)

**leader_elected**
```
{
  type:      "leader_elected",
  node:      string,           // Erlang node name
  term:      integer,          // Raft term
  ts:        ISO-8601
}
```

**quorum_lost / quorum_restored**
```
{ type: "quorum_lost" | "quorum_restored", node_count: integer, ts: ISO-8601 }
```

#### 4.4 Kernel Channel Events (`kernel:{id}` — inbound from client)

**advertise_capabilities** (client → server, topic `kernel:{id}`)
```
{
  type:   "advertise_capabilities",
  skills: string[]             // List of skill names this kernel can handle
}
```
Response (phx_reply):
```
{ status: "ok", response: { registered: string[] } }
```
The bridge stores the skills list in `AgentRegistry` under the kernel's `capabilities` key. This enables skill routing via `AgentRegistry.find_by_skill/1`.

#### 4.5 Skill Error (Dead-Letter) Events (`agent:lobby`)

**skill_error** (server → client, broadcast on agent:lobby)
```
{
  type:       "skill_error",
  request_id: string,          // Matches the original invoke request_id
  handler_id: string,          // The skill handler that was unavailable
  reason:     "handler_unavailable" | "timeout" | "rate_limited",
  ts:         ISO-8601
}
```
Emitted when a skill invocation cannot be delivered (handler not registered, dead-letter). The client MUST reject any pending `invokeSkill()` promise matching `request_id` with an error indicating the failure reason.

#### 4.6 Skill Evolution Events (`skill:evolution`)

**skill_promoted**
```
{
  type:           "skill_promoted",
  skill_id:       string,
  skill_name:     string,
  fitness_score:  float,
  originating_kernel: string,
  ts:             ISO-8601
}
```

**skill_demoted**
```
{
  type:       "skill_demoted",
  skill_id:   string,
  reason:     string,
  ts:         ISO-8601
}
```

#### 4.7 Audit Event Format

Every state-mutating operation MUST emit an audit event anchored to the Raft log index.

```
{
  type:           "audit",
  event:          string,          // e.g. "agent_registered", "skill_promoted"
  actor:          string,          // kernel_id or device_id
  target:         string,          // Agent ID, skill ID, etc.
  raft_log_index: integer,         // Provenance anchor (0 if bridge unavailable)
  payload:        map,             // Event-specific data
  ts:             ISO-8601
}
```

Audit events are forwarded to `AuditQueue` in the Node.js kernel for YAMO block anchoring.

---

### 5. HTTP/REST Endpoints

All HTTP endpoints use JSON (network layer exemption per RFC-0005 §1.1).

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness check → `{ status: "ok", uptime_ms: integer }` |
| `GET` | `/metrics` | Prometheus-compatible metrics |
| `GET` | `/agents` | Current agent registry snapshot |
| `GET` | `/agents/:id` | Single agent state |
| `POST` | `/auth/register` | Register device, get token |
| `POST` | `/auth/refresh` | Refresh device token |
| `POST` | `/rpc` | JSON-RPC 2.0 one-off commands (see §6) |

---

### 6. JSON-RPC 2.0 Gateway

One-off commands that do not require streaming use JSON-RPC 2.0 over HTTP POST to `/rpc`.

```json
{
  "jsonrpc": "2.0",
  "method": "agent.spawn",
  "params": { "skill": "skill_name", "context": {} },
  "id": 1
}
```

**Methods:**

| Method | Params | Description |
|---|---|---|
| `agent.spawn` | `{ skill, context }` | Spawn a new agent GenServer |
| `agent.kill` | `{ id, reason }` | Kill an agent by ID |
| `agent.get` | `{ id }` | Get current agent state |
| `cluster.status` | `{}` | Raft cluster health and quorum state |
| `kernel.list` | `{}` | All connected kernels with heartbeat status |

---

### 7. Streaming Response Protocol

Long-running operations (LLM inference, S-MORA retrieval) use **server-sent events** over the `agent:{id}` Phoenix Channel topic.

```
Server → Client: { type: "stream_start", request_id: string, ts: ISO-8601 }
Server → Client: { type: "stream_chunk", request_id: string, chunk: string, seq: integer }
Server → Client: { type: "stream_end",   request_id: string, total_chunks: integer, ts: ISO-8601 }
```

On error:
```
Server → Client: { type: "stream_error", request_id: string, error: string, ts: ISO-8601 }
```

Clients MUST handle out-of-order chunks using `seq`. Missing chunks within 5s trigger a `stream_retry` request.

---

### 8. Local-Only Mode Behaviour

When a kernel loses bridge connectivity it enters **local-only mode**:

- All reads served from local cache (stale tolerance: 500ms TTL)
- Writes to agent registry queued in-memory (max 1000 ops, oldest evicted)
- Commands requiring cluster consensus (`agent.spawn`, `agent.kill`) return error `{ code: -32001, message: "bridge_unavailable" }`
- Skill execution and memory retrieval continue uninterrupted
- Bridge reconnect attempted every 10 seconds with exponential backoff (max 60s)

---

## Backwards Compatibility

- **Breaking:** Replaces ad-hoc `ChannelManager` WebSocket payloads with typed MessagePack frames
- **Migration path:** `ChannelManager` adapters updated in Phase 2 to emit RFC-0008 compliant messages
- **Existing HTTP API** (`api.ts`) is unchanged in Phase 1; REST endpoints added incrementally

---

## Security Considerations

- **Ed25519** provides 128-bit security with small key and signature sizes (32/64 bytes)
- **Challenge-response** prevents replay attacks on connection establishment
- **Raft log index** provides tamper-evident audit provenance
- **Local-only mode** preserves availability during bridge partition (no data loss, queued writes)
- **MessagePack** reduces injection surface compared to JSON (binary format, no string evaluation)

---

## Reference Implementation

- **Coordination plane:** `yamo-bridge/` (Elixir/Phoenix, planned Phase 1)
- **Node.js client:** `yamo-os/lib/bridge/bridge-client.ts` (planned Phase 1)
- **Phase 0 PoC:** Single GenServer + bare Phoenix Channel, no Raft

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-20 | Initial draft — transport, device auth, Phoenix channel topics, message formats, HTTP endpoints, JSON-RPC, streaming, local-only mode |
| 0.1.1 | 2026-02-21 | Add `advertise_capabilities` inbound kernel event (§4.4) and `skill_error` dead-letter broadcast event (§4.5); renumber §4.5 Audit Event Format to §4.7 |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
