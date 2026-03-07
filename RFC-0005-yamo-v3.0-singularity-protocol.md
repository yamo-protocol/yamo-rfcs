# RFC-0005: YAMO v3.0 - The Singularity Protocol

**Status:** Draft
**Author:** Soverane Labs & Collaborative Swarm
**Created:** 2026-02-17
**Updated:** 2026-03-07
**Supercedes:** RFC-0001, RFC-0002, RFC-0003 (for v3.0 features)
**Related:** RFC-0006 (Autonomous Kernel — InterceptionEngine and StochasticSelector specs), RFC-0010 (Constitutional Value Hierarchy — runtime constitutional gating at micro level)

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

#### §2.1 Valid Agent Block Keys

In addition to the core agent block fields (`intent:`, `context:`, `constraints:`, `output:`, `log:`, `priority:`, `meta:`, `handoff:`), the following optional keys are valid in v3.0 agent blocks:

| Key | Purpose | Defined In |
|-----|---------|------------|
| `preserve:` | Named items that MUST be reproduced verbatim — conditional branches, exact paths, multi-option content | RFC-0003 §6.2 |
| `procedure:` | Ordered numbered steps that MUST NOT be collapsed to a summary — sequential checklists and workflows | RFC-0003 §6.3 |

Both keys address the **LLM-as-parser compression failure** modes identified during v3.0 production deployment: verbatim templates, ordered procedures, and conditional branches are systematically degraded when represented as flat constraint items. These keys signal to the executing LLM that the content is non-compressible.

See RFC-0003 §6 for the complete Verbatim Preservation Primitives specification, including the `;verbatim;` constraint modifier and YAML literal block scalar (`key: |`) patterns.

### 3. Formal Intent Semantics

Agent intents are no longer descriptive text; they are formal, underscored identifiers that map to the agent's cognitive capabilities.

### 4. Mandatory Meta-Reasoning

Every agent execution must produce a `meta` block containing:
- `hypothesis`: The expected outcome.
- `rationale`: The justification for the chosen path.
- `confidence`: Numeric score (0.0-1.0).
- `observation`: Post-execution findings.

### 5. InterceptionEngine Decision Matrix

The Singularity Kernel employs a multi-factor weighted scoring system to determine when a user command or agent intent should be intercepted by a synthesized skill.

#### §5.1 Weighted Scoring Formula

The final `weightedScore` for a skill candidate is calculated as follows:

```
weightedScore = 
  0.6 × semanticScore + 
  0.3 × reliability(skill) + 
  0.1 × useConfidence(skill)
```

Where:
- **semanticScore**: Vector similarity from LanceDB embedding search [0, 1].
- **reliability**: `success_rate × 0.5^(daysSinceLastUse / 30)`, floored at 0.3.
- **useConfidence**: `log(useCount + 1) / log(100 + 1)`, capped at 1.0.

#### §5.2 Decision Gates

Candidates are evaluated through a sequence of gates:
1.  **Parameter Check**: If required parameters are missing → **BLOCK**.
2.  **Self-Interception**: If the intent originates from the skill itself (recursion) → **BLOCK**.
3.  **Core Commands**: If the command is a core system command, `weightedScore` must be ≥ 0.85.
4.  **User Commands**: If the command is a user-defined intent, `weightedScore` must be ≥ 0.35.
5.  **Reliability Floor**: If `reliability` < 0.5 → **BLOCK** (unless overridden by manual bypass).

### 6. StochasticSelector & Shannon Entropy

To prevent "agentic stagnation" and ensure system diversity, the kernel uses probabilistic selection for non-critical skills, governed by Shannon Entropy.

#### §6.1 Normalized Shannon Entropy (H_norm)

Diversity in the candidate pool is measured via Normalized Shannon Entropy:

```
H_norm = -Σ p(i) × log2(p(i)) / log2(k)
```

Where:
- **k**: Number of unique skills in the candidate pool.
- **H_norm ∈ [0, 1]**: 0 = deterministic (exploitation), 1 = maximum diversity (exploration).

#### §6.2 Softmax Sampling

The selection probability for a skill is adjusted based on the current system entropy:

1.  **Exponential Scores**: `expScores = [exp(score / temp) for each candidate score]`.
2.  **Base Probabilities**: `probs = softmax(expScores)`.
3.  **Entropy Weighting**: `adjustedProbs = (1 - H_norm) × probs + H_norm / k`.

#### §6.3 Entropy Adaptation

The system automatically tunes its "curiosity" (target entropy) based on recent performance:
- **Explore**: If `successRate > 0.7` and `H_norm < 0.2` → Increase `targetEntropy` (encourage diversity).
- **Exploit**: If `successRate < 0.4` and `H_norm > 0.6` → Decrease `targetEntropy` (favor high-reliability skills).

## Rationale

Moving to a protocol-native architecture eliminates the "translation layer" between the agent's logic and its communication. LLMs are mathematically more reliable at generating and following a structured, semicolon-terminated syntax like YAMO than they are at managing nested JSON or long-form narrative prose.

## Backwards Compatibility

- **Breaking Change**: v3.0 agents cannot parse legacy JSON-based state from v1.x skills without an adapter.
- **Migration Path**: A `yamo-v3-upgrade` tool will be provided to transform v1.x skills into v3.0 Modular Translator format.

## Security Considerations

- **Prompt Injection**: By formalizing the syntax, we reduce the surface area for narrative-based prompt injection.
- **State Integrity**: Native YAMO state can be mathematically verified for consistency against the constitutional soul.

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-17 | Initial draft — Zero-JSON Mandate, YAMO Frontmatter, Formal Intent Semantics, Mandatory Meta-Reasoning |
| 0.1.1 | 2026-02-21 | Add RFC-0006 and RFC-0010 cross-references; §1.1 Scope Clarification was added to resolve Zero-JSON tension with TypeScript implementation layer |
| 0.1.2 | 2026-03-07 | Add §2.1 Valid Agent Block Keys — `preserve:` and `procedure:` formally recognized as optional v3.0 agent block fields; cross-reference RFC-0003 §6 Verbatim Preservation Primitives |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
