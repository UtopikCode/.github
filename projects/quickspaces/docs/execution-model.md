# Execution Model

Quickspaces is built around explicit separation between coordination, orchestration, and execution. This document describes the roles of each component and how they interact across different hosting platforms.

## Workspace

A **Workspace** is a first-class logical concept within Quickspaces.

A workspace represents:

- A developer's isolated development environment
- Tied to a specific GitHub repository and Git reference (branch, tag, or commit)
- Tracked and coordinated by the control plane over its lifetime
- Independent of the underlying execution infrastructure

### Workspace Identity and Execution

Workspace identity is persistent and owned by the control plane.

Workspace execution environment is replaceable:

- A workspace may be created on one execution environment
- Paused and moved to a different environment
- Recreated on fresh infrastructure while maintaining identity

In all cases:

- The workspace identity remains constant
- The workspace state is reconciled by the control plane
- The underlying execution infrastructure is an implementation detail

### What a Workspace Is NOT

A workspace is not:

- A VM
- A container
- A runner process
- A host or execution environment

These are infrastructure components that *host* workspaces, not workspaces themselves.

## Agent

An **Agent** is an orchestration component responsible for translating control-plane intent into platform-specific actions.

An agent:

- Calls platform-native APIs (AWS APIs, Proxmox APIs, TrueNAS APIs)
- Translates control-plane instructions into platform operations
- Monitors platform state and reports back to the control plane
- Is stateless with respect to workspace execution

### Agent Scope

An agent never:

- Executes user code
- Hosts workspaces
- Makes policy decisions
- Manages workspace state directly

An agent always:

- Operates under instructions from the control plane
- Translates those instructions into platform actions
- Reports execution results back to the control plane

### Agent Deployment Models

An agent may run as:

- A serverless function (AWS Lambda, etc.)
- A standalone binary
- A containerized service
- Any compute model supported by the target platform

### Platform Coverage

Every supported platform has a corresponding agent. An agent exists regardless of whether that platform has native execution capabilities.

## Runner (Execution Service)

A **Runner** is a Quickspaces-provided execution service responsible for running user code and managing execution environments.

A runner is required only on platforms that do not provide native execution capabilities.

### Runner Responsibilities

A runner:

- Creates and manages execution containers
- Mounts workspaces into containers
- Manages exposed ports and network interfaces
- Watches container health and lifetime
- Reports execution state to the control plane
- Executes commands issued by agents

### Runner Scope

A runner:

- May host multiple workspaces simultaneously
- Must remain available while workspaces are running
- Is not responsible for policy or scheduling decisions
- Does not coordinate with the control plane directly

A runner never:

- Makes decisions about which workspaces to run
- Determines policy or access control
- Manages cluster-wide resource allocation
- Orchestrates infrastructure

### When a Runner Is Required

A runner is required on platforms that do not offer native on-demand execution:

- TrueNAS SCALE (no native workspace execution)
- Other non-cloud hosting environments

A runner is NOT required on platforms with native execution APIs:

- AWS (uses EC2)
- Proxmox (uses VMs)

## Platform Responsibility Matrix

The following table describes the execution model across supported platforms:

| Platform | Native Execution | Agent | Runner |
|----------|------------------|-------|--------|
| AWS | EC2 VMs | Yes (serverless) | No |
| Proxmox | VMs | Yes (binary) | No |
| TrueNAS SCALE | None | Yes (containerized) | Yes (Docker) |

**Explanation:**

- **Native Execution:** the platform capability used to run workspaces
- **Agent:** orchestration service that translates control-plane intent
- **Runner:** Quickspaces-provided execution service (only when platform lacks native capability)

## Execution Model Summary

The Quickspaces execution model operates in three layers:

1. **Control Plane** – Issues coordination intent (desired workspace states, lifecycle commands)
2. **Agents** – Orchestrate infrastructure by calling platform APIs and translating control-plane intent into platform operations
3. **Execution Environments** – Run user code and workspaces
   - Native platform capability (EC2, VMs) where available
   - Quickspaces-provided runners where necessary

This model ensures:

- The control plane remains platform-agnostic
- Orchestration adapts to each platform's capabilities
- Execution is delegated to the platform when possible
- Quickspaces provides execution only when the platform cannot

## Repository Structure

Quickspaces is built from several core repositories, each with distinct responsibility:

### Control Plane Repository

- **Purpose:** coordination logic, state management, policy enforcement
- **Responsibility:** tracking workspace lifecycle, reconciliation, API surface
- **Deployment:** required in all deployments
- **Language/Stack:** Go

### Agent Repositories (Per Platform)

Each platform has a dedicated agent:

- **aws-agent**
  - Calls AWS APIs (EC2, IAM, etc.)
  - Manages infrastructure provisioning and deprovisioning
  - Deployment: serverless (Lambda)

- **proxmox-agent**
  - Calls Proxmox APIs
  - Manages VM lifecycle
  - Deployment: standalone binary

- **truenas-agent**
  - Calls TrueNAS APIs
  - Coordinates with the execution runner
  - May be embedded in the execution service
  - Deployment: containerized

**Requirement:** At least one agent must be available per supported platform in a deployment. Agents are optional depending on which platforms you target.

### Runner / Execution Service Repository

- **Purpose:** manages workspace execution for platforms without native capability
- **Responsibility:** container lifecycle, volume management, port exposure, health monitoring
- **Deployment:** required only for non-native platforms (e.g., TrueNAS SCALE)
- **Technology:** Docker-based

### VS Code Extension Repository

- **Purpose:** user interaction and workspace lifecycle UX
- **Responsibility:** workspace creation, connection, status reporting
- **Deployment:** distributed to VS Code marketplace
- **Required:** yes, for all users

### Documentation and Specification Repository

- **Purpose:** glossary, protocol specifications, architecture documentation
- **Contents:** this repository (.github)
- **Required:** yes, as a reference
- **Status:** evolving

## Repository Requirements by Deployment

### Cloud Deployments (AWS)

Required:
- Control Plane
- aws-agent
- VS Code Extension

Not required:
- Runner
- Other agents

### On-Premises VM (Proxmox)

Required:
- Control Plane
- proxmox-agent
- VS Code Extension

Not required:
- Runner
- Other agents

### On-Premises Container (TrueNAS SCALE)

Required:
- Control Plane
- truenas-agent
- Runner (execution service)
- VS Code Extension

Not required:
- Other agents
