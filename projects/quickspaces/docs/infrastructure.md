# Infrastructure Options

## Overview

Quickspaces supports multiple runner implementations, each tailored to its hosting technology. This document describes deployment options and their tradeoffs.

**Important**: Each runner implementation (AWS, TrueNAS, Proxmox, etc.) has a distinct design, cost model, and operational model. There is no "generic runner with configuration flags."

For detailed information on runner implementations, see [Runner Architecture](./runner-architecture.md).

---

## Cloud Providers

### AWS (Recommended for Cloud)

**Runner model**: Ephemeral, on-demand, one workspace per instance

**Architecture:**
- EC2 instances launched on-demand via Auto Scaling Group
- One instance per workspace (strict isolation)
- Instances destroyed when workspace ends
- Supports Spot Instances for cost savings

**Characteristics:**

- Runners scale to zero when no workspaces are active
- Each workspace is charged for the time it runs
- Spot instances provide 70% cost savings
- No idle compute charges

**Configuration:**
```yaml
runners:
  pool: aws-us-east-1
  implementation: ec2              # Technology identifier
  auto_scaling_group: quickspaces-runners
  instance_type: t3.medium         # Adjust based on workload
  spot_enabled: true               # Use Spot for cost savings (70% discount)
  min_capacity: 0                  # No always-on runners
  max_capacity: 100                # Maximum runners allowed
  scaling_policy: on-demand        # Launch instances as needed
```

**Cost example (100 developers):**
- t3.medium Spot: $0.0125/hour (70% discount from $0.0416 on-demand)
- Usage: 100 devs × 4 hours/day × 20 work days/month = 8,000 instance-hours
- **Monthly cost: $100** (infrastructure only), vs $300+ with always-on runners

**Benefits:**
- Scales to zero (no idle VMs)
- Cost aligns with usage
- Simple to deploy (infrastructure-as-code)
- Integrates with AWS services (IAM, VPC, CloudWatch, budget alerts)

**Considerations:**
- Spot instances can be interrupted (minimize impact: auto-replace)
- Startup takes 2-5 minutes (acceptable for dev workflows)
- Requires AWS account and billing setup
- Regional latency if far from users

---

## On-Premises & Home Lab

### Physical Machines / Bare Metal

**Runner model**: Long-lived, always-on, multiple workspaces per machine

**Architecture:**
- Linux machines with Docker and Quickspaces agent installed
- Machines remain running 24/7
- Multiple workspaces coexist on each machine

**Characteristics:**

- Hardware cost is amortized over multiple workspaces
- Multiple workspaces share the machine
- Maximizes utilization of purchased hardware

**Configuration:**
```yaml
runners:
  pool: on-prem-main
  implementation: physical          # Technology identifier
  hosts:
    - host1.example.com:22
    - host2.example.com:22
    - host3.example.com:22
  capacity_per_host: 4              # 4 concurrent workspaces per host
```

**Cost example (one $500 server, 3-year lifespan):**
- Hardware amortized: $0.02/hour ($500 ÷ 3 years ÷ 8760 hours/year)
- Electricity: ~$0.01/hour
- **Per-workspace cost**: $0.003-0.005/hour (with 4 concurrent workspaces)

**Benefits:**
- Much cheaper than cloud per-workspace
- Full control over data and security
- No network latency (local deployment)
- Can integrate with existing on-prem infrastructure

**Considerations:**
- Setup and ongoing maintenance burden
- Hardware can fail (remediation manual)
- Scaling requires purchasing new hardware
- No automatic replacement or failover

---

### Proxmox Virtual Environment

**Runner model**: On-demand VMs, ephemeral, one workspace per VM

**Architecture:**
- Quickspaces agent deployed on Proxmox cluster controller
- Creates VM on-demand for each workspace (from template)
- VMs destroyed when workspace ends
- Similar cost model to AWS, but on-prem

**Characteristics:**

- On-prem infrastructure with automated VM provisioning
- One VM per workspace (strict isolation)
- VM creation is sub-minute
- Easy resource cleanup (destroy VM when workspace ends)

**Configuration:**
```yaml
runners:
  pool: proxmox-ha
  implementation: proxmox           # Technology identifier
  proxmox_api: https://proxmox.example.com:8006
  node: pve-cluster
  template_vm: ubuntu-22-template   # Base VM image
  storage: local-lvm                # Storage backend
  cpu_per_vm: 4
  memory_per_vm: 8096               # MB
  disk_per_vm: 20                   # GB
```

