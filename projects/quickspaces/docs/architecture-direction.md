# Architectural Direction for Quickspaces

## Design Intent

Quickspaces is designed to balance extensibility with operational simplicity. The architecture is intended to allow multiple execution layers to coexist without forcing the control plane to become execution-aware.

The long-term intent is to preserve a clean separation between the control plane and execution environments, so that execution layers can be replaced or extended as infrastructure models change. The system is designed to support cost-aligned execution where billing follows activity and resource usage rather than uptime.

Operational simplicity is a first-order constraint. The architecture is intended to avoid unnecessary coupling, reduce the number of active runtime dependencies, and keep the core behavior predictable across self-hosted and managed deployments.

## Expected Areas of Evolution

### Execution environments

Quickspaces is expected to accommodate a broader range of execution environments over time. These are not feature add-ons, but classes of environments where workspaces can run with the same control plane semantics.

Examples of that evolution include:

- new host types for workspace execution
- execution surfaces that map to different infrastructure cost models
- environments where ephemeral workspaces are provisioned and reclaimed automatically

### Orchestration backends

The system is designed to allow orchestration backends to evolve separately from the core control plane. Different orchestration layers may be added as long as they preserve the same execution contract: ephemeral workspace lifecycle management, health visibility, and restart-safe reconciliation.

### Policy and governance models

Quickspaces may evolve to support broader policy models, but the expectation is that these models remain declarative and aligned with the control plane’s reconciliation-driven design. Future policy expansion is expected to cover organization-level governance, workspace placement, and access boundaries without turning the control plane into a general policy engine.

### Workspace lifecycles

Alternative workspace lifecycles are an anticipated area of evolution. The architecture is intentionally left open to support more than a single lifecycle model, such as strictly ephemeral sessions and longer-lived workspaces, provided the core control flow remains reconciliation-driven and cost-aware.

## Explicit Non-Goals (Forward-Looking)

Quickspaces is not intended to become:

- a general-purpose platform-as-a-service
- a full CI/CD system
- a generic container orchestration platform
- a replacement for cloud provider infrastructure

The intent is to remain focused on on-demand development environments, not to broaden into adjacent infrastructure domains that would undermine the project’s operational simplicity and control plane focus.

## Stability vs Flexibility

### Intended stability

The following aspects are intended to remain stable:

- the separation between control plane and execution
- the reconciliation-driven model for desired state
- the use of normalized workspace and repository contracts
- the constraint that provider integrations sit at the edge

### Intended flexibility

The following areas are intentionally fluid:

- the choice of execution backend
- orchestration layer implementations
- resource and cost alignment strategies for different hosts
- policy scope within the workspace lifecycle boundary

This distinction is meant to make it clear where contributors should preserve existing architectural expectations and where they may propose changes.

## Guidance for Contributors

This document is directional guidance, not a roadmap or commitment. It is intended to help determine whether a proposal aligns with the project’s architectural trajectory.

Contributors should use this guidance to evaluate proposals along these lines:

- does the change preserve the control plane / execution separation?
- does it avoid adding provider-specific logic into the core domain?
- does it keep operational complexity low?
- does it fit within the expected classes of evolution rather than expanding into unrelated infrastructure domains?

Proposals should be judged on whether they support the architecture’s intent and constraints, not on whether they add a particular feature.
