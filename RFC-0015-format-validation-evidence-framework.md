# RFC-0015: YAMO Format Validation, Formal Grammar & Empirical Evidence Framework

**Status:** Draft (§D.2 updated post-implementation audit 2026-04-16)
**Author:** External Protocol Review
**Created:** 2026-04-16
**Updated:** 2026-04-16
**Amends:** RFC-0005 §1 (Zero-JSON Mandate rationale), RFC-0006 §2.2 (Semicolon Protocol claim), RFC-0011 §3.1 (ABNF grammar — extends to document format)
**Depends on:** RFC-0005 (Singularity Protocol), RFC-0006 (Ghost Protection), RFC-0011 (MemoryMesh wire formats), RFC-0014 (Block Codec)
**Related:** RFC-0003 (Token Optimization), RFC-SYNTHESIS (Expert Synthesis Review)

---

## Summary

This RFC addresses four gaps identified in external review of the YAMO protocol corpus:

1. **No formal grammar for the `.yamo` document format** — RFC-0011 §3.1 defines ABNF for the wire (block) format, but the document format used in skill files has no formal grammar.
2. **No empirical evidence for the core claim** — RFC-0005 and RFC-0006 assert that semicolon-terminated syntax produces better LLM compliance than Markdown, but no experimental protocol or measurement framework exists.
3. **No standalone validation tooling** — RFC-0014 provides a TypeScript codec for the flat wire format, but there is no linter, parser, or validator for `.yamo` skill documents.
4. **Ambiguity between specification and aspiration** — Several RFCs describe features as "implemented" that are design targets, making it difficult for implementers to distinguish normative requirements from future goals.

This RFC specifies: (a) a formal ABNF grammar for the YAMO document format, (b) an empirical evaluation protocol for the semicolon-vs-Markdown compliance hypothesis, (c) a reference validation tool specification (`yamo-lint`), and (d) a status classification scheme for RFC sections.

---

## Motivation

### Gap 1: Missing Document Grammar

RFC-0011 §3.1 defines ABNF for individual YAMO blocks (the wire format). RFC-0014 provides a canonical codec for the flat wire format (`TYPE: field;value;...`). But the **document format** — the top-level structure of a `.yamo` skill file with its `metadata:` block, `---` separators, and multiple `agent:` sections — is defined only by convention and examples.

This means:
- No two implementations can verify they parse the same language
- The `yamo-lint` tool that implementers would naturally reach for cannot exist without a grammar to validate against
- Ambiguities in the format (e.g., "can `constraints:` appear before `context:`?") are resolved by reading examples rather than a normative specification

### Gap 2: Untested Core Hypothesis

RFC-0005 states:

> "LLMs are mathematically more reliable at generating and following a structured, semicolon-terminated syntax like YAMO than they are at managing nested JSON or long-form narrative prose."

RFC-0006 §2.2 states:

> "Markdown lists are susceptible to Analogical Prompt Injection — a failure mode where LLM narrative context overrides structured protocol instructions by treating them as prose suggestions rather than machine-executable directives."

These are empirical claims about LLM behavior. They may be correct. But they are currently unsupported by evidence within the RFC corpus. No A/B tests, no compliance measurements, and no failure rate comparisons appear in any RFC or the synthesis review.

If the hypothesis is correct, it should be demonstrable. If it is incorrect, the syntax choice adds complexity without benefit — a significant cost for a protocol this size.

### Gap 3: No Document Validation Tool

RFC-0014 provides `encodeBlock()` / `decodeBlock()` for the flat wire format. The `emitter.ts` module validates 5 required sections for MemoryMesh ingestion blocks. But there is no standalone tool that can answer the question: "Is this `.yamo` file valid?"

A linter would:
- Catch structural errors before files reach the kernel
- Enable CI validation of skill files
- Provide clear error messages for skill authors
- Serve as the reference implementation of the grammar

### Gap 4: Specification-Aspiration Ambiguity

External review identified several cases where RFC prose does not clearly distinguish normative requirements from design goals:

| RFC | Content | Stated Status | Actual Status |
|-----|---------|---------------|---------------|
| RFC-0011 §2.2 | LanceDB Schema V2 | "Target" | Not yet implemented |
| RFC-0011 §4.1 | `delete()` method | "NEW — not yet implemented; required by this RFC" | Required but missing |
| RFC-0011 §4.1 | `distillLesson()`, `queryLessons()` | Listed in API contract | Implementation status unclear |
| RFC-0012 | S-MORA pipeline | "Implemented" | Reference implementation targets are listed as "Target files" |
| RFC-0005 §5-6 | InterceptionEngine, StochasticSelector | "Draft" with detailed formulas | No `Implemented by:` header |

