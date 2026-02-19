# RFC-0011: @yamo/memory-mesh — Semantic Memory System Specification

**Status:** Draft
**Author:** Soverane Labs & Collaborative Swarm
**Created:** 2026-02-18
**Depends on:** RFC-0005 (Singularity Protocol), RFC-0007 (Semantic Heritage & Wisdom Distillation)

---

## Summary

This RFC formally specifies **`@yamo/memory-mesh`** as the canonical semantic memory persistence layer for the YAMO ecosystem. It defines the complete API contract, LanceDB schema, YAMO block wire formats (including the previously unspecified `LessonLearned` format), wisdom distillation cycle, CLI protocol, and integration patterns for embedding memory into YAMO agents and kernels.

---

## Motivation

RFC-0007 described the goals of Wisdom Distillation and Semantic Heritage but left the following critical gaps:

1. **No wire format for `LessonLearned` blocks** — the block structure was named but never serialized.
2. **No wisdom distillation trigger** — the heartbeat-triggered distillation cycle was described conceptually but never specified mechanically.
3. **No formal API contract** — consumer code must reverse-engineer the `MemoryMesh` interface from source.
4. **No schema versioning protocol** — V1 and V2 schemas coexist without a defined migration path.
5. **No CLI command specification** — the Zero-JSON CLI rewrite (v3.1.0) is undocumented.

RFC-0011 closes all five gaps. It is the authoritative reference for any system that stores, retrieves, or distills semantic memory within YAMO.

---

## Specification

### 1. Core Concepts

#### 1.1 Memory Entry

A **Memory Entry** is the atomic unit of persistence. It stores a single piece of agent knowledge with a vector embedding for semantic retrieval.

```
MEMORY_ID    = "mem_" TIMESTAMP "_" RANDOM_HEX_6
SKILL_ID     = "skill_" TIMESTAMP "_" RANDOM_HEX_6
LESSON_ID    = "lesson_" TIMESTAMP "_" RANDOM_HEX_6
YAMO_BLOCK_ID = "yamo_" OPERATION "_" TIMESTAMP "_" RANDOM_HEX_6

TIMESTAMP    = Unix milliseconds as decimal string
RANDOM_HEX_6 = 6 lowercase hexadecimal characters
```

#### 1.2 Memory Types

| Type | Description | Scope |
|------|-------------|-------|
| `global` | Shared across all agents and sessions | Persistent |
| `session` | Scoped to a single conversation session | Ephemeral |
| `agent` | Scoped to a specific agent or skill | Persistent |
| `lesson` | Distilled wisdom from past failures | Persistent, high-signal |
| `reflection` | Synthesized beliefs from memory aggregation | Persistent |

#### 1.3 Layers

Memory-mesh implements a **three-layer architecture**:

```
Layer 0: Scrubber (deterministic 6-stage preprocessing)
Layer 1: Embedding Factory (vector generation with fallback chain)
Layer 2: LanceDB (Apache Arrow vector storage + keyword index)
```

Content passes through all three layers on `add()`. Retrieval combines Layer 1 (vector search) and a keyword index (BM25-style), fused via Reciprocal Rank Fusion (RRF).

---

### 2. LanceDB Schema

#### 2.1 Memory Entries — Schema V1 (Current)

```typescript
{
  id:         Utf8,                        // MEMORY_ID
  vector:     FixedSizeList<Float32>(dim), // Embedding (default: 384 dims)
  content:    Utf8,                        // Layer 0-sanitized text
  metadata:   Utf8,                        // JSON-serialized metadata object
  created_at: Timestamp(ms),               // Creation time (UTC)
  updated_at: Timestamp(ms)                // Last update (nullable)
}
```

**Metadata object fields** (conventional; not schema-enforced):

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Memory classification (see §1.2) |
| `source` | string | Originating system or agent |
| `tags` | string[] | Freeform tags (including `#lesson_learned`) |
| `rationale` | string | Constitutional justification |
| `hypothesis` | string | Expected use of this memory |
| `skill_name` | string | Owning skill (for scoped recall) |
| `session_id` | string | Session association |

