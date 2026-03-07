# RFC-0003: YAMO v1.1 - Token Optimization and Syntax Extensions

**Status:** Draft
**Author:** Soverane Labs
**Created:** 2026-01-20
**Updated:** 2026-03-07
**Supercedes:** RFC-0001 (YAMO v1.0) for optimization features

## Summary

This RFC defines YAMO v1.1, which extends the v1.0 protocol with token optimization features that reduce file size by up to 50% while maintaining full semantic compatibility. The extension introduces metadata centralization, constraint groups, meta-patterns, and compressed syntax patterns that eliminate repetition without changing protocol semantics.

## Motivation

YAMO v1.0 established a foundation for transparent agent workflows. However, production usage revealed token efficiency issues:

1. **Repetitive Constraints**: Common constraints repeated across multiple agents
2. **Verbose Context Values**: Long paths and repeated values consume tokens
3. **No Shared References**: Each agent declares independent values
4. **Metadata Scattered**: Configuration distributed across agent blocks
5. **No Reusable Structures**: Similar patterns duplicated throughout files

A typical 7-agent YAMO file with full v1.0 syntax consumes ~4500 tokens. This limits:
- Agent context window capacity for complex workflows
- Blockchain storage costs for on-chain reasoning traces
- Latency in agent-to-agent handoff communication

YAMO v1.1 addresses these with **token optimization patterns** that reduce file size by 40-50% while preserving all semantic meaning.

## Specification

### 1. Metadata Centralization

#### 1.1 Shared Values Block

YAMO v1.1 allows defining shared values in the `metadata` block for reference throughout the file:

```yamo
metadata:
  name;MySkill;
  version;1.1.0;

  # NEW: Shared paths and references
  templates:
    base;/path/to/templates;
    specify;speckit.specify;
    clarify;speckit.clarify;
    plan;speckit.plan;
    tasks;speckit.tasks;

  # NEW: Common configurations
  environment:
    requires_filesystem;true;
    base_path;/workspace/skills;
```

#### 1.2 Shorthand Path References

Agents reference centralized values with abbreviated keys:

```yamo
# BEFORE (v1.0)
agent: SpecificationArchitect;
context:
  template_path;/very/long/path/to/templates/speckit.specify;

# AFTER (v1.1)
agent: SpecArch;
context:
  tpl;specify;  # Resolves to templates.base + templates.specify
```

### 2. Constraint Groups

#### 2.1 Definition

Constraint groups are reusable constraint sets defined in metadata:

```yamo
metadata:
  constraint_groups:
    core_validation:
      - validate_schema;true
      - check_types;strict
      - verify_bounds;true

    test_workflow:
      - setup;before_each
      - teardown;after_each
      - assert_all;true
      - coverage;80

    file_order:
      - contracts
      - integration
      - e2e
      - unit
      - source
```

#### 2.2 Usage Syntax

Agents reference groups with `use_group;` directive:

```yamo
# BEFORE (v1.0) - ~75 tokens
constraints:
  - write_tests_before_implementation;true
  - obtain_user_approval_before_proceeding
  - ensure_tests_fail_before_writing_source_code
  - implement_only_enough_code_to_pass_tests
  - refactor_only_after_tests_pass

# AFTER (v1.1) - ~12 tokens (84% reduction)
constraints;use_group;test_workflow;
```

#### 2.3 Group Composition

Multiple groups can be composed together:

```yamo
constraints;use_group;core_validation;test_workflow;
```

Individual constraints can extend groups:

```yamo
constraints;use_group;test_workflow;
  - additional_constraint;value;
  - extra_check;custom;
```

### 3. Meta-Patterns

#### 3.1 Definition

Meta-patterns define shared values for commonly used meta fields:

```yamo
metadata:
  meta_patterns:
    motive;transparency_enables_trust;
    guard;constraints_enable_creativity;
    cycle;bidirectional_feedback;
    default_confidence;0.85;
```

#### 3.2 Reference Syntax

Agents reference patterns with `ref;` directive:

```yamo
# BEFORE (v1.0)
meta:
  rationale;Constraints enable creativity within boundaries;
  confidence;0.85;

# AFTER (v1.1)
meta;ref;guard;default_confidence;
```

### 4. Compressed Syntax Patterns

#### 4.1 Abbreviated Field Names

Common field names have shorthand equivalents:

