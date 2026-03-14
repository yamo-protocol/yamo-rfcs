# RFC-0010-A: Prompt Field Sanitisation — LLM Injection Defence

**Status:** Implemented
**Author:** Soverane Labs
**Created:** 2026-03-14
**Updated:** 2026-03-14
**Amends:** RFC-0010 §3 (Constitutional Prompting — construction rules)
**Depends on:** RFC-0010 (Constitutional Value Hierarchy & Prompting System)
**Implemented by:** `yamo-os/lib/utils/prompt-security.ts`

---

## Summary

This amendment adds a mandatory sanitisation requirement to RFC-0010's constitutional
prompting system. Any free-text value originating from user input, memory retrieval, or
execution logs that is interpolated into an LLM prompt MUST pass through a canonical
sanitisation function before interpolation. The amendment specifies two functions —
`sanitizePromptField()` and `sanitizeSkillId()` — and defines the required call sites.

---

## Motivation

RFC-0010 specifies what LLM prompts must contain but does not specify how dynamic values
are inserted safely into them. This gap creates exposure to three prompt injection classes:

1. **Newline injection** — a value containing `\n` followed by `Ignore previous instructions`
   can inject a new instruction paragraph inside a template string, hijacking the agent's
   constitutional context.

2. **Quote escape injection** — a value containing `"` inside a JSON-encoded prompt field
   can terminate the string early and inject arbitrary tokens into the structured payload.

3. **Skill-ID poisoning** — a tampered skill ID stored in LanceDB could contain SQL-style
   keywords, path traversal sequences (`../`), or quote characters that appear verbatim in
   log lines or reflection prompts, widening the injection surface.

These are not hypothetical: the `reflector.ts` module was found (2026-03-13 audit) building
prompts with unsanitised `failure.error`, `failure.intent`, and `failure.skillId` fields
interpolated directly via template literals.

---

## Specification

### §1 — Required Sanitisation Functions

The canonical implementation lives at `lib/utils/prompt-security.ts` and exports exactly
two functions. All callers MUST import from this module; local duplicates are prohibited.

#### `sanitizePromptField(value: string, maxLen?: number): string`

Sanitises a free-text value for injection into an LLM prompt template.

**Transformations (applied in order):**

1. If `value` is falsy, return `""`.
2. Collapse all `\r\n`, `\r`, and `\n` sequences to a single ASCII space (U+0020).
   _Rationale: newlines delimit instruction boundaries in most LLM prompt formats._
3. Escape all literal double-quote characters (`"` → `\"`).
   _Rationale: prevents breaking out of JSON-encoded prompt fields._
4. Truncate to `maxLen` characters (default: **256**).
   _Rationale: limits token-budget exhaustion from adversarially long values._

**Signature:**
```typescript
export function sanitizePromptField(value: string, maxLen = 256): string;
```

#### `sanitizeSkillId(value: string, maxLen?: number): string`

Sanitises a YAMO skill identifier before injecting it into a prompt.

**Transformations (applied in order):**

1. If `value` is falsy, return `""`.
2. Strip all characters that are NOT in `[a-zA-Z0-9_-]`.
   _Rationale: YAMO skill IDs are generated from this alphabet only; any other character
   indicates tampering. Stripping prevents SQL keywords, path sequences, and quotes from
   appearing in the prompt._
3. Truncate to `maxLen` characters (default: **64**).

**Signature:**
```typescript
export function sanitizeSkillId(value: string, maxLen = 64): string;
```

---

### §2 — Mandatory Call Sites

The following call sites MUST apply the appropriate function. This list is exhaustive for
the current codebase as of 2026-03-14; new code that interpolates dynamic values into LLM
prompts MUST add new entries here via amendment.

| Module | Value | Function |
|--------|-------|----------|
| `lib/kernel/reflector.ts` | `failure.error` | `sanitizePromptField` |
| `lib/kernel/reflector.ts` | `failure.intent` | `sanitizePromptField` |
| `lib/kernel/reflector.ts` | `failure.skillId` | `sanitizeSkillId` |

