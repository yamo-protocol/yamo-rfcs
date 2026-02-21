# RFC-0013: YAMO Deployment and Operational Model

**Status:** Draft
**Author:** Soverane Labs
**Created:** 2026-02-21
**Updated:** 2026-02-21
**Depends on:** RFC-0008 (Wire Protocol — network interfaces), RFC-0009 (Workspace File Format — filesystem layout), RFC-0011 (MemoryMesh — LanceDB storage)
**Related:** RFC-0006 (Autonomous Kernel — kernel startup sequence), RFC-0004 (Gateway — device auth)

---

## Summary

This RFC specifies the **YAMO Deployment and Operational Model** — the canonical procedure for provisioning, starting, and operating a YAMO system. It covers system topology, the rationale for the polyglot architecture, service startup order, Ed25519 key lifecycle, Raft cluster bootstrap, LanceDB initialization, the observability contract, and the complete environment variable reference.

This RFC is the operational companion to RFC-0008 (which defines the wire protocol) and RFC-0009 (which defines the workspace layout). RFC-0008 specifies *what* messages flow between components; RFC-0013 specifies *how* to set those components up and run them.

---

## Motivation

The YAMO system comprises three language runtimes (Node.js, Elixir/BEAM, local native via LanceDB), communicating over WebSocket/Phoenix Channels with Ed25519 authentication and optional Raft consensus. No single RFC previously described:

1. **Why** the system uses multiple runtimes rather than a single language
2. **In what order** services must start and what each depends on
3. **How** cryptographic keys are provisioned and rotated
4. **How** a multi-node Raft cluster is bootstrapped from scratch
5. **What** the monitoring and alerting contract looks like end-to-end

Operators reading the existing RFCs could understand the protocol but not derive a runbook. RFC-0013 closes that gap.

---

## Specification

### 1. System Topology

```
┌─────────────────────────────────────────────────────────┐
│                    YAMO System                          │
│                                                         │
│  ┌──────────────────────────────────┐                   │
│  │       yamo-os (Node.js)          │                   │
│  │  ┌──────────┐  ┌──────────────┐ │                   │
│  │  │ YamoKernel│  │MemoryMesh   │ │                   │
│  │  │ (agent    │  │(@yamo/memory │ │                   │
│  │  │  runtime) │  │ -mesh)       │ │                   │
│  │  └─────┬─────┘  └──────┬───────┘ │                   │
│  │        │               │ LanceDB  │                   │
│  │        │ RFC-0008      │ (local)  │                   │
│  └────────┼───────────────┼──────────┘                   │
│           │ WS/Phoenix    │                              │
│  ┌────────▼───────────────┼──────────┐                   │
│  │    yamo-bridge (Elixir/BEAM)      │                   │
│  │  ┌──────────┐  ┌──────────────┐  │                   │
│  │  │ Phoenix   │  │ :ra Raft     │  │                   │
│  │  │ Channels  │  │ ClusterSup   │  │                   │
│  │  │ (WS)     │  │ (consensus)  │  │                   │
│  │  └──────────┘  └──────────────┘  │                   │
│  │  ┌──────────┐  ┌──────────────┐  │                   │
│  │  │ AgentReg  │  │ SkillPending │  │                   │
│  │  │ (ETS)    │  │ Registry(ETS)│  │                   │
│  │  └──────────┘  └──────────────┘  │                   │
│  └───────────────────────────────────┘                   │
│                                                         │
│  ┌──────────────────────────────────┐                   │
│  │  External (optional)             │                   │
│  │  - yamo-explorer (Next.js)       │                   │
│  │  - yamo-landing (web UI)         │                   │
│  │  - Nginx reverse proxy           │                   │
│  └──────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

**Component responsibilities:**

| Component | Runtime | Role |
|-----------|---------|------|
| `yamo-os` | Node.js / TypeScript | LLM agent execution, skill dispatch, MemoryMesh I/O |
| `yamo-bridge` | Elixir / BEAM (OTP) | Agent registry, skill routing, Raft consensus, rate limiting |
| LanceDB | Native (Rust, embedded) | Vector storage for MemoryMesh (runs in-process with yamo-os) |
| `yamo-explorer` | Next.js | Optional visualization UI — reads from yamo-bridge via RFC-0008 |
| Nginx | Native | Optional TLS termination and reverse proxy |

---

### 2. Polyglot Architecture Rationale

The system uses two primary runtimes (Node.js and Elixir) for complementary reasons:

**Node.js for the agent kernel (`yamo-os`):**
- LLM SDK ecosystem is JavaScript/TypeScript-first (Anthropic SDK, OpenAI SDK, LangChain)
- LanceDB's primary client is Node.js (Apache Arrow JS bindings)
- MemoryMesh (`@yamo/memory-mesh`) is a Node.js library; moving it to Elixir would require rewriting the entire vector search and embedding pipeline
- TypeScript provides strong typing for the complex skill/agent/kernel domain model

**Elixir/BEAM for the coordination plane (`yamo-bridge`):**
- Phoenix Channels are the de facto standard for real-time WebSocket multiplexing on the BEAM
- OTP supervision trees provide automatic crash recovery for agent registry and skill pending state — properties that are difficult to achieve in Node.js without external infrastructure
- `:ra` (Raft) is a battle-tested OTP library used in RabbitMQ; no equivalent exists in the Node.js ecosystem
- ETS (Erlang Term Storage) provides O(1) concurrent read/write for the agent registry without lock contention
- The BEAM's fault isolation means a misbehaving agent registration cannot crash the coordination plane

**Embedded LanceDB:**
- LanceDB runs in-process with `yamo-os` via native Node.js bindings — no separate database process required
- Local-first design aligns with YAMO's offline-capable, privacy-first architecture (RFC-0009 §4.4)
- For multi-kernel deployments, a shared LanceDB directory on a network filesystem (NFS/S3) can be used; this is not required for single-kernel deployments

**Operational implication:** The polyglot split means `yamo-os` and `yamo-bridge` are separate processes that communicate over the network. RFC-0008 defines their protocol boundary. This RFC defines how to start and connect them.

---

### 3. Service Startup Order

Services MUST start in the following order. Each step's dependency MUST be healthy before the next step proceeds.

```
Step 1: yamo-bridge
  - Start Phoenix endpoint (port 4001 by default)
  - Initialize ETS tables (AgentRegistry, SkillPendingRegistry, RateLimiter)
  - Bootstrap Raft cluster (see §5)
  - Health check: GET http://localhost:4001/health → { status: "ok" }

