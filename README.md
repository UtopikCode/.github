# Quickspaces

**On-demand, ephemeral development environments. Self-hosted. Docker-native. Cost-conscious.**

Quickspaces is an open-source, self-hostable alternative to GitHub Codespaces. It provides development environments that spin up on demand, tear down automatically.

---

## What It Does

Quickspaces provides **on-demand development environments** using Docker and VS Code. Instead of keeping servers running 24/7, workspaces spin up when needed and destroy automatically when idle.

**Key features:**
- ✅ Devcontainer-first (fully compatible with VS Code Dev Container spec)
- ✅ Docker Compose support (databases, caches, side services)
- ✅ Ephemeral by design (pay only for active usage)
- ✅ Cloud-agnostic (run on AWS, on-prem, NAS, or anywhere)
- ✅ Open-source and self-hostable

**How developers use it:**

1. Open a repository with `.devcontainer` config
2. Click "Create Workspace"
3. Full dev environment ready in minutes
4. Workspace auto-stops when idle
5. Resume later or destroy it

For more details, see [Why Quickspaces?](./docs/why-quickspaces.md) and [Architecture Overview](./docs/control-plane-architecture.md).

---

## Quick Start

- **Developers:** [Getting Started Guide](./docs/getting-started.md)
- **Self-Hosters:** [Deployment Guide](./docs/deployment.md)
- **Want to know why?** [Why Quickspaces?](./docs/why-quickspaces.md)
- **Deep dive:** [Control Plane Architecture](./docs/control-plane-architecture.md)

---

## Where Quickspaces Runs

Quickspaces uses **multiple runner implementations**, each tailored to its hosting technology:

- **AWS**: Ephemeral EC2 instances (one per workspace, pay-per-second, scales to zero)
- **TrueNAS**: Long-lived SCALE App (multiple workspaces, always-on, hardware ROI)
- **On-Prem**: Physical machines or VMs (multiple workspaces, fixed infrastructure cost)
- **Proxmox**: On-demand VM provisioning (one per workspace, elastic scaling)
- **Future**: Any infrastructure with a Quickspaces runner agent

**Key principle**: Infrastructure reality is respected, not abstracted. AWS is ephemeral because idle VMs cost money. TrueNAS is long-lived because hardware is persistent. Each runner is optimized for its platform.

For detailed architecture and why multiple implementations exist, see [Runner Architecture and Implementation](./docs/runner-architecture.md).

Execution backends are swappable—infrastructure is replaceable, not baked in.

---

## Features

- **Docker Compose native** — Full multi-container support
- **Local-like behavior** — localhost ports, file watching, same tools
- **Fast startup** — Minutes, not hours
- **Auto-idle** — Workspaces pause automatically
- **Deterministic** — Same `.devcontainer` config = consistent environments

See [Developer Experience Guide](./docs/developer-experience.md) for details.

---

## Project Status

Quickspaces is **early-stage but serious**.

- 🚀 Foundation architecture is solid and well-tested
- 🏗️ API and execution backends are functional
- 📝 Documentation and examples are growing
- 🔧 We're actively developing and optimizing

**We're looking for:**
- Feedback and real-world usage patterns
- Contributors (especially on execution backends and infrastructure providers)
- Architectural discussions and design input
- Battle-testing in diverse environments

---

## Documentation

Browse documentation by role or need:

### 📖 By Audience

| Who | Read This |
|-----|-----------|
| **Developer** | [Getting Started](./docs/getting-started.md) — Set up and use Quickspaces |
| **Self-Hoster** | [Deployment Guide](./docs/deployment.md) — Run your own control plane |
| **Infra Engineer** | [Control Plane Architecture](./docs/control-plane-architecture.md) — Deep technical dive |
| **Contributor** | [Contributing Guide](./CONTRIBUTING.md) — How to build and improve |

### 📚 All Docs

**Guides**
- [Why Quickspaces?](./docs/why-quickspaces.md) — Problem statement and design decisions
- [Developer Experience](./docs/developer-experience.md) — Using Quickspaces as a developer
- [Infrastructure Guide](./docs/infrastructure.md) — AWS, on-prem, and custom runners

**Technical**
- [Runner Architecture and Implementation](./docs/runner-architecture.md) — Why multiple implementations exist, deep dive on runner design
- [Control Plane Architecture](./docs/control-plane-architecture.md) — Orchestration layer, lifecycle, stateless design
- [Control Plane Architecture](./docs/control-plane-architecture.md) — Complete architecture reference
- [API Reference](./docs/api-reference.md) — REST API and authentication
- [Runner Protocol](./docs/runner-protocol.md) — Building custom runners

**Operations**
- [Deployment Guide](./docs/deployment.md) — Deploy control plane and runners
- [Configuration Reference](./docs/configuration.md) — All config options
- [Operations Runbook](./docs/operations.md) — Common tasks and troubleshooting

**Examples**
- [Sample Dev Environments](./examples/) — `.devcontainer` configs for common stacks
- [Docker Compose Examples](./examples/docker-compose/) — Multi-container setups

---

## Next Steps

👉 **First time here?** Start with [Getting Started](./docs/getting-started.md)

👉 **Want to deploy it?** Go to [Deployment Guide](./docs/deployment.md)

👉 **Curious about design?** Read [Control Plane Architecture](./docs/control-plane-architecture.md)

👉 **Want to contribute?** See [Contributing Guide](./CONTRIBUTING.md)

---

## Design Principles

- **Devcontainer-first** → Your `.devcontainer` spec is the source of truth
- **Docker-native** → Full Docker Compose support, no container runtimes
- **Infrastructure-agnostic** → AWS, on-prem, NAS, or bring your own
- **Ephemeral by design** → Compute only when active, destroy when done
- **Clean separation** → Control plane and execution are replaceable components
- **Cost-optimized** → Zero idle cost, predictable per-workspace billing

---

## Community & Support

- **GitHub Issues**: Report bugs, request features, discuss architecture
- **Discussions**: Ask questions, share use cases, brainstorm ideas
- **Pull Requests**: Contributions welcome (see CONTRIBUTING.md)

---

## License

Quickspaces is open-source under the [LICENSE](link-to-license). 

---

**Built by a team that cares about developer experience and operational simplicity.**
