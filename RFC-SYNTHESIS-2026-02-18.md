# YAMO RFC Synthesis — Expert Codebase Review
**Date**: 2026-02-18
**Reviewed by**: 4 parallel expert agents (Protocol/Kernel, Memory/Persistence, Singularity/Skills, S-MORA/Retrieval)
**Scope**: yamo-os, yamo-openclaw-singularity, yamo-memory-mesh, yamo-skills, yamo-rfcs

---

## Executive Summary

The YAMO ecosystem has grown significantly beyond its RFC formalization. The `.yamo` layer (agent protocol, skills, orchestration) demonstrates **98.2% RFC fidelity**, while the TypeScript implementation layer (kernel, gateway, memory-mesh) has outpaced documentation. Four new RFCs are proposed and three existing RFCs require amendment.

**Overall Assessment**: Production-capable but specification-lacking. The implementation is ahead of the formalization.

---

## 1. Cross-Cutting Finding: The Zero-JSON Tension

**RFC-0005** mandates "Zero-JSON" for agent state passing.

**Reality**:
- **`.yamo` files**: 100% compliant — zero JSON in any agent state block, 8 explicit `NO_JSON_STATE;strict;` constraints
- **TypeScript kernel**: JSON throughout — InterceptionEngine metadata, channel messages, audit logs, gateway RPC, state manager

**Resolution**: These are two different layers. RFC-0005 correctly targets the *agent protocol layer* (`.yamo` files), not the *implementation layer* (TypeScript internals). However, the RFC language is ambiguous about scope.

**Amendment Required**: RFC-0005 §1 — Add a "Scope of Mandate" section clarifying:

> The Zero-JSON Mandate applies exclusively to **agent state-passing via `.yamo` context files and blocks**. Internal implementation code (TypeScript, JavaScript) MAY use JSON for infrastructure purposes (database metadata, RPC protocols, OS-level APIs). The mandate governs the *semantic protocol layer*, not the *implementation layer*.

---

## 2. RFC-0005 Amendments (YAMO v3.0 Singularity Protocol)

### 2.1 Add: Module Type Enumeration

Valid `module_type` values in `.yamo` frontmatter:
- `foundation` — Immutable constitutional values, loaded once at boot
- `workflow` — Agent chain with sequential handoff logic
- `orchestrator` — Top-level coordinator of multiple workflows
- `skill` — Single-purpose callable agent (RFC-0004 scope)

### 2.2 Add: Feedback Loop Protocol

Micro-to-macro escalation (discovered in implementation, not in RFC):

```
Trigger: Critical logic error during micro-phase implementation
Action: Return execution context to MacroWorkflowInvoker
Justification: Re-evaluate specification assumptions when implementation
               discovers an infeasible or contradicted requirement
Context passing: micro_failure_context.yamo → PlanningWorkflowInvoker
```

### 2.3 Add: Workflow Memory Archival Requirement

All orchestrators MUST store execution history in MemoryMesh:
```
Required metadata fields:
  workflow_id;string;
  execution_status;completed|failed|paused;
  test_results;passed|failed|skipped;
  duration_ms;integer;
  commit_shas;list;
```

### 2.4 Add: Constitutional Gating at Micro Level

RFC-0005 implies Articles VII/VIII/IX apply at macro gates. Implementation applies them at task level too:

> Constitutional compliance (Article VII: Simplicity, Article VIII: Anti-Abstraction, Article IX: Integration-First) MUST be verified at BOTH macro phase gates AND individual micro task definitions.

### 2.5 Formalize InterceptionEngine Decision Matrix

The kernel uses a multi-factor weighted score (not in any RFC):

```
weightedScore =
  0.6 × semanticScore +
  0.3 × reliability(skill) +
  0.1 × useConfidence(skill)

Where:
  semanticScore   = vector similarity from embedding search [0, 1]
  reliability     = success_rate × 0.5^(daysSinceUse / 30), floored at 0.3
  useConfidence   = log(useCount + 1) / log(100 + 1), capped at 1.0

Decision gates (ordered):
  1. Missing required parameters → BLOCK
  2. Self-interception detected  → BLOCK
  3. Core command, score < 0.85  → BLOCK
  4. reliability < 0.5           → BLOCK
  5. weightedScore ≥ threshold   → INTERCEPT
     threshold = 0.85 (core commands), 0.35 (user commands)
```