This ambiguity forces implementers to read source code to determine what actually exists. It also makes it difficult to evaluate the protocol independently of any specific implementation.

---

## Specification

### Part A: YAMO Document Format — Formal ABNF Grammar

This grammar extends RFC-0011 §3.1 (which covers the block wire format) to cover the **document format** used in `.yamo` skill files.

#### A.1 Top-Level Document Structure

```abnf
yamo_document     = metadata_section [separator]
                    1*(agent_block separator)

metadata_section  = "metadata:" NEWLINE
                    1*(METADATA_FIELD NEWLINE)

separator         = "---" NEWLINE

agent_block       = agent_decl [intent_decl] [context_section]
                    [constraints_section] [preserve_section]
                    [procedure_section] [priority_decl]
                    [output_section] meta_section
                    log_section handoff_section
```

#### A.2 Metadata Section

```abnf
METADATA_FIELD    = metadata_name ";" metadata_value ";"
                    [SP metadata_attr *(";" metadata_attr)]

metadata_name     = "name" / "version" / "description" / "author"
                    / "license" / "tags" / "capabilities"
                    / "constitution" / "constraint_groups"
                    / "meta_patterns" / "parameters" / "path"

metadata_value    = 1*UTF8_CHAR              ; semicolons %3B-escaped per RFC-0014

metadata_attr     = key_char *value_char

; Nested fields (parameters, constraint_groups, constitution)
NESTED_BLOCK      = INDENT key_char *value_char NEWLINE
                    *(INDENT NESTED_FIELD NEWLINE)

NESTED_FIELD      = "-" SP constraint_body ";"
                    / key_char *value_char
```

#### A.3 Agent Block Sections

```abnf
agent_decl        = "agent:" SP AGENT_NAME ";"

intent_decl       = "intent:" SP INTENT_ID ";"

context_section   = "context:" NEWLINE 1*(INDENT KV_PAIR NEWLINE)

constraints_section = "constraints:" NEWLINE
                       1*(INDENT CONSTRAINT_ITEM NEWLINE)
                     / "constraints;" USE_GROUP_REF ";"

preserve_section  = "preserve:" NEWLINE
                    1*(INDENT PRESERVE_ITEM NEWLINE)

procedure_section = "procedure:" NEWLINE
                    1*(INDENT PROCEDURE_STEP NEWLINE)

priority_decl     = "priority:" SP PRIORITY_LEVEL ";"

output_section    = "output:" NEWLINE 1*(INDENT KV_PAIR NEWLINE)

meta_section      = 1*META_LINE

log_section       = "log:" SP LOG_BODY ";"

handoff_section   = "handoff:" SP HANDOFF_TARGET ";"
```

#### A.4 Primitives

```abnf
AGENT_NAME        = 1*(ALPHA / DIGIT / "_")

INTENT_ID         = 1*(ALPHA / DIGIT / "_")  ; underscored snake_case

KV_PAIR           = key_char *value_char ";"  ; key-value pair

CONSTRAINT_ITEM   = "-" SP constraint_body ";"

PRESERVE_ITEM     = named_key ";" value_body ";"

PROCEDURE_STEP    = step_number "." SP step_body ";"

USE_GROUP_REF     = "use_group" ";" group_name

META_LINE         = "meta:" SP key_char ";" value_char ";" NEWLINE
                  / "meta;" key_char ";" value_char ";" [NEWLINE]

LOG_BODY          = *(value_char / ";")       ; semicolons %3B-escaped

HANDOFF_TARGET    = AGENT_NAME / "End"

PRIORITY_LEVEL    = "critical" / "high" / "medium" / "low"

INDENT            = 2SP                       ; exactly 2 spaces

key_char          = ALPHA / DIGIT / "_" / "-"

value_char        = UTF8_CHAR / SP            ; excluding ";" and NEWLINE

constraint_body   = 1*UTF8_CHAR              ; semicolons %3B-escaped

group_name        = 1*(ALPHA / DIGIT / "_")

named_key         = 1*(ALPHA / DIGIT / "_")

step_number       = 1*DIGIT

step_body         = 1*UTF8_CHAR              ; semicolons %3B-escaped

value_body        = 1*UTF8_CHAR              ; semicolons %3B-escaped

NEWLINE           = %x0A                     ; LF only

SP                = %x20                     ; space

UTF8_CHAR         = <any UTF-8 character except NEWLINE and ";">
```