Step 2: yamo-os
  - Read workspace files (RFC-0009 startup sequence)
  - Pre-boot guard: check BOOTSTRAP.md, execute if present (RFC-0009 §6.3)
  - Connect to yamo-bridge via WebSocket (RFC-0008)
  - Authenticate via Ed25519 challenge-response (RFC-0008 §2.2)
  - Register kernel on kernel:{sessionId} channel
  - Advertise skill capabilities (RFC-0008 §4.4)
  - Begin serving agent requests

Step 3 (optional): yamo-explorer
  - Requires yamo-bridge to be healthy (step 1)
  - Connects to yamo-bridge via RFC-0008 WebSocket protocol
  - No dependency on yamo-os
```

**Readiness vs liveness distinction:**

| Check | Endpoint | Meaning |
|-------|----------|---------|
| Liveness | `GET /health` | Process is running and not deadlocked |
| Readiness | `GET /health?ready=1` | All dependencies connected (Raft cluster formed, channels open) |

`yamo-os` MUST NOT serve agent requests until its readiness check passes on `yamo-bridge`.

---

### 4. Ed25519 Key Provisioning

All yamo-bridge connections require Ed25519 authentication (RFC-0008 §2). This section specifies the key lifecycle.

#### 4.1 Key Generation

Generate a key pair for each `yamo-os` instance:

```bash
# Using Node.js @noble/ed25519 (reference implementation)
node -e "
const { utils } = require('@noble/ed25519');
const privKey = utils.randomPrivateKey();
const pubKey = await ed.getPublicKeyAsync(privKey);
console.log('YAMO_PRIVATE_KEY=' + Buffer.from(privKey).toString('base64'));
console.log('YAMO_PUBLIC_KEY=' + Buffer.from(pubKey).toString('base64'));
"

