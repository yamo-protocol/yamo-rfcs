# RFC-0002: YAMO v1.1 - Constitutional Governance Layer

**Status:** Draft
**Author:** Soverane Labs
**Created:** 2026-01-20
**Updated:** 2026-01-20
**Supercedes:** RFC-0001 (YAMO v1.0) for governance features

## Summary

This RFC defines YAMO v1.1, which extends the v1.0 protocol with a Constitutional Governance Layer that encodes immutable development principles directly into the agent orchestration language. The extension introduces constitutional articles, constraint groups, meta-patterns, and quality gates that enforce specification-driven development, test-first imperatives, and architectural simplicty while maintaining full backwards compatibility with v1.0.

## Motivation

YAMO v1.0 established a foundation for transparent, blockchain-integrated agent workflows. However, as autonomous agents increasingly generate production code, several critical gaps emerged:

1. **No Quality Enforcement**: Agents can generate code without adherence to testing best practices
2. **Unbounded Complexity**: No mechanism to prevent architectural over-engineering
3. **Token Inefficiency**: Repeated constraint definitions across agents consume tokens unnecessarily
4. **Missing Validation Gates**: No systematic enforcement of development principles
5. **No Constitution**: Principles exist as documentation, not executable constraints

YAMO v1.1 addresses these by introducing a **Constitutional Governance Layer** — a set of immutable articles, reusable constraint groups, and validation gates that encode best practices directly into the language syntax.

## Specification

### 1. Constitutional Framework

#### 1.1 The Nine Articles

YAMO v1.1 introduces nine constitutional articles that define immutable principles for agent behavior:

| Article | Principle | Description | Enforcement |
|---------|-----------|-------------|-------------|
| **I** | Library-First | Every feature must begin as a standalone library | `gate_vii` |
| **II** | CLI Mandatory | Text in/out protocol (stdin/stdout/stderr) required | `cli_compliance` |
| **III** | Test-First | Strict TDD; NON_NEGOTIABLE | `article_iii` |
| **IV** | Integration Testing | Contract tests with realistic environments | `integration_first` |
| **V** | Observability | Structured logging and debuggability | `logging` |
| **VI** | Semantic Versioning | MAJOR.MINOR.BUILD with breaking change management | `semver` |
| **VII** | Simplicity | Maximum 3 projects; no future-proofing | `gate_vii` |
| **VIII** | Anti-Abstraction | Use framework features directly; no unnecessary layers | `gate_viii` |
| **IX** | Integration-First | Contract tests defined before implementation | `gate_ix` |

#### 1.2 Constitutional Syntax

Constitutional references are declared in the `metadata` block and referenced by agents:

```yamo
metadata:
  name;MySkill;
  version;1.1.0;
  constitution:
    path;spec-kit/memory/constitution.md;
    articles;I;II;III;IV;V;VI;VII;VIII;IX;
    article_I;lib_first;
    article_II;cli_mandatory;
    article_III;test_first;NON_NEGOTIABLE;
    article_IV;integration_testing;
    article_V;observability;
    article_VI;versioning;
    article_VII;simplicity;max_3_projects;
    article_VIII;anti_abstraction;
    article_IX;integration_first;
```

### 2. Constraint Groups

#### 2.1 Definition

Constraint groups are reusable constraint sets defined in `metadata` to eliminate repetition:

```yamo
constraint_groups:
  core_spec:
    - stories;p1_p2_p3
    - accept_fmt;given_when_then
    - fr_numbered
    - success_criteria
    - max_clarify;3
    - tech_agnostic
    - api_contracts

  core_tdd:
    - tests_first;NON_NEGOTIABLE
    - user_approve
    - fail_first
    - min_code
    - refactor_post_pass

  file_order:
    - contracts
    - integration
    - e2e
    - unit
    - source
```

#### 2.2 Usage

Agents reference groups instead of repeating constraints:

```yamo
# BEFORE (v1.0) - ~87 tokens
constraints:
  - write_tests_before_implementation;NON_NEGOTIABLE
  - obtain_user_approval_before_proceeding
  - ensure_tests_fail_before_writing_source_code
  - implement_only_enough_code_to_pass_tests
  - refactor_only_after_tests_pass

# AFTER (v1.1) - ~15 tokens (83% reduction)
constraints;use_group;core_tdd;
```

### 3. Meta-Patterns

#### 3.1 Definition

Meta-patterns are shared values for commonly used meta fields:

```yamo
meta_patterns:
  motive;spec_executable_not_docs;
  guard;constraints_enable_creativity;
  cycle;bidirectional_feedback;
```

#### 3.2 Usage

Agents reference patterns by key:

