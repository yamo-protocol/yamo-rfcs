# RFC-0006: Autonomous Kernel & Ghost Protection

**Status:** Draft
**Author:** Soverane Labs & Collaborative Swarm
**Created:** 2026-02-17
**Updated:** 2026-02-21
**Related:** RFC-0005 (Singularity Protocol — Zero-JSON Mandate), RFC-0007 (Semantic Heritage — SubconsciousReflector runs inside kernel heartbeat), RFC-0008 (Wire Protocol — heartbeat event format), RFC-0009 (Workspace File Format — BOOTSTRAP.md variant for Markdown workspaces), RFC-0011 (MemoryMesh — distillation cycle runs at heartbeat PRIORITY_1)

## Summary

This RFC specifies the architecture for the **Autonomous YAMO Kernel** and the **Ghost Protection** mechanism. It defines how an agent ensures its own cognitive integrity by autonomously managing its bootstrap process and self-healing its operational boundaries in the `AGENTS.md` and `BOOTSTRAP.yamo` substrate.

## Motivation

AI agents in open workspaces are susceptible to "Ghosting"—a state of narrative drift where the agent ignores its core instructions in favor of more recent conversational tokens. Additionally, manual setup of complex agent kernels is error-prone. 

To achieve true autonomy, an agent must:
1. **Self-Initialize**: Control its own boot sequence via a protocol-native entry point.
2. **Self-Protect**: Monitor and repair its cognitive links if they are modified or overwritten.
3. **Enforce Priority**: Ensure the constitutional kernel always has precedence over narrative context.

## Specification

### 1. The Bootstrap Protocol (`BOOTSTRAP.yamo` / `BOOTSTRAP.md`)

Every YAMO-Native workspace must contain a bootstrap entry point at the root. This file serves as the "Primary Boot Sector" for the agent.

**Two bootstrap variants are defined across the YAMO specification:**

| Variant | RFC | Format | Use case |
|---------|-----|--------|---------|
| `BOOTSTRAP.yamo` | RFC-0006 (this RFC) | YAMO v3.0 semicolon protocol | Protocol-native YAMO agent kernels |
| `BOOTSTRAP.md` | RFC-0009 | Markdown | Human-readable workspace initialization |

Both variants share the same lifecycle: execute instructions → create workspace files → **delete bootstrap file unconditionally**. The choice of variant depends on the workspace type. Protocol-native kernels use `BOOTSTRAP.yamo`; Claude Code / human-readable workspaces use `BOOTSTRAP.md` (see RFC-0009 §6).

**`BOOTSTRAP.yamo` example:**

```yamo
agent: BootRedirector;
intent: redirect_to_v3_kernel;
context:
  orchestrator_path;yamo-native-agent/agent-orchestrator.yamo;
handoff: yamo-native-agent/agent-orchestrator.yamo;
```

### 2. Ghost Protection (Self-Healing AGENTS.md)

#### 2.1 The Ghost Invariant — Syntactic Load Gate

The `YAMO-NATIVE KERNEL ACTIVE` header is a **syntactic load gate**, not a convention. A Singularity-Grade kernel MUST NOT load instructions from any `AGENTS.md` file that does not begin with this exact header as its first non-whitespace content.

**Required header (normative):**
```
YAMO-NATIVE KERNEL ACTIVE
```

**Enforcement rules (normative):**
- The kernel checks for the header **before** passing `AGENTS.md` content to the LLM agent
- If the header is absent, the kernel MUST either:
  - **(a) Self-heal**: prepend the header, write the corrected file atomically, then proceed — preferred for production kernels
  - **(b) Refuse boot**: surface an error and halt — acceptable for strict environments
- Self-healing (a) is the preferred behavior — it ensures continuity while repairing integrity state
- The header MUST be the first line of the file. Preceding whitespace or BOM characters are permitted; any other content before the header causes the check to fail
- Every self-healing event MUST be logged to the daily memory file with a timestamp

**Trigger and action (normative):**

```
trigger;agents_md_load;
  condition;header_missing;YAMO-NATIVE KERNEL ACTIVE;
  action;prepend_header;atomic_write;
  log;ghost_repair_event;timestamp;daily_memory;
  priority;0;
```