# Using OpenSSL (alternative)
openssl genpkey -algorithm ed25519 -out yamo_private.pem
openssl pkey -in yamo_private.pem -pubout -out yamo_public.pem
```

#### 4.2 Key Registration

Register the public key with yamo-bridge before starting yamo-os:

```bash
curl -X POST http://localhost:4001/auth/register \
  -H 'Content-Type: application/json' \
  -d '{
    "public_key": "<base64-encoded-ed25519-pubkey>",
    "device_id": "kernel-node1",
    "label": "Production kernel node 1"
  }'
# Response: { "device_token": "...", "expires_at": "2026-02-22T00:00:00Z" }
```

Store the `device_token` in the environment. It expires in 24 hours and MUST be refreshed via `POST /auth/refresh` before expiry.

#### 4.3 Environment Variables for Keys

| Variable | Required | Description |
|----------|----------|-------------|
| `YAMO_PRIVATE_KEY` | Yes | Base64-encoded Ed25519 private key (32 bytes) |
| `YAMO_PUBLIC_KEY` | Yes | Base64-encoded Ed25519 public key (32 bytes) |
| `YAMO_DEVICE_TOKEN` | Yes | JWT device token from `/auth/register` |
| `YAMO_DEVICE_ID` | Yes | Stable device identifier (used in audit logs) |

**Security rules:**
- Private keys MUST NOT be committed to version control
- Private keys SHOULD be stored in environment secrets (GitHub Actions secrets, Kubernetes secrets, Vault)
- Key rotation: generate new key pair → register → update `YAMO_PRIVATE_KEY`/`YAMO_PUBLIC_KEY`/`YAMO_DEVICE_TOKEN` → restart `yamo-os`
- Old keys are automatically invalidated when the device token expires (24h TTL)

#### 4.4 Development Mode (Auth Disabled)

For local development, Ed25519 auth can be disabled:

```bash
# yamo-bridge: disable auth
YAMO_BRIDGE_AUTH_ENABLED=false mix phx.server

# yamo-os: skip key provisioning
YAMO_BRIDGE_AUTH_ENABLED=false npm start
```

Auth MUST be enabled in any environment accessible from outside `localhost`.

---

### 5. Raft Cluster Bootstrap

For single-kernel deployments, Raft runs in single-node mode (no configuration required). For multi-kernel deployments requiring consensus, follow this procedure.

#### 5.1 Single-Node Mode (Default)

Single-node mode requires no configuration. `yamo-bridge` bootstraps a single-member Raft cluster automatically on startup when `YAMO_BRIDGE_RAFT_NODES` is unset or contains only one entry.

```bash
# No Raft configuration needed — single node boots automatically
mix phx.server
```

#### 5.2 Multi-Node Cluster Bootstrap

For a 3-node cluster (minimum for fault tolerance — can survive 1 node failure):

**Prerequisites:**
- Each node must have a unique Erlang node name
- Nodes must be network-reachable to each other on the Erlang distribution port (default: 4369)
- Erlang cookie must be identical across all nodes

```bash
# Node 1
YAMO_BRIDGE_RAFT_NODES="yamo@node1,yamo@node2,yamo@node3" \
  RELEASE_NODE="yamo@node1" \
  RELEASE_COOKIE="shared_secret_cookie" \
  mix phx.server

# Node 2
YAMO_BRIDGE_RAFT_NODES="yamo@node1,yamo@node2,yamo@node3" \
  RELEASE_NODE="yamo@node2" \
  RELEASE_COOKIE="shared_secret_cookie" \
  mix phx.server

# Node 3
YAMO_BRIDGE_RAFT_NODES="yamo@node1,yamo@node2,yamo@node3" \
  RELEASE_NODE="yamo@node3" \
  RELEASE_COOKIE="shared_secret_cookie" \
  mix phx.server
```

**Bootstrap sequence (internal, per RFC-0008 §9):**

```
1. Parse YAMO_BRIDGE_RAFT_NODES → build server_id list
2. IF node count >= 2: multi-node mode
   a. Start :ra application
   b. :ra.start_cluster(:yamo_cluster, initial_members, YamoBridge.Raft.Machine, [])
   c. First listed node triggers election (becomes leader candidate)
   d. Cluster is healthy when quorum (⌊n/2⌋ + 1 nodes) have joined
3. ELSE: single-node mode (:ra.start_server with self as sole member)
```

**Verifying cluster health:**

```bash
curl http://localhost:4001/rpc \
  -d '{"jsonrpc":"2.0","method":"cluster.status","params":{},"id":1}'
