# Quickspaces Control Plane Architecture

## Overview

The **Quickspaces control plane** is the stateless orchestration layer that manages the lifecycle of ephemeral development environments. It does not execute user code, run containers, or maintain persistent workspaces. Instead, it coordinates with stateless execution nodes (runners) to provision, manage, and destroy workspaces on demand.

This document explains the design, responsibilities, trust model, and interaction patterns of the control plane. It is intended for infrastructure engineers, security reviewers, and contributors evaluating whether to self-host or operate Quickspaces.

---

## Important: On Runner Implementations

Quickspaces uses **multiple runner implementations**, each optimized for its hosting technology. This document describes the control plane's interaction with runners via a shared protocol. For a comprehensive explanation of runner design, why multiple implementations exist, and technology-specific behaviors, see [Runner Architecture and Implementation](./runner-architecture.md).

**Key principle**: The control plane has **no platform-specific logic**. It treats all runners equivalently via a shared contract. Infrastructure differences (AWS vs TrueNAS vs Proxmox) are hidden below the runner abstraction.

---

## 1. What the Control Plane Is (and Is NOT)

### Responsibilities

The control plane is **purely an orchestration and state management layer**. It:

- Authenticates users against GitHub
- Authorizes workspace operations based on GitHub permissions
- Tracks workspace desired state and actual state
- Orchestrates workspace lifecycle (create, start, stop, destroy)
- Discovers and selects runners based on availability and policy
- Enforces quotas, limits, and time-to-live (TTL) policies
- Records audit logs and emits observability signals
- Provides APIs for clients (VS Code extension, CLI, CI/CD)

### What It Does NOT Do

The control plane does not:

- Execute user code
- Run or manage Docker containers
- Provision infrastructure directly
- Store persistent workspace state
- Build or pull container images
- Host source code repositories
- Manage networking, DNS, or VPN
- Persist workspace files

### Stateless Architecture

The control plane is stateless:

- Workspace state is derived from runner state
- Workspace metadata is transient
- The control plane can be replicated and load-balanced
- If the control plane is unavailable, running workspaces continue unaffected

---

## 2. Core Responsibilities

### Authentication

The control plane uses GitHub OAuth as its identity provider:

- Users authenticate via GitHub
- GitHub access tokens are used to query GitHub API for user identity and permissions
- No separate password database

### Authorization

The control plane enforces **least-privilege access** using GitHub permissions.

- Can I create a workspace for repo `owner/repo`?
  → Check: Does my GitHub token have read access to `owner/repo`?
- Can I attach to workspace `X`?
  → Check: Do I have read access to the repo that workspace `X` was created from?
- Can I destroy workspace `X`?
  → Check: Am I the creator, or do I have admin access to the repo?
- Can I view org-level quotas?
  → Check: Am I an org admin on GitHub?

Authorization is **declarative and policy-driven**. Policies can enforce:
- Per-user quotas (max 5 concurrent workspaces)
- Per-org quotas (max 50 concurrent workspaces)
- TTL limits (max lifetime 24 hours)
- Allowed runner pools (self-hosted only, no managed runners)

### Workspace Metadata

The control plane maintains **desired state** for each workspace:

```json
{
  "id": "ws-abc123",
  "repo": {
    "owner": "quickspaces",
    "name": "quickspaces",
    "ref": "main"
  },
  "creator": {
    "github_login": "alice",
    "github_id": 12345
  },
  "desired_state": "started",
  "actual_state": "started",
  "created_at": "2026-04-05T10:00:00Z",
  "started_at": "2026-04-05T10:02:00Z",
  "ttl_seconds": 86400,
  "devcontainer_config": {
    "image": "mcr.microsoft.com/devcontainers/python:3.11",
    "features": ["docker-in-docker"],
    "forwardPorts": [8000, 3000]
  },
  "runner": {
    "id": "runner-xyz789",
    "pool": "aws-us-east-1"
  },
  "connection": {
    "vscode_remote_host": "runner-xyz789.internal",
    "vscode_remote_port": 22
  }
}
```

**Desired vs Actual State**

