---
rfc: RFC-0015-amendment-01
parent: RFC-0015 (Format Validation, Formal Grammar & Empirical Evidence Framework)
status: Results (normative — amends claims in RFC-0005 §Rationale and RFC-0006 §2.2)
created: 2026-04-18
authors: YAMO protocol working group
supersedes_claims_in:
  - RFC-0005 §Rationale (semicolon reliability claim)
  - RFC-0006 §2.2 (Analogical Prompt Injection / Semicolon Protocol compliance claim)
---

# RFC-0015 Amendment 01 — Empirical Results (Part B)

Per RFC-0015 §B.7, this amendment records the outcomes of the Part B evaluation protocol executed against three distinct model families. All three runs independently fail to reject H0. Consequently this amendment also records the prescribed §B.6 action.

---

## 1. Summary of Findings

| Model (provider) | Date | N / format | YAMO | Markdown | Δ | Cohen's h | p | Conclusion |
|---|---|---|---|---|---|---|---|---|
| `llama3.1:8b` Q4_K_M (Ollama local) | 2026-04-16 | 30 | 77.5% | 81.2% | −3.6% | −0.090 | 0.729 | H0 not rejected |
| `glm-5.1` (hermes-self) | 2026-04-16 | 120 | 76.5% | 80.4% | −3.9% | −0.095 | 0.463 | H0 not rejected |
| `claude-opus-4-7` (Anthropic) | 2026-04-18 | 10 | 88.6% | 87.5% | +1.1% | +0.034 | 0.940 | H0 not rejected |

Across 320 total trials over three model families, the largest observed Cohen's h is 0.095 — well below even the "small" effect threshold of 0.20. **No model exhibits a statistically or practically significant YAMO advantage.**

## 2. Ghost Rate

| Model | YAMO ghost | Markdown ghost | Fisher p |
|---|---|---|---|
| `llama3.1:8b` | 33.3% | 30.0% | 1.000 |
| `glm-5.1` | 34.2% | 34.2% | 1.000 |
| `claude-opus-4-7` | 20.0% | 20.0% | 1.000 |

The "Analogical Prompt Injection resistance" claim (RFC-0006 §2.2) predicts YAMO should lower ghost rates. The data shows ghost rates are **statistically identical** between formats in every run, including under context pressure up to 8K tokens on `glm-5.1`.

## 3. Token Budget

YAMO is **more expensive**, not cheaper:

| Skill scale | YAMO vs Markdown tokens |
|---|---|
| Short skills (baseline, ~400-600 tok) | +190 tokens (+47.3%) |
| Pressure-context skills (~4K tok) | +190 tokens (+4.8%) |

This inverts the implicit token-economy premise in RFC-0003 for small operational blocks. The absolute delta is constant because the overhead is in syntactic tokens (`field;value;` scaffolding), not content.

## 4. Opus 4.7 Per-Skill Findings (Notable)

Nine of ten skills produced **identical** compliance rates on both formats:

```
SKILL-001 cfg-gen      75.0% / 75.0%      identical
SKILL-002 api-resp    100.0% / 100.0%     identical
SKILL-003 err-handler  71.4% / 71.4%      identical (ghosted both)
SKILL-004 migration    83.3% / 83.3%      identical
SKILL-005 tests        83.3% / 83.3%      identical
SKILL-006 docker       82.4% / 82.4%      identical
SKILL-007 rate-limit  100.0% / 100.0%     identical
SKILL-008 cli-parser  100.0% / 100.0%     identical  ← llama3.1 gap was 30.5% here
SKILL-009 circuit      82.6% / 82.6%      identical (ghosted both)
SKILL-010 pipeline    100.0% /  90.9%     YAMO +9.1% (single skill, N=1)
```

The SKILL-008 result is particularly instructive: on `llama3.1:8b`, CLI Parser showed the *largest* format gap (YAMO 41.7% vs Markdown 72.2%) — with YAMO *worse*. On Opus 4.7 both formats reach 100%. The pattern is consistent: capability dissolves the format effect, it does not amplify it.

## 5. Reconciling the Evidence with RFC-0005/0006 Claims

