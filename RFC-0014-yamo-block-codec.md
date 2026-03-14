# RFC-0014: YAMO Block Codec — Canonical Serialiser/Parser

**Status:** Implemented
**Author:** Soverane Labs
**Created:** 2026-03-14
**Updated:** 2026-03-14
**Depends on:** RFC-0011 §3 (YAMO wire formats), RFC-0008 §4 (audit block format)
**Amends:** RFC-0011 §3.1 (field-value escape rule)
**Implemented by:** `yamo-os/lib/yamo/block-codec.ts`

---

## Summary

This RFC establishes a **canonical TypeScript codec** for the YAMO flat wire format and
amends RFC-0011 §3.1's escape rule from comma-based (lossy) to percent-encoding (lossless).
All writers MUST use `encodeBlock()` and all readers MUST use `decodeBlock()`. Hand-rolled
YAMO block string templates are prohibited.

---

## Motivation

The YAMO semicolon block format (`TYPE: field;value;field;value;...;`) is the load-bearing
data exchange layer between the kernel audit trail, failure analyzer, reflector, and the
YAMO-kernel bridge. Prior to this RFC it had no canonical implementation:

- `lib/kernel/kernel.ts` wrote blocks with hand-rolled string templates
- `lib/kernel/failure-analyzer.ts` parsed them with three independent regex calls
- `lib/gateway/yamo-kernel-bridge.ts` re-implemented YAMO parsing a third time
- Escaping was inconsistent: `%3B` in new kernel code vs. comma-escape in existing blocks and RFC-0011 §3.1

Every new writer was forced to re-implement the escaping rule, often getting it wrong.
Any field value containing a literal semicolon became a silent format error unless the
author remembered to apply the escape — and even then the escape scheme was ambiguous.

---

## Specification

### §1  Scope

This RFC covers the **flat wire format** (RFC-0008 audit events):

```
TYPE: field;value;field;value;...;
```

It does **not** cover the ABNF colon-separated multi-line format used by `lib/yamo/emitter.ts`
(MemoryMesh ingestion blocks). Those two formats are distinct and independently governed.

### §2  Field Escape Rule (amendment to RFC-0011 §3.1)

RFC-0011 §3.1 specified:

> "internal semicolons within values MUST be escaped as commas"

This rule is **amended**. The new rule:

> Literal semicolons within field values MUST be percent-encoded as `%3B`.

**Rationale:** Comma-escape is lossy — it cannot distinguish an original comma in the value
from an escaped semicolon. `%3B` encoding is reversible and lossless. The risk of collision
with existing `%3B` sequences in production data is negligible.

**Backward compatibility:** Existing LanceDB blocks written with comma-escape remain
readable. `decodeField()` only unescapes `%3B` sequences; commas are left as-is. The
FailureAnalyzer uses the error field only as an approximate dedup key, so residual commas
in old records do not cause functional regressions.

### §3  TypeScript Interface

```typescript
// lib/yamo/block-codec.ts

export type BlockType =
  | "retain" | "recall" | "reflect" | "synthesize" | "lesson";

export interface YamoBlock {
  type: BlockType;
  agent: string;         // REQUIRED — emitting agent name
  id?: string;           // skill ID (may be undefined for direct kernel executions)
  intent?: string;       // natural-language intent description
  input?: string;        // user input (capped at 100 chars on encode)
  status?: string;       // "started" | "completed" | "failed" | "error"
  error?: string;        // error message (semicolons %3B-escaped)
  handoff?: string;      // next agent / "End"
  [key: string]: string | undefined;  // extensible extra fields
}

// Encode a single field value: %3B-escape semicolons, cap length
export function encodeField(value: string, maxLen?: number): string;

// Decode a single field value: restore %3B → ";"
export function decodeField(value: string): string;

// Serialise a YamoBlock to the flat wire format string
// Canonical field order: agent, id, intent, input, status, error, handoff
// Extra fields are appended after canonical set in insertion order
export function encodeBlock(block: YamoBlock): string;

// Parse a flat wire-format YAMO block string into a YamoBlock
// Returns null (never throws) on malformed input
export function decodeBlock(raw: string): YamoBlock | null;

// Returns true if the raw string represents a failure block
// (contains "status;failed" or "status;error")
export function isFailureBlock(raw: string): boolean;

// Known YAMO response field names — for bridge response extraction
export const YAMO_FIELD_NAMES: readonly string[];

// Pre-compiled regex matching the start of any known YAMO field
export const YAMO_NEXT_FIELD_RE: RegExp;

// Extract the `response` field from an ABNF-style YAMO kernel output
export function extractResponseField(raw: string): string | null;
```

