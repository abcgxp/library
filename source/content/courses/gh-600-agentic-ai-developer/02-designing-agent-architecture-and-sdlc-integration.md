# Module 2: Designing Agent Architecture and SDLC Integration

This guide covers **Domain 1 (Prepare Agent Architecture and SDLC Processes)** and **Domain 2 (Implement Tool Use and Environment Interaction)** of Exam GH-600.

---

## What a Structured Task Contract is and why you use it

A **task contract** *(a clear definition of an agent's required inputs, expected outputs, and machine-testable success criteria)* prevents an agent from guessing what "done" looks like. Also keep two exam terms separate: **planning** *(the agent's capability to break a goal into steps)* produces a **plan** *(the saved, inspectable artifact in a PR or file that humans review)*. By default, an agent should only be assigned to SDLC stages where clear inputs and automated verification checks exist.

### How a task contract maps inputs to verifiable outputs

1. **Inputs (What the agent reads):** The Issue description, Dependabot/CVE alert details, allowed file paths, and `.github/copilot-instructions.md`.
2. **Outputs (What the agent produces):** A dedicated working branch, small commits, updated unit tests, and a Pull Request containing an inspectable **plan**.
3. **Success criteria (What proves it worked):** Enforceable CI gates that return pass/fail signals, such as unit tests (`npm test`) and security scans (**CodeQL**).

### Enforcing test and security success criteria in GitHub Actions

By default, an agent's claim that its code is secure means nothing until automated workflows verify the branch.

To verify both functional tests and CodeQL security scanning on every Pull Request targeting `main`:

```yaml
name: Verify Agent Task Contract
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  security-analysis:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - uses: github/codeql-action/analyze@v3
```

---

## Separating planning, reasoning, and execution