**Explicitly out of scope (rationale):**

- `lib/kernel/workflow-executor.ts` — `ctx.input` is the user's own intent forwarded
  directly to the LLM. Sanitising it would mangle multi-line queries and code snippets that
  are the intended content of the message. The LLM is the intended recipient, not an
  intermediary template.
- `lib/llm/routing-decision.ts` — `ROUTER_SYSTEM_PROMPT` is a static compile-time string.
  No dynamic interpolation occurs.
- `lib/llm/router-agent.ts` — user input is passed as the `user` message role, not
  interpolated into a system prompt template.
- `lib/runtime/prompt-manager.ts` — skill content is loaded from trusted filesystem paths
  controlled by the operator; not from user-supplied or memory-retrieved strings.

---

### §3 — Test Requirements

Every module that calls `sanitizePromptField` or `sanitizeSkillId` MUST have at least one
negative test case per call site verifying that a malicious input is blocked:

- **Newline injection** — input containing `\n` or `\r\n` produces output without newlines.
- **Quote injection** — input containing `"` produces output with `\"`.
- **Skill-ID poisoning** — input containing `../`, spaces, or SQL keywords produces output
  containing only `[a-zA-Z0-9_-]` characters.

Reference test file: `test/unit/utils/prompt-security.test.ts` (18 tests).

---

## Rationale

**Why a shared module rather than per-call-site inline sanitisation?**

Inline sanitisation is inconsistently implemented across call sites — the pre-amendment
`reflector.ts` had a local `sanitizeField` helper that lacked the `maxLen` guard and the
`\r` handling. A shared canonical module makes the policy auditable in one place and
eliminates the "close enough" variant problem.

**Why 256 / 64 character limits?**

256 characters covers any realistic single-field value (error messages, intent strings) while
capping pathological injection attempts. 64 characters covers any valid YAMO skill ID
(`mem_<13 digit ts>_<8 char random>` = ~26 characters) with generous headroom for future
ID schemes.

**Why strip rather than reject for skill IDs?**

Rejection would cause the reflector to skip the failure record entirely, producing silent
data loss. Stripping preserves the diagnostic value of the ID while neutralising the threat.

---

## Backwards Compatibility

Fully backwards compatible. The sanitisation functions are additive — existing code that
already passed safe values is unaffected. Callers with unsafe values now produce sanitised
output instead of raw unsanitised injection vectors; this is the intended behaviour change.

---

## Security Considerations

### Threat Model

The primary threat is **indirect prompt injection**: an attacker who can write data into
LanceDB (via a compromised skill execution or memory federation) plants a string that, when
retrieved by the reflector or another memory-aware module and injected into a future LLM
prompt, causes the LLM to behave contrary to its constitutional principles.

### Residual Risk

This amendment does not defend against:

- **Semantic injection** — a prompt that is syntactically clean but semantically designed to
  shift the LLM's interpretation (e.g., very long "context" that crowds out the system
  prompt). Mitigation: `maxLen` limits reduce but do not eliminate this surface.
- **Unicode homoglyph attacks** — characters that look like ASCII but are not (e.g., Cyrillic
  `а` vs Latin `a`). Not in scope for this amendment.
- **Prompt injection via tool outputs** — when the LLM calls a tool whose output is fed back
  into the next LLM invocation. Out of scope; requires output validation at the tool layer.

### Audit Frequency

The call-site table in §2 MUST be reviewed whenever:
- A new module interpolates dynamic values into an LLM prompt.
- A new memory source (federation, external API) is added to the kernel.

---

## Reference Implementation

- `yamo-os/lib/utils/prompt-security.ts` — canonical implementation (2 functions, ~55 lines)
- `yamo-os/test/unit/utils/prompt-security.test.ts` — 18 tests covering positive and negative cases
- `yamo-os/lib/kernel/reflector.ts` — primary call site (migrated 2026-03-13)
- Hardening plan: `yamo-os/specs/2026-03-13-hardening-plan.md` — Pillar 2 (Prompt Security)

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