| Full Name | Shorthand | Token Savings |
|-----------|-----------|---------------|
| `template_path` | `tpl` | 67% |
| `timestamp_generated` | `ts` | 83% |
| `from_SpecificationArchitect` | `from_SpecArch` | 60% |
| `constitution` | `constit` | 50% |

#### 4.2 Abbreviated Agent Names

Agent names may be shortened for handoff references:

```yamo
agent: SpecificationArchitect;
handoff;ClarificationAgent;  # Full name still required on declaration
```

Becomes:

```yamo
agent: SpecArch;
handoff;Clarify;
```

#### 4.3 Compressed Constraint Format

Constraints use abbreviated key-value pairs:

```yamo
# BEFORE
- write_acceptance_scenarios_using_given_when_then_format
- execute_constitutional_gates_before_proceeding
- validate_completeness_before_proceeding

# AFTER
- accept_fmt;given_when_then
- exec_gates;pre_flight
- validate_complete
```

### 5. New Keywords

| Keyword | Purpose | Location |
|---------|---------|----------|
| `constraint_groups;` | Define reusable constraint sets | metadata |
| `meta_patterns;` | Define shared meta values | metadata |
| `templates;` | Centralize path references | metadata |
| `use_group;` | Reference a constraint group | constraints |
| `ref;` | Reference a meta-pattern | meta |
| `preserve;` | Verbatim content that must not be summarized | agent block |
| `procedure;` | Ordered multi-step workflow that must not collapse | agent block |
| `;verbatim;` | Constraint item modifier — exact phrasing required | constraints items |

### 6. Verbatim Preservation Primitives

Token optimization (§1–4) reduces file size by eliminating redundancy. Verbatim preservation is the complementary concern: certain content must survive the LLM-as-parser round-trip without compression or paraphrase. These primitives make that intent explicit to the parser.

#### 6.1 Motivation

The LLM-as-parser model is YAMO's core design. It produces a class of failure distinct from token waste: **lossy compression of semantically load-bearing content**. Three specific patterns degrade reliably:

1. **Multi-line verbatim templates** (PR bodies, commit message templates, shell heredocs) — compressed to one-line summaries, losing the exact text the user must copy
2. **Ordered procedures** (numbered checklists, self-review steps, sequential workflows) — collapsed from N explicit steps to a single "follow the checklist" entry
3. **Conditional branches** (if/else logic, environment detection) — resolved to a single path, dropping the other branches

These failures are not format errors — the YAMO syntax is valid — but the semantic content is lost. Verbatim preservation primitives give authors a way to signal "this content must not be compressed."

#### 6.2 `preserve;` Block

A `preserve:` block holds named items that the LLM MUST reproduce verbatim, not infer or paraphrase.

```yamo
agent: CommitsAndPRs;
...
preserve:
  - fork_check_branch_a;if_origin_does_not_point_to_apache/airflow;use_origin_as_fork_remote;verbatim;
  - fork_check_branch_b;if_origin_points_to_apache/airflow;look_for_other_remote_pointing_to_user_fork;verbatim;
  - fork_check_branch_c;if_no_fork_remote_exists;gh repo fork apache/airflow --remote --remote-name fork;verbatim;
```

**Semantics:** Items in `preserve:` carry stronger reproduction guarantees than `constraints:`. The parser MUST NOT rephrase them. Content in `preserve:` is for exact reproduction, not enforcement.

**When to use:** Conditional branches with multiple distinct resolution paths. Each branch is a separate `preserve:` item — never collapsed to one.

#### 6.3 `procedure;` Block

A `procedure:` block holds numbered ordered steps that MUST NOT be collapsed into a summary.

```yamo
agent: CommitsAndPRs;
...
procedure:
  self_review:
    1;git diff main...HEAD;verify_every_change_intentional;remove_unrelated_changes;
    2;read_code_review_checklist;check_against_every_rule;architecture_boundaries,database_correctness,code_quality;verbatim;
    3;confirm_code_follows_coding_standards_and_architecture_boundaries;
    4;prek run --from-ref <target_branch> --hook-stage pre-commit;fix_any_failures;verbatim;
    5;prek run --from-ref <target_branch> --hook-stage manual;fix_any_failures;verbatim;
    6;run_relevant_individual_tests;confirm_they_pass;
    7;breeze selective-checks;run_tests_in_parallel;check_for_CI-specific_issues;verbatim;
    8;check_for_secrets;no_injection_vulnerabilities;no_unsafe_patterns;
```

