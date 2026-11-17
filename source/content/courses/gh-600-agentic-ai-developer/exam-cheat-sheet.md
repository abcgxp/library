# Exam GH-600 Cheat Sheet ("If You See X, Pick Y")

Use this rapid-fire reference right before taking **Exam GH-600 (*GitHub Certified: Agentic AI Developer*)**.

---

## 1. Architecture, Planning & SDLC Integration (Domains 1 & 6)

| If the Exam Question Mentions... (Exam Clue) | Pick This Answer | Why |
| :--- | :--- | :--- |
| Distinguishing an **AI Agent** from an **AI Assistant** or **Automation** | **Plans steps, calls tools, acts on branches, and iterates from system feedback** | Assistants only suggest code; automation follows fixed rules without revising plans. |
| How to evaluate the `Evaluate` step of `Plan → Act → Evaluate` | **Objective system signals** *(Actions tests, CodeQL, secret scanning, human review)* | Never trust the agent's self-reported confidence score. |
| Stopping **planless execution** from merging into `main` | **Make a plan check a required status check + require PR reviews** | Prompt instructions or an unenforced PR template alone can be bypassed. |
| High-risk or irreversible changes (`.github/workflows/`, `infra/`, `auth/`, prod) | **Plan-First Workflow** *(strict separation of planning and execution)* | Requires human approval of the structured plan before any code is written. |
| Low-to-medium risk, easily reversible changes (routine dependency bump) | **Plan-and-Execution Workflow** *(combined proposal in one PR)* | Generates plan and code together, validating before merge. |
| Stopping a planning agent from editing files early | **Capability limits (give the planning agent read-only tools only)** | Enforces boundaries at the tool layer rather than relying on polite prompts. |
| Passing data between steps in the **same job** vs. across **different jobs** | **`$GITHUB_OUTPUT`** (same job) vs. **`outputs:` + `needs:<job>.outputs`** (across jobs) | Runner VMs do not share memory or disk across separate jobs. |
| Preventing a PR job from failing when triggered manually via `workflow_dispatch` | **Add `if: github.event_name == 'pull_request'`** | Guards steps that expect `github.event.pull_request` payload fields. |

---

## 2. Tooling, `gh aw`, MCP & Execution Boundaries (Domain 2)

| If the Exam Question Mentions... (Exam Clue) | Pick This Answer | Why |
| :--- | :--- | :--- |
| Running open-ended repo tasks written in Markdown under guardrails | **GitHub Agentic Workflows (`gh aw`)** (`.md` compiled to `.lock.yml`) | Compiles natural-language Markdown + YAML frontmatter into a hardened Actions lock file. |
| Keeping a `gh aw` coding agent read-only while still letting it open a PR or Issue | **`safe-outputs:` frontmatter (`create-pull-request` / `create-issue`)** | Routes proposed writes through a separate, threat-scanned, permission-gated job. |
| Secret required to authenticate `gh aw` / Copilot CLI inside GitHub Actions | **`COPILOT_GITHUB_TOKEN`** *(fine-grained PAT with `Copilot Requests: Read`)* | Grants permission (`user_copilot_requests=read`) to invoke Copilot in CI. |
| Running Copilot CLI in a non-interactive CI job without hanging | **Pass `--no-ask-user`** (`npx @github/copilot-cli -p "..." --no-ask-user`) | Prevents the CLI from pausing for interactive human keyboard prompts. |
| Restricting Copilot CLI to a low-risk commit summary in the terminal | **`copilot -p "..." --allow-tool 'shell(git)'`** | Scopes tool execution strictly to read-only Git commands. |
| Pre-approving safe discovery tools for a custom CLI agent while gating edits | **`copilot agent run CodeAgent --allow-tool 'read,search'`** | Lets the agent read and search freely while prompting before file edits or shell runs. |
| An agent opens a PR in Actions, but downstream `on: pull_request` workflows don't run | **The step used `GITHUB_TOKEN` (switch to a GitHub App token or PAT)** | Events created with built-in `GITHUB_TOKEN` never trigger new workflow runs (loop protection). |
| Enforcing which MCP servers developers and agents can use across an org | **MCP Registry + MCP Allow List set to `Registry only`** | Blocks any MCP server (including local servers!) whose ID is not in the configured registry. |
| Entering an **Azure API Center** URL as your MCP Registry URL | **Enter the base URL only (do NOT append `/v0.1/servers`)** | Appending the endpoint path breaks GitHub's registry lookup. |
| Platform support for Local vs. Remote MCP servers | **Enterprise Cloud supports both; GHES supports Local MCP servers** | GitHub Enterprise Server (GHES) uses local MCP configuration (`http://localhost:3000`). |

---

## 3. Multi-Agent Orchestration & Custom Subagents (Domain 5)