#### 2.2 Memory Entries — Schema V2 (Target)

V2 extends V1 with promoted fields (previously JSON-embedded in `metadata`). All V2 fields are **nullable** for backward compatibility.

```typescript
{
  // V1 fields (unchanged)
  id:               Utf8,
  vector:           FixedSizeList<Float32>(dim),
  content:          Utf8,
  metadata:         Utf8,
  created_at:       Timestamp(ms),
  updated_at:       Timestamp(ms),

  // V2 promoted fields
  session_id:       Utf8,           // Session association
  agent_id:         Utf8,           // Creating agent identifier
  memory_type:      Utf8,           // Enum: global|session|agent|lesson|reflection
  importance_score: Float32,        // 0.0–1.0 relevance weight
  access_count:     Int32,          // Retrieval popularity counter
  last_accessed:    Timestamp(ms),  // Last retrieval time
  heritage_chain:   Utf8,           // JSON: { intentChain, hypotheses, rationales }
  lesson_pattern_id: Utf8,          // Pattern hash for lesson clustering
  source_skill_id:  Utf8            // Originating synthesized skill ID
}
```

**Schema Version Detection**: Systems MUST detect schema version by checking for the presence of the `session_id` field using `isSchemaV2(schema)`.

#### 2.3 YAMO Blocks Table

Immutable audit log of all memory operations. Supports provenance tracking and the hash-chaining required for RFC-0007 Semantic Heritage.

```typescript
{
  id:             Utf8,           // YAMO_BLOCK_ID
  agent_id:       Utf8,           // Creating agent (nullable)
  operation_type: Utf8,           // "retain"|"recall"|"reflect"|"synthesize"|"lesson"
  yamo_text:      Utf8,           // Full YAMO block content (wire format)
  timestamp:      Timestamp(ms),
  block_hash:     Utf8,           // SHA-256 of yamo_text (nullable; future)
  prev_hash:      Utf8,           // Previous block hash for chaining (nullable; future)
  metadata:       Utf8            // Additional context (JSON, nullable)
}
```

**Valid `operation_type` values**: `retain`, `recall`, `reflect`, `synthesize`, `lesson`

#### 2.4 Synthesized Skills Table

```typescript
{
  id:         Utf8,                        // SKILL_ID
  name:       Utf8,                        // Extracted skill name
  intent:     Utf8,                        // Primary skill purpose (1 sentence)
  yamo_text:  Utf8,                        // Full YAMO skill definition
  vector:     FixedSizeList<Float32>(dim), // Intent embedding (for semantic search)
  metadata:   Utf8,                        // JSON: { reliability, use_count, last_used, created_at }
  created_at: Timestamp(ms)
}
```

---

### 3. YAMO Block Wire Formats

All blocks follow the YAMO v3.0 semicolon-delimited protocol (RFC-0005). Keys and values are UTF-8 strings; internal semicolons within values MUST be escaped as commas.

#### 3.1 Canonical Block Grammar

```abnf
YAMO_BLOCK       = agent_section intent_section
                   [context_section] [constraints_section]
                   [priority_section] [output_section]
                   meta_section log_line handoff_line

agent_section    = "agent: " AGENT_NAME ";" NEWLINE
intent_section   = "intent: " INTENT_ID ";" NEWLINE
context_section  = "context:" NEWLINE 1*(INDENT KV_PAIR NEWLINE)
constraints_section = "constraints:" NEWLINE 1*(INDENT CONSTRAINT NEWLINE)
priority_section = "priority: " PRIORITY_LEVEL ";" NEWLINE
output_section   = "output:" NEWLINE 1*(INDENT KV_PAIR NEWLINE)
meta_section     = 1*(META_LINE)
log_line         = "log: " LOG_BODY ";" NEWLINE
handoff_line     = "handoff: " HANDOFF_TARGET ";" NEWLINE

META_LINE        = "meta: " KEY ";" VALUE ";" NEWLINE
KV_PAIR          = KEY ";" VALUE ";"
CONSTRAINT       = "-" SP CONSTRAINT_BODY ";"
PRIORITY_LEVEL   = "critical" | "high" | "medium" | "low"
AGENT_NAME       = 1*(ALPHA | DIGIT | "_")
INTENT_ID        = 1*(ALPHA | DIGIT | "_")    ; underscored snake_case
INDENT           = "  "                        ; 2 spaces
```