**Reasoning** *(the model's internal thought process)* is hidden, so your architecture must force the agent to output a visible **plan** before or alongside **execution** *(calling write tools to edit files or run commands)*. Mixing planning and execution with no inspectable plan is a major exam anti-pattern. By default, you should pick between a **Plan-First Workflow** and a **Plan-and-Execution Workflow** based on how risky or reversible the change is.

### Choosing between Plan-First and Plan-and-Execution Workflows

| Situation (Exam Clue) | Best Architectural Pattern | Why |
| :--- | :--- | :--- |
| Modifying infrastructure (`infra/`), CI workflows (`.github/workflows/`), authentication (`auth/`), or irreversible production settings | **Plan-First Workflow** *(strict separation: agent outputs a plan and pauses for human approval before writing code)* | Catches bad architectural ideas early before the agent touches sensitive files or wastes runner minutes. |
| Bumping a minor dependency version, fixing an isolated bug, or updating docs where changes are easy to revert | **Plan-and-Execution Workflow** *(combined proposal: agent submits both the plan and code diff in one PR)* | Speeds up delivery for low-to-medium risk tasks while still gating the final merge behind PR checks and review. |
| Stopping a planning agent from editing files early during the planning phase | **Enforce capability limits (read-only tool allowlist)** | Removes file-edit and write tools from the planning agent so it *cannot* modify code until handoff. |

### Enforcing a required Plan Gate in Pull Requests

By default, adding a Pull Request template (`.github/pull_request_template.md`) only shows blank headings—an agent or human can still delete them unless a workflow checks for the plan.

To standardize agent plans, create `.github/pull_request_template.md` with four required sections:

```markdown
## Goal
<!-- Link to the originating Issue and state the objective -->

## Plan
<!-- List the ordered steps and allowed file scope -->

## Risks
<!-- Document blast radius and rollback steps -->

## Validation
<!-- List automated tests and security checks run -->
```

To block any Pull Request from merging if the required plan artifact is missing, add a **Plan Gate** workflow (`.github/workflows/plan-gate.yml`) and mark `require-plan` as a required status check:

```yaml
name: Plan Gate
on:
  pull_request:
    branches: [main]

jobs:
  require-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Require plan artifact
        run: |
          if [ ! -f "Github/pull_request_template.md" ]; then
            echo "Github/pull_request_template.md is required for this pull request."
            exit 1
          fi
          echo "Github/pull_request_template.md found."
```

> [!WARNING]
> **Exam Watch-Out:** If an exam question asks for the *most reliable* way to stop **planless execution** from merging, never pick *"Ask the agent in the prompt to write a plan"* or *"Add a PR template"* by itself. You must make a **plan validation check a required status check** and **require pull request reviews**.

---

## Building reliable workflows with outputs, contexts, and defensive triggers

Multi-stage agent workflows often generate a plan in Job 1, write code in Job 2, and deploy in Job 3. Because each GitHub Actions job runs on a fresh virtual machine, files and environment variables from Job 1 vanish when Job 1 ends. By default, you pass small structured data between steps using **`$GITHUB_OUTPUT`** and across jobs using **`needs.<job_id>.outputs`**.

### Choosing between Step Outputs, Job Outputs, Artifacts, and Contexts

| Situation (Exam Clue) | Best GitHub Actions Mechanism | Why |
| :--- | :--- | :--- |
| Passing a value from Step A to Step B inside the **same job** | **Step output (`>> "$GITHUB_OUTPUT"`)** | Exposes `steps.<step_id>.outputs.<name>` to later steps in that same runner job. |
| Passing a short string (like a plan summary or risk tier) from the `plan` job to the `implement` job | **Job output (`outputs:` + `needs: plan`)** | Exposes `needs.plan.outputs.plan` across separate jobs running on different machines. |
| Passing a large JSON report, test log, or compiled bundle between jobs or for later audit | **Workflow Artifacts (`actions/upload-artifact@v4`)** | Saves whole files durably across jobs and keeps them after the workflow finishes. |
| Reading the triggering event name (`pull_request`), commit SHA, or actor username | **`github.*` context** (`github.event_name`, `github.sha`, `github.actor`) | Provides built-in read-only metadata about the workflow run and trigger event. |
| Reading reusable, non-secret configuration values set at the repo or org level | **`vars.*` context** | Reads repository or organization configuration variables across workflows. |

### Passing a structured plan from a `plan` job to an `implement` job

To capture a plan output in one job and read it in a downstream job using `needs:`:

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      plan: ${{ steps.generate_plan.outputs.plan }}
    steps:
      - id: generate_plan
        run: echo "plan=update-auth-middleware" >> "$GITHUB_OUTPUT"

  implement:
    needs: plan
    runs-on: ubuntu-latest
    steps:
      - run: echo "Executing approved plan: ${{ needs.plan.outputs.plan }}"
```

### Guarding PR-specific jobs when multiple triggers exist

By default, if a workflow has both `pull_request` and `workflow_dispatch` triggers, running it manually via `workflow_dispatch` will break any step that expects a Pull Request payload.

To prevent a PR validation job from failing when someone triggers the workflow manually, add a defensive `if: github.event_name == 'pull_request'` check:

```yaml
on:
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  validate-pr:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Validating PR #${{ github.event.pull_request.number }}"
```

---

## Controlling agent profiles, secrets boundaries, and lifecycle hooks

Even when an agent has the right prompt, you must constrain its runtime environment using three **agent profile boundaries**: its **capability boundary** *(which tools it may call)*, its **visibility boundary** *(whether users can pick it in the UI or only other agents can call it)*, and its **delegation boundary** *(which subagents it can hand work off to)*. You can also add **hooks** *(deterministic shell scripts in `.github/hooks/` that run before or after a tool call independently of the AI model)* to block dangerous actions.

### Blocking unsafe tool calls with a deterministic `.github/hooks/` rule

By default, an LLM might decide to call a destructive tool if a prompt is ambiguous or injected.

To block a `delete` tool call deterministically before it runs—regardless of what the AI model wants to do—configure a `pre-tool-use` hook in `.github/hooks/`:

```json
{
  "name": "block-high-risk-command",
  "trigger": "pre-tool-use",
  "run": "if [[ \"$TOOL\" == \"delete\" ]]; then echo 'Blocked unsafe command'; exit 1; fi"
}
```

### Choosing the right operational safeguard for tools, MCP, and deployments

| Situation (Exam Clue) | Best Operational Safeguard | Why |
| :--- | :--- | :--- |
| Connecting an agent to an external MCP server that requires an API token | **Inject the token at runtime from a scoped secret boundary and allowlist only needed tool namespaces** | Keeps credentials out of config files and stops the agent from calling dangerous wildcard MCP tools. |
| Deploying an agent-prepared release so you can roll back instantly if an error occurs | **Deploy an immutable commit SHA or release tag (never mutable `latest`)** | Guarantees you know the exact code version running in production and can revert cleanly. |
| An agent fails a required check twice in a row while trying to fix a bug | **Stop automated retries (bounded iteration) and escalate to a human with a structured summary** | Prevents infinite retry loops and gives the human reviewer the exact failure logs and attempted fixes. |
