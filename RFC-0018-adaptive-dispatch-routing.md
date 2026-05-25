# RFC-0018: Adaptive Dispatch Routing (Outcome-Learned Route Priors)

**Status:** Draft
**Author:** Soverane Labs
**Created:** 2026-05-25
**Updated:** 2026-05-25
**Depends on:** RFC-0016 (multi-model routing — the `RouterAgent`/`RoutingDecision` contract this augments), RFC-0017 (dispatch-bead lifecycle — `memory_dispatches` is the feedback data source)
**Companion:** memory-mesh Decision Context Graph (`decision_edges`, `recordOutcome`) — optional richer signals (§ Specification 5)
**Implemented by:** _(proposed)_ `yamo-os/lib/brain/kernel-brain.ts` (`routeStats()`), `yamo-os/lib/kernel/route-prior.ts` (new), `yamo-os/lib/llm/router-agent.ts`, `yamo-os/lib/kernel/kernel.ts` (close site)

---

## Summary

Make model-target routing **learn from its own history**. Today `RouterAgent`
picks a `target_agent` (`fast`/`reasoner`/`tool`/`reasoner+tool`) from static
rules and an LLM classifier; the outcome of that choice is recorded in
`memory_dispatches` (RFC-0017) but never read back. This RFC closes the loop: an
outcome-learned **route prior** — per `(intent, target)` success rate and latency,
derived from data already on disk — biases the routing decision, gated by
evidence and bounded by the existing deterministic `ruleBasedRoute()` floor.

## Motivation

- **The feedback data already exists, unused.** Every `execute()` writes a
  `memory_dispatches` row with `route_intent`, `route_target`, terminal `status`
  (`done`/`error`/`aborted`), and `latencyMs`. That is exactly
  `(intent → target → outcome)`. Nothing consumes it for routing.
- **Routing is static.** `routing-decision.ts` is rule/LLM-based; `RoutingMetrics`
  is explicitly *in-memory aggregate telemetry* — ephemeral, never persisted, never
  fed back. A target that consistently fails for an intent keeps getting chosen.
- **The pattern is already proven elsewhere in the kernel.** `InterceptionEngine`
  selects *skills* with a `60% semantic / 30% reliability / 10% use-confidence`
  score, and `EntropyMeter`/`StochasticSelector` already balance
  exploration/exploitation. This RFC brings that proven, outcome-weighted
  selection model to the *target-routing* decision.

## Specification

### 1. `KernelBrain.routeStats()` — the learned aggregate

A read-side aggregate over the existing `memory_dispatches` table (a scalar
sidecar, so a grouped scan is cheap):

```ts
interface RouteStat { successRate: number; p50LatencyMs: number; n: number; }
// success := status === 'done'
async routeStats(): Promise<Record<Intent, Record<RoutingTarget, RouteStat>>>;
```

No new capture is introduced — `routeStats()` reads what RFC-0017 already writes.

### 2. `RoutePrior` — hot in-memory model (`yamo-os/lib/kernel/route-prior.ts`)

EWMA success rate + p50 latency + sample count per `(intent, target)`:

- **Seed:** on kernel boot, `observe()` is primed from `routeStats()` — warm, not cold.
- **Update:** incremental `observe(intent, target, success, latencyMs)`.
- **Query:** `best(intent, { minSamples, minDelta })` → the materially-better target, or `null`.

```ts
class RoutePrior {
  observe(intent: Intent, target: RoutingTarget, success: boolean, latencyMs: number): void;
  best(intent: Intent, opts: { minSamples: number; minDelta: number }): RoutingTarget | null;
}
```

### 3. Close the loop at the dispatch terminal

`KernelBrain.closeDispatch(id, { status, latencyMs })` already fires on every
terminal state (`kernel.ts` close site). Fold the result into the prior,
fire-and-forget (never block the request):

```ts
this._routePrior.observe(intentOf(id), targetOf(id), status === "done", latencyMs);
```

### 4. `RouterAgent.route()` consults the prior — precedence stack