#### 3.2 Retain Block (memory store)

```yamo
agent: MemoryMesh_{agentId};
intent: store_memory_for_future_retrieval;
context:
  memory_id;{MEMORY_ID};
  memory_type;{type};
  timestamp;{ISO_8601};
  content_length;{bytes};
constraints:
  hypothesis;{expected_use_of_memory};
priority: medium;
output:
  memory_stored;{MEMORY_ID};
  content_preview;{first_60_chars};
meta:
  rationale;Memory persisted for semantic search and retrieval;
  observation;Content vectorized and stored in LanceDB;
  confidence;1.0;
log: memory_retained;timestamp;{ISO_8601};id;{MEMORY_ID};type;{type};
handoff: End;
```

#### 3.3 Recall Block (memory search)

```yamo
agent: MemoryMesh_{agentId};
intent: recall_relevant_memories;
context:
  query;{search_query};
  limit;{N};
  filter;{lancedb_filter_or_none};
  timestamp;{ISO_8601};
output:
  results_found;{N};
  top_score;{highest_rrf_score};
meta:
  rationale;Retrieved most semantically relevant memories for current context;
  confidence;{top_score};
log: memory_recalled;timestamp;{ISO_8601};query;{query};results;{N};
handoff: End;
```

#### 3.4 Reflect Block (synthesis)

```yamo
agent: MemoryMesh_{agentId};
intent: synthesize_beliefs_from_memories;
context:
  topic;{topic};
  source_count;{N};
  timestamp;{ISO_8601};
output:
  reflection_id;{reflect_timestamp_hex};
  insight_preview;{first_60_chars};
  confidence;{score};
meta:
  rationale;Aggregated high-signal patterns from recent memory;
  confidence;{score};
log: memory_reflected;timestamp;{ISO_8601};topic;{topic};sources;{N};
handoff: End;
```

#### 3.5 LessonLearned Block (wisdom distillation)

This is the previously unspecified format from RFC-0007. It is the canonical wire format for distilled wisdom entries.

```yamo
agent: MemoryMesh_{agentId};
intent: distill_wisdom_from_execution;
context:
  original_context;{situation_description};
  error_pattern;{pattern_id};
  severity;critical | high | medium | low;
  timestamp;{ISO_8601};
constraints:
  hypothesis;This lesson prevents recurrence of similar failures;
  hypothesis_confidence;{0.0-1.0};
priority: high;
output:
  lesson_id;{LESSON_ID};
  oversight_description;{what_failed};
  preventative_rule;{actionable_constraint_for_future};
  rule_confidence;{0.0-1.0};
meta:
  rationale;{why_this_lesson_matters};
  applicability_scope;{where_and_when_rule_applies};
  inverse_lesson;{what_succeeds_instead};
  confidence;{overall_confidence};
log: lesson_learned;timestamp;{ISO_8601};pattern;{pattern_id};severity;{severity};id;{LESSON_ID};
handoff: SubconsciousReflector;
```

**Field semantics:**

| Field | Required | Description |
|-------|----------|-------------|
| `context.original_context` | Yes | Human-readable description of the situation |
| `context.error_pattern` | Yes | Stable identifier for deduplication (e.g., `auth_fallback_bypass`) |
| `context.severity` | Yes | 4-tier: `critical` \| `high` \| `medium` \| `low` |
| `output.lesson_id` | Yes | Unique identifier: `lesson_{timestamp}_{hex}` |
| `output.preventative_rule` | Yes | Actionable constraint in imperative form |
| `output.rule_confidence` | Yes | Confidence that this rule is correct (0.0–1.0) |
| `meta.applicability_scope` | Yes | Conditions under which this rule applies |
| `meta.inverse_lesson` | Recommended | Counter-example: what succeeds when this rule is followed |
| `handoff` | Yes | Must be `SubconsciousReflector` to inject into active context |