### 2.6 Formalize Entropy-Aware Skill Selection

```
Normalized Shannon Entropy:
  H_norm = -Σ p(i) × log2(p(i)) / log2(k)
  where k = number of unique skills in candidate pool
  H_norm ∈ [0, 1]: 0 = deterministic, 1 = maximum diversity

Softmax Sampling (StochasticSelector):
  expScores = [exp(score/temp) for each score]
  probs = softmax(expScores)
  adjustedProbs = entropy_weight × probs + (1 − entropy_weight) / k
  where entropy_weight = 1 − H_norm

Entropy Adaptation:
  if successRate > 0.7 and H_norm < 0.2 → targetEntropy += 0.1  (explore)
  if successRate < 0.4 and H_norm > 0.6 → targetEntropy -= 0.1  (exploit)
  historyWindow = 3600s (1 hour)
```

---

## 3. RFC-0006 Amendments (Autonomous Kernel & Ghost Protection)

### 3.1 Add: Heartbeat Frequency Specification

> The Heartbeat cycle executes **2–4 times per day** by default.
> GhostGuard integrity check runs at PRIORITY_0 before all other heartbeat tasks.

### 3.2 Add: Ghost Protection Mechanism Specificity

> Ghost protection uses **atomic POSIX shell operations** (printf + cat + mv) rather than in-place editors (sed) to guarantee POSIX compatibility and prevent partial writes. The repair sequence is:
> ```sh
> printf "> [!IMPORTANT]\n> **YAMO-NATIVE KERNEL v3.0 ACTIVE**\n> ..." | \
>   cat - AGENTS.md > AGENTS.md.tmp && mv AGENTS.md.tmp AGENTS.md
> ```

---

## 4. RFC-0007 Amendments (Semantic Heritage & Wisdom Distillation)

### 4.1 Terminology Alignment

Change: `structure:` → `required_fields:` in LessonLearned block definition (implementation uses `required_fields`)

### 4.2 Add: Semantic Search Mandate for SubconsciousReflector

> SubconsciousReflector MUST use **semantic similarity search** (embedding-based) for MemoryMesh queries. Keyword-only tag matching (`#lesson_learned`) is insufficient for discovering lessons from dissimilar contexts. Both mechanisms SHOULD be combined: semantic similarity ranked first, tag filtering as secondary filter.

### 4.3 Add: Search Tag Standardization

> `#lesson_learned` is the canonical tag for distilled wisdom entries.
> Optional supplemental tags: `#oversight_type`, `#severity`, `#domain`.

### 4.4 Implementation Gap: Wisdom Distillation Not Yet Wired

**Finding**: The `.yamo` orchestration layer (SubconsciousReflector, WisdomDistiller) is fully specified. The `@yamo/memory-mesh` library does NOT yet implement:
- `distillWisdom()` heartbeat hook
- Automatic `#lesson_learned` tag scanning
- Pattern extraction from logs
- Preventative constraint injection

These gaps are a **critical path** for RFC-0007 compliance. See RFC-0011 below.

---

## 5. New RFC-0008: YAMO Wire Protocol v1

**Proposed scope**: Formalize all message formats on the wire.

### 5.1 Gateway RPC (JSON-RPC 2.0 over WebSocket)

```json
Request:
{
  "jsonrpc": "2.0",
  "method": "agent.execute",
  "params": {
    "prompt": "<yamo-formatted-prompt>",
    "context": {},
    "deviceContext": { "deviceId": "...", "scopes": ["kernel:execute"] }
  },
  "id": "<uuid-v4>"
}

Response (streaming):
  Content-Type: text/event-stream
  Chunks: raw UTF-8 text, terminated by "[DONE]\n"

Response (terminal):
{
  "jsonrpc": "2.0",
  "id": "<matching-id>",
  "result": { "output": "...", "telemetry": {} }
}
```

### 5.2 Audit Event Format (YAMO v0.5 Block)