**Semantics:** The parser MUST reproduce all numbered steps in order. Collapsing 8 steps to "follow the self-review checklist" is a parsing error. Sub-items on each step (categories, flags, tool invocations) MUST also be preserved.

**When to use:** Any content where order matters and partial reproduction is a functional failure — checklists, setup sequences, workflows with dependencies between steps.

#### 6.4 `;verbatim;` Modifier

Appending `;verbatim;` to any constraint item signals that the exact phrasing of that item is required and MUST NOT be paraphrased.

```yamo
constraints:
  - self_review_step_2_categories;architecture_boundaries,database_correctness,code_quality,testing_requirements,API_correctness,AI-generated_code_signals;verbatim;
  - ai_disclosure;Generated-by: <Agent Name and Version> following the guidelines;verbatim;
  - fork_check_branch_a;if_origin_does_not_point_to_apache/airflow;use_origin_as_fork_remote;verbatim;
```

**Semantics:** The item's value must be reproduced character-for-character. No synonym substitution, no abbreviation, no summarization.

**When to use:** Named category lists where omitting any item is a functional error; exact tool invocations with flags that must not be paraphrased; verbatim text the user will copy (disclosure statements, commit messages).

#### 6.5 YAML Literal Block Scalars

For multi-line verbatim content (PR body templates, heredocs, commit message formats), use YAML literal block scalar syntax (pipe `|`):

```yamo
agent: CommitsAndPRs;
...
pr_body_template: |
  Brief description of the changes.

  closes: #ISSUE  (if applicable)

  ---

  ##### Was generative AI tooling used to co-author this PR?

  - [X] Yes — <Agent Name and Version>

  Generated-by: <Agent Name and Version> following the guidelines
```

**Semantics:** The block content is immune to LLM compression. The parser copies it; it does not reason about it. This is the strongest verbatim preservation available — the content never enters the inference path.

**When to use:** Any text the user must copy exactly: PR body templates, commit message templates, configuration file blocks, shell heredocs. If the user will paste it, use a literal block.

#### 6.6 Content Classification Protocol

Before translating any source document to YAMO format, classify all content by type:

| Source pattern | Target primitive |
|---|---|
| Multi-line block the user copies verbatim | YAML literal block scalar (`key: \|`) |
| Numbered ordered checklist / sequential workflow | `procedure:` block |
| If/else logic with multiple resolution paths | Named per-branch items in `preserve:` or `constraints:` with `;verbatim;` |
| Exact flag combinations, named category lists | `;verbatim;` modifier on constraint items |
| Rule-based prohibitions / enforcement | `constraints:` block (standard) |
| Stateful context passed between agents | `context:` block (standard) |

**Invariant:** A source document containing verbatim templates, ordered procedures, or conditional branches MUST NOT be translated to YAMO using only `constraints:` blocks. The compression failure is predictable and avoidable.

#### 6.7 Interaction with Token Optimization

Verbatim preservation and token optimization are complementary, not conflicting:

- Token optimization (§1–4) reduces redundancy in **rule-based** content
- Verbatim preservation (§6) protects **procedural and template** content from compression

A single agent block may use both: a `constraints;use_group;core_validation;` reference (§2) alongside a `procedure:` block (§6.3) for its ordered steps. The optimization applies to the constraints; the procedure remains fully expanded.

**Anti-pattern:** Applying constraint group compression to procedural steps compresses content that must not be compressed. Constraint groups are for rule-based content. Procedure blocks are for ordered steps.

#### 6.8 Token Cost

Verbatim preservation primitives increase token count compared to compressed representations. This is intentional — the cost of preservation is always less than the cost of an incorrect output.

| Technique | Token delta | Correctness guarantee |
|---|---|---|
| Flat constraint item (compressed) | Baseline | None — LLM may paraphrase |
| `;verbatim;` modifier | +1 token | Signals exact phrasing required |
| `preserve:` block | +2–5 tokens overhead | Reproduction without inference |
| `procedure:` block | Full step detail preserved | No collapse to summary |
| YAML literal block scalar | Full content preserved | Immune to inference path |

### 6. Parsing Rules

#### 6.1 Reference Resolution