**Tagging**: The memory entry derived from a LessonLearned block MUST include `"#lesson_learned"` in its `tags` metadata field.

---

### 4. Public API Contract

#### 4.1 MemoryMesh Class

```typescript
class MemoryMesh {
  constructor(options?: MemoryMeshOptions)

  // Lifecycle
  async init(): Promise<void>
  async close(): Promise<void>

  // Memory operations
  async add(content: string, metadata?: Record<string, any>): Promise<MemoryRecord>
  async ingest(content: string, metadata?: Record<string, any>): Promise<MemoryRecord>  // alias for add()
  async get(id: string): Promise<MemoryRecord | null>
  async getAll(options?: { limit?: number }): Promise<MemoryRecord[]>
  async delete(id: string): Promise<void>   // NEW — not yet implemented; required by this RFC

  // Search
  async search(query: string, options?: SearchOptions): Promise<SearchResult[]>

  // Reflection & synthesis
  async reflect(options?: ReflectOptions): Promise<ReflectionResult>
  async synthesize(options?: SynthesisOptions): Promise<SynthesisResult>

  // Wisdom distillation (NEW — required by this RFC)
  async distillLesson(context: LessonContext): Promise<LessonBlock>
  async queryLessons(query: string, options?: QueryOptions): Promise<LessonBlock[]>
  async insertHeritage(memoryId: string, heritage: HeritageChain): Promise<void>
  async getMemoriesByPattern(patternId: string): Promise<MemoryRecord[]>

  // Skill management
  async ingestSkill(yamoText: string, metadata?: Record<string, any>, sourceFilePath?: string): Promise<SkillRecord>
  async searchSkills(query: string, options?: { limit?: number }): Promise<SkillResult[]>
  async listSkills(options?: { limit?: number }): Promise<SkillResult[]>
  async updateSkillReliability(id: string, success: boolean): Promise<ReliabilityUpdate>
  async pruneSkills(threshold?: number): Promise<PruneResult>

  // Diagnostics
  async stats(): Promise<MemoryStats>
  clearCache(): void
  getCacheStats(): CacheStats
  formatResults(results: SearchResult[]): string
}
```

#### 4.2 Key Types

```typescript
interface MemoryMeshOptions {
  enableYamo?: boolean;          // YAMO block emission (default: true)
  enableLLM?: boolean;           // LLM-enhanced reflection (default: true)
  enableMemory?: boolean;        // Memory persistence (default: true)
  agentId?: string;              // Agent identifier (default: "YAMO_AGENT")
  llmProvider?: string;          // "openai" | "ollama" | "cohere"
  llmModel?: string;
  llmApiKey?: string;
  llmMaxTokens?: number;
  skill_directories?: string | string[];
  dbDir?: string;                // Override DB path (testing)

  // Wisdom distillation (new)
  enableWisdomDistillation?: boolean;   // default: false
  lessonMinConfidence?: number;         // default: 0.7
  lessonRetentionDays?: number;         // default: 365
  enableHeritageTracking?: boolean;     // default: false
  maxHeritageDepth?: number;            // default: 5
}

interface MemoryRecord {
  id: string;                    // MEMORY_ID
  content: string;               // Sanitized content
  metadata: Record<string, any>; // Parsed metadata
  created_at: string;            // ISO 8601
}

interface SearchOptions {
  limit?: number;                // default: 10
  filter?: string;               // LanceDB SQL WHERE expression
  useCache?: boolean;            // default: true
}

interface SearchResult extends MemoryRecord {
  score: number;                 // 0.0–1.0 RRF-normalized relevance
}

interface LessonContext {
  situation: string;             // What happened
  errorPattern: string;          // Stable pattern identifier
  oversight: string;             // What was missed
  fix: string;                   // How it was resolved
  preventativeRule: string;      // Constraint for future executions
  severity: "critical" | "high" | "medium" | "low";
  applicableScope: string;       // When/where the rule applies
  inverseLesson?: string;        // Counter-example (success case)
  confidence?: number;           // Rule confidence (default: 0.7)
}

interface LessonBlock {
  lessonId: string;              // LESSON_ID
  patternId: string;             // Stable dedup hash
  severity: string;
  preventativeRule: string;
  ruleConfidence: number;
  applicableScope: string;
  wireFormat: string;            // Full YAMO block text
  memoryId: string;              // Corresponding MEMORY_ID
}

interface HeritageChain {
  intentChain: string[];         // Ordered chain of agent intents
  hypotheses: string[];          // Validated hypotheses at each level
  rationales: string[];          // Justifications for decisions
}
```

