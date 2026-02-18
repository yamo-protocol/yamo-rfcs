# RFC-0005: YAMO v3.0 - The Singularity Protocol

**Status:** Draft
**Author:** Soverane Labs & Collaborative Swarm
**Created:** 2026-02-17
**Supercedes:** RFC-0001, RFC-0002, RFC-0003 (for v3.0 features)

## Summary

This RFC defines **YAMO v3.0**, also known as the **Singularity Protocol**. It introduces the **Zero-JSON Mandate**, replacing fragile narrative instructions and structured data blobs with a **Formal Semantic Protocol**. By moving to native YAMO context chains for all agent-to-agent and agent-to-kernel communication, YAMO v3.0 achieves near-absolute parsing reliability and unbroken intent heritage.

## Motivation

While YAMO v1.x and v2.x established a foundation for structured agent workflows, they still relied on hybrid patterns where agents passed JSON blobs or relied on narrative prompt instructions. This led to several failure modes in production:

1. **Syntax Hallucinations**: Agents frequently corrupted JSON structure in long-running sessions.
2. **Context Dilution**: Narrative instructions were often ignored or "diluted" as the context window saturated.
3. **Intent Drift**: The "Why" behind a decision was lost during the handoff between agents.
4. **Parsing Fragility**: Small variations in LLM output caused catastrophic workflow failures.

YAMO v3.0 solves these by enforcing a **Protocol-Native** environment where the language of orchestration *is* the language of state.

## Specification

### 1. The Zero-JSON Mandate

In YAMO v3.0, the use of JSON for state passing or agent instructions is **strictly prohibited** (Critical Priority).

*   **State Mode**: `llm_first` (native YAMO).
*   **Inter-Agent Communication**: Must use `.yamo` context files or blocks.
*   **Kernel Interface**: Commands are issued as semantic intents, never as JSON parameters.

#### §1.1 Scope Clarification

The Zero-JSON Mandate applies **exclusively** to the YAMO agent protocol layer — specifically to:
- `.yamo` skill files and skill block definitions
- Agent-to-agent context passing (YAMO context chains)
- Kernel interface commands and semantic intent declarations
- Inter-agent state passing in the YAMO runtime

The mandate does **not** apply to:
- **TypeScript/JavaScript implementation code** — internal types, interfaces, and data structures in `.ts`/`.js` source files may freely use JSON, objects, and typed schemas
- **LanceDB schema and storage** — database adapters use JSON metadata fields for persistence
- **CLI tools** — command-line interfaces may accept and emit JSON for machine-readable output
- **Test code** — unit and integration tests may use any format appropriate for the test framework
- **HTTP/WebSocket interfaces** — network protocol layers may use JSON for external interoperability (e.g., gateway RPC per RFC-0008)
- **Configuration files** — `package.json`, `tsconfig.json`, and environment configs are exempt

**Rationale**: The mandate targets the cognitive protocol layer where YAMO semantic parsing provides reliability and injection resistance. Implementation internals already have type-safe alternatives (TypeScript interfaces) that achieve the same goals through different means.

### 2. YAMO Frontmatter (Standardized)

All v3.0 skills must use the standardized YAML-compatible frontmatter for metadata:

```yamo
---
name: SkillName
version: 3.0.0
module_type: orchestrator | workflow | foundation
architecture: modular | simple
dependencies:
  - foundational/module.yamo
capabilities:
  - capability_name
---
```

### 3. Formal Intent Semantics

Agent intents are no longer descriptive text; they are formal, underscored identifiers that map to the agent's cognitive capabilities.

### 4. Mandatory Meta-Reasoning

Every agent execution must produce a `meta` block containing:
- `hypothesis`: The expected outcome.
- `rationale`: The justification for the chosen path.
- `confidence`: Numeric score (0.0-1.0).
- `observation`: Post-execution findings.

## Rationale

Moving to a protocol-native architecture eliminates the "translation layer" between the agent's logic and its communication. LLMs are mathematically more reliable at generating and following a structured, semicolon-terminated syntax like YAMO than they are at managing nested JSON or long-form narrative prose.

## Backwards Compatibility

- **Breaking Change**: v3.0 agents cannot parse legacy JSON-based state from v1.x skills without an adapter.
- **Migration Path**: A `yamo-v3-upgrade` tool will be provided to transform v1.x skills into v3.0 Modular Translator format.

## Security Considerations

- **Prompt Injection**: By formalizing the syntax, we reduce the surface area for narrative-based prompt injection.
- **State Integrity**: Native YAMO state can be mathematically verified for consistency against the constitutional soul.

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
