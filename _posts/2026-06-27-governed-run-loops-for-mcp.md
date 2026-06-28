---
layout: post
title: "Governing the Run Loop: Why Enterprise AI Agents Must Think Inside the Box"
date: 2026-06-27 18:00:00 +0800
categories: [AAIF, Security]
topics: [agentic-safety]
projects: [mcp, agentgateway]
author: vincent_caldeira
external_repo: "https://github.com/caldeirav/mcp-governed-runloop"
image: "/assets/images/og/governed-run-loops-for-mcp.png"
description: "How to design a dynamic, multi-layered control plane using agentgateway to intercept, validate, and secure Model Context Protocol tool execution."
---

In many consumer applications, AI agents are praised for their unconstrained, creative planning. However, when we transition **Multi-Agent Systems (MAS)** to regulated financial environments, this unstructured behavior is an operational hazard. Letting an agent write arbitrary SQL databases or call critical banking APIs without determinism is a recipe for catastrophic system failures.

For agents to be enterprise-ready, they must **think inside the box**. That means their cognitive loops must run inside strict, structured architectural boundaries where planning is auditable and tool executions are safe.

This post walks through how to configure **agentgateway** as an architectural control plane that enforces boundaries over the **Model Context Protocol (MCP)**.

---

### The Cybernetic Control Loop Model

To secure agentic loops, we apply classical systems control theory. Rather than treating safety as an alignment layer inside a black-box model, we construct a closed feedback control loop:

$$\mathcal{S} = \{ \mathcal{A}, \mathcal{G}, \mathcal{M}, \mathcal{E} \}$$

Where:

* $\mathcal{A}$ is the autonomous agent engine generating tool-call requests.
* $\mathcal{G}$ is our real-time intercepting control plane gateway (**agentgateway**).
* $\mathcal{M}$ represents the machine-readable security policies (e.g., matching the FINOS AI Governance framework).
* $\mathcal{E}$ is the target execution pool (the MCP servers).

When the agent attempts to call a tool, the transaction vector $\mathbf{T}_{req}$ must pass a compliance check against our boundary policy state $\mathbf{P}$:

$$\mathbf{T}_{req} \cdot \mathbf{P} \ge \tau$$

If the boundary condition is violated, the transaction is intercepted, blocked, or dynamically routed to a validation correction handler.

---

### The Reference Architecture

To implement this containment box, we establish a centralized gateway layer between our agent runtime and our individual MCP servers:

```
+------------------+     [Tool Call Request]     +------------------+
|   Agent Engine   | --------------------------> |  agentgateway    |
|  (Dynamic Loop)  | <-------------------------- |  (Control Plane) |
+------------------+     [Sanitized / Safe]      +------------------+
                                                          |
                                              [Policy Match: Match/Block]
                                                          |
                                                          v
                                                 +-------------------+
                                                 |    MCP Server     |
                                                 | (Enterprise Apps) |
                                                 +-------------------+
```

---

### Step-by-Step Configuration Implementation

To operationalize this pattern, we configure our control plane policy to explicitly restrict an agent's execution parameters when accessing an MCP filesystem server.

Here is the policy payload we write inside our `agentgateway-config.json` module:

```json
{
  "control_plane": {
    "version": "1.0.0",
    "boundaries": [
      {
        "id": "containment-box-filesystem",
        "mcp_server": "filesystem-connector",
        "allowed_tools": ["read_file", "list_directory"],
        "blocked_tools": ["write_file", "delete_file", "execute_command"],
        "enforcement": "deny_and_report",
        "audit_target": "/var/log/agentgateway/audit.json"
      }
    ]
  }
}
```

By placing this configuration inside the routing layer, we guarantee that even if the underlying LLM attempts to generate a writing tool-call instruction because of prompt injection, the tool execution vector is blocked natively by the architectural boundaries.

The full container configurations and functional testing suites for this setup are available in the open-source sandbox at [caldeirav/mcp-governed-runloop](https://github.com/caldeirav/mcp-governed-runloop).