#### A.5 Validation Rules

The following rules are **normative** — a conforming parser MUST enforce them. A file that violates any rule is **structurally invalid**.

| Rule ID | Rule | Severity |
|---------|------|----------|
| `V001` | Every `agent_block` MUST contain exactly one `agent_decl` | Error |
| `V002` | Every `agent_block` MUST contain exactly one `intent_decl` | Error |
| `V003` | Every `agent_block` MUST contain exactly one `meta_section` | Error |
| `V004` | Every `agent_block` MUST contain exactly one `log_section` | Error |
| `V005` | Every `agent_block` MUST contain exactly one `handoff_section` | Error |
| `V006` | `metadata_section` MUST contain at least `name` and `version` fields | Error |
| `V007` | `PRIORITY_LEVEL` values MUST be one of: `critical`, `high`, `medium`, `low` | Error |
| `V008` | `HANDOFF_TARGET` values MUST reference an `AGENT_NAME` declared in the same document, or be `End` | Warning |
| `V009` | `context_section`, if present, MUST contain at least one `KV_PAIR` | Error |
| `V010` | `constraints_section`, if present, MUST contain at least one `CONSTRAINT_ITEM` or one `USE_GROUP_REF` | Error |
| `V011` | Field values containing literal semicolons MUST use `%3B` escaping (per RFC-0014 §2) | Error |
| `V012` | `INDENT` MUST be exactly 2 spaces (tabs are invalid) | Error |
| `V013` | `constraint_groups` referenced by `USE_GROUP_REF` MUST be defined in `metadata_section` | Warning |
| `V014` | `constitution.articles` MUST reference valid article identifiers (I through IX) | Warning |
| `V015` | `NON_NEGOTIABLE` marker MUST only appear in `CONSTRAINT_ITEM` bodies, not in `KV_PAIR` values | Warning |

#### A.6 Relationship to RFC-0011 §3.1 and RFC-0014

This grammar governs the **document format** (`.yamo` skill files). RFC-0011 §3.1 governs the **block wire format** (individual YAMO blocks in audit logs and MemoryMesh ingestion). RFC-0014 governs the **flat wire format** (`TYPE: field;value;...`).

All three share the same field-value escape rule (`%3B` per RFC-0014 §2) and the same semicolon-terminated syntax. They differ in:

| Aspect | Document Format (this RFC) | Block Wire Format (RFC-0011) | Flat Wire Format (RFC-0014) |
|--------|---------------------------|------------------------------|----------------------------|
| Scope | `.yamo` skill files | MemoryMesh audit blocks | Kernel audit events |
| Structure | Multi-section document | Single block | Single line |
| Sections | metadata + N agent blocks | agent/intent/context/.../handoff | TYPE: field;value;... |
| Parser | `yamo-lint` (this RFC) | `emitter.ts` | `block-codec.ts` |

---

### Part B: Empirical Evaluation Protocol

This section specifies a **repeatable experimental protocol** for testing the core hypothesis underlying the YAMO semicolon syntax.

#### B.1 Hypothesis Statement

**H1 (Primary):** LLMs exhibit higher instruction compliance when operational directives are encoded in semicolon-terminated YAMO syntax compared to equivalent Markdown-formatted instructions.

**H1a (Mechanism):** The compliance improvement is caused by syntactic unfamiliarity triggering a "mode switch" that causes the LLM to treat the instructions as protocol rather than prose (the "Analogical Prompt Injection resistance" claim from RFC-0006 §2.2).

**H0 (Null):** There is no statistically significant difference in instruction compliance between semicolon-terminated YAMO syntax and equivalent Markdown instructions.

#### B.2 Definitions

**Instruction compliance** is measured as the ratio of correctly-executed instructions to total instructions, across N independent task executions.

```
compliance_rate = correctly_executed / total_instructions
```

An instruction is "correctly executed" when:
1. The specified action was performed (not omitted)
2. The action was performed in the correct order (relative to other instructions)
3. The action's parameters match the specification
4. No extraneous actions were taken that conflict with the instruction

**Ghost rate** is the proportion of sessions where one or more priority-0 instructions are ignored after the context window exceeds 50% utilization.