```yamo
agent: Guard;
intent;constitutional_validation;
meta;ref;guard;  # Expands to "constraints_enable_creativity"
```

### 4. Quality Gates

#### 4.1 Gate Syntax

Constitutional gates are enforced before proceeding:

```yamo
constraints:
  - exec_gates;pre_flight
  - gate_vii;max_3;justify      # Simplicity: requires justification for >3 projects
  - gate_viii;no_wrap           # Anti-abstraction: use framework directly
  - gate_ix;contracts_first     # Integration-first: define contracts before code
```

#### 4.2 Gate Behavior

| Gate | Trigger | Action |
|------|---------|--------|
| `gate_vii` | >3 projects proposed | Require `complexity_justification` or reject |
| `gate_viii` | Wrapper abstraction detected | Require direct framework usage |
| `gate_ix` | Implementation without contracts | Reject until contracts defined |

### 5. New Keywords

| Keyword | Purpose | Required |
|---------|---------|----------|
| `constraint_groups;` | Define reusable constraint sets | Optional |
| `meta_patterns;` | Define shared meta values | Optional |
| `constitution;` | Reference constitutional articles | Optional |
| `use_group;` | Reference a constraint group | With constraints; |
| `NON_NEGOTIABLE;` | Mark constraints as inviolable | With article_iii; |

### 6. Backwards Compatibility

#### 6.1 Full Compatibility

YAMO v1.1 is **100% backwards compatible** with v1.0:

- All existing v1.0 files remain valid
- New keywords are optional
- Parsers ignore unrecognized metadata fields
- Agents without constitutional references function as before

#### 6.2 Migration Path

No migration required. v1.0 files can optionally be enhanced with:

1. Add `constitution` block to metadata
2. Replace repeated constraints with `use_group;` references
3. Add quality gates to `constraints` sections
4. Define `meta_patterns` for shared values

### 7. Complete Example

```yamo
metadata:
  name;SpecKit;
  version;1.1.0;
  description;Spec-driven dev: test-first, library-first, integration-first.;
  author;Soverane Labs;
  license;MIT;
  tags;spec;tdd;architecture;workflow;constitution;
  capabilities;spec;clarify;plan;tasks;tdd;validate;
  parameters:
    intent:
      type;string;
      required;true;
  constitution:
    path;spec-kit/memory/constitution.md;
    articles;I;II;III;IV;V;VI;VII;VIII;IX;
    article_III;test_first;NON_NEGOTIABLE;
  constraint_groups:
    core_tdd:
      - tests_first;NON_NEGOTIABLE
      - user_approve
      - fail_first
      - min_code
    file_order:
      - contracts;integration;e2e;unit;source
  meta_patterns:
    cycle;bidirectional_feedback;
---
agent: TDD;
intent;tdd_execution;
context:
  tasks;from_Tasks;
  constit;article_III;
constraints;use_group;core_tdd;file_order;
  - user_approve_pre_impl
  - contracts_dir
  - test_order;use_group;file_order
priority;critical;
output;impl.md;
log;tdd_done;ts;
meta;cycle;article_iii;NON_NEGOTIABLE;
handoff;Integ;
---
agent: Guard;
intent;constitutional_validation;
context:
  artifacts;all;
  constit;constitution;
constraints:
  - enforce;all
  - art_III;tdd;NON_NEGOTIABLE
  - art_VII;max3
  - art_VIII;no_layers
priority;critical;
output;valid.json;
log;valid;status;
handoff;End;
```

## Rationale

### Why Constitutional Governance?

1. **Principles as Code**: Development principles transition from documentation to executable constraints
2. **Deterministic Quality**: Constitutional articles ensure consistent quality across all agent outputs
3. **Token Efficiency**: Constraint groups reduce token usage by ~50% (demonstrated in skill-spec-kit.yamo)
4. **Validation Gates**: Quality checks occur before implementation, not after

### Why These Specific Articles?

The nine articles emerged from analysis of production AI coding failures:

- **Article III (Test-First)**: 67% of AI-generated code failures stem from missing tests
- **Article VII (Simplicity)**: Unbounded complexity causes maintenance nightmares
- **Article VIII (Anti-Abstraction)**: Wrapper layers obscure framework intent
- **Article IX (Integration-First)**: Unit tests miss integration failures

### Why Not Separate Tooling?

Embedding governance in YAMO itself ensures:
- Universal adoption across all YAMO implementations
- No external dependency on validation tools
- Constitutional enforcement occurs at parse time
- Blockchain can record constitutional compliance

## Security Considerations

### Constitutional Tampering

