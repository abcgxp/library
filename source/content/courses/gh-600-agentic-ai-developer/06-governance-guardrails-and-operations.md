# Module 6: Governance, Guardrails, and Operations

This guide covers **Domain 6 (Implement Guardrails and Accountability)** as well as operational governance topics from **Domains 1, 2, and 5** of Exam GH-600.

---

## What Risk-Based Autonomy is and why you use it

**Risk-based autonomy** *(matching how much freedom an agent gets to how much damage its action could cause)* lets low-risk tasks run fast while keeping high-risk changes behind human approval. Without clear **action boundaries** *(hard limits on which files, tools, and environments an agent can touch)*, an agent could rewrite security rules or deploy broken code straight to production. By default, you should start agents at low autonomy and expand access in phases as your guardrails prove reliable.

### How the 4-Phase Progressive Autonomy Rollout works

```text
Phase 1: Propose Only       ──► Agent opens branches & PRs; humans review and merge all changes
Phase 2: Low-Risk Auto      ──► Agent merges low-risk PRs (docs/formatting) after required checks pass
Phase 3: Staging Execution  ──► Agent deploys automatically to staging environments with guardrails
Phase 4: Gated Production   ──► Agent deploys to production strictly behind Environment approval gates
```

### Choosing the right autonomy level and governance control by risk tier

| Situation (Exam Clue) | Autonomy Level & Risk Tier | Required GitHub Control |
| :--- | :--- | :--- |
| Summarizing commits, classifying issues, or inspecting logs without changing files | **Read-only autonomy** (Low Risk) | Read-only token permissions (`contents: read`) and read-only tool allowlists. |
| Fixing typos in documentation or formatting code where failures are easy to revert | **Execute with guardrails** (Low Risk) | Automated checks with optional auto-merge once all status checks pass. |
| Updating package dependencies or refactoring application code | **Propose-only autonomy** (Medium Risk) | Pull Request + **Required status checks** + standard code review. |
| Editing *"small diff, big consequence"* paths: `.github/workflows/`, `infra/`, or `security/` | **Human-authorized execution** (High Risk) | **`CODEOWNERS` review** + explicit human approval + strict policy checks. |
| Deploying to production or reading production secrets | **Human-authorized execution** (Critical Risk) | **Protected GitHub Environment** with required reviewers and audit evidence. |

### Routing workflow jobs dynamically by machine-readable risk level

By default, prose risk notes inside a PR comment cannot stop a workflow job from running.

To pause high-risk agent changes for human approval while letting low-risk plans run automatically, have your planning step output a JSON file (`out/plan.json` with `{"risk": "low"}`) and route downstream jobs with `if:` conditions and a protected `environment`:

```yaml
name: Risk-Routed Agent Execution
on: workflow_dispatch

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      risk: ${{ steps.read.outputs.risk }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: plan
          path: out
      - id: read
        run: echo "risk=$(jq -r .risk out/plan.json)" >> "$GITHUB_OUTPUT"

  apply_auto:
    needs: plan
    if: ${{ needs.plan.outputs.risk == 'low' }}
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/apply-changes.sh

  apply_with_approval:
    needs: plan
    if: ${{ needs.plan.outputs.risk != 'low' }}
    environment: approval-required
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/apply-changes.sh
```

> [!WARNING]
> **Exam Watch-Out:** Any edit to `.github/workflows/`, `infra/`, or `security/` is **always High Risk**, even if it is a 1-line change. Never allow an agent to auto-merge changes to workflow files without human `CODEOWNERS` approval.

---

## Enforcing governance with non-bypassable GitHub controls

Writing rules in `README.md` or system prompts is an anti-pattern called **"controls in docs only"** *(relying on polite instructions instead of platform locks that actually block bad actions)*. To make governance **non-bypassable** *(impossible for an agent to skip, even if it hallucinates or misbehaves)*, you must enforce rules across three layers:

1. **Pre-action constraints:** Restrict `GITHUB_TOKEN` permissions and tool allowlists before the agent runs.
2. **In-action validation:** Enforce required status checks, local CLI hooks, and guardrail workflows during execution.
3. **Post-action traceability:** Upload durable artifacts and audit logs after execution.

### Routing mandatory domain reviews with `CODEOWNERS`

By default, anyone with write access can review a pull request unless you map sensitive folders to specific teams in a `CODEOWNERS` file.

To require the security and platform teams to review agent edits to workflows, infrastructure, and security files, create `.github/CODEOWNERS`:

```text
*                     @core-team
/security/            @security-team
/infra/               @platform-team
/.github/workflows/   @platform-team
```

> [!WARNING]
> **Exam Watch-Out:** Creating a `CODEOWNERS` file by itself only *requests* a review—it does not block merging! To make it an actual gate, you must enable **"Require review from Code Owners"** inside your **Branch Protection rule** or **Repository Ruleset**.

### Blocking dangerous shell commands with a Copilot CLI `preToolUse` hook

By default, an agent with bash access can attempt any shell command available in its runner environment.

To block destructive commands (`sudo`, `rm -rf /`, or piping `curl` straight to `bash`) before the shell tool executes them, configure a `preToolUse` hook in your repository hook policy:

```json
{
  "hook": "preToolUse",
  "tools": ["shell(bash)"],
  "rules": [
    { "match": "sudo", "permissionDecision": "deny" },
    { "match": "rm -rf /", "permissionDecision": "deny" },
    { "match": "curl .*\\|\\s*bash", "permissionDecision": "deny" }
  ]
}
```

### Blocking bot self-approval with a guardrail workflow

By default, you should never let an autonomous agent open and approve its own pull request without human oversight.

To fail a check if `github-copilot[bot]` owns the PR or if no human `User` has approved it, add a guardrail workflow using `actions/github-script@v7`:

```yaml
name: Guardrail - Require Human Approval
on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  verify_human_oversight:
    runs-on: ubuntu-latest
    steps:
      - name: Block bot-owned PRs
        if: ${{ github.event.pull_request.user.login == 'github-copilot[bot]' }}
        run: |
          echo "A human must own this pull request."
          exit 1

      - name: Verify at least one human approval
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            const pull_number = context.payload.pull_request.number;
            const { data: reviews } = await github.rest.pulls.listReviews({
              owner, repo, pull_number
            });
            const hasHumanApproval = reviews.some(r =>
              r.state === "APPROVED" &&
              r.user &&
              r.user.type === "User" &&
              r.user.login !== "github-copilot[bot]"
            );
            if (!hasHumanApproval) {
              core.setFailed("At least one human user approval is required.");
            }
```

---

## Designing human-in-the-loop workflows and least-privilege access

**Human-in-the-loop (HITL)** *(pausing automation at high-impact moments so a person can verify and approve the action)* keeps humans accountable without slowing down routine coding. If you ask humans to approve every minor formatting step, you cause **rubber-stamping** *(reviewers blindly clicking "Approve" because they are tired of too many pop-ups)*. Pair targeted human approvals with **least privilege** *(giving each workflow job and CLI session only the bare minimum permissions required for its task)*.

### Scoping Copilot CLI tool permissions from the command line

By default, granting an agent unrestricted terminal tools increases the blast radius if it misinterprets a prompt.

To let Copilot CLI summarize recent commits using only read-only Git commands:

```bash
copilot -p "Summarize the last 5 commits" --allow-tool 'shell(git)'
```

To run a custom agent (`CodeAgent`) with pre-approved file reading and search while still requiring human confirmation for file edits or shell commands:

```bash
copilot agent run CodeAgent --allow-tool 'read,search'
```

### Enforcing job-level least privilege for `GITHUB_TOKEN`

By default, setting broad write permissions at the top of a workflow file gives every job—including untrusted test or lint steps—the power to modify your repository.

To enforce least privilege, set `contents: read` at the workflow root and grant `contents: write` and `pull-requests: write` only on the specific job that creates or updates pull requests:

```yaml
name: Least-Privilege Agent Workflow
on: workflow_dispatch

# 1. Default to read-only across the entire workflow
permissions:
  contents: read

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/analyze-repo.sh

  update_artifacts:
    needs: analyze
    runs-on: ubuntu-latest
    # 2. Elevate permissions ONLY on the job that mutates state
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/create-agent-pr.sh
```

### Preventing overlapping production deployments with `concurrency`

By default, if two workflows trigger back-to-back, GitHub Actions can run both production deployments at the same time and corrupt your environment state.

To gate production deployments behind human approval and automatically cancel older queued runs when a newer run starts, combine `environment: production` with `concurrency`:

```yaml
jobs:
  deploy_prod:
    runs-on: ubuntu-latest
    environment:
      name: production
    concurrency:
      group: production
      cancel-in-progress: true
    steps:
      - run: ./scripts/deploy.sh
```

---

## Maintaining auditability and preventing governance drift

**Evidence-first governance** *(treating any agent run without saved proof of what it planned, tested, and changed as a failed run)* ensures you can audit every autonomous action later. Over time, teams also face **governance drift** *(when safety controls quietly weaken because someone renames a required status check, widens token permissions, or leaves `CODEOWNERS` outdated)*. By default, console logs alone are not enough for high-risk actions—your workflows must upload structured evidence artifacts.

### Uploading durable execution evidence in a workflow

By default, files generated inside a runner disappear as soon as the job finishes.

To save a machine-readable execution report (`report.json`) as a durable workflow artifact for auditing, use `actions/upload-artifact@v4`:

```yaml
      - name: Upload durable execution evidence
        uses: actions/upload-artifact@v4
        with:
          name: execution-report
          path: report.json
```

### Choosing the right mitigation for common governance anti-patterns

| Governance Anti-Pattern (Exam Clue) | What Breaks | Best GitHub Fix |
| :--- | :--- | :--- |
| **Unbounded autonomy** *(agent deploys to production or edits workflows without approval)* | Bad changes hit production before humans notice. | Add **Protected Environments** with required reviewers and **Repository Rulesets**. |
| **Excess permissions** *(workflow uses write-all `GITHUB_TOKEN` and repo-wide prod secrets)* | A compromised or confused analysis step can overwrite code or leak prod keys. | Set `contents: read` at the workflow root, elevate per job, and scope secrets to the `production` **Environment**. |
| **Missing audit trail** *(agent logs only to console output with no saved artifacts)* | You cannot prove what plan the agent followed during a security incident. | Enforce **evidence-first workflows** that upload `plan.json` and `report.json` via `actions/upload-artifact@v4`. |
| **Bypass paths exist** *(agents or admins can push straight to `main` or skip checks)* | Branch rules become optional when an agent pushes directly. | Enable **Rulesets** that block direct pushes and require status checks for all actors. |
| **Rubber-stamping** *(humans approve 50 tiny agent checks a day without reading them)* | Reviewers miss real bugs due to approval fatigue. | Automate low-risk checks, attach summary evidence artifacts to PRs, and save human gates for **merge, prod deploy, and secret access**. |
| **Check drift / Governance drift** *(someone renames a CI job and the required check stops matching)* | PRs merge without validation or get permanently stuck waiting for an old job name. | Treat renaming or deleting required status checks as a **High-Risk** change and audit rulesets on a regular schedule. |

> [!IMPORTANT]
> **Rule of Thumb (Governance Review Cadence & Failure Handling):**
> - **Weekly:** Review failed agent workflow runs and repeated policy violations.
> - **Monthly:** Audit workflow `permissions:` blocks and secret scopes.
> - **Quarterly:** Audit repository rulesets, `CODEOWNERS`, environment reviewers, and artifact retention policies.
> - **Security Failures:** Use **bounded retries** only for temporary network glitches. **Never automatically retry a security check failure**—stop and escalate to a human immediately.
