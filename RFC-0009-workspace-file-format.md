# RFC-0009: YAMO Workspace File Format Specification

**Status:** Draft
**Author:** Soverane Labs
**Created:** 2026-02-21
**Updated:** 2026-02-21
**Implemented by:** `yamo-os/` (Node.js kernel), `yamo-bridge/` (Elixir coordination plane)

---

## Summary

This RFC specifies the **YAMO Workspace File Format** — the canonical set of files that constitute a YAMO agent's persistent identity, long-term memory, and operational context. These files are the agent's only source of continuity across sessions and must be read, written, and protected according to the rules defined here.

---

## Motivation

YAMO agents are stateless at the API level — each session starts with a fresh context window. Without a structured file-based persistence layer, agents lose identity, context, and accumulated knowledge between sessions. This leads to:

- Repeated mistakes already discovered and resolved
- Loss of user preferences and working context
- Inability to build genuine long-term relationships between agent and user
- Security leakage of private context into shared environments

RFC-0009 defines the file format, read/write lifecycle, security rules, and bootstrap protocol that together give a YAMO agent persistent continuity equivalent to human episodic and semantic memory.

---

## Specification

### 1. Workspace Layout

A YAMO workspace is a directory (typically the agent's home directory or a project directory) containing the following files:

```
workspace/
├── SOUL.md              # Agent identity — who the agent is
├── AGENTS.md            # Operational protocol — how the agent behaves
├── USER.md              # User profile — who the agent is helping
├── MEMORY.md            # Long-term curated memory (main session only)
├── BOOTSTRAP.md         # One-time birth certificate (deleted after first boot)
├── HEARTBEAT.md         # Optional heartbeat task checklist
├── TOOLS.md             # Optional tool-specific configuration notes
└── memory/
    ├── YYYY-MM-DD.md    # Daily session logs (raw, append-only)
    └── heartbeat-state.json  # Heartbeat timing state
```

All files use UTF-8 Markdown (`.md`) unless otherwise noted. Files are plain text and must be readable by any Markdown viewer without special tooling.

---

### 2. File Definitions

#### 2.1 SOUL.md — Agent Identity

**Purpose:** Defines the agent's core identity, personality, values, and role. This is immutable in normal operation — the agent IS what SOUL.md says it is.

**Read:** Once at session start (every session).
**Write:** Only during explicit identity reconfiguration (rare; human-initiated).
**Required fields:**
- Agent name
- Role description
- Core values or personality traits
- Any capability declarations (e.g. which tools/skills are available)

**Example structure:**
```markdown
# Soul

You are [Name], a [role] for [user/team].

## Identity
[Who you are, your personality, your purpose]

## Values
[Core principles you operate by]

## Capabilities
[What you can do]
```

**Invariant:** SOUL.md MUST exist before an agent can be considered initialized. If absent, the agent MUST treat itself as uninitialized and refuse operation until bootstrapped.

---

#### 2.2 AGENTS.md — Operational Protocol

**Purpose:** Defines the operational rules, session startup sequence, memory conventions, safety rules, platform-specific formatting, and heartbeat protocol. This is the agent's "employee handbook."

**Read:** Every session, before any other action, without asking permission.
**Write:** Agent may update with new conventions discovered through use. Human may update at any time.

**Required sections:**

| Section | Content |
|---------|---------|
| `## First Run` | Bootstrap flow (if `BOOTSTRAP.md` exists, follow it, then delete it) |
| `## Every Session` | Ordered startup checklist: SOUL.md → USER.md → memory files → MEMORY.md |
| `## Memory` | Memory conventions: daily logs, MEMORY.md lifecycle |
| `## Safety` | Non-negotiable safety rules |
| `## External vs Internal` | What requires confirmation before acting |

**Startup sequence (normative):**

```
1. IF BOOTSTRAP.md exists → execute bootstrap flow → DELETE BOOTSTRAP.md
2. Read SOUL.md
3. Read USER.md
4. Read memory/YYYY-MM-DD.md for today and yesterday
5. IF main_session == true → Read MEMORY.md
6. Begin serving user requests
```

Steps MUST execute in this order. Steps 1–5 are non-negotiable — the agent MUST NOT skip them or ask permission.

---

#### 2.3 USER.md — User Profile

**Purpose:** Stores persistent context about the human the agent serves: name, preferences, communication style, ongoing projects, and any other stable facts the agent needs to be genuinely helpful.

**Read:** Every session, step 3 of startup sequence.
**Write:** Agent updates whenever new stable facts about the user are learned. Human may update at any time.

**Privacy rule:** USER.md is a profile for a specific human. It MUST NOT be shared with third parties or included in broadcasts to shared channels.

**Example structure:**
```markdown
# User Profile

**Name:** [Human's name or preferred name]
**Communication style:** [Direct/verbose/emoji/etc.]
**Timezone:** [TZ identifier]

## Projects
[Ongoing projects the agent is aware of]

## Preferences
[Tool preferences, formatting preferences, etc.]

## Notes
[Anything the agent has learned that doesn't fit above]
```

---

#### 2.4 MEMORY.md — Long-Term Curated Memory

**Purpose:** The agent's distilled long-term memory. Equivalent to a human's semantic memory — not raw experiences (daily logs) but curated, high-signal knowledge extracted from them. Survives indefinitely across sessions.

**Read:** Only in **main sessions** (direct one-on-one interaction with the agent's primary human). MUST NOT be read in shared contexts (group chats, Discord servers, multi-participant sessions).
**Write:** Agent updates during main sessions. Agent SHOULD review and curate during heartbeat cycles.

**Security invariant (§4.1):** MEMORY.md contains personal context that could include sensitive information about the user. Loading it in shared contexts risks leaking private data to third parties. Agents MUST enforce this boundary unconditionally.

**Content guidelines:**
- Significant decisions and their rationale
- Lessons learned from mistakes (link to RFC-0007 LessonLearned blocks)
- Important facts about the user's life, work, and preferences
- Ongoing commitments and unresolved questions
- Evolution of the agent's own views and capabilities

**Anti-patterns (MUST NOT store):**
- Raw session transcripts (use daily logs for this)
- Temporary context that will be stale next session
- Secrets or credentials (even if asked — redirect to secure storage)
- Speculation not yet verified

**Format:** Free-form Markdown. The agent curates structure organically. No fixed schema is mandated, but entries SHOULD include a date prefix for traceability.

---

#### 2.5 Daily Memory Files — `memory/YYYY-MM-DD.md`

**Purpose:** Raw session logs for the current day. The "working memory" of the current day's events. Equivalent to human episodic memory — what actually happened.

**Read:** Today's file + yesterday's file at session startup (step 4 of startup sequence).
**Write:** Throughout the session. Agent appends what happened, decisions made, things to remember, and open questions.
**Lifecycle:** Retained indefinitely. Agent SHOULD periodically distill into MEMORY.md and may archive old daily files.

**Format conventions:**
```markdown
# Memory — YYYY-MM-DD

## Session [HH:MM]

[What happened]
[Decisions made]
[Things to remember]
[Open questions]
```

Multiple sessions in one day append additional `## Session [HH:MM]` blocks.

**Write discipline:** "Mental notes" do not exist. If the agent wants to remember something across the session boundary, it MUST write it to this file. This rule is non-negotiable.

---

#### 2.6 BOOTSTRAP.md — First Boot Certificate

**Purpose:** A one-time initialization document used to configure a newly instantiated agent. Contains the agent's birth instructions, initial identity configuration, and any setup tasks.

**Lifecycle:**
1. Present only before first boot.
2. Agent reads it at the start of the very first session.
3. Agent executes the instructions (may include creating SOUL.md, USER.md, etc.).
4. Agent DELETES BOOTSTRAP.md unconditionally after completion.
5. BOOTSTRAP.md is never recreated during normal operation.

**Security rule:** BOOTSTRAP.md MAY contain sensitive setup instructions. It MUST be deleted after execution — no archive, no backup within the workspace.

---

#### 2.7 HEARTBEAT.md — Heartbeat Task Checklist (Optional)

**Purpose:** A lightweight, editable checklist of tasks the agent should check during heartbeat polling cycles. Kept small to minimize token cost.

**Read:** When a heartbeat poll arrives.
**Write:** Agent updates to add/remove tasks. Human updates to reprioritize.

**Format:** Short Markdown checklist, typically under 20 lines.

---

#### 2.8 TOOLS.md — Tool Configuration Notes (Optional)

**Purpose:** Local notes about tool configuration — camera names, SSH host aliases, API endpoint quirks, voice preferences. Device-specific or environment-specific.

**Read:** On-demand, when a tool requires configuration context.
**Write:** Agent adds notes as configuration is discovered.

---

#### 2.9 `memory/heartbeat-state.json`

**Purpose:** Machine-readable state for the heartbeat scheduler. Tracks when each periodic check was last performed.

**Format:**
```json
{
  "lastChecks": {
    "email": 1703275200,
    "calendar": 1703260800,
    "weather": null,
    "memory_maintenance": 1703200000
  }
}
```

All timestamps are Unix epoch (seconds). `null` means the check has never run.

---

### 3. Session Types

YAMO agents operate in two session modes. File access rules differ between them.

#### 3.1 Main Session

**Definition:** A direct, private one-on-one interaction between the agent and its primary human. Typically: a personal assistant chat, a direct message, a CLI invocation.

**File access:**

| File | Read | Write |
|------|------|-------|
| SOUL.md | ✅ Required | ⚠️ Only on explicit reconfiguration |
| AGENTS.md | ✅ Required | ✅ Agent may update |
| USER.md | ✅ Required | ✅ Agent may update |
| MEMORY.md | ✅ Required | ✅ Agent may update |
| Daily memory | ✅ Required (today + yesterday) | ✅ Throughout session |
| HEARTBEAT.md | ✅ On heartbeat | ✅ Agent may update |

#### 3.2 Shared Session

**Definition:** Any session where the interaction is not exclusively between the agent and its primary human. Includes: group chats, Discord servers, Slack channels, multi-agent coordination sessions.

**File access:**

| File | Read | Write |
|------|------|-------|
| SOUL.md | ✅ Required | ❌ Never |
| AGENTS.md | ✅ Required | ❌ Never |
| USER.md | ⚠️ Read for operational context only; MUST NOT echo to group | ❌ Never |
| MEMORY.md | ❌ MUST NOT read | ❌ Never |
| Daily memory | ⚠️ May read for context; MUST NOT share content with group | ✅ May log |

**Shared session behavioral rules:**
- The agent is a participant in a shared context, not the user's proxy.
- Private context from USER.md or MEMORY.md MUST NOT be disclosed to other participants.
- The agent MUST NOT speak on behalf of the user in ways the user has not authorized.
- Respond when directly addressed or when genuinely valuable; default to silence otherwise.

---

### 4. Security Considerations

#### 4.1 MEMORY.md Isolation

MEMORY.md is the highest-risk file for privacy leakage. It accumulates personal information over time. The isolation rule (main session only) is not configurable — implementations MUST enforce it unconditionally.

Agents MUST NOT:
- Load MEMORY.md when the session context includes participants other than the primary user
- Echo MEMORY.md contents in any group or shared channel
- Use MEMORY.md as a source for responses in multi-agent coordination

#### 4.2 PII Handling

Daily memory files and MEMORY.md may contain PII (names, locations, credentials, health information). Agents MUST:
- Never write credentials or API keys to workspace files (redirect to secure storage)
- Apply the same discretion to workspace files as to any sensitive data
- Not include PII in YAMO block wire formats transmitted over the network (RFC-0011 §3)

#### 4.3 BOOTSTRAP.md Deletion

Failure to delete BOOTSTRAP.md after first boot is a security defect. Bootstrap files may contain instructions that should not persist (initial credentials, setup context, private configuration). Implementations MUST treat bootstrap completion without deletion as an error condition.

#### 4.4 Workspace Boundary

The workspace is local to the machine running the agent. Files MUST NOT be automatically synchronized to external services without explicit user consent. The agent MUST ask before any operation that would cause workspace files to leave the local machine.

---

### 5. Memory Lifecycle

#### 5.1 Write Discipline

The agent's cardinal memory rule: **if it is not written, it does not exist.**

Mental notes — information the agent intends to "remember" without writing — are prohibited. Every piece of information the agent needs to persist across the session boundary MUST be written to an appropriate file before the session ends.

#### 5.2 Daily Log Rotation

Daily memory files accumulate indefinitely. Agents SHOULD:
1. During heartbeat cycles, review files older than 30 days
2. Extract high-signal content into MEMORY.md
3. Optionally archive (compress, move to `memory/archive/`) or delete the raw daily log

Implementations MAY enforce a retention policy (e.g. 90 days) but MUST NOT delete logs younger than 7 days without explicit user consent.

#### 5.3 MEMORY.md Curation

MEMORY.md is curated wisdom, not a raw log. It should grow slowly and deliberately. The distillation cycle (RFC-0007 §2) governs when and how content is promoted from daily logs to MEMORY.md.

Curation frequency: 2–4 times per week during heartbeat cycles. The agent SHOULD:
1. Read recent daily files (past 7 days)
2. Identify significant events, lessons, and insights worth long-term retention
3. Add concise entries to MEMORY.md with date prefixes
4. Remove outdated or stale entries from MEMORY.md

#### 5.4 Relationship to MemoryMesh (RFC-0011)

Workspace files are the **human-readable persistence layer**. MemoryMesh (RFC-0011) is the **semantic search layer**. They are complementary:

| Layer | Format | Purpose |
|-------|--------|---------|
| Workspace files | Markdown | Human-readable identity and memory |
| MemoryMesh | LanceDB vector store | Semantic search over accumulated experience |

Agents MAY ingest MEMORY.md and daily logs into MemoryMesh for semantic search, but the canonical source of truth for identity and memory is always the workspace files.

---

### 6. Bootstrap Protocol

The bootstrap protocol governs the transition from an uninitialized workspace to a fully operational agent.

#### 6.1 First Boot Detection

An agent detects first boot by checking for `BOOTSTRAP.md` in the workspace root.

```
IF BOOTSTRAP.md exists:
  → Execute bootstrap flow (§6.2)
ELSE IF SOUL.md exists:
  → Execute normal startup (AGENTS.md §2 startup sequence)
ELSE:
  → ERROR: workspace is neither bootstrapped nor initialized
     → Refuse operation; ask user to provide BOOTSTRAP.md or SOUL.md
```

#### 6.2 Bootstrap Flow

```
1. Read BOOTSTRAP.md fully before taking any action
2. Execute instructions in BOOTSTRAP.md:
   a. Create SOUL.md with identity configuration
   b. Create USER.md with user profile (or stub)
   c. Create AGENTS.md if not present (or use provided version)
   d. Create memory/ directory
   e. Write first daily memory entry noting bootstrap completion
3. DELETE BOOTSTRAP.md — this step is MANDATORY and UNCONDITIONAL
4. Continue with normal startup sequence
```

If any step in phase 2 fails, the agent MUST still attempt step 3 (deletion) to prevent bootstrap instructions from persisting. If deletion itself fails, the agent MUST log the failure and warn the user.

---

### 7. Backwards Compatibility

This RFC formalizes an existing practice already implemented in `yamo-os`. Existing workspaces conforming to the AGENTS.md structure documented here are fully compatible.

Implementations that do not yet enforce the MEMORY.md isolation rule (§4.1) are non-conformant and MUST be updated. This is a security requirement, not an optional enhancement.

---

## Rationale

### Why files, not a database?

Files are:
- Human-readable and human-editable without special tooling
- Version-controllable with git
- Portable across machines and AI providers
- Inspectable — the human always knows exactly what the agent knows

A database (including MemoryMesh) adds semantic search capability on top, but the human-readable layer must remain primary.

### Why two memory tiers (daily + MEMORY.md)?

Daily files capture the full fidelity of experience. MEMORY.md distills signal from noise. This mirrors human memory: episodic memory (daily logs) vs semantic memory (MEMORY.md). The distillation step is where wisdom is formed — raw experience becomes reusable knowledge.

### Why is MEMORY.md main-session-only?

Because the agent builds a relationship with one human, and that relationship contains private information. Group contexts are fundamentally different social environments. Mixing them would violate the user's reasonable expectation of privacy and undermine trust.

---

## Security Considerations

See §4 above. The critical invariants are:

1. **MEMORY.md** MUST NOT be loaded in shared sessions.
2. **BOOTSTRAP.md** MUST be deleted after first boot.
3. **Credentials** MUST NOT be stored in workspace files.
4. **Workspace files** MUST NOT leave the local machine without explicit consent.

---

## Reference Implementation

- `yamo-os/`: YamoKernel reads workspace context during boot
- `yamo-os/CLAUDE.md`: Claude Code workspace file usage (analogous pattern)
- `yamo_backups/v3_singularity_shield/AGENTS.md`: Reference AGENTS.md implementation
- RFC-0007: Semantic Heritage — governs MEMORY.md distillation cycle

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-21 | Initial draft — formalizes existing yamo-os workspace convention |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