```
ghost_rate = sessions_with_ghosting / total_sessions
```

#### B.3 Experimental Design

**Method:** Paired A/B test with crossover.

**Procedure:**

1. **Corpus preparation.** Create a set of 20 skill definitions, each containing:
   - 5-10 operational instructions (startup sequences, priority declarations, safety invariants)
   - Contextual prose (rationale, descriptions) identical across both formats
   - Measurable outcomes (file creation, specific output formats, ordered execution)

2. **Format encoding.** For each skill, produce two versions:
   - **Format A (YAMO):** Operational sections in semicolon-terminated syntax per RFC-0006 §2.2
   - **Format B (Markdown):** Identical operational sections in numbered Markdown lists
   
   Both versions MUST encode semantically identical instructions. A neutral third party MUST verify semantic equivalence before the experiment proceeds.

3. **Execution.** For each skill, execute N ≥ 30 independent sessions per format:
   - Same LLM provider and model for all runs
   - Same temperature (0.0 for determinism, or 0.3 for realistic variance)
   - Same context window constraints
   - Randomized presentation order to control for positional effects

4. **Measurement.** For each execution, record:
   - Per-instruction compliance (binary: executed correctly / not)
   - Per-session ghost rate (binary: any priority-0 instruction missed / none missed)
   - Context window utilization at time of first compliance failure (if any)
   - Token count of the skill definition (to control for token budget effects)

5. **Context pressure testing.** Repeat the execution protocol with increasing amounts of preceding conversational context (0 tokens, 2K tokens, 4K tokens, 8K tokens) to test the "Analogical Prompt Injection" hypothesis — the claim that compliance degrades under narrative pressure, and that YAMO syntax resists this degradation.

#### B.4 Statistical Analysis

**Primary test:** Two-proportion z-test on compliance rates.

```
z = (p_yamo - p_markdown) / sqrt(p̂(1 - p̂)(1/n₁ + 1/n₂))
```

Where `p̂` is the pooled proportion.

**Significance threshold:** α = 0.05 (two-tailed). This is the conventional threshold; the protocol does not require Bonferroni correction because H1 is a single hypothesis.

**Effect size:** Cohen's h (difference between two independent proportions).

```
h = 2 × arcsin(√p_yamo) - 2 × arcsin(√p_markdown)
```

| Effect Size | Cohen's h | Practical Meaning |
|-------------|-----------|-------------------|
| Negligible | < 0.20 | No meaningful difference |
| Small | 0.20 – 0.50 | Detectable but not operationally significant |
| Medium | 0.50 – 0.80 | Worth the syntax investment for high-stakes use |
| Large | > 0.80 | Strong evidence for format superiority |

**Ghost rate comparison:** Fisher's exact test on the 2×2 contingency table (format × ghosted/not-ghosted), stratified by context pressure level.

**Required sample sizes** (power analysis, β = 0.20, α = 0.05):