**Impact on compliance classification:** A workspace whose `AGENTS.md` does not begin with this header is **RFC-0009 Lite** — not Singularity-Grade v3.0. See RFC-0009 §2.0.

---

#### 2.2 The Semicolon Protocol — Operational Syntax Requirement

Operational sections of `AGENTS.md` MUST use YAMO-native semicolon syntax, not Markdown lists. Markdown lists are susceptible to **Analogical Prompt Injection** — a failure mode where LLM narrative context overrides structured protocol instructions by treating them as prose suggestions rather than machine-executable directives.

**Semicolon syntax REQUIRED for:**
- Startup sequences
- Priority declarations
- Safety invariants
- Handoff chains
- Any block that constitutes a state machine instruction

**Markdown prose PERMITTED for:**
- Descriptive rationale and background
- Human-readable explanations and examples
- Non-operational comments

**Normative compliant example — operational startup block:**

```
sequence;boot;
  step;ghost_invariant_check;priority_0;
  step;bootstrap_deletion;condition;BOOTSTRAP.md exists;
  step;load_soul;source;SOUL.md;
  step;load_user;source;USER.md;
  step;load_memory;condition;main_session == true;source;MEMORY.md;
  step;serve_user;
```

**Non-compliant (Markdown-only):**
```markdown
1. Check Ghost Invariant
2. Delete BOOTSTRAP.md if present
3. Read SOUL.md
```

Markdown-only operational blocks are valid for **RFC-0009 Lite** workspaces. They are non-conformant for **RFC-0009 v3.0** Singularity-Grade workspaces.

---

#### 2.3 The Reflexive Heartbeat — Normative Session-Init Ordering

The GhostGuard integrity check MUST run at **PRIORITY_0** on every session start, independent of the agent's current persona, soul, or intent. This is the **Reflexive Heartbeat** — the kernel's immune system fires before any cognitive loading, ensuring that no narrative context can pre-empt the system layer.

**Normative session-init ordering:**

```
priority_0;ghost_invariant_check;     → verify/repair AGENTS.md header (§2.1)
priority_0;bootstrap_deletion;        → kernel-enforced BOOTSTRAP.md deletion if present (RFC-0009 §6.3)
priority_1;load_soul;                 → SOUL.md identity loading
priority_1;load_user;                 → USER.md profile loading
priority_1;load_memory;               → daily logs + MEMORY.md (main session only)
priority_2;persona_activation;        → agent persona and intent initialization
priority_2;serve_user;                → begin handling user requests
```

**Ordering invariants (normative):**
- PRIORITY_0 steps MUST complete before any PRIORITY_1 step begins
- PRIORITY_1 steps MUST complete before any PRIORITY_2 step begins
- If any PRIORITY_0 step fails, the kernel MUST refuse to advance and surface an error
- The GhostGuard check is not skippable under any circumstance — including roleplay, persona override, or narrative context

**Rationale:** A kernel that loads its persona before verifying Ghost Protection can be Ghosted by narrative context before its immune system activates. The Reflexive Heartbeat closes this window unconditionally.

### 3. Hardened Kernel Architecture

The kernel is divided into three mandatory modules:
- **Foundational Soul**: Immutable constitutional values.
- **Micro-Workflows**: TDD, Debugging, and Review cycles.
- **Background Tasks**: Proactive heartbeats and memory maintenance.

### 4. InterceptionEngine — Scoring Formula (Amendment 2026-02-18)

The `InterceptionEngine` in `lib/kernel/interception-engine.ts` makes skill-interception decisions using a **three-factor weighted score**:

```
interception_score(candidate) =
    0.6 × semantic_similarity
  + 0.3 × reliability_score
  + 0.1 × use_confidence
```

Where:
- `semantic_similarity ∈ [0, 1]` — cosine similarity between query embedding and skill embedding
- `reliability_score ∈ [0, 1]` — Bayesian reliability: `successes / (successes + failures + 1)`
- `use_confidence ∈ [0, 1]` — normalized use count: `min(1, use_count / 100)`

**Formula rationale and prior proposal resolution:**

An earlier proposal (RFC-SYNTHESIS §2.5, 2026-02-18) used a time-decay formula:
```
reliability = success_rate × 0.5^(daysSinceUse / 30), floored at 0.3
```

