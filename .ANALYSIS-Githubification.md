# .ANALYSIS-Githubification.md

## How `githubification-NemoClaw` Could Become a GitHub Action Based Mechanism

---

## 1. What This Repo Is Today

[NemoClaw](https://github.com/NVIDIA/NemoClaw) is NVIDIA's reference stack for running [OpenClaw](https://openclaw.ai) agents safely inside [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) sandboxes. It is not an AI agent. It is **the security infrastructure that makes AI agents safe to run unattended**.

The repo is organized as a dual-language system:

| Layer | Technology | Purpose |
|---|---|---|
| **TypeScript CLI plugin** (`nemoclaw/`) | Node.js 22 | User-facing commands: `onboard`, `connect`, `status`, `migrate`, `eject`, `logs` |
| **Python blueprint** (`nemoclaw-blueprint/`) | Python 3.11 + PyYAML | Orchestration: sandbox lifecycle, inference routing, policy application |
| **OpenShell runtime** | External binary | Enforcement: Landlock, seccomp, network namespaces, inference gateway |

The four stages of the blueprint lifecycle are:

```
resolve artifact → verify digest → plan resources → apply through OpenShell CLI
```

Currently NemoClaw runs **always-on on a host machine** — it is never triggered by a GitHub event. The `nemoclaw` CLI is the only entry point. GitHub is used only for standard open source project management (issues, PRs, CI).

**Githubification status**: Pre-Githubification. Strategy: Not yet determined.

---

## 2. What Githubification Means

Per [japer-technology/githubification](https://github.com/japer-technology/githubification):

> *Githubification is the act of converting a repository into GitHub-as-infrastructure. Instead of cloning the repo and running the software elsewhere, the repo becomes something that runs on GitHub itself via GitHub Actions.*

The reference pattern, established by [github-minimum-intelligence](https://github.com/japer-technology/github-minimum-intelligence), maps four GitHub primitives:

| GitHub Primitive | Role |
|---|---|
| **GitHub Actions** | Compute — the runtime that executes the agent |
| **Git** | Memory — state, history, and configuration committed to the repo |
| **GitHub Issues** | Interface — conversation threads, command triggers |
| **GitHub Secrets** | Credentials — API keys, provider tokens |

For most repos, this mapping is sufficient. For NemoClaw, it is necessary but insufficient — because NemoClaw's value proposition is **security**, not execution.

---

## 3. The Core Githubification Challenge

> **When a system's primary value is security and isolation, Githubification must preserve those guarantees — or it destroys the reason the system exists.**

NemoClaw's five security pillars each present a distinct challenge on ephemeral GitHub Actions runners:

| NemoClaw Capability | Githubification Challenge |
|---|---|
| **Landlock + seccomp sandbox** | GitHub Actions runners provide container-level isolation (job boundaries), but not per-path filesystem policy. The same enforcement granularity requires running inside a Docker container with custom seccomp profiles. |
| **Network namespace isolation** | GitHub Actions has no native network namespace. Outbound traffic is unrestricted unless iptables rules or a userspace proxy are applied at workflow start. |
| **Declarative YAML network policy** | `openclaw-sandbox.yaml` lists every allowed endpoint. On an ephemeral runner, enforcement requires a policy enforcement step that applies and tears down rules within the workflow's lifetime. |
| **Inference routing through OpenShell gateway** | On a persistent host, OpenShell intercepts model calls transparently. On an ephemeral runner, the gateway must be started, configured, and torn down within the workflow lifetime — or replaced by a GitHub Actions-native proxy step. |
| **Blueprint digest verification** | The blueprint artifact is immutable and digest-verified before execution. This supply chain safety mechanism must survive the transition to Actions — meaning the blueprint must be checked out from a pinned ref or downloaded from a verified release. |

The conclusion: a partial Githubification that runs OpenClaw on a GitHub Actions runner without the sandbox, policy engine, and inference gateway has not Githubified NemoClaw — it has bypassed it.

---

## 4. What Githubification Unlocks

Before designing the architecture, it is worth stating what NemoClaw-as-GitHub-Action would make possible that the host-based model cannot:

1. **Zero local infrastructure** — No machine needs OpenShell installed. Operators trigger sandbox operations from GitHub Issues or workflow dispatch.
2. **Auditable operations** — Every sandbox plan, apply, and rollback is a committed workflow run with full logs, associated to a git ref.
3. **PR-gated policy changes** — Network policy YAML (`openclaw-sandbox.yaml`) is reviewed through pull requests before it governs any sandbox.
4. **Issue-driven lifecycle** — Open an issue to provision a sandbox. Comment to change policy. Close the issue to destroy the sandbox.
5. **Blueprint version control** — The `blueprint.yaml` pinned digest is a committed file. Rolling back to a previous blueprint version is a git revert.
6. **Ephemeral-by-default** — Each GitHub Actions job is a fresh runner. The sandbox's ephemeral nature aligns naturally with this model instead of fighting it.

---

## 5. Proposed Architecture

The Githubified NemoClaw maps its existing components to GitHub primitives as follows:

```
GitHub Issue (command trigger)
    │
    ▼
GitHub Actions Workflow (compute runtime)
    │
    ├── Step: Checkout blueprint at pinned ref + verify digest
    ├── Step: Apply network policy (iptables / tproxy)
    ├── Step: Start inference proxy (OpenShell gateway OR lightweight substitute)
    ├── Step: Run blueprint runner.py (plan → apply → status → rollback)
    ├── Step: Commit run state to .nemoclaw-state/ (git memory)
    └── Step: Post result as Issue comment
         │
         ▼
         Git commit (state saved) + Issue comment (user sees result)
```

### GitHub Primitive Mapping

| NemoClaw Component | GitHub Primitive |
|---|---|
| `nemoclaw onboard` wizard | Workflow dispatch with `inputs:` (provider, model, endpoint, profile) |
| `nemoclaw <name> status` | Workflow triggered by issue comment `/nemoclaw status` |
| `blueprint.yaml` | Committed file in repo, pinned version tracked in git |
| Blueprint artifact fetch + digest verify | `actions/checkout` at tagged release + `sha256sum` step |
| `openclaw-sandbox.yaml` policy | Committed YAML, enforced by iptables step in workflow |
| `runner.py plan` / `apply` / `rollback` | Steps in a single Actions job |
| `~/.nemoclaw/state/runs/` | `.nemoclaw-state/` directory committed to repo |
| `NVIDIA_API_KEY` / `OPENAI_API_KEY` | GitHub Repository Secrets |
| OpenClaw TUI / CLI | Issues interface (comment-driven) |

---

## 6. Workflow Design

### 6.1 Trigger Patterns

Three trigger patterns cover the full NemoClaw lifecycle:

#### Pattern A — Workflow Dispatch (Initial Onboarding)

```yaml
on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Blueprint action'
        required: true
        type: choice
        options: [plan, apply, status, rollback]
      profile:
        description: 'Inference profile'
        default: 'default'
        type: choice
        options: [default, ncp, nim-local, vllm]
      run_id:
        description: 'Run ID (for status/rollback)'
        required: false
```

#### Pattern B — Issue Comment (Conversational Lifecycle)

```yaml
on:
  issue_comment:
    types: [created]
```

Recognized commands parsed from issue body:
- `/nemoclaw plan [--profile default]`
- `/nemoclaw apply [--profile default]`
- `/nemoclaw status [--run-id nc-20260324-...]`
- `/nemoclaw rollback --run-id nc-20260324-...`
- `/nemoclaw policy show`
- `/nemoclaw policy update <preset>`

#### Pattern C — Pull Request (Policy Review Gate)

```yaml
on:
  pull_request:
    paths:
      - 'nemoclaw-blueprint/policies/**'
      - 'nemoclaw-blueprint/blueprint.yaml'
```

Changes to policy YAML or blueprint definition trigger a `plan --dry-run` validation. Policy changes cannot land without a passing PR check.

### 6.2 Job Structure

```yaml
jobs:
  nemoclaw-action:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # commit run state
      issues: write          # post result comment
    steps:
      - name: Authorize sender
        # Only repo owners, members, collaborators may trigger
        # Same pattern as github-minimum-intelligence

      - name: Checkout at pinned blueprint ref
        uses: actions/checkout@v4
        with:
          ref: ${{ env.BLUEPRINT_REF }}

      - name: Verify blueprint digest
        run: |
          ACTUAL=$(sha256sum nemoclaw-blueprint/blueprint.yaml | awk '{print $1}')
          EXPECTED=$(grep '^digest:' nemoclaw-blueprint/blueprint.yaml | awk '{print $2}')
          [ "$ACTUAL" = "$EXPECTED" ] || { echo "::error::Blueprint digest mismatch"; exit 1; }

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install blueprint dependencies
        run: |
          cd nemoclaw-blueprint
          pip install pyyaml

      - name: Apply network policy
        # Enforce declarative YAML policy via iptables
        # All outbound traffic blocked except policy-listed endpoints
        run: bash .github/scripts/apply-network-policy.sh

      - name: Start inference proxy
        # Lightweight substitute for OpenShell gateway on ephemeral runner
        # Routes inference calls to provider endpoint with credential injection
        env:
          NVIDIA_API_KEY: ${{ secrets.NVIDIA_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: bash .github/scripts/start-inference-proxy.sh

      - name: Run blueprint action
        id: blueprint
        env:
          NEMOCLAW_BLUEPRINT_PATH: nemoclaw-blueprint
        run: |
          python3 nemoclaw-blueprint/orchestrator/runner.py \
            ${{ env.ACTION }} \
            --profile ${{ env.PROFILE }} \
            ${{ env.RUN_ID && format('--run-id {0}', env.RUN_ID) || '' }}

      - name: Commit run state
        run: |
          git config user.name "nemoclaw[bot]"
          git config user.email "nemoclaw@users.noreply.github.com"
          git add .nemoclaw-state/
          git diff --staged --quiet || git commit -m "nemoclaw: run state ${{ env.RUN_ID }}"
          git push

      - name: Post result comment
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `👍 NemoClaw \`${{ env.ACTION }}\` complete.\n\n\`\`\`\n${{ steps.blueprint.outputs.result }}\`\`\``
            })
```

---

## 7. Security Preservation Strategy

The primary risk in Githubification is losing the security guarantees that make NemoClaw valuable. The following strategy preserves them at each layer:

### 7.1 Filesystem Isolation → Job Boundary + Docker Step

GitHub Actions jobs run in isolated VMs. For per-path filesystem enforcement equivalent to Landlock, the blueprint runner executes inside a Docker container with a custom seccomp profile:

```yaml
- name: Run blueprint in isolated container
  run: |
    docker run \
      --security-opt seccomp=nemoclaw-blueprint/policies/seccomp.json \
      --read-only \
      --tmpfs /tmp \
      --volume ${{ github.workspace }}/nemoclaw-blueprint:/blueprint:ro \
      python:3.11-slim \
      python3 /blueprint/orchestrator/runner.py ${{ env.ACTION }}
```

### 7.2 Network Namespace → iptables Policy Enforcement

The `openclaw-sandbox.yaml` policy lists every permitted outbound endpoint. A setup script translates this to iptables rules at workflow start:

```bash
# Default: block all outbound
iptables -P OUTPUT DROP
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow only policy-listed endpoints
for endpoint in $(parse_policy_endpoints openclaw-sandbox.yaml); do
  iptables -A OUTPUT -d "$endpoint" -j ACCEPT
done
```

This is functionally equivalent to the OpenShell network namespace for the duration of the job.

### 7.3 Inference Routing → Credential-Injecting Proxy Step

The OpenShell inference gateway intercepts model calls and routes them to the configured provider. On an ephemeral runner, a lightweight HTTP proxy step achieves the same result:

- The proxy binds to `localhost:8080` and forwards to the configured inference endpoint
- Provider credentials are injected from GitHub Secrets, never exposed to the blueprint runner
- The blueprint runner's `OPENAI_BASE_URL` is set to `http://localhost:8080/v1`

### 7.4 Blueprint Digest Verification → sha256sum Step

The existing digest verification logic in `runner.py` depends on the blueprint being fetched from a registry. On GitHub Actions, the blueprint is checked out from the repository at a tagged ref. The digest verification step runs `sha256sum` against `blueprint.yaml` before any action is taken.

### 7.5 Authorization → Sender Check

Following the `github-minimum-intelligence` pattern, the workflow checks that the trigger actor is a repository owner, member, or collaborator before running any blueprint action:

```javascript
const { data: permission } = await github.rest.repos.getCollaboratorPermissionLevel({
  owner: context.repo.owner,
  repo: context.repo.repo,
  username: context.actor
});
if (!['admin', 'write', 'maintain'].includes(permission.permission)) {
  // Post 👎 comment and exit
}
```

---

## 8. State Management

All run state is committed to the repository under `.nemoclaw-state/`, mirroring the `~/.nemoclaw/state/` structure from the host-based model:

```
.nemoclaw-state/
  runs/
    nc-20260324-143022-a1b2c3d4/
      plan.json          # Profile, sandbox config, inference config
      rolled_back        # Written by rollback action
  issue-map.json         # Maps issue numbers to run IDs (parallel to GMI pattern)
```

This gives the Githubified NemoClaw full recall of every prior run — a git-tracked audit trail that the host-based `~/.nemoclaw/` directory cannot provide across machines or operators.

---

## 9. The `.GITNEMOCLAW/` Folder Convention

Following the conventions established in [githubification](https://github.com/japer-technology/githubification) for the wrapping strategy (as used in OpenClaw Githubification), the Githubification layer lives in `.GITNEMOCLAW/`:

```
.GITNEMOCLAW/
  README.md              # Githubification overview and usage
  AGENTS.md              # Agent identity (if personality is desired)
  lifecycle/
    entrypoint.ts        # Issue/dispatch trigger handler (analogous to GMI lifecycle/agent.ts)
    authorize.ts         # Sender authorization check
    parse-command.ts     # Issue comment command parser
    post-result.ts       # Issue comment result poster
  scripts/
    apply-network-policy.sh   # Translates openclaw-sandbox.yaml → iptables rules
    start-inference-proxy.sh  # Starts credential-injecting inference proxy
  VERSION                # Githubification layer version (separate from NemoClaw version)
```

The `.GITNEMOCLAW/` folder does not modify any file in `nemoclaw/` or `nemoclaw-blueprint/`. The Githubification layer wraps the existing system without touching it — upstream NemoClaw changes can be pulled without conflicts.

---

## 10. What Is Gained and What Is Lost

### Gained

| Capability | Description |
|---|---|
| **Zero host infrastructure** | No machine needs OpenShell or Node.js. GitHub is the runtime. |
| **Audit trail** | Every plan, apply, and rollback is a git-committed event with full logs. |
| **Policy-as-code review** | Network policy changes go through PR review before taking effect. |
| **Issue-driven operations** | Non-technical operators can trigger blueprint actions from GitHub Issues. |
| **Ephemeral-by-default** | Each run starts from a clean state. No accumulated host drift. |
| **Multi-operator** | Any authorized collaborator can trigger actions without SSH access to a server. |

### Lost / Traded

| Capability | Trade-off |
|---|---|
| **Always-on sandbox** | GitHub Actions is event-driven and ephemeral. There is no persistent always-on OpenClaw instance. A Githubified NemoClaw is a lifecycle manager, not a persistent sandbox. |
| **OpenShell TUI** | The interactive approval TUI (`openshell term`) cannot run in a headless CI environment. Policy approvals must be committed to the repo instead. |
| **Native Landlock enforcement** | Landlock is a Linux kernel feature unavailable in GitHub Actions runners. The seccomp+Docker substitute provides comparable but not identical isolation. |
| **Real-time network interception** | OpenShell intercepts connections at the network namespace level. iptables enforcement on a runner is applied before execution, not interception-on-demand. |
| **Sub-minute response time** | GitHub Actions runner startup adds latency. An issue comment may take 30–90 seconds to receive a response, versus sub-second for a local `nemoclaw status` call. |

---

## 11. Implementation Roadmap

### Phase 1 — Foundation (Wrapping, No Security Enforcement)

1. Create `.GITNEMOCLAW/` folder structure
2. Add `.github/workflows/nemoclaw-action.yaml` with workflow dispatch trigger
3. Wire `runner.py plan` and `runner.py apply` as workflow steps (no network policy yet)
4. Commit run state to `.nemoclaw-state/` and push
5. Validate the blueprint lifecycle runs end-to-end on an Actions runner

### Phase 2 — Authorization and Issue Interface

1. Add sender authorization check (owner/member/collaborator only)
2. Add issue comment trigger with command parser (`/nemoclaw plan`, `/nemoclaw status`, etc.)
3. Add issue-to-run-id mapping in `.nemoclaw-state/issue-map.json`
4. Post 🚀 / 👍 / 👎 lifecycle comments (same pattern as github-minimum-intelligence)

### Phase 3 — Security Enforcement

1. Implement `scripts/apply-network-policy.sh` — translate `openclaw-sandbox.yaml` to iptables rules
2. Implement `scripts/start-inference-proxy.sh` — credential-injecting inference proxy
3. Add Docker-based isolation step for blueprint runner with custom seccomp profile
4. Add blueprint digest verification step
5. Add PR-gated policy validation workflow (triggers on changes to `policies/**`)

### Phase 4 — Policy-as-Code Review Gate

1. Add workflow that runs `runner.py plan --dry-run` on any PR touching `blueprint.yaml` or `policies/**`
2. Add policy preset selection via issue comment (`/nemoclaw policy use pypi-preset`)
3. Commit policy change results to run state for audit

### Phase 5 — Personality and Governance (Optional)

1. Add `AGENTS.md` with NemoClaw agent identity
2. Add a `/nemoclaw hatch` command (following GMI's hatching pattern) for guided configuration
3. Add DEFCON-style readiness levels (GMI pattern) for policy strictness

---

## 12. Relationship to the Githubification Ecosystem

NemoClaw sits at a unique position in the [githubification winners ranking](https://github.com/japer-technology/githubification/blob/main/.githubification/winners.md) (#19 of 20): it is the only subject that is **security infrastructure rather than an agent**. This means:

- The Githubification strategy is **Wrapping** (same as OpenClaw #5) — the existing `nemoclaw-blueprint/` and `nemoclaw/` systems are not modified
- The value delivered through Githubification is **lifecycle management and audit**, not always-on agent execution
- The security gap (ephemeral runners vs. OpenShell) is the primary design constraint, not a secondary concern
- A fully Githubified NemoClaw is a **CI/CD system for AI agent sandbox policy**, not a replacement for the OpenShell runtime

The closest analogue in the ecosystem is **OpenClaw** (github-openclaw, #5): a complex production-grade system Githubified without modifying a single line of its source, using a wrapper folder. NemoClaw should follow the same pattern — `.GITNEMOCLAW/` alongside untouched `nemoclaw/` and `nemoclaw-blueprint/`.

---

## 13. Summary

| Dimension | Current State | Githubified State |
|---|---|---|
| **Trigger** | `nemoclaw` CLI on host | GitHub Issue comment or workflow dispatch |
| **Runtime** | OpenShell on persistent host | GitHub Actions ephemeral runner |
| **Sandbox** | Landlock + seccomp + netns via OpenShell | Docker + seccomp + iptables policy enforcement |
| **Inference** | OpenShell gateway, transparent interception | Credential-injecting proxy step |
| **State** | `~/.nemoclaw/state/` on host | `.nemoclaw-state/` committed to repo |
| **Policy** | YAML file on host, applied at runtime | YAML file in repo, PR-reviewed, applied at job start |
| **Audit** | Host filesystem logs | Git history + Actions run logs |
| **Authorization** | Host access (SSH / local shell) | GitHub collaborator permission check |
| **Always-on** | Yes (persistent sandbox) | No (ephemeral, event-triggered) |
| **Code changes** | N/A | Zero changes to `nemoclaw/` or `nemoclaw-blueprint/` |

The Githubification of NemoClaw converts it from **a tool you run on a machine** into **a mechanism that runs on GitHub** — preserving its security intent through GitHub-native equivalents, gaining auditability and zero-infrastructure operation, and accepting the trade-off that always-on sandboxed execution remains an on-host concern.