| If the Exam Question Mentions... (Exam Clue) | Pick This Answer | Why |
| :--- | :--- | :--- |
| Most effective coordination primitive for multiple agents in GitHub | **Pull requests and surrounding GitHub artifacts (branches, checks, reviews)** | Avoids hidden agent-to-agent chat channels and keeps all handoffs auditable. |
| Running a security workflow only after the main CI workflow finishes | **Sequential orchestration (`on: workflow_run` with `types: [completed]`)** | Chains separate workflows in strict sequence. |
| Running two read-only analysis agents in parallel and merging their outputs | **Fan-Out / Fan-In (`needs: [spec_analyzer, risk_reviewer]`)** | Fans out parallel analysis jobs and fans in to a single `plan_merger` job. |
| Canceling superseded runs of the same workflow on a branch without canceling other workflows | **`concurrency: group: ${{ github.workflow }}-${{ github.ref }}` + `cancel-in-progress: true`** | Using only `${{ github.ref }}` is a trap that cancels unrelated workflows on that branch! |
| Detecting textual merge conflicts with `main` automatically in CI | **`git fetch origin main && git merge --no-commit origin/main`** | Fails fast as a required status check before a human wastes time reviewing. |
| An agent PR conflicts **twice** after rebase, or a required check fails **twice** with the same error | **Stop retries and escalate to a human reviewer (2-strike rule)** | Prevents infinite non-converging loops (**flapping**). |
| Blocking an `.agent.md` profile from being called as a subagent by another agent | **`disable-model-invocation: true`** | Prevents programmatic subagent invocation in `agents:` / `handoffs:`. |
| Hiding an `.agent.md` profile from the human UI dropdown while allowing subagent calls | **`user-invocable: false`** | Keeps internal helper subagents out of the user-facing picker. |
| Repository ruleset requiring **signed commits** blocks Copilot from pushing to `copilot/*` | **Add `"GitHub Copilot"` as a bypass actor in the Repository Ruleset** | Lets Copilot push to automation branches while still requiring humans to sign commits. |
| Investigating who deleted a missing workflow evidence artifact | **Search Organization Audit Log for `action:artifact.destroy`** | Shows the actor, timestamp, and repository for deleted Actions artifacts. |
| Streaming live logs of a stalled or running agent task in the terminal | **`gh agent-task view --log --follow`** | Follows the real-time agent session log output from the GitHub CLI. |

---

## 4. Memory, State, Evaluation & Governance (Domains 3, 4 & 6)

| If the Exam Question Mentions... (Exam Clue) | Pick This Answer | Why |
| :--- | :--- | :--- |
| Authoritative **source of truth** in agentic workflows | **External memory** *(GitHub Issues, Pull Requests, and durable artifacts)* | Short-term prompt memory is ephemeral and lost when the session ends. |
| Primary **central state anchor** for tracking progress across sessions | **Pull Request** *(PR description, commits, and Checks tab)* | Connects the plan, incremental commits, and CI validation in one place. |
| Agent commits diverge from original Issue acceptance criteria or earlier decisions | **Context drift** *(fix by re-anchoring to the Issue and PR description)* | Restores alignment with the canonical source of truth. |
| Sharing state between IDE, CLI, and GitHub Actions without stale/conflicting context | **Pass canonical GitHub identifiers** *(PR number, branch name, commit SHA, run URL)* | Never copy-paste raw chat transcripts between tools. |
| Classifying why an agent failed: bad logic vs. broken CLI flag vs. outdated PR state | **Reasoning error** vs. **Tool misuse** vs. **Context issue** | Maps directly to tuning **instructions**, **tool config**, or **memory/state**. |
| Routing mandatory specialist reviews for `.github/workflows/`, `infra/`, or `/security/` | **`CODEOWNERS` + "Require review from Code Owners" enabled in Rulesets** | `CODEOWNERS` alone is advisory until enforced by Branch Protection / Rulesets. |
| Requiring human approval before production deploy or production secret access | **Protected GitHub Environment with required reviewers** | Pauses the deployment job and withholds environment secrets until approved. |
| Enforcing least privilege for `GITHUB_TOKEN` in a multi-job workflow | **`permissions: contents: read` at workflow root; elevate `write` per job** | Ensures read-only analysis jobs cannot mutate code or PRs. |
| Preventing two production deployments from running at the same time | **`concurrency: group: production` + `cancel-in-progress: true`** | Serializes production releases and cancels superseded queued runs. |
| Blocking `sudo` or `rm -rf /` deterministically before the shell tool runs | **`preToolUse` hook in `.github/hooks/` with `"permissionDecision": "deny"`** | Runs independently of the LLM's reasoning to block destructive commands. |

---

## 5. Hard Numbers, Retention Limits & Cadences to Memorize

| Metric / Setting | Official Exam Value |
| :--- | :--- |
| **Default GitHub Actions log & artifact retention** | **90 days** |
| **Configurable retention range (Public repositories)** | **1 to 90 days** |
| **Configurable retention maximum (Private repositories)** | **Up to 400 days** |
| **Manual artifact deletion in the Actions tab** | **Permanent** (cannot be restored; audited via `action:artifact.destroy`) |
| **Multi-agent retry / conflict escalation threshold** | **2 failures** (escalate after 2 merge conflicts or 2 identical check failures) |
| **Parallel agent session support by mode** | **Copilot in the cloud:** Yes \| **Copilot CLI:** Yes \| **Local IDE:** No (serial only) |
| **Required MCP Registry v0.1 endpoints** | `GET /v0.1/servers`, `GET /v0.1/servers/{serverName}/versions/latest`, `GET /v0.1/servers/{serverName}/versions/{version}` |
| **Governance review cadence** | **Weekly:** Failed runs/violations \| **Monthly:** Permissions & secrets \| **Quarterly:** Rulesets, `CODEOWNERS` & environments |