**Cost:**
- Infrastructure cost (server hardware, network, power)
- Per-workspace: Hardware cost amortized across VM lifetime
- VM creation is instant (faster than AWS)

**Benefits:**
- On-prem with automatic per-workspace provisioning
- Full isolation (one VM per workspace)
- Medium complexity (more than Bare Metal, less than K8s)
- Great for medium-sized teams wanting cloud-like elasticity

**Considerations:**
- Requires Proxmox expertise and setup
- Must maintain template VMs
- Licensing: Proxmox is open-source, but support/subscriptions optional
- Hardware cost upfront (not pay-per-hour like cloud)

---

### TrueNAS / NAS Appliances

**Runner model**: Long-lived App, 24/7, multiple workspaces per runner

**Architecture:**
- Installed as TrueNAS SCALE App (using Apps ecosystem)
- Runs continuously on TrueNAS hardware
- Multiple workspaces can coexist on one TrueNAS instance
- Leverages TrueNAS persistence, ZFS, and resilience

**Characteristics:**

- Long-lived runner deployed as TrueNAS SCALE App
- Multiple workspaces per runner (shared hardware)
- Workspace data is persistent (backed by ZFS)
- TrueNAS provides storage resilience and backups

**Deployment:**
1. Open TrueNAS SCALE UI → Apps
2. Add Quickspaces Runner App
3. Configure:
   - Control plane URL
   - Runner pool name
   - API auth token
   - Concurrent workspace capacity (3-10 typical for NAS appliances)

The TrueNAS App framework manages networking, persistence, health monitoring, and restarts.

**Configuration:**
```yaml
runners:
  pool: truenas-scale
  implementation: truenas          # Technology identifier
  host: truenas.example.com
  capacity: 4                      # 4 concurrent workspaces on this TrueNAS
  app_name: quickspaces-runner     # TrueNAS App name
  storage_pool: example-pool       # ZFS pool for workspace data
```

**Cost example (small team, 4 workspaces concurrently):**
- TrueNAS appliance: $2,000-5,000 (one-time)
- Electricity: ~$50-100/month
- **Per-workspace amortized cost**: <$15/month across team

**Benefits:**
- Excellent cost efficiency (amortize hardware across many users)
- Data resilience (ZFS backups, snapshots)
- Suitable for small/medium teams and home labs
- Data stays on-prem (privacy and compliance)
- Docker-native (App framework)

**Considerations:**
- Capacity limited by NAS hardware (not cloud-scale)
- Not suitable for >100 concurrent workspaces (single machine limitation)
- Always-on power and network required
- Operator must maintain NAS health and storage

---

## Hybrid Setup

Combine multiple runner implementations in one control plane:

```yaml
runners:
  pools:
    - name: aws-prod
      implementation: ec2
      priority: 1                # Try this first (burst capacity)
      capacity: 100
    
    - name: on-prem-base
      implementation: physical
      priority: 2                # Use second (baseline capacity, cheap)
      capacity: 10
    
    - name: truenas-lab
      implementation: truenas
      priority: 3                # Last resort
      capacity: 4
```

**Workspace assignment (by priority):**
1. Control plane checks available capacity in each pool
2. Picks first pool with available slots
3. If multiple pools have capacity, selects by priority
4. If all full, returns "No available runners"

**Use cases:**

- **Startups**: AWS-only (scales with demand, pay-per-use)
- **Small teams**: TrueNAS or bare metal only (low fixed cost)
- **Mature orgs**: AWS + on-prem hybrid (cheap baseline + burst capacity)
- **Compliance**: Sensitive code on-prem, non-sensitive on AWS

---

## Comparison Matrix