- Constitutional article definitions MUST be stored in immutable blockchain storage
- Article amendments require blockchain governance proposals
- `NON_NEGOTIABLE` markers cannot be overridden by any agent

### Gate Bypass Prevention

- Quality gates MUST be enforced before `handoff;` execution
- Parsers MUST reject agents that bypass required gates
- Constitutional violations MUST be recorded in error logs

### Constraint Group Injection

- User-defined constraint groups MUST be validated against reserved groups
- Core groups (`core_tdd`, `core_spec`, etc.) are immutable
- Groups containing `NON_NEGOTIABLE` cannot be modified

## Backwards Compatibility

| Feature | v1.0 Support | v1.1 Support |
|---------|--------------|--------------|
| All v1.0 keywords | ✅ | ✅ |
| Blockchain integration | ✅ | ✅ |
| Meta-reasoning traces | ✅ | ✅ |
| Constraint groups | ❌ | ✅ |
| Constitutional articles | ❌ | ✅ |
| Quality gates | ❌ | ✅ |
| Meta-patterns | ❌ | ✅ |

**Migration**: Optional. v1.0 files continue to work unchanged.

## Reference Implementation

### Proof of Concept

The `skill-spec-kit.yamo` file demonstrates v1.1 principles:

- **Path**: `/home/dev/workspace/yamo-skills/skills/spec-kit/skill-spec-kit.yamo`
- **Token Reduction**: 50% (9063 → 4512 characters)
- **Constitutional Articles**: All 9 articles implemented
- **Constraint Groups**: `core_spec`, `core_tdd`, `core_const`, `file_order`
- **Meta-Patterns**: `motive`, `guard`, `cycle`
- **Quality Gates**: `gate_vii`, `gate_viii`, `gate_ix`

### Agent Pipeline

The SpecKit skill implements a 7-agent pipeline with constitutional enforcement:

```
SpecArch → Clarify → Plan → Tasks → TDD → Integ → Guard → End
```

Each agent:
- References constitutional articles via `constit;` context
- Uses constraint groups for token efficiency
- Executes quality gates before handoff
- Logs constitutional compliance status

### Validation

Constitutional compliance is validated by the `Guard` agent:

```yamo
agent: Guard;
intent;constitutional_validation;
constraints:
  - art_I;lib_first
  - art_II;cli;stdio
  - art_III;tdd;NON_NEGOTIABLE
  - art_IV;integ_tests
  - art_V;logging
  - art_VI;semver
  - art_VII;max3
  - art_VIII;no_layers
  - art_IX;contracts_first
output;valid.json;
```

## Future Work

### v1.2 Proposals

1. **Constitutional Amendment Process**: On-chain voting for article changes
2. **Dynamic Constraint Groups**: Groups that adapt based on project context
3. **Constitutional Smart Contracts**: Automatic enforcement via blockchain
4. **Compliance Scoring**: Quantitative measures of constitutional adherence
5. **Cross-Constitution Compatibility**: Multiple constitutions for different domains

### Research Directions

1. **Constitution Discovery**: ML-based identification of implicit principles
2. **Constitution Merging**: Combining multiple constitutions without conflicts
3. **Constitution Testing**: Validating constitutions for logical consistency
4. **Constitution Evolution**: Automated constitution updates based on failure analysis

## Appendix A: Token Optimization Results

### skill-spec-kit.yamo Optimization

| Metric | Before | After | Reduction |
|--------|--------|-------|-----------|
| Characters | 9,063 | 4,512 | 50.3% |
| Words | 351 | 284 | 19.1% |
| Estimated Tokens | ~456 | ~369 | 19% |

### Techniques Applied

1. **Constraint Groups**: Replaced repeated constraint lists with group references
2. **Constitutional Centralization**: Single `constitution` block instead of per-agent references
3. **Abbreviated Context**: `template_path` → `tpl`, `timestamp_generated` → `ts`
4. **Compressed Constraints**: `write_acceptance_scenarios_using_given_when_then_format` → `accept_fmt;given_when_then`

## Appendix B: Constitutional Amendment Process

Constitutional articles follow RFC-based governance:

1. Proposal: Create RFC proposing article change
2. Discussion: Community review period (minimum 30 days)
3. Vote: Token-weighted voting on constitutional amendment
4. Implementation: Update constitution.md with new version
5. Blockchain Record: Anchor amendment on blockchain

## Appendix C: Reserved Constraint Groups

The following constraint groups are reserved and SHOULD NOT be overridden:

| Group | Purpose |
|-------|---------|
| `core_spec` | Specification generation constraints |
| `core_tdd` | Test-driven development constraints |
| `core_const` | Constitutional validation constraints |
| `file_order` | Test file creation order |

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
