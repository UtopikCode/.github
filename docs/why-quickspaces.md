# Why Quickspaces?

## The Problem

GitHub Codespaces is an excellent managed service, but it has costs and constraints:

- **Managed SaaS pricing adds up** — Running many workspaces or large builds can become expensive
- **Idle cost is high** — Runners stay alive between uses, consuming resources
- **Vendor lock-in** — You can't run Codespaces on your own infrastructure
- **Limited customization** — Hard to integrate with existing infrastructure or tooling

Traditional CI/CD platforms face the same issue: developers expect always-on resources, and operations teams struggle with underutilized compute.

## The Solution

Quickspaces is built on a simple principle: **pay only for compute you're actively using**.

### Key Insights

**1. Ephemeral VMs are simpler and cheaper than idle runners**

Traditional CI/CD systems (GitHub Actions, GitLab CI) maintain pools of always-on runners:
- Even when sleeping, runners consume resources (compute, storage, network)
- Cost accumulates whether they're used or not

Quickspaces allocates compute only when a workspace is active:
- No unused runners sitting idle
- Workspaces destroy automatically when not in use
- Cost is proportional to actual usage

**2. Docker Compose requires a real Docker daemon**

Serverless container platforms (AWS Fargate, Google Cloud Run) don't work well for development:
- They can't run docker-compose (no daemon access)
- They're optimized for stateless functions, not long-running interactive sessions
- They lack features like local port binding and volume mounts

Real Docker (on a VM or host) is the right choice for dev environments:
- Full Docker and docker-compose support
- Local-like behavior (localhost ports, file watching, mounts)
- Direct control over the environment

**3. VM-per-workspace is simpler than container-per-workspace**

Some platforms try to run multiple workspaces on one host (Kubernetes, Docker Swarm). This adds complexity:
- "Noisy neighbor" problems (one workspace consuming all CPU affects others)
- Complex resource quarantine
- Harder troubleshooting

One workspace per VM is simpler:
- Complete isolation (security, performance, debugging)
- Easier resource accounting
- No complex orchestration needed

## Design Decisions

### Why Self-Hosted?

GitHub Codespaces is great if you trust GitHub with your code and infrastructure. But many teams need:
- **On-premises control** — Code never leaves their network
- **Cost predictability** — Own the infrastructure, own the costs
- **Compliance** — Meet data residency and security requirements
- **Integration** — Connect with existing CI/CD, VPN, or custom tooling

Quickspaces is open-source and self-hostable, so teams can run it how they want.

### Why Open-Source?

- **Transparency** — Audit exactly how workspaces are provisioned and destroyed
- **Extensibility** — Add custom runners or backends for your infrastructure
- **Trust** — No vendor lock-in; the code is yours
- **Community** — Benefit from contributions and improvements from other teams

## Cost Comparison

### Scenario: 100 developers, 4 hours per day

**GitHub Codespaces (or always-on runners)**
- 100 VMs × $0.05/hour × 24 hours × 20 days = **$2,400/month**
- (Workspaces run even when idle or asleep)

**Quickspaces with ephemeral VMs**
- 100 developers × 4 hours/day × 20 days × $0.50/hour = **$40/month**
- (Workspaces only exist when active)

**Savings: 98%**

(Real costs depend on instance size, region, and usage patterns. This is illustrative.)

## When Quickspaces Makes Sense

✅ **Good fit:**
- Teams with many developers (more idle cost savings)
- Frequent short development sessions (create, code, destroy)
- Cost-conscious orgs
- Need to run on own infrastructure
- Want full docker-compose support

❌ **Not a good fit:**
- Long-lived persistent development servers
- Teams that only have a few developers (small cost savings)
- Managed service preferred (let someone else operate it)

## What About Always-On Workspaces?

Quickspaces supports long-lived workspaces too:
- Workspace can stay `idle` instead of being destroyed
- Billing reduces when idle (optional)
- Resume by connecting again

But the architecture is optimized for ephemeral usage, so this is a secondary use case.

---

**Next:** [Getting Started Guide](./getting-started.md) or [Deployment Guide](./deployment.md)
