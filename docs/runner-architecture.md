# Runner Architecture

## Overview

Quickspaces uses **multiple runner implementations**, each dedicated to a specific hosting technology. Each runner is optimized for its underlying platform's execution model and cost structure.

The control plane is platform-agnostic. Runners implement a shared contract for registration, heartbeat, work dispatch, and status reporting. This separation allows the control plane to remain free of platform-specific logic.

## Runner Implementations

Quickspaces defines two primary runner implementations:

1. **AWS EC2 Runner** — Ephemeral, one workspace per instance
2. **TrueNAS SCALE Runner** — Long-lived, multiple workspaces per runner

Each implementation has a distinct lifecycle model, cost model, and scaling behavior aligned to its underlying platform.

---

## Shared Runner Contract

All runners implement a shared protocol with the control plane. This protocol is identical across all runner implementations.

### Registration

Runners register with the control plane at startup:

```
POST /api/runners/register
{
  "id": "runner-abc-123",
  "pool": "aws-us-east-1",
  "implementation": "aws-ec2",
  "capacity": 10,
  "available_slots": 8,
  "os": "linux",
  "arch": "x86_64",
  "docker_version": "24.0.6",
  "labels": ["gpu=false", "ssd=true"]
}
```

### Heartbeat

Runners maintain a continuous heartbeat with the control plane every 30 seconds:

```
{
  "runner_id": "runner-abc-123",
  "available_slots": 7,
  "workspaces": [
    {
      "id": "ws-123",
      "actual_state": "started",
      "cpu_usage": "45%",
      "memory_usage": "2.5G"
    }
  ]
}
```

### Work Assignment

The control plane assigns workspaces to runners via polling:

```
GET /api/runners/{runner_id}/work
→ Returns:
{
  "workspace_id": "ws-123",
  "action": "bootstrap",
  "repo": "owner/repo",
  "ref": "main",
  "devcontainer_config": { ... }
}
```

### Status Reporting

Runners report workspace state to the control plane:

```
POST /api/workspaces/{workspace_id}/status
{
  "actual_state": "started",
  "connection": {
    "ssh_host": "runner-abc.internal",
    "ssh_port": 2222,
    "ssh_user": "quickspaces"
  },
  "metrics": { ... }
}
```

The control plane treats all runners equivalently via this shared contract.

---

## TrueNAS SCALE Runner

### Overview

The TrueNAS runner executes as a long-lived application within the TrueNAS SCALE App framework. It hosts multiple workspaces on a single TrueNAS instance and runs continuously.

### Deployment

The runner is deployed via the TrueNAS App mechanism:

```
TrueNAS UI → Apps → Install Quickspaces Runner

Configuration:
  - Control plane URL
  - Runner pool name
  - Auth token
  - Concurrent workspace capacity
```

The TrueNAS App framework manages container orchestration, networking, persistent storage, and monitoring.

### Workspace Model

Multiple workspaces coexist on a single TrueNAS runner:

```
TrueNAS Hardware
├── Docker Daemon (managed by App)
│   ├── Workspace 1 (user alice, repo A)
│   ├── Workspace 2 (user bob, repo B)
│   ├── Workspace 3 (user charlie, repo C)
│   └── ...
└── App health monitoring
```

Each workspace has:
- Isolated Docker containers and networks
- Per-workspace SSH port (2222, 2223, 2224, etc.)
- Mounted home directory or workspace directory
- Consistent identity (`quickspaces` user)

### Lifecycle

**Workspace Creation**

- Control plane selects TrueNAS runner
- Runner has available slots
- Runner spawns Docker containers for the workspace
- Workspace becomes available for connection

**Workspace Runtime**

- Workspace runs persistently or until destroyed
- Runner does not evict idle workspaces
- Storage persists across stops and restarts
- TrueNAS backup and resilience policies apply to workspace data

**Workspace Termination**

- User or TTL policy triggers destroy
- Runner stops and removes containers
- Storage is cleaned up

### Runner Lifecycle

The TrueNAS runner:
- Runs 24/7 on the TrueNAS system
- Maintains continuous registration with the control plane
- Hosts multiple concurrent workspaces
- Reports heartbeat and metrics continuously

---

## AWS EC2 Runner

### Overview

On AWS, Quickspaces runs workspaces as ephemeral EC2 instances. Each instance is dedicated to a single workspace. Instances are created on demand and terminated immediately when the workspace ends.

### Architecture

Workspaces on AWS run on EC2 instances managed by an Auto Scaling Group:

```hcl
resource "aws_autoscaling_group" "quickspaces_runners" {
  name                = "quickspaces-runners"
  min_size            = 0              # No always-on instances
  desired_capacity    = 0              # Start at zero
  max_size            = 100            # Maximum running instances
  vpc_zone_identifier = var.availability_zones
  launch_template {
    id = aws_launch_template.runner.id
  }
}

resource "aws_launch_template" "runner" {
  image_id      = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  user_data     = base64encode(file("runner-startup.sh"))
  # Installs: Docker, devcontainer support, Quickspaces agent
}
```

### Workspace Model

Each EC2 instance hosts exactly one workspace:

```
AWS Compute Pool
├── EC2 Instance (Workspace 1, user alice)
├── EC2 Instance (Workspace 2, user bob)
├── EC2 Instance (Workspace 3, user charlie)
└── ...
```

Each instance provides:
- Dedicated EC2 instance (full resource allocation)
- Clean Linux environment
- SSH access on port 22
- Ephemeral root volume (no persistence after termination)

### Instance Lifecycle

**Creation**

1. User initiates workspace creation
2. Control plane selects AWS pool
3. ASG launches a new EC2 instance
4. Instance boots, installs Quickspaces agent
5. Agent registers with control plane
6. Control plane assigns workspace to instance
7. Instance clones repository and starts devcontainer
8. Developer connects via SSH

**Runtime**

- Instance executes user code
- Instance charges per second of uptime

**Termination**

1. User destroys workspace or TTL expires
2. Control plane transitions workspace to "destroyed"
3. Instance gracefully shuts down containers
4. Instance is terminated
5. Billing stops immediately

### Idle Handling

When a workspace is idle:

- Control plane detects inactivity (no activity for N minutes)
- Control plane can signal to ASG for capacity reduction
- Idle instances can be terminated to reduce costs

---

## Additional Runner Implementations

Quickspaces supports runner implementations for other hosting platforms.

### Proxmox Runner

- **Deployment**: Quickspaces agent on Proxmox cluster
- **Lifecycle**: VMs created on-demand per workspace, destroyed when workspace ends
- **Workspace hosting**: One workspace per VM
- **Resource model**: Infrastructure cost fixed; VM provisioning overhead paid per-workspace

### Physical / Bare Metal Runner

- **Deployment**: Quickspaces agent installed on physical machines
- **Lifecycle**: Long-lived; machines remain powered
- **Workspace hosting**: Multiple workspaces per machine
- **Resource model**: Infrastructure cost fixed

---

## Control Plane: Platform-Agnostic Design

### Control Plane Responsibilities

The control plane:
- Maintains a registry of available runners
- Tracks runner health via heartbeat
- Assigns workspaces to runners based on availability and policy
- Orchestrates workspace lifecycle state transitions
- Remains free of platform-specific logic

### Runner Selection Algorithm

```python
def select_runner(pool_name, capacity_needed):
    runners = get_runners_in_pool(pool_name)
    available = runners.filter(available_slots >= capacity_needed)
    return available.sort_by(available_slots).first()
```

The algorithm:
1. Filters runners by pool
2. Filters by capacity
3. Selects runner with highest availability

### Extensibility

To add a new runner implementation:

1. Create a runner agent that implements the shared protocol:
   - Registration
   - Heartbeat
   - Work dispatch polling
   - Workspace lifecycle
   - Status reporting

2. Provide deployment tooling (manifests, scripts, documentation)

3. **No changes to the control plane required**

The control plane remains generic across all runner implementations.

---

## Comparison: Runner Characteristics

| Aspect | TrueNAS | AWS EC2 | Proxmox | Physical |
|--------|---------|---------|---------|----------|
| **Deployment** | TrueNAS App | ASG + Terraform | Agent on cluster | Agent on machine |
| **Runner Lifecycle** | Long-lived (24/7) | Ephemeral (per-workspace) | Ephemeral (per-workspace) | Long-lived (24/7) |
| **Workspaces per Runner** | Multiple | One | One | Multiple |
| **Cost Model** | Fixed (hardware) | Per-second (opex) | Fixed (hardware) | Fixed (hardware) |
| **Scaling** | Manual | Automatic (ASG) | Semi-automatic | Manual |
| **Startup Time** | < 1 second | 2-5 minutes | < 1 minute | < 1 second |
| **Data Custody** | On-prem | Cloud | On-prem | On-prem |

---

## Hybrid Deployments

Quickspaces supports multiple runner pools within a single control plane. Workspaces can be assigned to different pools based on availability and priority:

```yaml
runners:
  pools:
    - name: aws-us-east-1
      implementation: ec2
      priority: 1
    - name: truenas-lab
      implementation: truenas
      priority: 2
    - name: proxmox-staging
      implementation: proxmox
      priority: 3
```

Assignment logic:
1. Select pool by priority
2. Assign workspace to available runner in pool
3. Fallback to next priority pool if no capacity