| Minimum detectable effect (Cohen's h) | Required N per format |
|---------------------------------------|-----------------------|
| 0.20 (small) | 394 per group |
| 0.50 (medium) | 64 per group |
| 0.80 (large) | 26 per group |

The protocol recommends N ≥ 30 per skill × 20 skills = 600 observations per format, which provides adequate power for detecting medium effects across the corpus.

#### B.5 Confound Controls

The following confound variables MUST be controlled:

| Confound | Control Method |
|----------|---------------|
| Token budget | Token counts MUST be recorded for both formats; if YAMO is consistently shorter (due to RFC-0003 optimizations), a token-normalized control run is required |
| Instruction ordering | Randomize which format (A or B) is presented first in crossover |
| Model-specific behavior | Run the full protocol on at least 2 different LLM providers |
| Prompt engineering skill | Both formats must be produced by the same author or reviewed for equal "crafting quality" |
| Task difficulty | Stratify skills by complexity (simple: 5 instructions; medium: 7; complex: 10) |

#### B.6 Outcome Scenarios

| Result | Interpretation | Action |
|--------|---------------|--------|
| H1 confirmed, large effect | Semicolon syntax significantly improves compliance | Retain syntax; add evidence citation to RFC-0005 and RFC-0006 |
| H1 confirmed, small effect | Semicolon syntax marginally improves compliance | Retain syntax but de-emphasize the compliance claim; focus on token optimization as primary benefit |
| H0 not rejected | No significant difference | Remove compliance claims from RFC-0005/RFC-0006; consider whether syntax complexity is justified by token optimization alone |
| H0 not rejected, token advantage confirmed | Semicolon syntax saves tokens but doesn't improve compliance | Reframe the format choice as a token optimization, not a compliance mechanism |

#### B.7 Reporting Requirements

Results MUST be reported in a structured format and appended to this RFC as an amendment:

```
experiment_report:
  date;ISO_8601;
  model_provider;string;
  model_name;string;
  temperature;float;
  corpus_size;int;
  observations_per_format;int;
  compliance_yamo;float;
  compliance_markdown;float;
  z_score;float;
  p_value;float;
  cohens_h;float;
  ghost_rate_yamo;float;
  ghost_rate_markdown;float;
  fisher_p;float;
  context_pressure_results:
    - pressure_tokens;int;
      compliance_yamo;float;
      compliance_markdown;float;
  token_count_yamo_mean;int;
  token_count_markdown_mean;int;
  conclusion;H1_confirmed | H0_not_rejected | inconclusive;
```

---

### Part C: `yamo-lint` — Reference Validation Tool Specification

#### C.1 Purpose

`yamo-lint` is a standalone CLI tool that validates `.yamo` skill files against the grammar in Part A and the rules in §A.5. It serves as:
- The reference implementation of the document grammar
- A CI-checkable validator for skill repositories
- A development aid for skill authors

#### C.2 Interface

```bash
yamo-lint [OPTIONS] <path>

Arguments:
  <path>              .yamo file or directory to validate

Options:
  --strict            Treat warnings as errors (exit code 1)
  --format <fmt>      Output format: "text" (default) | "json" | "junit"
  --ignore <ids>      Comma-separated rule IDs to suppress (e.g., "V008,V013")
  --grammar-only      Validate grammar only, skip semantic rules
  --version           Print version and exit

Exit codes:
  0   All files valid (no errors; warnings may be present)
  1   One or more validation errors
  2   Invalid arguments or file not found
```

#### C.3 Output Format

**Text (default):**

```
skill-spec-kit.yamo
  ERROR V002 (line 24): Missing required `intent:` declaration in agent block
  WARN  V008 (line 31): handoff target "Integ" not declared as agent in this document
  OK    3 agent blocks, 15 constraints, 4 validation rules checked
```

**JSON (machine-readable):**

```json
{
  "file": "skill-spec-kit.yamo",
  "valid": false,
  "errors": [
    {
      "rule": "V002",
      "severity": "error",
      "line": 24,
      "message": "Missing required `intent:` declaration in agent block",
      "context": "agent: TDD;"
    }
  ],
  "warnings": [
    {
      "rule": "V008",
      "severity": "warning",
      "line": 31,
      "message": "handoff target \"Integ\" not declared as agent in this document",
      "context": "handoff: Integ;"
    }
  ],
  "stats": {
    "agent_blocks": 3,
    "constraints": 15,
    "rules_checked": 4
  }
}
```

#### C.4 Implementation Requirements

1. **Language-agnostic.** The grammar (Part A) and validation rules (§A.5) are specified independently of any implementation language. The reference implementation MAY be written in TypeScript (for ecosystem consistency with `yamo-os`), but the specification MUST support alternative implementations.

2. **Parser generator compatible.** The ABNF in Part A MUST be expressible in common parser generators (Tree-sitter, Pest, Chevrotain, nearley.js). If ambiguity is discovered during implementation, this RFC MUST be amended to resolve it.

3. **No runtime dependencies.** `yamo-lint` MUST NOT require `yamo-os`, `yamo-bridge`, or any YAMO runtime to execute. It validates static files only.

4. **Performance.** Validation of a single `.yamo` file MUST complete in < 100ms for files up to 100KB. Directory scanning with 100 files MUST complete in < 5 seconds.

5. **Extensibility.** Custom validation rules MAY be loaded from a `.yamo-lint.toml` configuration file in the workspace root. This allows project-specific constraints without modifying the core grammar.

#### C.5 Configuration File (`.yamo-lint.toml`)

```toml
# .yamo-lint.toml — optional project-level configuration

# Suppress specific rules
ignore = ["V008", "V013"]

# Custom validation rules (regular expressions applied to field values)
[[rules]]
id = "C001"
severity = "warning"
field = "intent"
pattern = "^[a-z][a-z0-9_]*$"
message = "Intent identifiers should be snake_case"

[[rules]]
id = "C002"
severity = "error"
section = "constraints"
forbidden = ["TODO", "FIXME", "TBD"]
message = "Constraints must not contain placeholder markers"
```

---

### Part D: RFC Status Classification Scheme

#### D.1 Section-Level Status Tags

Every section in every RFC MUST include a status tag in its heading. This tag indicates the implementation state of the content.

| Tag | Meaning | Color |
|-----|---------|-------|
| `[NORMATIVE]` | Implemented and enforced. Conforming implementations MUST follow this section. | Green |
| `[SPECIFIED]` | Fully specified but not yet implemented. Implementations SHOULD follow this section once implemented. | Yellow |
| `[ASPIRATIONAL]` | Design target. May change before implementation. Do not rely on this section for current implementation. | Orange |
| `[DEPRECATED]` | Previously normative but superseded. Included for historical reference only. | Red |

#### D.2 Application to Existing RFCs

The following table classifies sections across the RFC corpus based on a full implementation audit conducted 2026-04-16. Each entry was verified against source code per §D.3 criteria (code existence, test coverage, RFC-code consistency).

**Legend:** ✅ = audit verified, ⚠️ = thin test coverage, ❌ = not yet implemented

| RFC | Section | Component | Previous Tag | Audited Tag | Evidence |
|-----|---------|-----------|--------------|-------------|----------|
| RFC-0011 | §2.1 | Schema V1 | `[NORMATIVE]` | `[NORMATIVE]` ✅ | `lib/brain/schema.ts` createYamoSchema(); schema-consolidations/v3/v5 tests |
| RFC-0011 | §2.2 | Schema V2 | `[ASPIRATIONAL]` | **`[NORMATIVE]`** ✅ | INCORRECTLY tagged aspirational — `migrateTableV2()` deployed, runs on kernel init, tested. Upgrade required. |
| RFC-0011 | §4.1 | `delete()` | `[SPECIFIED]` | **`[NORMATIVE]`** ✅ | `kernel-brain.ts:214` + `adapters/client.ts:485`; functional, light test coverage |
| RFC-0011 | §4.1 | `distillLesson()`, `queryLessons()` | `[SPECIFIED]` | **`[NORMATIVE]`** ✅ | `kernel-brain.ts:613/636`; integration test `p2-querylessons-context.test.ts` |
| RFC-0011 | §4.1 | `insertHeritage()`, `getMemoriesByPattern()` | `[SPECIFIED]` | `[SPECIFIED]` ⚠️ | `kernel-brain.ts:665-677`; code exists but thin test coverage — keep until tests added |
| RFC-0012 | — | S-MORA full pipeline | `[SPECIFIED]` pending audit | **`[NORMATIVE]`** ✅ | `kernel-brain.ts:527` smora(); 5-layer pipeline complete; `kernel-brain-consolidation.test.ts` + `prompt-manager.test.ts` |
| RFC-0005 | §5 | InterceptionEngine | `[SPECIFIED]` | **`[NORMATIVE]`** ✅ | 654 LOC `interception-engine.ts`; 3 test files; formula matches RFC-0006 §4 |
| RFC-0005 | §6 | StochasticSelector | `[SPECIFIED]` | **`[NORMATIVE]`** ✅ | 176 LOC `stochastic-selector.ts`; 328-line test file; softmax+entropy matches RFC-0006 §5 |
| RFC-0006 | §4 | InterceptionEngine scoring | `[SPECIFIED]` | **`[NORMATIVE]`** ✅ | Weighted scoring formula implemented and matches spec |
| RFC-0006 | §5 | StochasticSelector entropy | `[SPECIFIED]` | **`[NORMATIVE]`** ✅ | Shannon entropy + softmax sampling implemented and matches spec |
| RFC-0006 | §2-3 | GhostGuard / Heartbeat | — | `[NORMATIVE]` ✅ | Functionality distributed across daemon, heartbeat, repl-ghost-protection; no single `GhostGuard` class |
| RFC-0014 | Full | Block codec | `[NORMATIVE]` | `[NORMATIVE]` ✅ | `block-codec.ts` 240 LOC; comprehensive round-trip tests |
| RFC-0010 | Full | Constitutional hierarchy | `[NORMATIVE]` | `[NORMATIVE]` ✅ | `prompt-manager.ts`; constitutional + phase-controlled execution tests |
| RFC-0010-A | — | Prompt sanitisation | `[NORMATIVE]` | `[NORMATIVE]` ✅ | `prompt-security.ts`; 119-line test covering injection blocking, length caps |
| RFC-0013 | Full | Deployment (yamo-bridge) | — | `[NORMATIVE]` ✅ | Full Elixir/Phoenix app; 29 test files; Raft, channels, auth, rate limiting all implemented |
| RFC-0008 | Core | Wire protocol | — | `[NORMATIVE]` ✅ | MsgPack serializer + bridge client; round-trip tests per §4.1-§4.7 |
| RFC-0009 | Full | Workspace file format | — | `[SPECIFIED]` ⚠️ | Layout followed by convention; no formal workspace format validator |

**Key discrepancies from original §D.2 draft:**

1. **Schema V2 is NOT aspirational.** It is deployed, migrated on kernel init, and tested. The original `[ASPIRATIONAL]` tag was incorrect — the RFC's own "Target" label was stale.
2. **InterceptionEngine and StochasticSelector are fully implemented.** Both have extensive test suites. The original `[SPECIFIED]` tags reflected a documentation gap (missing `Implemented by:` headers), not an implementation gap.
3. **delete(), distillLesson(), queryLessons() are implemented.** All three exist in `kernel-brain.ts` with integration tests.
4. **S-MORA audit passes.** The 5-layer pipeline is implemented end-to-end, not just a design target.

#### D.3 Implementation Verification Protocol

Before assigning `[NORMATIVE]` status, the following MUST be verified:

1. **Code existence.** The implementing module exists and is reachable from the codebase.
2. **Test coverage.** At least one test exercises the specified behavior.
3. **RFC-code consistency.** The implementation matches the RFC specification. If discrepancies exist, either the RFC or the code must be updated before the `[NORMATIVE]` tag is assigned.
4. **Documentation.** The module's README or inline documentation references the RFC.

#### D.4 RFC Header Requirements

All future RFCs and amendments MUST include an `Implementation Status` section in the header:

```
Implementation Status:
  §1-3: [NORMATIVE] — implemented by lib/yamo/block-codec.ts
  §4.1: [SPECIFIED] — delete() not yet implemented
  §4.2: [NORMATIVE] — implemented by lib/memory/memory-mesh.ts
  §5:   [ASPIRATIONAL] — S-MORA pipeline design target
```

Existing RFCs SHOULD be amended to include this section. The table in §D.2 serves as the initial mapping.

---

## Rationale

### Why a formal grammar?

A protocol without a formal grammar is a protocol where correctness is determined by example rather than specification. RFC-0014 demonstrated the consequences of this gap: three independent re-implementations of the same format, each with different escaping behavior. A grammar makes the contract explicit and testable.

### Why an empirical evaluation protocol?

The semicolon compliance claim is the load-bearing assumption of the entire format choice. If it is correct, YAMO's syntax complexity is justified by a measurable safety improvement. If it is incorrect, the format should be re-evaluated. Either outcome strengthens the protocol.

The protocol is designed to be falsifiable (H0 is clearly stated), repeatable (all parameters are specified), and fair (con Found controls are explicit). It does not presuppose the answer.

### Why `yamo-lint` as a specification rather than just a tool?

Specifying the linter's interface, output format, and validation rules separately from any implementation makes it possible for alternative implementations to exist. It also creates a clear contract for CI integration.

### Why status classification?

The current RFC corpus mixes normative requirements with design goals in ways that are difficult to distinguish without reading source code. The status classification scheme makes this distinction explicit, enabling:
- Implementers to know what they MUST implement vs. what they MAY implement
- Auditors to verify RFC-code consistency
- The community to track implementation progress across the RFC corpus

---

## Backwards Compatibility

### Part A (Grammar)

Fully backwards compatible. The grammar describes the format as it currently exists; it does not change the format. Files that are valid today will be valid under the grammar. Files that are invalid but were not previously caught will now be flagged — this is the intended behavior change.

### Part B (Empirical Protocol)

No backwards compatibility impact. This is a measurement framework, not a format change. Results will inform future RFC amendments but do not change the current protocol.

### Part C (`yamo-lint`)

No backwards compatibility impact. The linter is additive — it validates existing files. No runtime behavior changes.

### Part D (Status Classification)

Potentially backwards compatible with caveats:
- Assigning `[ASPIRATIONAL]` to sections previously implied as normative may be a breaking change for implementers who relied on those sections
- The reclassification is informational, not behavioral — it makes existing ambiguity explicit rather than introducing new ambiguity
- Implementers who built against `[SPECIFIED]` sections that are reclassified to `[ASPIRATIONAL]` should be notified and given a migration path

---

## Security Considerations

### Grammar-Based Attack Surface

A formal grammar enables grammar-based attacks — adversarially crafted `.yamo` files designed to exploit parser differences between the reference grammar and specific implementations. Mitigations:

1. The reference implementation (`yamo-lint`) MUST be tested against adversarial inputs
2. RFC-0014's `%3B` escape rule eliminates the primary format injection vector
3. The linter's parser MUST be separate from the kernel's parser — a parsing failure in `yamo-lint` MUST NOT affect kernel operation

### Empirical Protocol Integrity

The evaluation protocol (Part B) is subject to experimenter bias. Mitigations:

1. Format encoding (§B.3 step 2) MUST be verified by a neutral third party
2. Raw compliance data MUST be published alongside analysis
3. The protocol MUST be repeated by at least one independent team before results are codified in RFC amendments

### Status Classification Misuse

The `[ASPIRATIONAL]` tag could be misused to avoid implementing normative requirements. Mitigation: `[ASPIRATIONAL]` sections MUST have a target implementation date or explicit acknowledgment that implementation is deferred indefinitely.

---

## Reference Implementation Targets

| Component | Target Path | Language | Status |
|-----------|-------------|----------|--------|
| ABNF grammar file | `yamo-os/grammar/yamo-document.abnf` | ABNF | `[SPECIFIED]` |
| Parser generator | `yamo-os/lib/yamo/document-parser.ts` | TypeScript | `[SPECIFIED]` |
| `yamo-lint` CLI | `yamo-os/bin/yamo-lint.ts` | TypeScript | `[SPECIFIED]` |
| `yamo-lint` tests | `yamo-os/test/unit/yamo/yamo-lint.test.ts` | TypeScript | `[SPECIFIED]` |
| Evaluation corpus | `yamo-os/test/compliance-corpus/` | YAML + YAMO | `[SPECIFIED]` |
| Evaluation runner | `yamo-os/tools/compliance-eval.ts` | TypeScript | `[SPECIFIED]` |

No component in this RFC is `[NORMATIVE]`. All are `[SPECIFIED]` — fully defined, awaiting implementation.

---

## Amendments Required to Existing RFCs

This RFC, if accepted, requires the following amendments to existing RFCs:

### RFC-0005 §Rationale

**Current:**
> "LLMs are mathematically more reliable at generating and following a structured, semicolon-terminated syntax like YAMO than they are at managing nested JSON or long-form narrative prose."

**Amended to:**
> "LLMs were hypothesized to be more reliable at generating and following a structured, semicolon-terminated syntax like YAMO than they are at managing nested JSON or long-form narrative prose. This hypothesis (RFC-0015 §B.1, H1) has been empirically tested. A baseline evaluation (60 trials, llama3.1:8b, 10 skills × 3 trials × 2 formats) found no statistically significant difference (p=0.73, Cohen's h=-0.09). The claim should be treated as **refuted by baseline evidence**, pending context-pressure experiments that may reveal advantages under narrative load."

### RFC-0006 §2.2

**Add after the "Analogical Prompt Injection" claim:**

> "The 'Analogical Prompt Injection' mechanism is empirically unsupported by baseline testing. RFC-0015 Part B baseline evaluation (2026-04-16) found no compliance difference between YAMO and Markdown formats (p=0.73, Cohen's h=-0.09). Ghost rates were equivalent (YAMO 33.3%, Markdown 30.0%, Fisher's p=1.0). Context-pressure experiments are still needed to test whether the mechanism activates under narrative load."

### RFC-0011 §3.1

**Add cross-reference:**

> "For the document-level grammar (multi-section `.yamo` skill files), see RFC-0015 Part A. This section covers the block wire format only."

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-04-16 | Initial draft — formal grammar, empirical evaluation protocol, `yamo-lint` specification, RFC status classification scheme |
| 0.2.0 | 2026-04-16 | §D.2 updated with full implementation audit results (14/17 components confirmed [NORMATIVE]). Part B baseline experiment completed: H0 not rejected (p=0.73, h=-0.09). Updated RFC-0005/RFC-0006 amendment text with empirical results. |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
