# RFC-0004: YAMO Gateway & Workspace Specification

**Status:** Draft  
**Author:** ConstitutionalOptimizer & Collaborative Swarm  
**Created:** 2026-02-14  
**Updated:** 2026-02-14  

## Summary

This RFC proposes the transformation of the YAMO Daemon into a multiplexed **YAMO Gateway** and the standardization of a **YAMO Workspace** structure. It introduces a distributed **Node Protocol** for remote actuation and a simplified **Pairing UX** for secure device authorization, drawing from the superior user-experience patterns established by the OpenClaw platform.

## Motivation

Current YAMO-OS infrastructure excels in autonomous reasoning and blockchain provenance but lacks the accessibility required for seamless multi-device interaction. Headless VPS deployments are difficult to manage without a standardized remote interface. By adopting a Gateway architecture and a strict Workspace model, we enable: 
1. **Remote Heads:** Secure dashboard and chat interfaces on any device.
2. **Distributed I/O:** Using phones/laptops as actuators (nodes) for a centralized server brain.
3. **Soul Portability:** Decoupling personality (Workspace) from the core logic (OS).

## Specification

### 1. The YAMO Gateway
The existing daemon shall be replaced by a multiplexed WebSocket + HTTP Gateway listening on a single secure port (default: 3333).
- **Protocol:** `yamo-protocol/v1` (JSON-RPC over WS).
- **Multiplexing:** Serves the Visualization API, real-time Agent Logs, and WebChat UI simultaneously.
- **Trusted Proxies:** Native support for `X-Forwarded-For` to enable secure Nginx/Tailscale reverse proxying.

### 2. Standardized YAMO Workspace (`~/.yamo/workspace/ `)
A canonical directory structure for agent identity and continuity:
- `SOUL.md`: The primary Constitutional Prompt and personality definition.
- `AGENTS.md`: Operational boundaries and project-specific focus areas.
- `USER.md`: Context regarding the human user (Timezone, Preferences).
- `MEMORY.md`: Distilled long-term semantic memory (linked to MemoryMesh).
- `skills/`: User-defined AgentSkills.

### 3. YAMO Node Protocol
Allows remote devices to register as "Capability Providers."
- **Node Registration:** Nodes announce capabilities (e.g., `camera.snap`, `location.get`) to the Gateway.
- **Remote Invocation:** The Gateway executes tool calls by proxying them to the relevant Node via the WS connection.

### 4. Pairing UX
- **Command:** `yamo pair` generates a 6-character, short-lived alpha-numeric code.
- **Handshake:** Devices enter the code in the UI; the Gateway performs a public-key exchange and adds the Device ID to `identity/allowFrom.json`.

## Rationale

We chose the Gateway model over the Daemon model to facilitate real-time streaming of thoughts and blocks, which is critical for user trust in autonomous systems. The Workspace model is chosen to ensure that "Soul" data is human-readable and easily portable between OS versions.

## Backwards Compatibility

- **Breaking Change:** This requires a migration of existing config files to the new Workspace structure.
- **Migration Path:** A `yamo-migrate` tool will be provided to move existing prompts into `SOUL.md` and `AGENTS.md`.

## Security Considerations

- **Token Handshake:** All non-localhost connections require a 64-character Gateway Token.
- **Path Obscurity:** The Gateway supports custom URL prefixes (e.g., `/yamo-hash/`) to prevent bot scanning.
- **Capability Gating:** Nodes must be explicitly approved before their tools are made available to the agent.

## Reference Implementation

Elements of this specification are currently running as a prototype in the **OpenClaw-YAMO Bridge** deployment (Feb 2026).

## Supersession Note

The **YAMO Workspace** structure introduced in §2 of this RFC has been superseded and fully specified by **RFC-0009** (YAMO Workspace File Format Specification). RFC-0009 is the authoritative reference for `SOUL.md`, `AGENTS.md`, `USER.md`, `MEMORY.md`, `BOOTSTRAP.md`, and daily memory files. The Gateway and Node Protocol sections of this RFC remain active.

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 0.1.0 | 2026-02-14 | Initial draft — YAMO Gateway, Workspace, Node Protocol, Pairing UX |
| 0.1.1 | 2026-02-21 | Add RFC-0009 supersession note for workspace file definitions |

---

## Copyright

Copyright and related rights waived via [MIT License](https://opensource.org/licenses/MIT).