### §4  Canonical Field Order

When serialising, canonical fields are emitted in this fixed order:

```
agent → id → intent → input → status → error → handoff
```

Extra (non-canonical) fields are appended after the canonical set in object insertion order.
The `type` field governs the block prefix (`TYPE:`) and is never emitted as a key-value pair.

### §5  Length Caps

| Field  | Max encoded length |
|--------|--------------------|
| input  | 100 characters     |
| id     | 256 characters     |
| all other fields | 256 characters |
| extra field keys | 64 characters |

These caps prevent token-budget exhaustion when blocks flow into LLM prompts.

### §6  Blank Value Handling

A blank field value (empty string or whitespace-only) is preserved during parsing.
The token filter in `decodeBlock` MUST NOT remove blank value tokens — doing so shifts
all subsequent key-value pairs, causing fields to be misattributed.

Callers that need to detect "no value set" for fields like `id` MUST trim and check for
empty string themselves:

```typescript
const skillId = block?.id?.trim() || undefined;
```

### §7  Usage Rules

1. **All YAMO block writers MUST call `encodeBlock()`.** String template literals that
   produce `TYPE: field;value;...` format are prohibited in `lib/`.

2. **All YAMO block readers MUST call `decodeBlock()`.** Ad-hoc regex parsing of the flat
   wire format is prohibited.

3. **`isFailureBlock()` MUST be used** to filter failure entries in the audit log. Direct
   string matching for `"status;failed"` outside the codec is a code smell.

4. **`encodeField()` and `decodeField()` are exposed** for callers that need to encode
   individual values (e.g., dynamic field construction). They MUST NOT be bypassed with
   `replace(/;/g, ...)` calls.

---

## Rationale

### Why a dedicated codec module?

The alternative — documentation-only — had already failed. Multiple independent
re-implementations existed despite clear format documentation in RFC-0008 and RFC-0011.
A single importable module makes the correct path the easy path.

### Why percent-encoding over URI encoding?

Only semicolons need escaping in this format. Full URI encoding (`encodeURIComponent`)
would escape spaces, colons, and other characters unnecessarily, making blocks less readable
in logs and debug output. `%3B` is the minimal encoding that solves the problem.

### Why not JSON?

RFC-0005 (Zero-JSON Mandate) applies to the agent protocol layer. The flat wire format
predates JSON in the YAMO protocol stack and is optimised for streaming line-by-line
processing. Converting existing audit blocks to JSON would require a breaking migration of
all LanceDB `yamo_blocks` records.

---

## Backwards Compatibility

- **New blocks** written with `encodeBlock()` use `%3B` escaping.
- **Old blocks** in LanceDB written with comma-escape remain readable: `decodeField()`
  only un-escapes `%3B`; commas pass through unchanged.
- **Type changes**: `getSystemLoad()` on `YamoKernel` returns `number` (scalar). The new
  `getSystemMetrics()` method returns `{cpuPercent, memoryMB, uptimeSeconds}`. Bridge
  consumers MUST call `getSystemMetrics()` for structured values.

---

## Security Considerations

The codec is a defence-in-depth layer against **format injection**: a malicious input
containing a literal `;` could inject extra key-value pairs into a YAMO block if the value
were interpolated without escaping. `encodeField()` eliminates this vector by converting
all literal semicolons to `%3B` before the value is embedded in the wire format.

The codec does **not** sanitise field values for LLM prompt safety. That responsibility
belongs to `lib/utils/prompt-security.ts` (RFC-0010-A). Both layers are required:
the codec prevents format corruption; prompt-security prevents prompt injection.

---

## Reference Implementation

- **Codec:** `yamo-os/lib/yamo/block-codec.ts`
- **Tests:** `yamo-os/test/unit/yamo/block-codec.test.ts` (37 tests, 100% line coverage)
- **Migrated writers:**
  - `yamo-os/lib/kernel/kernel.ts` — 6 audit blocks migrated to `encodeBlock()`
  - `yamo-os/lib/kernel/failure-analyzer.ts` — parsing migrated to `decodeBlock()`
  - `yamo-os/lib/gateway/yamo-kernel-bridge.ts` — response parsing uses `extractResponseField()`
- **Companion RFC:** RFC-0010-A (Prompt Field Sanitisation — see `lib/utils/prompt-security.ts`)

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
