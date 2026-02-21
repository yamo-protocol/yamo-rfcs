# RFC-0007: Semantic Heritage & Wisdom Distillation

**Status:** Draft
**Author:** Soverane Labs & Collaborative Swarm
**Created:** 2026-02-17
**Updated:** 2026-02-21
**Related:** RFC-0006 (Autonomous Kernel — heartbeat trigger for distillation cycle), RFC-0010 (Constitutional Value Hierarchy — FINAL phase reasoning produces LessonLearned candidates), RFC-0011 (MemoryMesh — implementation of wisdom distillation, canonical LessonLearned wire format), RFC-0012 (S-MORA — enhanced lesson retrieval via HyDE-Lite + heritage-aware reranking)

## Summary

This RFC formalizes the **Semantic Heritage** protocol and the **Wisdom Distillation** workflow. It defines how an agent preserves the "Why" behind its actions across long-running sessions and how it autonomously converts transient logs into persistent, actionable "Lessons Learned" within the MemoryMesh.

## Motivation

Most AI agents suffer from "Episodic Amnesia"—they lose the rationale for a decision once it leaves the active context window. Furthermore, raw execution logs are often too noisy to be useful for future reasoning.

To achieve senior-level reliability, an agent needs:
1. **Heritage Tracking**: A way to inherit intent from high-level specifications down to granular code implementations.
2. **Wisdom Distillation**: A mechanism to extract high-signal lessons from low-signal logs.
3. **Proactive Reflection**: A "Subconscious" process that consults past failures before starting new tasks.

## Specification

### 1. Semantic Heritage Protocol

Every YAMO v3.0 interaction must maintain a `heritage` chain in its context:

```yamo
semantic_heritage:
  intent_chain: [macro_spec, implementation_plan, tdd_cycle];
  hypotheses: [list_of_validated_hypotheses];
  rationales: [unbroken_chain_of_justifications];
```

This ensures that a `TDDAgent` knows exactly *why* a specific test was requested by the `SpecificationArchitect`.

### 2. Wisdom Distillation Workflow

The `MemoryKernel` module must autonomously execute a distillation cycle (Heartbeat triggered):

1. **Scan**: Analyze recent `yamo_blocks` for `#lesson_learned` tags or failure patterns.
2. **Distill**: Synthesize raw logs into a structured `LessonLearned` block.
3. **Ingest**: Store the distilled wisdom in the MemoryMesh with appropriate semantic embeddings.

### 3. The LessonLearned Block

> **Authoritative wire format**: The conceptual structure below is a summary. The **canonical, fully-specified LessonLearned YAMO block wire format** is defined in **RFC-0011 §3.5**. Implementations MUST use the RFC-0011 §3.5 format for serialization, storage, and inter-agent exchange.

**Conceptual field mapping** (use RFC-0011 §3.5 for the normative definition):

```yamo
concept: LessonLearned;
required_fields:
  context;situation_description;
  oversight;what_was_missed;
  fix;how_it_was_resolved;
  constraint;preventative_rule_for_future;
  tags;["#lesson_learned"];
```

> **Terminology note**: The field container was originally named `structure:`. RFC-0011 uses `output:` and `context:` per standard YAMO block grammar. `required_fields:` is used above for conceptual clarity; see RFC-0011 §3.5 for the normative field layout.

### 4. Subconscious Reflection

Before any high-priority `handoff;`, the kernel should invoke the `SubconsciousReflector`:
- **Query**: Search MemoryMesh for similar past oversights.
- **Inject**: Automatically append preventative constraints to the current agent's context.

## Rationale

Semantic Heritage prevents "Intent Decay" in long-running projects. Wisdom Distillation ensures the agent's MemoryMesh remains high-density and actionable, rather than becoming a graveyard of raw logs.

## Security Considerations

- **Privacy**: The distillation process must invoke the "Layer 0 Scrubber" to ensure PII and secrets are never persisted to long-term wisdom stores.

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-17 | Initial draft — Semantic Heritage Protocol, Wisdom Distillation Workflow, LessonLearned Block, SubconsciousReflector |
| 0.1.1 | 2026-02-21 | Add RFC cross-references; update LessonLearned §3 to point to RFC-0011 §3.5 as canonical wire format; add `required_fields:` terminology note |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