# Expected: { "result": { "leader": "yamo@node1", "members": [...], "quorum": true } }
```

#### 5.3 Node Recovery

When a node crashes and restarts, it rejoins the existing cluster automatically — `:ra` persists cluster membership in `/tmp/yamo_ra_data` (configurable via `YAMO_RA_DATA_DIR`). The node does NOT need to be listed first or trigger an election on restart.

```bash
# Override WAL data directory (recommended for production: use persistent storage)
YAMO_RA_DATA_DIR=/var/lib/yamo/raft mix phx.server
```

---

### 6. LanceDB Initialization

LanceDB is embedded in `yamo-os` via `@lancedb/lancedb` Node.js bindings. No separate database process is required.

#### 6.1 Database Directory

LanceDB stores data in a local directory. Default: `~/.yamo/memory-mesh/db`.

```bash
# Override via environment variable
YAMO_MEMORY_DB_DIR=/var/lib/yamo/lancedb npm start
```

#### 6.2 First-Run Initialization

On first start, `MemoryMesh.init()` creates the database directory and all required tables:

```
~/.yamo/memory-mesh/db/
├── memory_entries/          # V1 or V2 schema (RFC-0011 §2)
├── yamo_blocks/             # Audit log (RFC-0011 §2.3)
└── synthesized_skills/      # Skill table (RFC-0011 §2.4)
```

No manual schema migration is required for first-run. Schema detection (`isSchemaV2()`) handles V1→V2 migration automatically on startup (RFC-0011 §11).

#### 6.3 Backup and Restore

LanceDB stores data as Apache Arrow IPC files. To back up:

```bash
# Backup
tar -czf yamo-memory-$(date +%Y%m%d).tar.gz ~/.yamo/memory-mesh/db/