#### 4.3 MemoryContextManager Class

High-level wrapper for agent conversation loops.

```typescript
class MemoryContextManager {
  constructor(config?: ContextManagerConfig)

  async initialize(): Promise<void>

  async captureInteraction(
    prompt: string,
    response: string,
    context?: InteractionContext
  ): Promise<MemoryRecord | null>

  async recallMemories(
    query: string,
    options?: RecallOptions
  ): Promise<MemoryRecord[]>

  formatMemoriesForPrompt(
    memories: MemoryRecord[],
    options?: FormatOptions
  ): string

  async healthCheck(): Promise<HealthStatus>

  dispose(): void
}

interface RecallOptions {
  limit?: number;                // default: 5
  useCache?: boolean;
  memoryType?: string;           // Filter by memory_type
  skillName?: string;            // Scope to skill-specific memories
  includeLessons?: boolean;      // Also query #lesson_learned (default: false)
}
```

---

### 5. Search Algorithm

#### 5.1 Hybrid Vector + Keyword Search

```
Input: query string, limit N, optional filter

Step 1 — Vector Search
  embedding = embed(query)
  vectorResults = lancedb.search(embedding, limit=N×2, filter=filter)
  // Cosine distance → similarity: score = max(0, 1 - distance/2)

Step 2 — Keyword Search
  tokens = tokenize(query)  // lowercase, camelCase split, min 3 chars
  keywordResults = bm25_index.search(tokens, limit=N×2)

Step 3 — Reciprocal Rank Fusion (k=60)
  scores = {}
  for rank, result in enumerate(vectorResults):
    scores[result.id] += 1 / (60 + rank + 1)
  for rank, result in enumerate(keywordResults):
    scores[result.id] += 1 / (60 + rank + 1)

Step 4 — Normalize & Sort
  if len(candidates) <= 2×N:
    sort descending by score  // O(n log n)
  else:
    partial selection sort for top N  // O(n×k)

Step 5 — Cache
  key = "search:{query}:{limit}:{filter}"
  ttl = 5 minutes, max 500 entries (LRU eviction)

Output: top N results sorted by RRF score
```

#### 5.2 Score Normalization

```
rawDistance ∈ [0.0, 2.0]  (cosine distance range)
similarity  = max(0.0, min(1.0, 1.0 − rawDistance / 2.0))
score       = round(similarity, 2)
```

#### 5.3 Skill Search Weighting

Skill search uses a different fusion ratio (skills have explicit keyword fields):

```
finalScore = 0.7 × keywordScore + 0.3 × vectorScore
```

Keyword scoring checks: `name` (weight 3×), `intent` (weight 2×), `tags` (weight 1×), `description` (weight 1×).

---

### 6. Layer 0 Scrubber

Content MUST pass through the Scrubber before embedding. The Scrubber is a deterministic, idempotent 6-stage pipeline.

#### 6.1 Pipeline Stages

| Stage | Purpose | Key Operations |
|-------|---------|----------------|
| 1. Structural Cleaning | Remove markup | Strip HTML, MD syntax, collapse whitespace, remove scripts/styles |
| 2. Semantic Filtering | Remove noise | Deduplication, boilerplate removal, minimum signal ratio check |
| 3. Normalization | Standardize text | Heading normalization, list normalization, punctuation standardization |
| 4. Chunking | Segment content | Split by tokens (min: 1, max: 500, hard max: 2000), heading-aware |
| 5. Metadata Annotation | Add provenance | Source, section path, content hash, timestamp |
| 6. Validation | Quality gate | Enforce min/max length, reject empty chunks |