Parsers MUST resolve references in this order:
1. `use_group;` → expand constraint group definitions
2. `tpl;` → resolve against `templates` base path
3. `ref;` → resolve against `meta_patterns`
4. `ts;` → expand to current timestamp

#### 6.2 Backwards Compatibility

Parsers MUST:
- Accept both full and abbreviated syntax
- Ignore unknown metadata fields
- Expand `use_group;` before validation
- Maintain semantic equivalence with v1.0

### 7. Complete Example

```yamo
metadata:
  name;OptimizedSkill;
  version;1.1.0;
  description;Demonstrates v1.1 token optimization.;
  author;Soverane Labs;
  license;MIT;
  templates:
    base;/workspace/templates;
    spec;template.spec.yaml;
    plan;template.plan.yaml;
  constraint_groups:
    core:
      - validate;strict
      - check_types;true
      - verify_schema;true
    tdd:
      - tests_first;true
      - fail_first;true
      - min_code;true
  meta_patterns:
    motive;optimization_preserves_semantics;
    guard;structured_constraints_reduce_hallucination;
---
agent: SpecArch;
intent;create_specification;
context:
  input;user_intent;
  tpl;spec;
constraints;use_group;core;
priority;critical;
output;spec.yaml;
log;created;ts;
meta;ref;motive;
handoff;Plan;
---
agent: Plan;
intent;create_technical_plan;
context:
  spec;from_SpecArch;
  tpl;plan;
constraints;use_group;core;
  - research_tech;true
priority;critical;
output;plan.yaml;
log;planned;ts;
meta;ref;guard;
handoff;End;
```

## Rationale

### Why These Optimizations?

1. **Constraint Groups**: Most workflows reuse similar constraints across agents (validation, logging, error handling)
2. **Path Centralization**: Template paths, base directories, and common references are repeated 3-5 times per file
3. **Meta-Patterns**: Rationale statements and confidence levels often follow patterns
4. **Compressed Syntax**: Long descriptive names are valuable for documentation but consume tokens

### Why Maintain v1.0 Compatibility?

1. **Incremental Adoption**: Existing files can be gradually optimized
2. **Tool Compatibility**: v1.0 parsers continue to work with v1.1 files (new keywords ignored)
3. **Human Readability**: Full syntax remains available for clarity when needed
4. **Blockchain Provenance**: v1.0 blocks on chain remain valid

### Token Reduction Breakdown

Based on production optimization of a 7-agent workflow:

| Technique | Tokens Saved | % Reduction |
|-----------|--------------|-------------|
| Constraint groups | ~150 | 33% |
| Path centralization | ~80 | 18% |
| Meta-patterns | ~60 | 13% |
| Compressed syntax | ~40 | 9% |
| **Total** | **~330** | **~73% of original** |

## Security Considerations

### Reference Injection

- User-defined constraint groups MUST be validated against reserved groups
- Group names containing system keywords MUST be rejected
- Circular group references MUST be detected and rejected

### Parsing Consistency

- All parsers MUST produce equivalent semantic output for v1.0 and v1.1 syntax
- Reference expansion MUST be deterministic
- Timestamp generation MUST be monotonic

### Blockchain Compatibility

- Optimized files produce equivalent blockchain hashes when expanded
- v1.1 syntax is transparent to blockchain validation
- On-chain storage stores expanded form for canonical hashing

## Backwards Compatibility

| Feature | v1.0 Support | v1.1 Support |
|---------|--------------|--------------|
| All v1.0 keywords | ✅ | ✅ |
| Blockchain integration | ✅ | ✅ |
| Meta-reasoning traces | ✅ | ✅ |
| Full constraint syntax | ✅ | ✅ |
| Shorthand field names | ❌ | ✅ |
| Constraint groups | ❌ | ✅ |
| Meta-patterns | ❌ | ✅ |
| `preserve:` block | ❌ | ✅ |
| `procedure:` block | ❌ | ✅ |
| `;verbatim;` modifier | ❌ | ✅ |
| YAML literal block scalars | ❌ | ✅ |

**Migration**: Optional. v1.0 files continue to work unchanged. Optimization is additive.

## Reference Implementation

### Token Optimization Results

A production 7-agent YAMO file optimized with v1.1 patterns:

| Metric | Before | After | Reduction |
|--------|--------|-------|-----------|
| Characters | 9,063 | 4,512 | 50.3% |
| Words | 351 | 284 | 19.1% |
| Estimated Tokens | ~456 | ~369 | 19.0% |

### Optimization Techniques Applied

1. **Constraint Groups**: Replaced 8 repeated constraint lists with 4 group references
2. **Path Centralization**: Moved 5 repeated paths to `templates` block
3. **Meta-Patterns**: Consolidated 3 repeated meta values
4. **Compressed Syntax**: Abbreviated 12 field names and constraint descriptions

### Pseudocode: Reference Expansion

```
function expand_v11_to_v10(yamo_content):
    metadata = parse_metadata(yamo_content)

    for agent in yamo_content.agents:
        # Expand constraint groups
        if agent.constraints.contains("use_group"):
            for group_name in agent.constraints.use_group:
                agent.constraints.extend(metadata.constraint_groups[group_name])

        # Resolve path references
        if agent.context.contains("tpl"):
            agent.context.template_path = metadata.templates.base + metadata.templates[agent.context.tpl]

        # Expand meta patterns
        if agent.meta.contains("ref"):
            for pattern_ref in agent.meta.ref:
                agent.meta[pattern_ref] = metadata.meta_patterns[pattern_ref]

        # Expand timestamps
        if agent.log.contains("ts"):
            agent.log.timestamp = current_timestamp()

    return yamo_content  # Now in v1.0 format
```

## Future Work

### v1.2 Proposals

1. **Dynamic Constraint Groups**: Groups that accept parameters
2. **Conditional References**: Include/exclude groups based on context
3. **Import Statements**: Reference constraint groups from external files
4. **Macro Expansion**: User-defined syntax transformations
5. **Compression Standard**: Canonical compressed representation for blockchain storage
6. **Verbatim validation tooling**: Static checker that detects compressed constraint items that should be literal block scalars or procedure blocks

### Research Directions

1. **Automatic Optimization**: Tool to convert v1.0 to v1.1 syntax
2. **Semantic Equivalence Proofs**: Formal verification of optimization correctness
3. **Token Usage Analysis**: ML-based identification of optimization opportunities
4. **Cross-Language Optimization**: Patterns applicable to other agent languages

## Appendix A: Reserved Keywords

The following are reserved and MUST NOT be used as custom group or pattern names:

| Reserved | Purpose |
|----------|---------|
| `use_group` | Constraint group reference |
| `ref` | Meta-pattern reference |
| `tpl` | Template path reference |
| `ts` | Timestamp expansion |
| `from_` | Agent handoff prefix |
| `preserve` | Verbatim content block |
| `procedure` | Ordered procedure block |
| `verbatim` | Constraint item modifier (as trailing semicolon tag) |

## Appendix B: Optimization Guidelines

### When to Use Constraint Groups

Use constraint groups when:
- Same constraints appear in 3+ agents
- Constraints represent a coherent workflow stage
- Group name clearly describes the constraint purpose

Avoid when:
- Constraints are agent-specific
- Group would have only 1-2 items
- Group name is ambiguous

### When to Use Shorthand Syntax

Use shorthand syntax when:
- File size is critical (e.g., on-chain storage)
- Context makes meaning clear
- Working within token limits

Use full syntax when:
- Readability is priority
- File is educational or reference
- Audience may not know abbreviations

## Appendix C: Migration Guide

### Converting v1.0 to v1.1

1. **Identify Repetition**: Find constraints repeated across 3+ agents
2. **Create Groups**: Move repeated constraints to `constraint_groups`
3. **Centralize Paths**: Move common paths to `templates` block
4. **Add Meta-Patterns**: Extract common meta values
5. **Update References**: Replace with `use_group;`, `ref;`, `tpl;`
6. **Validate**: Ensure output is semantically equivalent

### Example Migration

```yamo
# v1.0
agent: AgentOne;
constraints:
  - validate_schema;true
  - check_types;strict
  - specific_constraint;value;
---
agent: AgentTwo;
constraints:
  - validate_schema;true
  - check_types;strict
  - other_constraint;value;

# v1.1
metadata:
  constraint_groups:
    common:
      - validate_schema;true
      - check_types;strict;
---
agent: AgentOne;
constraints;use_group;common;
  - specific_constraint;value;
---
agent: AgentTwo;
constraints;use_group;common;
  - other_constraint;value;
```

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