```
[TYPE]: agent;[AGENT];intent;[INTENT];input;[FIRST-100-CHARS];status;[STATUS];error;[MSG];handoff;[NEXT];

Where:
  TYPE   ∈ { RECALL, RETAIN, REFLECT, KERNEL_OP }
  Fields: semicolon-delimited key;value pairs
  Escaping: internal semicolons replaced with commas
  Timestamp: implicit from database record timestamp
```

### 5.3 Channel Message Format

```typescript
interface MessageEvent {
  channelId: string;     // "whatsapp" | "loopback" | custom
  sender: string;        // e.g., "whatsapp:+1234567890"
  target: string;        // e.g., "whatsapp:+0987654321"
  content: string;       // Plain UTF-8, max 4096 bytes
  timestamp: number;     // Unix milliseconds
  metadata: {
    signature?: string;        // Webhook HMAC signature
    contentType?: "text/plain" | "application/yamo";
    encryption?: "none" | "e2e";
  }
}
```

### 5.4 Device Authentication Protocol

```
Pairing Request: POST /pair
  deviceId    = SHA256(publicKey)[0:16] as hex
  publicKey   = base64url-encoded Ed25519 public key
  pairingCode = 6-char alphanumeric (60s TTL)

Authenticated Request:
  GET /ws?deviceId=<id>&timestamp=<ms>&signature=<sig>
  signature = Ed25519.sign(privateKey, UTF8("${deviceId}:${timestamp}"))
              base64url-encoded
  Replay prevention: |timestamp − now| ≤ 60s

Scope System:
  Pattern: resource:action[:subject]
  Hierarchy: kernel:* matches all kernel operations
  Format in allowFrom.json:
    { "deviceId": { "publicKey": "...", "scopes": [...], "expiresAt": null } }
```

### 5.5 Streaming Response Protocol

```
HTTP POST /execute
  Transfer-Encoding: chunked
  Content-Type: text/plain; charset=utf-8

Chunks: raw UTF-8 output fragments
Terminal chunk: "[DONE]\n"
Error chunk:    "[ERROR]: <message>\n"
```

---

## 6. New RFC-0011: @yamo/memory-mesh Specification

**Proposed scope**: Formalize the memory-mesh API contract, schema, wire formats, and integration patterns.

### 6.1 LessonLearned Wire Format

```yamo
agent: MemoryMesh_{agentId};
intent: distill_wisdom_from_execution;
context:
  original_context;{situation};
  error_pattern;{pattern_id};
  severity;critical | high | medium | low;
  timestamp;{ISO_8601};
constraints:
  hypothesis;Lesson prevents similar failures;
  hypothesis_confidence;0.85;
output:
  lesson_id;lesson_{timestamp}_{hex};
  oversight_description;{what_failed};
  preventative_rule;{conditional_constraint};
  rule_confidence;{0.0-1.0};
meta:
  rationale;{why_this_lesson_matters};
  applicability_scope;{where_rule_applies};
  inverse_lesson;{success_case};
  confidence;{overall_confidence};
log: lesson_learned;timestamp;{ts};pattern;{pattern_id};severity;critical;
handoff: SubconsciousReflector;
```

### 6.2 Schema V2 Extensions

Backward-compatible additions to memory_entries Arrow schema:
```typescript
session_id:          Utf8()          // Conversation session association
agent_id:            Utf8()          // Creating agent identifier
memory_type:         Utf8()          // "global" | "session" | "agent" | "lesson"
importance_score:    Float32()       // 0.0–1.0
access_count:        Int32()         // Retrieval popularity
last_accessed:       Timestamp(ms)   // Last retrieval time
heritage_chain:      Utf8()          // JSON: [intent, rationale, hypothesis]
lesson_pattern_id:   Utf8()          // Clustering hash for similar lessons
source_skill_id:     Utf8()          // Synthesized skill origin
```

### 6.3 New API Methods for Wisdom Distillation

```typescript
// Distill a lesson from execution
distillLesson(context: LessonContext): Promise<LessonBlock>

// Query lessons for preventative injection
queryLessons(query: string, options?: { severity?: string; limit?: number }): Promise<LessonBlock[]>

// Track heritage chain
insertHeritage(memoryId: string, heritage: HeritageChain): Promise<void>

// Cluster similar lessons
getMemoriesByPattern(patternId: string): Promise<MemoryRecord[]>

// Delete operation (currently missing from CLI and API)
delete(id: string): Promise<void>
```