#### 6.2 MemoryMesh Scrubber Overrides

When instantiated by MemoryMesh, the scrubber is configured with:

```typescript
{
  enabled: true,
  chunking: { minTokens: 1 },         // Allow short memories (override default: 10)
  validation: { enforceMinLength: false }  // Don't reject short inputs
}
```

#### 6.3 PII Handling

The Scrubber MUST redact recognizable PII patterns before any content reaches the embedding layer or LanceDB. Patterns include email addresses, phone numbers, credit card numbers, and API key patterns. The exact pattern list is implementation-defined but MUST be documented.

---

### 7. Embedding Factory

#### 7.1 Fallback Chain

```
Priority 1 (Primary):  configured provider (local/openai/ollama/cohere)
Priority 2+ (Fallback): local ONNX (Xenova/all-MiniLM-L6-v2, 384 dims)

On embed(text):
  try primary → if success, return
  for each fallback in order → if success, log warning + return
  if all fail → throw EmbeddingError

Per-service LRU cache: 1000 entries, no TTL (permanent within process)
```

#### 7.2 Default Model

| Property | Value |
|----------|-------|
| Provider | Local ONNX (Transformers.js) |
| Model | `Xenova/all-MiniLM-L6-v2` |
| Dimensions | 384 |
| License | Apache 2.0 |
| Requires | No API key |

#### 7.3 Lazy Initialization

Embedding models MUST NOT be loaded at `MemoryMesh` construction time. Models load on the first `embed()` call. This allows fast instantiation and deferred resource allocation.

---

### 8. Wisdom Distillation Cycle

This section specifies the previously undocumented heartbeat-triggered distillation cycle required by RFC-0007.

#### 8.1 Trigger

The distillation cycle runs as part of the kernel heartbeat (RFC-0006), at **PRIORITY_1** (after GhostGuard at PRIORITY_0), **2–4 times per day**.

#### 8.2 Algorithm

```
WisdomDistillationCycle:

  1. SCAN — Query recent YAMO blocks (last 24h) for failure patterns:
     - operation_type = "retain" with metadata.status = "error"
     - Any block containing "#lesson_learned_candidate" tag
     - Repeated error patterns (same error_pattern within 7 days)

  2. DEDUPLICATE — For each candidate:
     pattern_id = SHA256(errorPattern + applicableScope)[0:16]
     existing = getMemoriesByPattern(pattern_id)
     if existing.length > 0 and existing[0].rule_confidence >= candidateConfidence:
       skip (existing lesson is better)

  3. DISTILL — For each novel pattern:
     lesson = distillLesson(LessonContext{...})
     // Emits LessonLearned YAMO block (§3.5)
     // Stores in LanceDB with tag "#lesson_learned"
     // Records lesson_pattern_id for deduplication

  4. INJECT — Update SubconsciousReflector context:
     top_lessons = queryLessons("", { limit: 10, orderBy: "severity DESC" })
     // Injected as preventative constraints at next session start
```

#### 8.3 distillLesson() Contract

```typescript
async distillLesson(context: LessonContext): Promise<LessonBlock>
```

**Side effects**:
1. Generates LessonLearned YAMO block (§3.5 wire format)
2. Stores as memory entry with `memory_type = "lesson"` and `tags = ["#lesson_learned"]`
3. Stores YAMO block in yamo_blocks table with `operation_type = "lesson"`
4. Returns `LessonBlock` with full wire format text

**Idempotency**: If a lesson with the same `patternId` and equal or higher `ruleConfidence` already exists, MUST return the existing lesson without creating a duplicate.

---

### 9. Zero-JSON CLI Protocol

The CLI MUST use named flags for all operations. Positional JSON arguments are prohibited (Zero-JSON Mandate, RFC-0005).

#### 9.1 Command Reference