# Restore
tar -xzf yamo-memory-20260221.tar.gz -C ~/
```

For multi-kernel deployments sharing a LanceDB instance, the database directory can be placed on a network filesystem. Concurrent writes from multiple `yamo-os` instances are NOT safe without external locking — this configuration is not recommended for current versions.

---

### 7. Observability Contract

#### 7.1 Health Endpoints

Both `yamo-bridge` and `yamo-os` expose health endpoints.

**yamo-bridge (Elixir):**

```
GET /health
Response: { "status": "ok", "uptime_ms": <integer> }
HTTP 200: healthy
HTTP 503: unhealthy (Raft cluster not formed, ETS tables not ready)
```

**yamo-os (Node.js):**

```
GET /health
Response: { "status": "ok" | "degraded" | "error", "bridge": "connected" | "disconnected", "uptime_ms": <integer> }
HTTP 200: ok or degraded (bridge disconnected — local-only mode)
HTTP 503: error (kernel not initialized)
```

#### 7.2 Metrics Endpoint

```
GET /metrics
Content-Type: text/plain; version=0.0.4 (Prometheus format)
```

**Required metrics (both services):**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `yamo_agents_active` | Gauge | `kernel_id` | Currently active agents |
| `yamo_skill_invocations_total` | Counter | `skill_name`, `status` | Total skill invocations |
| `yamo_skill_rate_limited_total` | Counter | `session_id` | Rate limit hits |
| `yamo_bridge_connected` | Gauge | `kernel_id` | 1 if bridge connected, 0 if local-only |
| `yamo_raft_leader` | Gauge | `node` | 1 if this node is Raft leader |
| `yamo_memory_entries_total` | Gauge | — | Total MemoryMesh entries |
| `yamo_heartbeat_last_run_ts` | Gauge | — | Unix timestamp of last heartbeat |

**Phase 0 limitation — `yamo_memory_entries_total`:**
LanceDB/MemoryMesh runs in-process with `yamo-os` (Node.js). `yamo-bridge` (Elixir) has
no direct read path into LanceDB and therefore always reports `yamo_memory_entries_total 0`.
The authoritative value is exposed by `yamo-os GET /metrics`.

Phase 1+ resolution: `yamo-os` SHOULD expose `GET /memory/count` (JSON: `{"count": N}`);
`yamo-bridge` polls this endpoint on a configurable interval and caches via
`Metrics.set(:memory_entries_total, n)`. Until then, consumers MUST use `yamo-os /metrics`
for this gauge.

#### 7.3 Log Levels and Format

Both services use structured logging with the following levels:

| Level | Elixir | Node.js | Usage |
|-------|--------|---------|-------|
| `debug` | `:debug` | `debug` | Detailed protocol traces (disabled in production) |
| `info` | `:info` | `info` | Normal operations (skill invocations, agent registration) |
| `warning` | `:warning` | `warn` | Degraded state (local-only mode, rate limit hits) |
| `error` | `:error` | `error` | Failures (bridge disconnect, skill error, auth failure) |

**yamo-bridge log format** (Elixir Logger):
```
HH:MM:SS.mmm [kernel_id=<id>] [agent_id=<id>] [level] message
```

**yamo-os log format** (JSON structured):
```json
{ "ts": "ISO-8601", "level": "info", "msg": "...", "kernelId": "...", "agentId": "..." }
```

#### 7.4 Alerting Recommendations

| Alert | Condition | Severity |
|-------|-----------|----------|
| Bridge disconnected | `yamo_bridge_connected == 0` for > 60s | High |
| Raft quorum lost | `cluster.status` returns `quorum: false` | Critical |
| Rate limit saturation | `yamo_skill_rate_limited_total` rate > 10/min per session | Warning |
| Heartbeat overdue | `yamo_heartbeat_last_run_ts` > 12h ago | Warning |
| Memory growth | `yamo_memory_entries_total` growth > 10k/day | Info |

---

### 8. Environment Variable Reference

Complete reference for all runtime-configurable parameters across `yamo-bridge` and `yamo-os`.

#### 8.1 yamo-bridge Environment Variables

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `SECRET_KEY_BASE` | (dev default) | **Yes (prod)** | Phoenix endpoint secret — MUST be changed in production (64+ chars) |
| `YAMO_BRIDGE_AUTH_ENABLED` | `false` | No | Enable Ed25519 device authentication (RFC-0008 §2) |
| `YAMO_SKILL_RATE_LIMIT` | `60` | No | Max skill invocations per 60-second window per session |
| `YAMO_BRIDGE_RAFT_NODES` | `""` | No | Comma-separated Erlang node names for multi-node Raft (e.g. `yamo@node1,yamo@node2,yamo@node3`) |
| `YAMO_RA_DATA_DIR` | `/tmp/yamo_ra_data` | No | `:ra` WAL data directory — use persistent storage in production |
| `PORT` | `4001` | No | HTTP/WebSocket listen port |
| `RELEASE_NODE` | — | Yes (multi-node) | Erlang node name (e.g. `yamo@node1`) |
| `RELEASE_COOKIE` | — | Yes (multi-node) | Erlang distribution cookie — MUST match across all cluster nodes |

#### 8.2 yamo-os Environment Variables

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `YAMO_BRIDGE_URL` | `ws://localhost:4001` | No | WebSocket URL for yamo-bridge connection |
| `YAMO_BRIDGE_AUTH_ENABLED` | `false` | No | Enable Ed25519 auth on bridge connection |
| `YAMO_PRIVATE_KEY` | — | Yes (auth enabled) | Base64-encoded Ed25519 private key |
| `YAMO_PUBLIC_KEY` | — | Yes (auth enabled) | Base64-encoded Ed25519 public key |
| `YAMO_DEVICE_TOKEN` | — | Yes (auth enabled) | JWT device token from `/auth/register` |
| `YAMO_DEVICE_ID` | — | Yes (auth enabled) | Stable device identifier |
| `YAMO_MEMORY_DB_DIR` | `~/.yamo/memory-mesh/db` | No | LanceDB storage directory |
| `YAMO_WORKSPACE_DIR` | `~/.yamo/workspace` | No | Workspace files root (RFC-0009) |
| `YAMO_SKILL_RATE_LIMIT` | `60` | No | Client-side rate limit awareness (matches bridge default) |
| `YAMO_ENABLE_LLM` | `false` | No | Enable LLM-enhanced reflection in MemoryMesh (RFC-0011 §7) |
| `YAMO_LLM_PROVIDER` | — | If LLM enabled | `openai` \| `ollama` \| `cohere` |
| `YAMO_LLM_API_KEY` | — | If cloud LLM | API key for LLM provider |
| `SMORA_ENABLED` | `false` | No | Enable S-MORA 5-layer RAG pipeline (RFC-0012) |
| `SMORA_HYDE_ENABLED` | `true` | No | Enable HyDE-Lite query expansion |
| `ANTHROPIC_API_KEY` | — | Yes (agent exec) | Anthropic API key for LLM agent execution |

