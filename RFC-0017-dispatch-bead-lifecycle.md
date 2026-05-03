# RFC-0017: Dispatch-Bead Lifecycle Records

**Status:** Implemented (Phase 1+2+3 items 1+3); Phase 3 item 2 deferred
**Author:** Soverane Labs
**Created:** 2026-05-02
**Updated:** 2026-05-02
**Depends on:** RFC-0010-A (sanitisation), RFC-0014 (sidecar table precedent — `memory_consolidations`)
**Implemented by:** `yamo-os/lib/brain/schema.ts`, `yamo-os/lib/brain/kernel-brain.ts`, `yamo-os/lib/kernel/kernel.ts`, `yamo-os/lib/interface/gateway.ts`

---

## Summary

Every top-level `kernel.execute()` call produces one queryable, persistent row in a new
LanceDB sidecar table `memory_dispatches`. Rows track lifecycle (`running` → `done` /
`error` / `aborted`), routing decision, recursion lineage via `parent_id`, and memory
linkage to the `memory_entries` rows produced under the dispatch. All writes are
fire-and-forget so the dispatch hot path is never blocked. The system is inspired by the
dispatch-as-data principle from Steve Yegge's Gas City SDK (Beads): work is a first-class
queryable record, not a transient log line.

---

## Motivation

Prior to this RFC, dispatch state lived only in transient sources:

- `pino` logs (rotated, unindexed, lossy)
- `runtime/data/lancedb/yamo_blocks.lance/` (per-event audit, no dispatch envelope)
- `memory_entries.lance/` (per-memory rows, no link back to the dispatch that produced
  them)

The kernel had no canonical answer to questions an operator routinely asks:

- "What did the system do for request X, end-to-end?"
- "Which memory entries were written by dispatch Y, and which were consumed?"
- "Of the last 1000 dispatches, what was the routing-target distribution?"
- "Show me the lineage of this skill-synthesis recursion."