| Criterion | AWS EC2 | Physical | Proxmox | TrueNAS |
|-----------|---------|----------|---------|---------|
| **Deployment** | ASG + K8s/Terraform | SSH + agent binary | Qemu/KVM + agent | TrueNAS App |
| **Runner model** | Ephemeral (per-workspace) | Long-lived | Ephemeral (per-VM) | Long-lived |
| **Workspace per runner** | 1 | Many (3-10) | 1 | Many (3-10) |
| **Cost structure** | Operating expenditure (OpEx) | Capital expenditure (CapEx) | CapEx | CapEx |
| **Billing** | Pay-per-second | Fixed infrastructure | Fixed infrastructure | Fixed infrastructure |
| **Scaling** | Automatic (auto-scaling group) | Manual (buy hardware) | Semi-automatic (VMaaS) | Manual (add hardware) |
| **Startup time** | 2-5 min | <1 sec* | <30 sec | <1 sec* |
| **Data/Privacy** | Limited (cloud) | ✓ Full (on-prem) | ✓ Full (on-prem) | ✓ Full (on-prem) |
| **Suitable for** | Scaling organizations | Home lab, small team | Medium organization | Small team, home lab |
| **Operational complexity** | Medium | Low | High | Low |
| **Cost efficiency** | ✓ At any scale | ✓ After amortization | ✓ After amortization | ✓ After amortization |

*Startup: "Instant" assumes machine is already on; time is container/VM provisioning, not hardware boot.

---

## Design Patterns

### Pattern 1: Pure Cloud (Scaling Organizations)

- Single AWS account with auto-scaling
- Pay-per-use model ensures no wasted capacity
- Good for: Startups, rapidly growing teams, variable workload
- **Why**: Cloud scales with demand; no upfront hardware cost

### Pattern 2: On-Prem Only (Small Team / Privacy-Focused)

- Dedicated hardware (Bare Metal, Proxmox, or NAS)
- Fixed infrastructure cost amortized across users
- Good for: Compliance requirements, privacy-sensitive work, cost-predictability
- **Why**: Keep data under your control; low per-user cost at scale

### Pattern 3: Hybrid (Mature Organization)

- Base capacity on on-prem or TrueNAS (cheap, always-on)
- Burst to AWS cloud during peaks (expensive but on-demand)
- Best cost/performance trade-off
- Good for: Predictable baseline + variable spikes
- **Why**: Minimize per-user cost while handling unpredictable demand

### Pattern 4: Multi-Region (Global Organization)

- Runners in different regions (AWS us-east-1, AWS eu-west-1, on-prem Asia)
- Low latency for all developers worldwide
- Resilience: If one region down, fall back to another
- **Why**: Global distribution, disaster recovery, latency optimization

---

## Adding New Runner Implementations

The Quickspaces architecture is extensible. To add a new hosting technology (Kubernetes, Lambda, Nomad, OpenStack, etc.):

1. **Implement a runner binary/agent** that:
   - Registers with the control plane
   - Maintains heartbeat (30-second updates)
   - Polls for workspace assignments
   - Handles workspace lifecycle (bootstrap, start, stop, destroy)
   - Reports status and metrics
   
2. **Write deployment tooling** (manifests, scripts, docs)

3. **Document the model** (architecture diagram, cost analysis, tradeoffs)

4. **Zero changes to the control plane** ← This is the key

The control plane has **no platform-specific logic**. It treats all runners equivalently. This allows new implementations to be added independently.

---

## Implementation Decisions Summary

**Why not a "universal" runner?**

A single runner that works on all platforms would require:
- Compromises that satisfy no one
- Efficiency losses (feature bloat, unnecessary abstractions)
- Cloud platforms paying for always-on idle resources
- On-prem platforms paying for VM startup/shutdown overhead
- Control plane bloated with platform-specific logic

**Why multiple implementations?**

- **AWS runner is ephemeral** because AWS charges by the second (minimize idle costs)
- **TrueNAS runner is long-lived** because hardware is persistent (maximize hardware ROI)
- **Physical runners share workspaces** because hardware is bought upfront (maximize utilization)
- **Proxmox runners are ephemeral** because VMs are elastic (clean isolation, easy cleanup)
- **Control plane is agnostic** because it only uses a shared protocol (no platform lock-in)

**The result**: Cost aligns with infrastructure. Efficiency improves. Operations become simpler.

---

## Future Backends

Quickspaces architecture supports adding new hosting technologies without changing the control plane:

- **Kubernetes** — For teams already using K8s
- **Nomad** — HashiCorp's orchestrator
- **Lambda** — AWS serverless functions
- **OpenStack** — Open-source cloud
- **Packet / Equinix** — Bare metal cloud
- **Your own platform** — Build a Quickspaces runner agent for your infrastructure

---

## References

- [Runner Architecture and Implementation](./runner-architecture.md) — Deep dive on why multiple runners exist
- [Control Plane Architecture](./control-plane-architecture.md) — How the control plane orchestrates runners
- [Why Quickspaces?](./why-quickspaces.md) — Design philosophy