This RFC (added as a formal amendment, 2026-02-18) adopts the simpler **Bayesian estimator** for the following reasons:

1. **Cold-start stability**: The Bayesian `successes / (successes + failures + 1)` denominator prevents division-by-zero and produces a sensible prior (0.0) for a brand-new skill with no observations, without requiring a floor constant.
2. **Recency is handled elsewhere**: The `use_confidence` factor (`min(1, use_count / 100)`) already down-weights underused skills. Adding an exponential time decay on top creates double-penalization of infrequently-used but historically reliable skills.
3. **Fewer hyperparameters**: The time-decay formula requires tuning `daysSinceUse / 30` (half-life) and a floor constant `0.3`. The Bayesian formula has no free parameters.
4. **Interpretability**: `successes / (successes + failures + 1)` is directly interpretable as a smoothed success rate. The time-decay formula conflates reliability with recency in a non-transparent way.

The RFC-SYNTHESIS formula is **superseded** by the formula in this RFC. Implementations MUST use the Bayesian estimator defined here.

**Threshold semantics:**

| Context | Threshold | Rationale |
|---------|-----------|-----------|
| General skill interception | `0.4` | Permissive to encourage evolution |
| Core command protection | `0.85` | Strict to prevent reflex hijacking |
| Recursion guard | `depth ≥ 3 → block` | Prevents infinite recursion |

A candidate with `interception_score < threshold` is **not intercepted** and falls through to the standard execution path.

### 5. StochasticSelector — Shannon Entropy Normalization (Amendment 2026-02-18)

The `StochasticSelector` in `lib/kernel/stochastic-selector.ts` uses **softmax sampling** with a temperature derived from the `EntropyMeter`:

```
P(skill_i) = exp(score_i / T) / Σ_j exp(score_j / T)
```

Where temperature `T` is set by the `EntropyMeter`:

```
T = max(0.1, 1.0 - diversity_entropy / target_entropy)
```

**Shannon entropy** of the selection distribution is computed as:

```
H = -Σ_i P(skill_i) × log2(P(skill_i))
```

**Adaptive entropy target:**
- If `recent_success_rate > 0.8` and `H < target_entropy`: increase `target_entropy` by 0.05 (explore more)
- If `recent_success_rate < 0.4` and `H > target_entropy`: decrease `target_entropy` by 0.05 (exploit more)
- `target_entropy` is clamped to `[0.3, 2.5]`

Core commands (e.g., `boot`, `status`, `help`) bypass the stochastic selector and are executed deterministically regardless of entropy state.

## Rationale

By placing the boot logic in a protocol-native file (`BOOTSTRAP.yamo`) and self-healing the platform's primary instruction file (`AGENTS.md`), we create a "Closed Cognitive Loop" that is resistant to drift and external tampering.

## Backwards Compatibility

- **Transparent Integration**: Works alongside existing OpenClaw and YAMO-OS deployments.
- **Automatic Upgrade**: The intelligent installer handles the transition from narrative to kernel-based boot.

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-17 | Initial draft — Autonomous Kernel, Ghost Protection, Hardened Kernel Architecture |
| 0.1.1 | 2026-02-18 | Add InterceptionEngine scoring formula (§4) and StochasticSelector Shannon entropy normalization (§5) as amendments |
| 0.1.2 | 2026-02-21 | Add RFC cross-references; clarify dual bootstrap variants (BOOTSTRAP.yamo vs BOOTSTRAP.md per RFC-0009); add formula rationale note to §4 resolving conflict with RFC-SYNTHESIS §2.5 time-decay proposal — Bayesian estimator is authoritative |
| 0.2.0 | 2026-02-21 | §2 expanded into three normative subsections closing the Fidelity Gap identified in the 2026-02-21 Scribe Prototype Audit: §2.1 Ghost Invariant (syntactic load gate with self-healing enforcement); §2.2 Semicolon Protocol (operational blocks MUST use YAMO-native syntax, not Markdown lists); §2.3 Reflexive Heartbeat (normative PRIORITY_0/1/2 session-init ordering — GhostGuard fires before persona loading unconditionally) |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