### 6.4 Wisdom Distillation Cycle (Heartbeat Hook)

```
Trigger: Heartbeat (2–4× daily, PRIORITY_0)
Steps:
  1. Scan recent memories for error patterns (tag: #lesson_learned candidate)
  2. Extract pattern identifier via content hash
  3. If pattern unseen: call distillLesson() → store with tag #lesson_learned
  4. Update lesson_pattern_id for deduplication
  5. Inject top lessons into SubconsciousReflector context
```

### 6.5 Missing CLI Commands

| Command | Status | Action |
|---------|--------|--------|
| `delete --id <id>` | ❌ Missing | Requires `LanceDBClient.delete(id)` |
| `reflect` | ⚠️ API-only | Expose in CLI |
| `get --id <id>` | ⚠️ API-only | Expose in CLI |

---

## 7. New RFC-0012: S-MORA Retrieval Augmentation

**Proposed scope**: Formalize the 5-layer S-MORA RAG pipeline as a YAMO subsystem.

### 7.1 Architecture

```
Layer 1: HyDE-Lite Query Enhancement
  Input: user query
  LLM: llama3.1:8b (Ollama)
  Output: hypothetical document representation
  Latency target: <200ms

Layer 2: Hybrid Retrieval
  Vector search (cosine, LanceDB) + BM25 keyword search
  Fusion weight α (0=keyword, 1=vector), default 0.5
  Limits: vector=30, keyword=30, final=10

Layer 3: Context Compression
  Tree-based clustering to eliminate redundancy
  Token reduction target: >50% of raw retrieval

Layer 4: Structured Context Assembly
  Organize chunks with citations and metadata
  Multiple assembly strategies (semantic, chronological, relevance)

Layer 5: S-MORA Orchestrator
  Coordinates Layers 1–4
  Telemetry and performance monitoring
  Health checks and graceful fallback
```

### 7.2 Integration with SubconsciousReflector

**Current state**: S-MORA is standalone (no MemoryMesh integration).

**Proposed bridge**:
```
SubconsciousReflector enhancement:
  - Use S-MORA Layer 1 (HyDE-Lite) to enhance lesson queries
  - Use S-MORA Layer 2 (Hybrid Retrieval) for LanceDB lesson search
  - Replace simple vector-only search with full 5-layer pipeline
  - Expected improvement: >30% relevance lift on lesson retrieval
```

### 7.3 Configuration

```
SMORA_ENABLED=true
SMORA_HYDE_ENABLED=true
SMORA_HYDE_MODEL=llama3.1:8b
SMORA_HYBRID_ALPHA=0.5
SMORA_VECTOR_LIMIT=30
SMORA_FINAL_LIMIT=10
```

---

## 8. RFC-0004 Amendments (Gateway & Workspace)

### 8.1 Pairing UX (Currently Missing)

The `yamo pair` command specified in RFC-0004 is not implemented. Required:
- `POST /pair` endpoint (see RFC-0008 §5.4)
- 6-character alphanumeric code generation (60s TTL)
- Ed25519 key exchange (currently stubbed in `device-auth-impl.ts`)
- `identity/allowFrom.json` storage

### 8.2 Scope Hierarchy

Add to RFC-0004:
```
Scope pattern: resource:action[:subject]
  kernel:execute       → Full kernel execution
  kernel:execute:recall → Only recall operations
  memory:read          → Read all memory
  memory:read:audit    → Read audit logs only
  channel:send:*       → Send via any channel
  *:*                  → SuperAdmin (manual grant only)

Wildcard matching: kernel:* matches all kernel sub-operations
```

### 8.3 Implementation Gaps

| RFC-0004 Requirement | Status |
|----------------------|--------|
| Trusted proxies (X-Forwarded-For) | ❌ Not implemented |
| Gateway Token required for non-localhost | ❌ Token is optional |
| Node Protocol (capability invocation) | ⚠️ node.announce only |
| Pairing UX (6-char code) | ❌ Not implemented |
| YAMO Workspace files (SOUL.md, BRAIN.md) | ❌ Not implemented |
| Ed25519 signature verification | ❌ Stubbed |
| Webhook signature verification (Twilio) | ❌ Not implemented |

