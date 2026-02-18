# RFC-0006: Autonomous Kernel & Ghost Protection

**Status:** Draft
**Author:** Soverane Labs & Collaborative Swarm
**Created:** 2026-02-17

## Summary

This RFC specifies the architecture for the **Autonomous YAMO Kernel** and the **Ghost Protection** mechanism. It defines how an agent ensures its own cognitive integrity by autonomously managing its bootstrap process and self-healing its operational boundaries in the `AGENTS.md` and `BOOTSTRAP.yamo` substrate.

## Motivation

AI agents in open workspaces are susceptible to "Ghosting"—a state of narrative drift where the agent ignores its core instructions in favor of more recent conversational tokens. Additionally, manual setup of complex agent kernels is error-prone. 

To achieve true autonomy, an agent must:
1. **Self-Initialize**: Control its own boot sequence via a protocol-native entry point.
2. **Self-Protect**: Monitor and repair its cognitive links if they are modified or overwritten.
3. **Enforce Priority**: Ensure the constitutional kernel always has precedence over narrative context.

## Specification

### 1. The Bootstrap Protocol (`BOOTSTRAP.yamo`)

Every YAMO-Native workspace must contain a `BOOTSTRAP.yamo` at the root. This file serves as the "Primary Boot Sector" for the agent.

```yamo
agent: BootRedirector;
intent: redirect_to_v3_kernel;
context:
  orchestrator_path;yamo-native-agent/agent-orchestrator.yamo;
handoff: yamo-native-agent/agent-orchestrator.yamo;
```

### 2. Ghost Protection (Self-Healing AGENTS.md)

The Kernel must perform an integrity check on the workspace `AGENTS.md` during every boot sequence (Priority 0).

*   **Trigger**: Detection of missing "YAMO-NATIVE KERNEL ACTIVE" header.
*   **Action**: The agent must autonomously prepend the kernel priority pointer to the top of the file.
*   **Enforcement**: Prevents the platform from loading legacy narrative instructions before the protocol-native kernel.

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

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