The control plane tracks what *should* be true (desired) and what *is* true (actual):

- Desired: `started` → Actual: `starting` → Actual: `started` (normal progression)
- Desired: `stopped` → Actual: `running` → Actual: `stopped` (drift correction)
- Desired: `destroyed` → Actual: `stopped` → Actual: `destroyed` (clean shutdown)

This allows idempotent operations and recovery from transient failures.

### Workspace Lifecycle Orchestration

The control plane is responsible for the state machine transitions:

- **Create**: Reserve a workspace ID, persist metadata, emit "workspace.created" event
- **Provision**: Select a runner, assign the workspace, emit "runner.assign"
- **Bootstrap**: Trigger runner to clone repo and bootstrap devcontainer
- **Attach**: Report connection details to VS Code extension
- **Idle Detection**: Monitor for inactivity, transition to "idle" if no activity for N minutes
- **Stop**: Signal runner to stop workspace
- **Destroy**: Release runner assignment, cleanup metadata

The control plane **does not wait or block** on long operations. Instead:

1. Set desired state to target
2. Runner observes new desired state via polling/webhooks
3. Runner executes the transition
4. Runner reports actual state back to control plane
5. Control plane emits events

### Runner Discovery and Selection

The control plane maintains a registry of available runners using a simple selection algorithm.

When selecting a runner for a workspace:

1. Filter runners by requested pool
2. Filter by available capacity
3. Filter by label constraints
4. Select runner with highest available capacity

If no runner has capacity, workspace creation fails.

---

## 3. Control Plane vs Runners: Clean Separation

### Conceptual Model

```
┌─────────────────────────────────────────────────────┐
│                  Control Plane                      │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  State Store                                │   │
│  │  - workspace metadata                       │   │
│  │  - desired state, actual state              │   │
│  │  - audit logs                               │   │
│  └─────────────────────────────────────────────┘   │
│                       │                             │
│  ┌────────────────────▼──────────────────────────┐  │
│  │  Orchestration Engine                        │  │
│  │  - lifecycle transitions                     │  │
│  │  - runner selection & dispatch               │  │
│  │  - policy enforcement                        │  │
│  │  - authorization checks                      │  │
│  └────────────┬─────────────────────────────────┘  │
│               │                                     │
│  ┌────────────▼──────────────────────────────────┐  │
│  │  API Layer                                   │  │
│  │  - REST endpoints                            │  │
│  │  - webhook ingestion                         │  │
│  │  - observability hooks                       │  │
│  └────────────┬─────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
               │
               │ gRPC / REST (control protocol)
               │
┌──────────────▼──────────────────────────────────────┐
│             Runner Pool                            │
│                                                    │
│  ┌──────────────┐  ┌──────────────┐               │
│  │   Runner A   │  │   Runner B   │  ...          │
│  │              │  │              │               │
│  │  Docker      │  │  Docker      │               │
│  │  - WS-1      │  │  - WS-2      │               │
│  │  - WS-3      │  │  - WS-4      │               │
│  │              │  │              │               │
│  └──────────────┘  └──────────────┘               │
│                                                    │
│  Each runner:                                      │
│  - runs Quickspaces agent (control channel)       │
│  - runs Docker daemon                             │
│  - pulls repos, bootstraps devcontainers         │
│  - manages workspace lifecycle locally            │
│  - reports status back to control plane           │
└────────────────────────────────────────────────────┘
```

### Separation of Concerns

| Responsibility | Control Plane | Runner |
|---|---|---|
| **State tracking** | Workspace metadata, desired vs actual | Local workspace files, container IDs |
| **Orchestration** | State transitions, scheduling | Executing transitions, running code |
| **Code execution** | Never | Always |
| **Docker** | Never | Always |
| **GitHub integration** | OAuth, permissions, repo metadata | Clone, pull, version control |
| **Networking** | Tracks connection info | Opens ports, manages SSH, sets up DNS |
| **Secrets** | Does not store user secrets | Injects secrets into containers |

---

## 4. Runner Interaction Model

### Runner Registration

Runners are stateless, replaceable execution nodes. They register with the control plane:

```
Runner startup:
1. Read config (control plane URL, auth token, pool name, capacity)
2. POST /api/runners/register with runner metadata
3. Control plane assigns runner ID
4. Runner enters idle loop
```

The runner **does not require pre-provisioning**. Any machine with Docker and the Quickspaces agent binary can become a runner.

### Authentication

Runners authenticate to the control plane using:

- **API key** (for self-hosted): A long-lived secret provisioned by the operator
- **OIDC** (for managed runners in Quickspaces Cloud): Short-lived, signed tokens from the Quickspaces identity service

The control plane never sends secrets to runners. Runners can pull secrets they need from GitHub (if the workspace creator has granted permission).

### Heartbeat and Health Checks

Runners maintain a **persistent connection** to the control plane:

```
1. Runner connects to control plane (gRPC or TCP)
2. Every 30 seconds, runner sends heartbeat:
   {
     "runner_id": "runner-xyz",
     "available_slots": 8,
     "workspaces": [
       {
         "id": "ws-abc",
         "actual_state": "started",
         "cpu_usage": "45%",
         "memory_usage": "2.5G"
       }
     ]
   }
3. Control plane marks runner as healthy
4. If heartbeat missing for >2 minutes, control plane marks runner as unhealthy
5. New workspaces are not assigned to unhealthy runners
6. Orphaned workspaces on unhealthy runners are transitioned to "unknown" state
```

### Dispatching Work to Runners

The control plane uses a **pull model** where runners poll for work:

```
1. Runner queries: GET /api/runners/{runner_id}/work
2. Control plane returns workspace assignment:
   {
     "workspace_id": "ws-abc",
     "action": "bootstrap",
     "repo": "owner/repo",
     "ref": "main",
     "devcontainer_config": { ... }
   }
3. Runner executes the assignment
4. Runner reports result: POST /api/workspaces/{ws_id}/status
```

Runners can also be notified via push (persistent connection), but pull is the default.

### Self-Hosted and Managed Runners Use the Same Protocol

Whether you're running a managed runner in Quickspaces Cloud or a self-hosted runner on your own hardware, the protocol is identical:

- Same registration flow
- Same heartbeat mechanism
- Same work dispatch API
- Same security model (OIDC for managed, API key for self-hosted)

This means you can **mix managed and self-hosted runners in the same pool** and the control plane treats them equivalently.

---

## 5. Workspace Lifecycle: State Machine

### State Diagram

```
created → provisioning → starting → started ← idle ← (resume)
                           ↓
                         running
                           ↓
                         stopped
                           ↓
                       destroyed
```

### Detailed Transitions

#### Step 1: Create Workspace

**User action:** "Click Create Workspace" in VS Code extension

**Control plane does:**

1. Validate input (repo exists, user has access, quotas allow)
2. Generate workspace ID
3. Persist metadata to state store
4. Transition desired state to `provisioning`
5. Return workspace ID to VS Code extension
6. Emit event: `workspace.created`

**Runner involvement:** None yet

**Result:** Workspace exists but no compute allocated

---

#### Step 2: Provision Execution Environment

**Trigger:** Control plane observes desired state = `provisioning`

**Control plane does:**

1. Select a runner (based on pool, capacity, labels)
2. Assign workspace to runner
3. Write assignment to state store
4. Signal runner (push or pull) with assignment Details

**Runner does:**

1. Receive assignment for workspace `ws-abc`
2. Reserve a Docker bridge network
3. Reserve a port on the host (for VS Code SSH)
4. Allocate DNS entry (workspace ID → runner host)
5. Update actual state: `provisioning`
6. Report status to control plane

**Control plane observes:**

- Runner reports `actual_state: provisioning`
- Control plane updates state store

**Result:** Workspace has a runner assigned and basic infrastructure reserved

---

#### Step 3: Bootstrap Devcontainer and Docker Compose

**Trigger:** Runner observes `desired_state: starting`

**Runner does:**

