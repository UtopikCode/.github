# What Is a Workspace?

In Quickspaces, a **workspace** is the fundamental unit of development.

A workspace represents a **developer's active development environment** for a specific GitHub repository and reference (branch or commit), managed and tracked by the Quickspaces control plane.

A workspace is a **logical concept**, not a specific piece of infrastructure.

## A Workspace Is

- A development session tied to:
  - a GitHub repository
  - a Git reference (branch, tag, or commit)
- A consistent environment defined by:
  - `.devcontainer` configuration
  - optional `docker-compose` services
- A lifecycle object tracked over time:
  - created
  - started
  - stopped
  - destroyed

## A Workspace Is NOT

- Not a runner
- Not a VM
- Not a container
- Not an AWS or TrueNAS resource

Those are **implementation details** used to *host* a workspace.

## Workspace Lifetime

A workspace may outlive any individual execution environment.

For example:

- On AWS, a workspace may be hosted on an ephemeral EC2 instance that is created and destroyed on demand.
- On TrueNAS, a workspace may run on a long-lived host as one of multiple active environments.

In all cases:

- The **workspace identity and state** are owned by the control plane.
- The **execution environment** is replaceable.

## Workspace State

Each workspace has two tracked states:

- **Desired state** – What the user intends (running, stopped, deleted).
- **Actual state** – What the execution layer has currently achieved.

The control plane reconciles these states by issuing commands to the appropriate runner or orchestration agent.

## Workspace Contents

A workspace may include:

- The repository filesystem
- Devcontainer tooling
- Multiple containers (via docker-compose)
- Exposed ports
- Temporary runtime state

Persistent data policies are defined by the hosting backend and configuration.

## Why Workspaces Are Central

Everything in Quickspaces revolves around workspaces:

- Runners exist to host workspaces
- Orchestration agents exist to create environments for workspaces
- The control plane exists to coordinate workspace intent and lifecycle
