---
title: "GH-600: Agentic AI Developer"
---

# GitHub Certified: Agentic AI Developer (Exam GH-600) Study Guide

**Exam GH-600 (*Developing in Agentic AI Systems*)** tests your ability to design, run, supervise, and govern autonomous AI agents inside GitHub workflows without breaking security or delivery speed. By default, GitHub acts as both the **System of Record** *(the permanent log of every plan, commit, check, and artifact)* and the **Control Plane** *(the policy layer that enforces rulesets, checks, permissions, and human approvals)*.

```text
courses/gh-600-agentic-ai-developer/
├── index.md                                                  # Exam overview & Key Terms Translator
├── 01-foundations-of-agentic-ai.md                           # Module 1.1: Plan → Act → Evaluate & Contributor Model
├── 02-designing-agent-architecture-and-sdlc-integration.md   # Module 1.2: Task Contracts, Plan Gates & Workflow Outputs
├── 03-tooling-mcp-and-execution-environments.md              # Module 1.3: gh aw, MCP Registries/Allowlists & Copilot CLI
├── 04-multi-agent-systems-and-orchestration.md               # Module 2.1: Fan-Out/Fan-In, Concurrency & .agent.md Handoffs
├── 05-memory-state-and-evaluation.md                         # Module 2.2: External Memory, State Anchors & Failure Tuning
├── 06-governance-guardrails-and-operations.md                # Module 2.3: Risk Tiers, Least Privilege & preToolUse Hooks
└── exam-cheat-sheet.md                                       # "If You See X, Pick Y" Decision Tables & Hard Limits
```

---

## Exam details and domain weights

| Exam Detail | Requirement |
| :--- | :--- |
| **Exam Code & Title** | **GH-600**: *Developing in Agentic AI Systems* |
| **Training Course** | **GH-600T00-A** (2 Learning Paths, 6 Modules, 52 Units) |
| **Duration & Passing Score** | **120 minutes** \| **700 / 1000** |

| Exam Domain | Weight | Primary Study Guide Modules |
| :--- | :--- | :--- |
| **1. Prepare agent architecture and SDLC processes** | **15–20%** | [Module 1](./01-foundations-of-agentic-ai.md), [Module 2](./02-designing-agent-architecture-and-sdlc-integration.md) |
| **2. Implement Tool Use and Environment Interaction** | **20–25%** | [Module 2](./02-designing-agent-architecture-and-sdlc-integration.md), [Module 3](./03-tooling-mcp-and-execution-environments.md) |
| **3. Manage Memory, State, and Execution** | **10–15%** | [Module 5](./05-memory-state-and-evaluation.md) |
| **4. Perform Evaluation, Error Analysis, and Tuning** | **15–20%** | [Module 1](./01-foundations-of-agentic-ai.md), [Module 5](./05-memory-state-and-evaluation.md) |
| **5. Orchestrate Multi-Agent Coordination** | **15–20%** | [Module 4](./04-multi-agent-systems-and-orchestration.md) |
| **6. Implement Guardrails and Accountability** | **10–15%** | [Module 1](./01-foundations-of-agentic-ai.md), [Module 6](./06-governance-guardrails-and-operations.md) |

---

## Key Exam Terms Translator

| Official Exam Term | Plain-English Meaning |
| :--- | :--- |
| **Plan → Act → Evaluate loop** | The core agent cycle: write a visible plan, edit code on an isolated branch, and verify results with automated checks and human review. |
| **System of Record** | GitHub's role in saving a permanent history of Issues, commits, Pull Requests, check logs, and artifacts. |
| **Control Plane** | GitHub's role in blocking unsafe actions using rulesets, required status checks, `CODEOWNERS`, permissions, and Environments. |
| **Contributor Model** | Treating an AI agent like a human developer whose Pull Requests must pass the exact same CI checks and code reviews. |
| **Plan-First Workflow** | High-risk workflow pattern where the agent outputs a structured plan and pauses for human approval before writing any code. |
| **GitHub Agentic Workflows (`gh aw`)** | Markdown workflows (`.md`) with YAML frontmatter that compile via `gh aw compile` into hardened `.lock.yml` Actions files. |
| **`safe-outputs`** | A frontmatter block in `gh aw` that keeps the coding agent read-only and routes PR or issue creation through a separate gated job. |
| **Model Context Protocol (MCP)** | Open standard connecting agents to external or local tools, governed by **MCP Servers**, **MCP Registries** (`v0.1`), and **Allow Lists**. |
| **Fan-Out / Fan-In Orchestration** | Running independent read-only analysis jobs in parallel (fan-out) and combining their results in a downstream `needs:` job (fan-in). |
| **External Memory** | Durable records stored outside the agent in GitHub **Issues**, **Pull Requests**, and **artifacts** (the authoritative **source of truth**). |
| **Central State Anchor** | The **Pull Request**, which ties together the implementation plan, commit progress, review comments, and status check results. |
| **Context Drift** | When an agent's ongoing commits slowly wander away from the original Issue acceptance criteria or earlier PR decisions. |
| **Risk-Based Autonomy** | Matching an agent's freedom to its blast radius and reversibility across 4 tiers: **Low**, **Medium**, **High**, and **Critical**. |
| **`preToolUse` Hook** | A deterministic rule in `.github/hooks/` that blocks destructive shell commands (`sudo`, `rm -rf /`) before the tool runs. |
| **Governance Drift / Check Drift** | When safety gates quietly weaken over time because someone renames a required status check, widens permissions, or leaves `CODEOWNERS` stale. |

> [!TIP]
> Ready for a rapid exam review? Jump straight to the [Exam GH-600 Cheat Sheet](./exam-cheat-sheet.md) for *"If you see X, pick Y"* tables, CLI commands, and hard retention limits.