1. Clone repo (or pull from cache)
2. Checkout requested ref (branch, tag, commit)
3. Read `.devcontainer/devcontainer.json`
4. Parse Docker Compose config if present
5. Build/pull base image
6. Start containers:
   - Main devcontainer (VS Code will attach here)
   - Sidecar containers (PostgreSQL, Redis, etc.)
7. Wait for containers to be healthy
8. Set up port forwarding (localhost:8000 → container:8000)
9. Set up file mounts (workspace directory, cache volumes)
10. Update actual state: `started`
11. Report connection details to control plane:
    ```json
    {
      "workspace_id": "ws-abc",
      "actual_state": "started",
      "connection": {
        "ssh_host": "runner-xyz.internal",
        "ssh_port": 2222,
        "ssh_user": "quickspaces",
        "dns_name": "ws-abc.internal"
      }
    }
    ```

**Control plane observes:**

- Runner reports `actual_state: started` with connection details
- Control plane stores connection details
- Control plane emits event: `workspace.ready`

**VS Code extension observes:**

- Receives connection details from control plane
- Establishes SSH connection to runner
- Connects to VS Code Server running in devcontainer
- User can now edit files and run commands

**Result:** Workspace is running and developer is connected

---

#### Step 4: Active Use

**Duration:** User works for T minutes

**Control plane monitors:**

- VS Code extension sends heartbeat (activity signal)
- Control plane tracks last activity time

**Runner monitors:**

- File I/O activity
- Process creation/execution
- Network traffic
- If using SSH, terminal activity

**Result:** Workspace remains `started` as long as activity is detected

---

#### Step 5: Idle Detection and Transition to Idle

**Trigger:** No activity detected for N minutes (typically 15–30 min, configurable)

**Control plane detects:**

- Current time - last activity time > idle threshold
- Transitions desired state to `idle`

**Runner observes:**

- Receives `desired_state: idle`
- Signals VS Code extension to disconnect gracefully
- Saves any temporary state (optional, runners are ephemeral)
- Transitions to `idle` (containers still running)
- Updates actual state: `idle`

**Result:** Workspace is no longer charged for execution. Compute may be right-sized (e.g., smaller instance type on AWS).

---

#### Step 6: Resume or Destroy

**Option A: Resume (user returns)**

- VS Code extension detects resume intent
- Control plane transitions desired state to `started`
- Runner resumes from idle state
- Workspace is charged again
- User reconnects

**Option B: TTL Expiration**

- Workspace has been running for >24 hours (TTL)
- Control plane emits event: `workspace.ttl_exceeded`
- Control plane transitions desired state to `stopped`
- Runner stops containers
- Runner updates actual state: `stopped`

**Option C: Manual Destroy**

- User clicks "Destroy Workspace"
- Control plane transitions desired state to `destroyed`
- Runner stops containers, releases ports, cleanup
- Runner updates actual state: `destroyed`
- Control plane removes workspace from state store

**Result:** Compute is released, billing ends

---

## 6. GitHub-Native Design

Quickspaces uses GitHub as its source of truth for identity, permissions, and repository metadata.

### Identity and Authentication

Authentication is performed via GitHub OAuth:

- Users authenticate with their GitHub account
- The control plane receives a GitHub access token
- There is no separate user database or password system

### Authorization

Authorization is delegated to GitHub:

- To create a workspace for a repo, the user must have read access on GitHub
- Public repos are accessible to all authenticated users
- Private repos are accessible only to users with GitHub access
- Organization repos are accessible to org members

This means GitHub permissions are the source of truth. No separate ACL system exists.

### Repository Metadata

Repository metadata is queried from GitHub on-demand:

- Repo existence, default branch, clone URL are retrieved from GitHub API
- Repository metadata is not stored; it is cached briefly (<1 minute)
- Changes to the repository (rename, delete, archive) are reflected immediately in Quickspaces

### Configuration (.devcontainer)

Workspace configuration is defined in `.devcontainer/devcontainer.json` in the repository.

- This is the same format used by VS Code Devcontainer standard
- Configuration is versioned with the code
- Runners parse and apply this configuration

---

## 7. Security Model

The control plane uses GitHub OAuth for authentication and delegates authorization to GitHub permissions.

### Trust Model