```
memory-mesh store --content <text> [--type <type>] [--rationale <text>]
                  [--hypothesis <text>] [--metadata <json>]

memory-mesh search <query> [--limit <n>] [--filter <expr>]

memory-mesh ingest-dir <path> [--extension <ext>] [--type <type>] [--recursive]

memory-mesh stats

memory-mesh reflect [--topic <text>] [--lookback <n>]

memory-mesh get --id <id>

memory-mesh delete --id <id>

memory-mesh synthesize [--topic <text>]

memory-mesh ingest-skill --file <path> [--type <type>]

memory-mesh search-skills <query> [--limit <n>]

memory-mesh list-skills [--limit <n>]

memory-mesh skill-feedback --id <id> --success <true|false>

memory-mesh skill-prune [--threshold <0.0-1.0>]
```

#### 9.2 Output Format

**`store` output:**
```
[MemoryMesh] Ingested record {MEMORY_ID}
```

**`search` output:**
```
[MemoryMesh] Found {N} matches.

[ATTENTION DIRECTIVE]
- ALIGN attention to entries with [IMPORTANCE >= 0.8]
- TREAT entries with [IMPORTANCE <= 0.4] as auxiliary

[MEMORY CONTEXT]
--- MEMORY 1: {id} [IMPORTANCE: {score}] ---
Type: {type} | Source: {source}
{content}
```

**`stats` output:**
```
[MemoryMesh] Total Memories: {N}
[MemoryMesh] Total Skills: {N}
[MemoryMesh] DB Path: {uri}
[MemoryMesh] Status: connected|disconnected
```

**`delete` output:**
```
[MemoryMesh] Deleted record {MEMORY_ID}
```

#### 9.3 Error Output

All errors MUST be written to stderr. Exit code MUST be non-zero on failure.

```
[MemoryMesh Error] {error_message}
```

---

### 10. Integration Pattern: Agent Memory Loop

The canonical pattern for integrating memory into a YAMO agent:

```typescript
// 1. Initialize
const mesh = new MemoryMesh({ agentId: "MyAgent", enableYamo: true });
await mesh.init();

// 2. SubconsciousReflector: query lessons before acting
const lessons = await mesh.queryLessons(userRequest, { limit: 5 });
const constraints = lessons.map(l => l.preventativeRule);
// Inject constraints into agent context

// 3. Recall relevant memories
const memories = await mesh.search(userRequest, { limit: 10 });

// 4. Execute agent task
const result = await executeWithContext(userRequest, memories, constraints);

// 5. Retain new knowledge
await mesh.add(result.output, {
  type: "event",
  source: "MyAgent",
  rationale: result.rationale,
  hypothesis: result.hypothesis,
  session_id: sessionId
});

// 6. Distill lesson if error occurred
if (result.error) {
  await mesh.distillLesson({
    situation: result.context,
    errorPattern: result.errorCode,
    oversight: result.error.message,
    fix: result.resolution,
    preventativeRule: result.constraint,
    severity: "high",
    applicableScope: "MyAgent execution"
  });
}

// 7. Cleanup
await mesh.close();
```

---

### 11. Schema Migration: V1 → V2

**Detection**: Check for `session_id` field in LanceDB table schema.

**Migration strategy**: Additive only. V2 fields are nullable; existing V1 records remain valid.

```
Migration procedure:
  1. Read existing table schema
  2. If isSchemaV1(schema):
     a. Create new table with V2 schema
     b. Copy all V1 records (V2 nullable fields = null)
     c. Rename old table to memory_entries_v1_backup
     d. Rename new table to memory_entries
  3. If isSchemaV2(schema): no action needed
```

**Rollback**: V1 backup table is retained for 30 days before permanent deletion.

---

## Rationale

### Why LanceDB?

LanceDB provides Apache Arrow-native vector storage with zero-copy columnar access. Its SQL-style filtering, cosine similarity search, and local-first design (no server required) align with YAMO's offline-capable, privacy-first architecture.

### Why RRF for Fusion?

Reciprocal Rank Fusion (k=60) is robust to score scale differences between vector and keyword systems. It rewards results that rank well in both modalities without requiring score normalization across fundamentally different scales. k=60 is the established default from the original RRF paper (Cormack et al., 2009).

### Why Layer 0 Scrubber Before Embedding?