Without a persistent envelope, these questions could only be reconstructed by greping
logs, which is brittle and incomplete. Dispatches are also the natural anchor for replay
and verifier loops (cycle-7 candidate in the yamo-dispatch skill: "does this answer cite
anything not in the memory_in / grounding sources?") — none of which can be implemented
without a queryable dispatch record.

The Gas City SDK demonstrates the broader principle: when work is a first-class
queryable record, observability, replay, and federation become straightforward; when it
isn't, every cross-cutting concern (audit, retry, attribution) is reinvented per
subsystem.

---

## Specification

### §1  Storage — `memory_dispatches` table

Scalar-only LanceDB sidecar mirroring the `memory_consolidations` precedent (RFC-0014
companion table). No vector column — dispatches are observability + replay records, not
retrieval targets. Created with `stable` storage format. Lives at
`runtime/data/lancedb/memory_dispatches.lance/`.

#### §1.1  Schema (Phase 1 columns — created at table init)

| Field             | Type   | Null | Notes                                                    |
| ----------------- | ------ | ---- | -------------------------------------------------------- |
| `id`              | Utf8   | no   | `disp_<uuid>`                                            |
| `session_id`      | Utf8   | no   | from `kernel.sessionId`                                  |
| `agent_id`        | Utf8   | no   | from `kernel.agentId`                                    |
| `created_at`      | Utf8   | no   | ISO-8601                                                 |
| `depth_entered`   | Int32  | no   | recursion depth at open (0 for top-level)                |
| `status`          | Utf8   | no   | `running` \| `done` \| `error` \| `aborted`              |
| `prompt_sha256`   | Utf8   | no   | hex sha-256 of canonical prompt                          |
| `closed_at`       | Utf8   | yes  | ISO-8601 on terminal                                     |
| `prompt_preview`  | Utf8   | yes  | first 256 chars via `sanitizePromptField`                |
| `result_sha256`   | Utf8   | yes  | hex sha-256 of result on `done`                          |
| `result_preview`  | Utf8   | yes  | first 256 chars of result, sanitized                     |
| `result_bytes`    | Int64  | yes  | `Buffer.byteLength(result, "utf8")`                      |
| `latency_ms`      | Int64  | yes  | wall time from open to close                             |
| `error_class`     | Utf8   | yes  | `error.constructor.name` on `error`                      |
| `error_message`   | Utf8   | yes  | `error.message`, sanitized                               |
| `flags`           | Utf8   | yes  | JSON object: `{ workflow_tool, ground, ground_memory, …}`|

#### §1.2  Schema (Phase 2 columns — added via `migrateDispatchesTableV2`)

| Field             | Type   | Null | Notes                                                    |
| ----------------- | ------ | ---- | -------------------------------------------------------- |
| `parent_id`       | Utf8   | yes  | dispatch id of parent for recursive calls                |
| `route_target`    | Utf8   | yes  | `RoutingDecision.target_agent`                           |
| `route_source`    | Utf8   | yes  | `llm` / `rules` (`RoutingDecision._source`)              |
| `route_intent`    | Utf8   | yes  | `RoutingDecision.intent`                                 |
| `provider_key`    | Utf8   | yes  | resolved provider for reasoner/tool roles                |
| `memory_in_ids`   | Utf8   | yes  | JSON array of `memory_entries.id` consumed (Phase 3)     |
| `memory_out_ids`  | Utf8   | yes  | JSON array of `memory_entries.id` written by this dispatch |

`migrateDispatchesTableV2(table)` is idempotent — it inspects the current schema and only
adds missing columns. New columns are nullable so existing rows from Phase 1 deployments
remain valid without backfill.

### §2  Lifecycle

#### §2.1  Open

`KernelBrain.openDispatch({ prompt, sessionId, agentId, depthEntered, flags? })` is
called by the gateway at the start of `POST /execute`, before `kernel.execute()`. It:

1. Generates `id = "disp_" + crypto.randomUUID()`.
2. Returns the id synchronously.
3. Fires a non-blocking `_dispatchesTable.add([row])` with `status: "running"`,
   `prompt_sha256`, sanitized `prompt_preview`, and either `null` flags or
   `JSON.stringify(flags)`.
4. On LanceDB write failure, calls `logger.warn` with the id and continues — never
   throws or blocks the caller.

The id is threaded through `kernel.execute()` via `params._dispatchId` so downstream
hooks (routing capture, memory linkage, recursive children) can refer to it without
crossing the dispatch boundary.

#### §2.2  Close

`KernelBrain.closeDispatch(id, { status, result?, err?, latencyMs })` is called by the
gateway after `kernel.execute()` resolves or throws. It updates the dispatch row with:

- `status` (terminal: `done` / `error` / `aborted`)
- `closed_at` (ISO-8601)
- `latency_ms`
- On success: `result_sha256`, sanitized `result_preview`, `result_bytes`
- On failure: `error_class` (constructor name), sanitized `error_message`

Write is fire-and-forget. Failures `logger.warn` and continue.

#### §2.3  Status transitions

```
running ──► done       (kernel.execute resolved, result captured)
        ──► error      (kernel.execute threw, error captured)
        ──► aborted    (caller cancelled — currently unused, reserved)
```

Once terminal, rows are immutable except via `updateDispatch` for late field stamping
(see §3). No transitions out of terminal states.

### §3  Routing capture

After `RouterAgent.route()` resolves at depth 0 in `kernel.execute()`, and after rule-
based override sanity-checks complete, the kernel calls
`brain.updateDispatch(params._dispatchId, { route_target, route_source, route_intent })`.
When the resolved provider is set for `reasoner` / `tool` targets, a second update
stamps `provider_key`. Both writes are fire-and-forget.

Routing capture is depth-0 only because `RouterAgent.route()` is gated to depth 0; child
dispatches inherit no router decision (they execute under the routing already chosen by
their parent's flow).

### §4  Recursion lineage

The kernel exposes a private helper `_executeChild(input, params, depth, stack, stream)`
that wraps a recursive `kernel.execute()` call:

1. Reads `params._dispatchId` as `parentId`.
2. If absent, falls through to plain `execute()` — zero overhead for callers without a
   tracked dispatch.
3. If present: opens a new child dispatch (`brain.openDispatch`), stamps
   `parent_id = parentId` via `updateDispatch`, and replaces `params._dispatchId`
   with the child id for the recursive call.
4. Wraps the recursive call in `try/catch`, calling `closeDispatch` with `done` (and
   the result) or `error` on the way out.

`_executeChild` is wired at two recursive sites in `kernel.ts`:

- The **EvolutionGovernor closure** (synthesis recursion)
- The **YamoExecutor recursive `execute()` tool** (in-tool recursion)

The **WorkflowEngine tool closure** (REASONER_TOOL_WORKFLOW step 2) is **not** wired —
that closure hardcodes `{ _workflowTool: true }` at `WorkflowEngine` construction time
and does not propagate caller `params`. Threading parent context through it would
require either (a) extending the `toolExecutor: (input: string) => Promise<string>`
signature to accept parent context (rippling through `WorkflowEngine` and
`WorkflowExecutor`), or (b) using `AsyncLocalStorage` so the closure reads the active
dispatch from `als.getStore()` at invocation time. This RFC defers that choice to
Phase 3.

### §5  Memory linkage

#### §5.1  Outbound (`memory_out_ids`)

`KernelBrain.add()` checks `metadata._dispatchId`. When the value is a non-empty string,
after the underlying `_mesh.add()` resolves with the new memory id, `add()` calls
`appendDispatchMemory(id, "out", [newId])` to link the write back to the originating
dispatch.

The kernel propagates `metadata._dispatchId` from `_emitBlock(type, text, traceId,
dispatchId?)` — which now accepts a `dispatchId` 4th argument — to `brain.add` via
`metadata._dispatchId`. Wired at four call sites in the main execution path:

- `recall` (execute start)
- `retain` (execute completion)
- `retain` (execute failure)
- `retain` (fast-path completion in `_executeFast`)

`_executeFast` itself accepts the dispatch id as a 5th parameter and threads it to
`_emitBlock`.

#### §5.2  Append serialization

`appendDispatchMemory(id, kind, memoryIds)` is read-modify-write: read the current
JSON array, append, write back. Concurrent appends within the same dispatch must
serialize to avoid lost updates.

The implementation uses an in-memory chained-promise queue keyed by
`(dispatchId, column)`:

```typescript
private _dispatchAppendQueue: Map<string, Promise<void>> = new Map();
```

Each call chains onto the previous promise for its key. The queue entry is removed
in `.finally` after the chain head resolves and equals the entry, preventing memory
growth across long-lived dispatches.

This was preferred over `AsyncLocalStorage` for Phase 2 because the serialization
boundary is per-(id, column) rather than per-async-context — concurrent dispatches
on different ids never block each other.

#### §5.3  Inbound (`memory_in_ids`)

Deferred to Phase 3. Requires a `_dispatchId` parameter threaded through
`KernelBrain.search()` / `KernelBrain.smora()` and injection at the kernel's search
call sites (grounding flows, retrieval-fan-out, skill scanner). The append path will
reuse `appendDispatchMemory(id, "in", […ids])`.

### §6  Read API

All HTTP routes use the existing `clampedLimit` helper (RFC unrelated; lives in
`lib/interface/route-schemas.ts`) with the `ADMIN_LIMIT_MAX` ceiling.

| Route                          | Purpose                                                  |
| ------------------------------ | -------------------------------------------------------- |
| `GET /dispatches`              | List recent dispatches, newest-first by `created_at`. Query: `limit` (default 50, max 1000), `status` (filter). |
| `GET /dispatches/:id`          | Single row, or 404 when absent.                          |
| `GET /dispatches/:id/trace`    | BFS traversal: root + descendants via `parent_id`. Query: `maxNodes` (default 256, max 1000). 404 when root absent. |

Brain methods backing the routes: `listDispatches`, `getDispatch`, `getDispatchTrace`.

---

## Rationale

### Sidecar table over `memory_entries` columns

`memory_entries` is the V5 vector + scalar table on the hot retrieval path. Adding
~15 columns for dispatch data — mostly null for non-dispatch rows — would inflate every
row, complicate the V2/V3/V4/V5 migration chain, and entangle retrieval-side queries
with audit-side queries.

The `memory_consolidations` precedent (RFC-0014 Phase 2 companion) established the
sidecar pattern for scalar-only structured records. `memory_dispatches` follows the
exact same pattern — separate file, scalar-only, no vector column, identical idempotent
create-or-open helper. Future schema evolution stays local to the sidecar.

### Fire-and-forget LanceDB writes

The dispatch hot path is `POST /execute` — user-facing latency. Synchronous LanceDB
writes would add tens of milliseconds per dispatch even for cache-hit cases, and would
turn LanceDB I/O failures into 5xx errors for the user.

The pattern is cribbed from `_populateV2Columns` in `kernel-brain.ts`: write the row,
ignore the returned promise, log on rejection. Failures surface in
`/stats.columnWriteErrors` and the structured logger but never propagate to the user
request.

### Per-(id, column) append queue over AsyncLocalStorage

`appendDispatchMemory` is the only operation that needs serialization, and the
serialization boundary is naturally per-`(dispatchId, column)`: appends to dispatch A's
`memory_out_ids` are independent of appends to dispatch B's. A `Map<string, Promise>`
keyed by that tuple gives correct serialization with O(1) overhead and no context
plumbing.

`AsyncLocalStorage` would force every async hop in the kernel to opt into context
propagation and would couple the dispatch concern to the async-context concern of the
runtime. It remains the right choice for §4 workflow-tool lineage (where the
serialization need is per-call-stack, not per-row), and is named explicitly as a Phase 3
candidate.

### Parent_id threading at propagating sites only

`kernel.execute()` is a 1100-line method with many return paths. Wrapping its body in a
top-level open-child / close-child block would either require structural refactoring
(extracting the body into `_executeInternal`) or risk subtle regressions in the existing
return paths.

The local alternative — wrapping each recursive call site with `_executeChild` — is
surgical: only sites that propagate `params` need touching, and the helper is a
straight pass-through when no parent dispatch is active. It misses the WorkflowEngine
tool closure (which doesn't propagate `params`), but that gap is now an explicit Phase 3
item rather than an invisible structural compromise.

### Porting the principle, not Dolt

Gas City uses Dolt (SQL + Git semantics) as its substrate, citing federatable / queryable
/ versioned history as the differentiator vs. LangGraph / CrewAI / AutoGen. yamo-os
already has a vector substrate (LanceDB) chosen for memory mesh workloads. Bolting
Dolt on for dispatch records would introduce a second persistence engine, dual write
paths, and operational overhead disproportionate to the benefit.

This RFC ports the *principle* — every dispatch is a queryable, persistent row — without
adopting the substrate. LanceDB's columnar SQL-style queries are sufficient for the
read API (§6); versioning and federation, if ever needed, are layerable on top
(snapshot column, replication via memory mesh).

---

## Backwards Compatibility

**Fully backwards compatible.** This RFC is purely additive:

- New table — no impact on `memory_entries`, `yamo_blocks`, `synthesized_skills`, or
  `memory_consolidations`.
- `migrateDispatchesTableV2` is idempotent — Phase 1 deployments upgrade transparently
  on next boot.
- `_emitBlock` gains an optional 4th parameter (`dispatchId`) — undefined-by-default, so
  existing callers compile and run unchanged.
- `_executeFast` gains an optional 5th parameter (`dispatchId`) — same.
- `kernel.execute` signature unchanged. New behavior gated on `params._dispatchId`
  presence.
- HTTP API — new routes only; `POST /execute` external behavior unchanged (response
  body, status codes, streaming).

Test impact: 39 new tests added; 2 existing assertions in `active-routing.test.ts`
updated for `_executeFast`'s new arity.

No data migration is required. Dispatches start at deploy time; pre-existing memory
rows have no `_dispatchId` and remain valid.

---

## Security Considerations

### Prompt / result content

Full prompts and results are **not stored** on dispatch rows — only sha256 hashes and
sanitized previews (first 256 chars, multi-line collapsed, quotes escaped via
`sanitizePromptField` from RFC-0010-A). Full content remains in its original streams
(LLM logs, response bodies) which already have their own retention and PII handling.

This deliberately limits the new PII surface: an operator querying
`memory_dispatches` sees previews and hashes, never full content.

### Error message content

`error.message` is also sanitized via `sanitizePromptField` before write. Stack traces
are not stored.

### SQL injection on `where` clauses

`getDispatch`, `closeDispatch`, `updateDispatch`, and `appendDispatchMemory` all build
`where` clauses of the form `id = '${safeId}'` where `safeId = id.replace(/'/g, "''")`.
This matches the pre-existing pattern in `getSessionMemories` and is sufficient for
single-quoted string literals. The brain's `appendDispatchMemory` also escapes the
column name via static dispatch (`kind === "in"` → `"memory_in_ids"`) — no caller-
supplied column names cross the boundary.

### Resource exhaustion

`/dispatches` and `/dispatches/:id/trace` are admin-clamped via `clampedLimit(...,
ADMIN_LIMIT_MAX)` (max 1000 per response). `getDispatchTrace` enforces `maxNodes`
on the BFS (default 256, max 1000) so a deep recursive lineage cannot fan-out without
bound.

### Authentication

`/dispatches*` routes inherit the gateway's existing device-auth middleware (RFC-0008
§7) when enabled. No new auth surface.

---

## Reference Implementation

- **Schema:** `yamo-os/lib/brain/schema.ts` — `createDispatchesSchema`, `createDispatchesTable`, `migrateDispatchesTableV2`
- **Brain:** `yamo-os/lib/brain/kernel-brain.ts` — `openDispatch`, `closeDispatch`, `updateDispatch`, `appendDispatchMemory`, `getDispatch`, `listDispatches`, `getDispatchTrace`; auto-link in `add()`
- **Kernel:** `yamo-os/lib/kernel/kernel.ts` — public `agentId` getter; `_emitBlock` 4th arg; routing capture; `_executeChild` helper; wiring at EvolutionGovernor closure and YamoExecutor recursive call; `_executeFast` 5th arg
- **Gateway:** `yamo-os/lib/interface/gateway.ts` — `POST /execute` open/close wrap with stream-tee'd result capture; `GET /dispatches`, `GET /dispatches/:id`, `GET /dispatches/:id/trace`
- **Tests:** `yamo-os/test/unit/brain/schema-dispatches.test.ts` (16 tests), `yamo-os/test/unit/brain/kernel-brain-dispatches.test.ts` (23 tests)
- **Commit:** `yamo-os@260b77b` (2026-05-02)
- **Companion RFC:** RFC-0010-A (`sanitizePromptField` for previews + error messages); RFC-0014 (sidecar table precedent — `memory_consolidations`)

### Phase 3 work items

1. **Inbound memory linkage (`memory_in_ids`).** ✅ **Implemented `2e15c6e` (2026-05-03).**
   `KernelBrain.search()`, `.smora()`, and `.searchSkills()` now read `_dispatchId` from
   options and call `appendDispatchMemory("in", ids)`. `kernel.execute()` skill-scan site
   passes `params._dispatchId` to `searchSkills`.

2. **Workflow-tool closure parent context.** Either change the
   `toolExecutor: (input: string) => Promise<string>` signature to accept parent context
   (rippling through `WorkflowEngine` / `WorkflowExecutor` / `workflow-types`), or adopt
   `AsyncLocalStorage.run(dispatchId, …)` at the top of `kernel.execute` so the existing
   closure can read the active dispatch via `als.getStore()`. The latter is more
   ambitious but generalizes to any future site where `params` doesn't propagate
   (including RFC-0007 §1 `insertHeritage`).

3. **Replay endpoint (`/dispatches/:id/replay`).** ✅ **Implemented `2e15c6e` (2026-05-03).**
   `GET /dispatches/:id/replay` resolves `memory_in_ids` → `brain.get()` for each id,
   wraps content in `<MEMORY>` blocks (same format as `yamo-dispatch --ground-memory`),
   returns `grounded_prompt` ready for re-dispatch or verifier use.

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