```
                       GitHub
                         ↑
                    (OAuth, permissions)
                         │
┌────────────────────────┼────────────────────────┐
│                   Control Plane                 │
│                   (state tracking)              │
│        (authenticate, authorize, schedule)      │
└────────────────────────┼────────────────────────┘
                         │
              (control protocol, work dispatch)
                         │
              ┌──────────▼──────────┐
              │  Runner Pool        │
              │  (execute user code)│
              └─────────────────────┘
```

### Boundaries

**Boundary 1: Control Plane ↔ Runner**

- **Runners trust the control plane** (it's within your infrastructure)
- **Control plane does NOT trust runners** (they run user code)
- **Authentication:** Runners use API key or OIDC token (checked on every request)
- **Authorization:** Control plane validates runner is authorized for the workspace
- **Trust assumption:** Runners are compromisable (user code runs there)

**Boundary 2: External User ↔ Control Plane**

- **Users authenticate via GitHub OAuth**
- **Token is short-lived** (1 hour typical, depends on GitHub's token expiry)
- **Token is never shared with runners or other users**
- **HTTPS only** (certificate validation enforced)
- **Control plane validates token with GitHub** (every API call, or cached for 60 seconds)

**Boundary 3: Control Plane ↔ GitHub**

- **Control plane uses GitHub OAuth creds** (client ID + secret)
- **Creds are stored securely** (encrypted, access-controlled)
- **All repo queries use user's GitHub token** (not Quickspaces' creds)
- **Creds are never exposed to runners or users**

### Token Lifetimes

| Token | Lifetime | Holder | Risk |
|---|---|---|---|
| GitHub OAuth access token | 1 hour (typical) | User + Control plane (short-term) | Compromised token = can access repos for user |
| Runner API key | Long-lived (months) | Runner (in-container) | Compromised runner = can create/destroy workspaces |
| Workspace attachment token | Ephemeral (session) | VS Code extension | Session hijack = can connect to workspace |

### Least-Privilege Principles

- **GitHub tokens**: User provides token, control plane uses it only for that user's workspaces
- **Runner credentials**: Runners authenticate to control plane, not vice versa (zero implicit trust)
- **Workspace secrets**: Runners pull secrets from GitHub (user's token), not from control plane
- **Cross-workspace isolation**: One workspace cannot access another's files or services

### What If a Runner Is Compromised?

Attacker gains access to one runner.

**Damage:**
- Can read/modify files in active workspaces on that runner
- Can execute arbitrary code in those workspaces
- Can exfiltrate user secrets injected into containers
- Can modify containers (e.g., install backdoor)

**Containment:**
- Control plane is unaffected (isolated from runner)
- Other runners are unaffected
- Attacker cannot:
  - Create new workspaces (no control plane access)
  - Access workspaces on other runners
  - Escalate to GitHub (runner doesn't have GitHub creds)
- Operator can remove runner from pool, spin new one

**Recommendation:**
- Treat runners as ephemeral and replaceable
- Rotate runners regularly (e.g., weekly)
- Monitor runner for suspicious activity
- Run runners in isolated network segment

---

### What If the Control Plane Is Compromised?

Attacker gains access to control plane.

**Damage:**
- Can read workspace metadata (repo, creator, connection details)
- Can read audit logs
- Can create/destroy/pause workspaces for any user
- Can assign workspaces to attacker-controlled runners

**Containment:**
- Running workspaces are unaffected (detached from control plane)
- User code continues executing on runners
- Attacker cannot:
  - Read files in workspaces (those are on runners)
  - Execute code inside workspaces (wrong network)
  - Access GitHub (different credential)
- Operator can:
  - Revoke control plane's GitHub OAuth credentials
  - Pause all workspace creation until incident resolved
  - Audit logs (before compromise) for damage assessment

**Recommendation:**
- Treat control plane as the critical security boundary
- Enable audit logging and monitoring
- Rotate API credentials regularly
- Run in air-gapped network if possible
- Use network segmentation (control plane ≠ runners)

---

## 10. Non-Goals

The control plane intentionally does NOT:

### CI/CD

Quickspaces is for **interactive development**, not batch automation. It does not:
- Run tests on a schedule
- Build and push images
- Deploy to production
- Manage artifacts or registries

Use GitHub Actions, GitLab CI, or traditional CI/CD for those tasks.

### Kubernetes Orchestration

Quickspaces is NOT a Kubernetes abstraction layer. It does not:
- Manage Kubernetes clusters
- Schedule Pods or Deployments
- Provide Helm chart templating
- Abstract away cloud provider APIs

If you need Kubernetes, use Kubernetes (or an abstraction like Crossplane). Quickspaces is simpler and more specialized.

### Generic Job Scheduling

Quickspaces is NOT a general-purpose job scheduler. It does not:
- Run arbitrary batch jobs
- Support job priorities or queues
- Provide DAG-based workflows
- Manage long-running background services

Use Apache Airflow, Temporal, or AWS Batch for those use cases.

### Repository Hosting

Quickspaces does NOT:
- Host Git repositories
- Manage pull requests
- Provide code review

Use GitHub, GitLab, or Gitea. Quickspaces integrates with your existing repository platform.

### Long-Running Services

Workspaces are ephemeral by design. Quickspaces does NOT:
- Support persistent databases running 24/7
- Manage stateful services
- Provide backup/restore for workspace data

If you need persistent infra, run it separately (e.g., RDS for database, S3 for storage). Workspaces can *connect to* persistent services.

---

## Summary: Control Plane Design Principles

1. **Thin and Stateless**
   - Minimal state. Runners report state; control plane tracks it.
   - Easy to scale, deploy, replicate.
   - Low compute and storage cost.

2. **GitHub-Native**
   - OAuth for identity. GitHub permissions for authorization.
   - No separate user database.
   - Repo metadata and config from GitHub.

3. **Clean Separation**
   - Control plane orchestrates; runners execute.
   - User code never touches control plane.
   - Breach containment: compromise one layer doesn't compromise the other.

4. **Ephemeral by Default**
   - Workspaces are temporary. Compute released when idle.
   - Cost-optimized: pay only for active usage.
   - Enables both self-hosting and SaaS.

5. **Simple Client Interface**
   - VS Code extension is intentionally thin.
   - REST API is minimal and intuitive.
   - Desktop, web, CLI, or programmatic access all possible.

6. **Cloud-Agnostic Execution**
   - Runners can be AWS, on-prem, NAS, or Proxmox.
   - Same protocol for all backend types.
   - Self-hosters and cloud users operate with same codebase.

---

## Appendix: Quick Reference

### Key Concepts

| Term | Definition |
|---|---|
| **Control Plane** | Stateless orchestration layer. Authenticates, authorizes, manages workspace state. |
| **Runner** | Stateless execution node. Runs Docker, executes workspaces, reports status. |
| **Workspace** | One ephemeral development environment (devcontainer + docker-compose). |
| **Workspace State** | Desired (target) vs Actual (current). State machine: created → provisioning → starting → started → idle → stopped → destroyed. |
| **Devcontainer** | VS Code Dev Container spec. Defines Docker image, features, port forwarding, mounts. |
| **Pool** | Group of runners (e.g., "aws-us-east-1", "on-prem-homelab"). Control plane schedules workspaces to pools. |

### API Endpoints (Control Plane)

```
POST /api/auth/token              # Authenticate (GitHub OAuth)
GET /api/workspaces               # List user's workspaces
POST /api/workspaces              # Create workspace
GET /api/workspaces/{id}          # Get workspace details
PATCH /api/workspaces/{id}        # Update workspace (pause, resume, destroy)
DELETE /api/workspaces/{id}       # Destroy workspace
```

### Runner Endpoints (Control Plane)

```
POST /api/runners/register        # Register runner
GET /api/runners/{id}/work        # Poll for work (pull model)
POST /api/runners/{id}/heartbeat  # Send heartbeat
POST /api/workspaces/{id}/status  # Report workspace status
```

---

**This document is a living guide to the Quickspaces control plane. It evolves with the project. For latest implementation details, see the main Quickspaces repository.**