`RouterAgent` gains an injectable `routePrior` (mirroring the existing injectable
`llmFactory`). Decision precedence — **floor → LLM → evidence-gated nudge**:

```
(a) Deterministic floor:  a confident ruleBasedRoute() result is returned as-is. (UNCHANGED)
(b) Cold/thin prior:      n < MIN_SAMPLES for (intent, chosen target) → return the
                          LLM/rule decision unchanged.   (behaviour == today)
(c) Evidence gate:        routePrior.best(intent, { minSamples, minDelta })
                          overrides only on a well-sampled, material success delta.
(d) ε-exploration:        epsilon-pick between the prior's best and the original
                          target, reusing the StochasticSelector softmax, so the
                          estimate stays honest.  decision._source = "prior".
```

Tunables (defaults): `MIN_SAMPLES = 20`, `minDelta = 0.15`, `epsilon` sourced
from `EntropyMeter` (exploration rises when success is high and diversity low).

### 5. Optional richer signals (memory-mesh DCG)

- **Skill outcomes:** when a dispatch selected a synthesized skill, call
  memory-mesh `recordOutcome(skillMemoryId, { status })` on close → feeds
  `importance_score`, which `InterceptionEngine`'s 30% reliability term already
  reads. Closes the *skill* loop with the same mechanism.
- **Route lineage:** record routing decisions as `decision_edges` (`depends-on`
  the routed-from context; `supersedes` a prior failed route) so
  `decisionLineage()` can answer "why did routing change for this intent?"

### 6. Rollout via existing `routing_mode`

1. **shadow:** log where the prior *would* diverge from rules/LLM; accumulate
   `memory_dispatches`; change nothing.
2. **validate:** confirm via `routeStats()` that the prior beats the status quo.
3. **active:** enable the nudge in live dispatch.

## Rationale

- **Why a prior over retraining the LLM router?** The data is structured
  `(intent, target, outcome)` — a cheap aggregate beats fine-tuning, stays
  interpretable, and degrades gracefully (cold prior == today's behaviour).
- **Why keep `ruleBasedRoute()` as a hard floor?** Safety and determinism:
  confident rules (e.g. `chit_chat → fast`, shell patterns → `tool`) must never
  be overridden by a statistical estimate. The learner only touches the
  *ambiguous* (LLM-decided) cases.
- **Why ε-exploration?** Pure exploitation freezes a possibly-wrong early
  estimate. The kernel already has the softmax machinery; reuse it.
- **Why the prior in yamo-os, not the memory-mesh package?** `memory_dispatches`
  is a yamo-os brain table (RFC-0017). The generic memory-mesh package owns the
  *reusable* primitives (`recordOutcome`, `decision_edges`); the dispatch-specific
  aggregate belongs with the dispatch table. Clean system-of-record (transactional
  dispatch state) vs system-of-intelligence (learned prior) split.

## Backwards Compatibility

**Fully backwards compatible.** No schema changes (`routeStats()` reads existing
`memory_dispatches`). With `routing_mode` unchanged or the prior cold, behaviour
is byte-identical to today. `RoutingMetrics` is retained unchanged as the
observability view; the prior is a separate decision input. Default-off until
explicitly flipped to `active` after shadow validation.

## Security Considerations

- **Poisoning:** an attacker who can drive many failing dispatches for an intent
  could depress a target's success rate. Mitigations: the `ruleBasedRoute()` floor
  is immune; `minSamples`/`minDelta` gates resist small-sample skew; per-target
  routing only changes *which local/zai model* runs, not *whether* a request runs.
- **Prompt fields:** the prior keys on `(intent, target)` enums only — no free text
  — so it introduces no new injection surface. Existing `sanitizePromptField` on
  `memory_dispatches` is unchanged.
- **Resource bounds:** `routeStats()` must use `clampedLimit` semantics / a bounded
  group scan; the EWMA prior is O(intents × targets) in memory.

## Reference Implementation

Proposed, tracked in beads (see epic). Foundation piece is
`KernelBrain.routeStats()`; integration is `RoutePrior` + `RouterAgent`.
No PRs yet.

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