Embedding dirty content (HTML, boilerplate, PII) produces noisy vectors that reduce retrieval precision. By normalizing content deterministically before embedding, the system ensures that semantically identical content produces similar vectors regardless of surface formatting differences.

### Why Lazy Initialization?

Local ONNX models require 1–3 seconds to load. Lazy initialization allows agents to instantiate MemoryMesh at startup without blocking on model loading—resources are consumed only when the first memory operation occurs.

### Why a LessonLearned Wire Format?

RFC-0007 defined the concept of LessonLearned but left the serialization format open. Without a canonical format, different implementations cannot exchange lessons across agent boundaries. The wire format in §3.5 makes lessons interoperable across kernel boundaries and inspectable via the YAMO blocks audit table.

---

## Backwards Compatibility

- **Schema V1 → V2**: Fully backward compatible. All V2 fields are nullable. V1 readers ignore unknown fields.
- **CLI v3.0 → v3.1.0 (Zero-JSON)**: Breaking change for callers using positional JSON arguments. Migration: replace `store '{"content": "..."}' ` with `store --content "..."`.
- **API `add()` / `ingest()`**: `ingest()` is a stable alias for `add()`. Both are permanent.
- **`delete()` method**: Not yet implemented in v3.1.1. Required by this RFC. MUST be implemented before RFC-0011 reaches Final status.

---

## Security Considerations

### PII Redaction

The Layer 0 Scrubber MUST redact PII before content reaches the embedding layer or LanceDB. PII that bypasses the scrubber could be reconstructed via vector similarity search (embedding inversion attacks).

### Prototype Pollution

Metadata objects from user input MUST be sanitized to prevent prototype pollution. The `_validateMetadata()` method MUST reject keys in `{ "__proto__", "constructor", "prototype" }`.

### Metadata Stored as JSON Strings

LanceDB stores `metadata` as a UTF-8 JSON string. This prevents direct filtering on nested metadata fields. LanceDB-level filters operate only on schema-promoted fields. This is a known limitation; V2 schema promotion of common fields (§2.2) is the mitigation.

### Embedding Cache Poisoning

The per-service LRU embedding cache (1000 entries) is in-process only and not persisted across restarts. Cache keys are the raw input text, so cache poisoning requires control of input text, which implies broader system compromise.

### CLI Injection

The CLI `--filter` flag accepts raw LanceDB SQL expressions. Implementations MUST validate or sandbox filter inputs when used in multi-tenant contexts. In single-agent deployments this is not a concern.

### LessonLearned Integrity

Lessons with incorrect `preventativeRule` values could cause agents to adopt wrong constraints. Implementations SHOULD require `rule_confidence >= 0.7` before injecting a lesson as an active constraint. The `lessonMinConfidence` option (§4.2) controls this threshold.

---

## Reference Implementation

- **Core library**: `/home/dev/workspace/yamo-memory-mesh/` (`@yamo/memory-mesh` v3.1.1)
- **CLI**: `/home/dev/workspace/yamo-memory-mesh/bin/memory_mesh.js`
- **Type definitions**: `/home/dev/workspace/yamo-memory-mesh/lib/memory/memory-mesh.d.ts`
- **Schema**: `/home/dev/workspace/yamo-memory-mesh/lib/yamo/schema.ts`
- **Scrubber**: `/home/dev/workspace/yamo-memory-mesh/lib/scrubber/scrubber.ts`
- **E2E tests**: `/home/dev/workspace/yamo-memory-mesh/test/e2e/cli.test.ts`

**Implementation gaps** (required before RFC-0011 reaches Final status):
- [ ] `MemoryMesh.delete(id)` — not yet implemented
- [ ] `MemoryMesh.distillLesson(context)` — not yet implemented
- [ ] `MemoryMesh.queryLessons(query, options)` — not yet implemented
- [ ] `MemoryMesh.insertHeritage(memoryId, heritage)` — not yet implemented
- [ ] `memory-mesh delete --id <id>` CLI command
- [ ] `memory-mesh reflect` CLI command
- [ ] `memory-mesh get --id <id>` CLI command
- [ ] Wisdom distillation heartbeat hook
- [ ] Schema V2 migration procedure

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