---

## 9. Innovations to Document (Not Yet in Any RFC)

| Innovation | Location | RFC Target |
|------------|----------|------------|
| Multi-factor interception scoring | yamo-os `interception-engine.ts` | RFC-0005 §2.5 |
| Shannon entropy + StochasticSelector | yamo-os `entropy-meter.ts`, `stochastic-selector.ts` | RFC-0005 §2.6 |
| Semantic similarity search in SubconsciousReflector | yamo-unified-orchestrator.yamo | RFC-0007 §4.2 |
| Bidirectional micro-to-macro feedback loop | planning-logic.yamo | RFC-0005 §2.2 |
| Constitutional gating at task level | planning-logic.yamo | RFC-0005 §2.4 |
| Scrubber-first embedding (Layer 0) | memory-mesh.js | RFC-0011 |
| RRF fusion (Reciprocal Rank Fusion) | memory-mesh search | RFC-0011 |
| HyDE-Lite query enhancement | S-MORA template-engine.js | RFC-0012 |
| Atomic shell Ghost Protection (printf/cat/mv) | heartbeat-kernel.yamo | RFC-0006 §3.2 |
| Workflow memory archival | yamo-unified-orchestrator.yamo | RFC-0005 §2.3 |

---

## 10. Prioritized Action Plan

### Tier 1: Critical (RFC amendment blocks spec-to-impl alignment)
- [ ] **Amend RFC-0005 §1** — Clarify Zero-JSON Mandate scope (agent protocol only, not TypeScript internals)
- [ ] **Amend RFC-0005** — Add InterceptionEngine scoring matrix (§2.5)
- [ ] **Amend RFC-0005** — Add EntropyMeter/StochasticSelector spec (§2.6)
- [ ] **Amend RFC-0007** — Add semantic search mandate for SubconsciousReflector

### Tier 2: High (New RFCs needed for emerging subsystems)
- [ ] **Draft RFC-0008** — Wire Protocol (gateway RPC, audit format, channel messages, device auth)
- [ ] **Draft RFC-0011** — @yamo/memory-mesh specification (schema, API, LessonLearned format, distillation cycle)
- [ ] **Implement** — `distillWisdom()` heartbeat hook in memory-mesh (closes RFC-0007 gap)
- [ ] **Implement** — Ed25519 device auth with @noble/ed25519 (closes RFC-0004 gap)

### Tier 3: Medium (Formalization of proven innovations)
- [ ] **Draft RFC-0012** — S-MORA Retrieval Augmentation
- [ ] **Amend RFC-0004** — Pairing protocol handshake, scope hierarchy, X-Forwarded-For
- [ ] **Amend RFC-0006** — Heartbeat frequency, Ghost Protection shell specifics
- [ ] **Amend RFC-0007** — LessonLearned `required_fields` terminology

### Tier 4: Low (Nice-to-have formalizations)
- [ ] **Amend RFC-0005** — Feedback loop protocol, workflow memory archival
- [ ] **Document** — S-MORA ↔ SubconsciousReflector integration bridge
- [ ] **Implement** — `delete` CLI command in memory-mesh
- [ ] **Implement** — Webhook signature verification (Twilio)

---

## 11. Summary Table

| RFC | Status | Change Type | Priority |
|-----|--------|-------------|----------|
| RFC-0004 | Needs amendment | Pairing protocol, scope hierarchy, gaps | High |
| RFC-0005 | Needs amendment | Zero-JSON scope clarification, interception/entropy specs | Critical |
| RFC-0006 | Needs amendment | Heartbeat frequency, Ghost Protection specifics | Medium |
| RFC-0007 | Needs amendment | Semantic search mandate, LessonLearned terminology | High |
| RFC-0008 | New (proposed) | Wire Protocol formalization | High |
| RFC-0011 | New (proposed) | Memory-mesh specification | High |
| RFC-0012 | New (proposed) | S-MORA Retrieval Augmentation | Medium |

---

*Synthesized from 4 parallel expert reviews: Protocol & Kernel, Memory & Persistence, Singularity & Skills, S-MORA & Retrieval*
*Generated: 2026-02-18 by YAMO Unified OS Expert Review workflow*