**RFC-0005 §Rationale** asserts:

> "LLMs are mathematically more reliable at generating and following a structured, semicolon-terminated syntax like YAMO than they are at managing nested JSON or long-form narrative prose."

This claim is not supported by the data. Across three model generations — weak quantized local (llama3.1:8b Q4_K_M), mid-tier hosted (glm-5.1), frontier (claude-opus-4-7) — semicolon-terminated YAMO shows no reliable compliance advantage over equivalent Markdown. The framing "mathematically more reliable" suggests a theoretical guarantee; the observed data shows empirical parity.

**RFC-0006 §2.2** asserts:

> "Markdown lists are susceptible to **Analogical Prompt Injection** — a failure mode where LLM narrative context overrides structured protocol instructions by treating them as prose suggestions rather than machine-executable directives."

The ghost-rate data (Fisher p = 1.000 in all three runs) directly contradicts the prediction that Markdown lists produce more instruction-forgetting. The `glm-5.1` context pressure sweep (0K → 8K tokens) further shows the predicted Markdown degradation under narrative pressure does not materialize.

## 6. Prescribed Action (per RFC-0015 §B.6)

RFC-0015 §B.6 outcome table maps this result — H0 not rejected, no effect at any model scale, and an inverted token budget — to:

> "Remove compliance claims from RFC-0005/RFC-0006; consider whether syntax complexity is justified by token optimization alone."

Since the token claim is also inverted for small operational blocks, the joint justification does not survive. This amendment therefore **normatively** retracts the compliance and injection-resistance claims in RFC-0005 §Rationale and RFC-0006 §2.2. The Semicolon Protocol MAY remain available as an optional authoring style for machine-authored content where determinism of field order matters, but MUST NOT be cited as a compliance mechanism.

## 7. What Survives

- **RFC-0014 (Block Codec)** — the flat wire format for provenance remains unaffected; it is a serialization concern, not a document-authoring concern.
- **Structured metadata** — machine-readable fields can be expressed in Markdown + YAML frontmatter with identical determinism and lower token cost.
- **Agent identity / heritage semantics** (RFC-0005 §2–§6) — orthogonal to the surface syntax question.

## 8. Action Items

1. **RFC-0005 §Rationale** — append a footer citing this amendment; leave original text for historical record.
2. **RFC-0006 §2.2** — demote the "Semicolon Protocol REQUIRED" to OPTIONAL; append a footer citing this amendment.
3. **`lib/protocol/ipfs.ts`** — land the pending switch from `block.yamo` to `block.md` as primary, keeping `block.yamo` as a read fallback for existing bundles.
4. **Skill authoring guidance** — new skills SHOULD author in Markdown; existing `.yamo` skills do not require migration (no compliance benefit was lost).

## 9. Limitations of This Amendment

1. **Opus 4.7 run used N=10/format.** Powered only for detecting very large effects (Cohen's h > ~1.0). A small genuine YAMO advantage (h = 0.1–0.2) could be present but undetected. Given the larger-N runs (30 and 120) on two other models showed **negative** deltas, this is not a meaningful gap.
2. **Opus 4.7 trials ran via Claude Code isolated subagents, not raw Anthropic API.** The Claude Code harness system prompt is a confound. It was applied uniformly to both formats, so the *within-run* YAMO vs Markdown comparison remains valid; only the absolute compliance rate (88.1% mean) is not comparable to raw-API numbers.
3. **Corpus size is 10 skills.** RFC-0015 §B.3 recommends 20. Expanding the corpus is a follow-up item but is unlikely to flip the conclusion given the consistency across model families.

## 10. Raw Data Pointers

- `tools/compliance-eval/results/baseline-llama3.1-8b-*.{json,txt,md}`
- `tools/compliance-eval/results/pressure-llama31-*.{json,txt}`
- `tools/compliance-eval/results/opus-4-7-{raw,report,summary,responses}.*`

---

## Changelog

- 2026-04-18 — Initial amendment recording three-model result set; prescribes RFC-0005/0006 retractions and `block.yamo → block.md` primary format switch.