#### 8.3 Nginx Reverse Proxy (Optional)

For TLS termination and public access, place Nginx in front of yamo-bridge:

```nginx
server {
    listen 443 ssl;
    server_name yamo.example.com;

    ssl_certificate     /etc/ssl/yamo.crt;
    ssl_certificate_key /etc/ssl/yamo.key;

    location / {
        proxy_pass http://localhost:4001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`X-Forwarded-For` is passed to yamo-bridge for rate limiting and audit logging. Configure `check_origin: false` in `yamo-bridge` config when behind a reverse proxy.

---

### 9. Minimal Single-Node Quickstart

```bash
# 1. Start yamo-bridge (no auth, no Raft config needed)
cd yamo-bridge
mix deps.get && mix phx.server &

# 2. Wait for bridge readiness
curl http://localhost:4001/health  # → { "status": "ok" }

# 3. Initialize workspace
mkdir -p ~/.yamo/workspace
# Create SOUL.md, AGENTS.md, USER.md (see RFC-0009 §2)

# 4. Start yamo-os
cd yamo-os
npm install
ANTHROPIC_API_KEY=sk-... npm start
```

For production deployments with auth enabled, follow §4 (key provisioning) before starting yamo-os.

---

## Rationale

### Why not Docker Compose as the canonical deployment?

Docker Compose is an implementation detail, not a specification. This RFC specifies the operational model at the protocol level — service dependencies, startup order, health checks, environment variables. Any orchestration tool (Docker Compose, Kubernetes, systemd) can implement this spec. A Docker Compose reference implementation is provided separately in the repository.

### Why is the Raft data directory `/tmp` by default?

The default `/tmp/yamo_ra_data` is intentional for development — it makes it easy to wipe state and start fresh. Production deployments MUST set `YAMO_RA_DATA_DIR` to a persistent path. The default is chosen to be safe-to-lose in dev, not appropriate for production.

### Why is LanceDB not a separate service?

LanceDB's embedded model is a deliberate architectural choice: no network hop for memory operations, no separate process to manage, no authentication layer between the agent and its memory. The cost is that multiple `yamo-os` instances cannot share one LanceDB safely without external coordination. For multi-kernel deployments requiring shared memory, RFC-0011 §5.4 describes MemoryMesh's relationship to workspace files; a future RFC will specify distributed memory coordination.

---

## Backwards Compatibility

This RFC is additive documentation — it formalizes operational procedures that are already implemented. No code changes are required. Existing deployments conforming to the environment variable conventions described here are fully compatible.

---

## Security Considerations

- **`SECRET_KEY_BASE`** MUST be changed from the development default in any production deployment. A compromised secret key allows session forgery against the Phoenix endpoint.
- **`RELEASE_COOKIE`** MUST be a strong random value in multi-node deployments. The Erlang distribution protocol provides no application-level encryption; all nodes with the same cookie can execute arbitrary Erlang on each other.
- **`YAMO_RA_DATA_DIR`** should use a path with restricted permissions (`chmod 700`). The Raft WAL contains agent registry state that should not be world-readable.
- **LanceDB directory** should use restricted permissions. Memory entries may contain PII (RFC-0011 §6.3).
- **Private keys** (`YAMO_PRIVATE_KEY`) MUST be stored in a secrets manager in production, not in `.env` files committed to version control.

---

## Reference Implementation

- `yamo-bridge/config/config.exs` — bridge environment variable handling
- `yamo-bridge/lib/yamo_bridge/application.ex` — service supervision tree and startup order
- `yamo-bridge/lib/yamo_bridge/raft/cluster_supervisor.ex` — Raft bootstrap procedure
- `yamo-os/lib/kernel/kernel.ts` — `YamoKernel` startup sequence
- `yamo-os/lib/bridge/bridge-client.ts` — bridge connection and auth
- `yamo-memory-mesh/` — LanceDB initialization (`MemoryMesh.init()`)

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-21 | Initial draft — system topology, polyglot rationale, startup order, Ed25519 key lifecycle, Raft cluster bootstrap, LanceDB initialization, observability contract, environment variable reference |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
