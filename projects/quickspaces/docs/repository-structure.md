# Repository Structure

Quickspaces is built from several core repositories, each with distinct responsibility. This document describes each repository, its purpose, and deployment requirements.

## Core Repositories

### Control Plane Repository

**Purpose:** coordination logic, state management, policy enforcement

**Responsibility:**
- Tracking workspace lifecycle
- Reconciliation logic
- API surface and coordination
- State persistence

**Deployment:** required in all deployments

**Language/Stack:** Go

### Agent Repositories (Per Platform)

Each supported platform has a dedicated agent. Agents are orchestration components that translate control-plane intent into platform-specific actions.

#### AWS Agent (`aws-agent`)

**Purpose:** Orchestration for AWS hosting

**Responsibility:**
- Calls AWS APIs (EC2, IAM, VPC, etc.)
- Manages infrastructure provisioning and deprovisioning
- Reports platform state back to control plane

**Deployment:** serverless (AWS Lambda)

**Required for:** AWS deployments

#### Proxmox Agent (`proxmox-agent`)

**Purpose:** Orchestration for Proxmox hosting

**Responsibility:**
- Calls Proxmox APIs
- Manages VM lifecycle
- Reports state to control plane

**Deployment:** standalone binary

**Required for:** Proxmox deployments

#### TrueNAS Agent (`truenas-agent`)

**Purpose:** Orchestration for TrueNAS SCALE hosting

**Responsibility:**
- Calls TrueNAS APIs
- Coordinates with the execution runner
- May be embedded in the execution service

**Deployment:** containerized

**Required for:** TrueNAS SCALE deployments

### Runner / Execution Service Repository

**Purpose:** manages workspace execution for platforms without native capability

**Responsibility:**
- Container lifecycle management
- Volume management and workspace mounting
- Port exposure and networking
- Health monitoring and reporting
- Command execution under agent instructions

**Deployment:** required only for non-native platforms (e.g., TrueNAS SCALE)

**Technology:** Docker-based

**Required for:** Non-native execution platforms

### VS Code Extension Repository

**Purpose:** user interaction and workspace lifecycle UX

**Responsibility:**
- Workspace creation and deletion UI
- Connection management to workspaces
- Status reporting and monitoring
- Git reference selection

**Deployment:** distributed to VS Code Marketplace

**Required:** yes, for all users

### Documentation and Specification Repository

**Purpose:** glossary, protocol specifications, architecture documentation

**Contents:** this repository (.github)

**Responsibility:**
- Execution model documentation
- Architecture and design decisions
- API specifications
- Contributing guidelines

**Required:** yes, as a reference for development and understanding

## Repository Requirements by Deployment

### AWS Deployment

**Required:**
- Control Plane
- aws-agent
- VS Code Extension
- Documentation

**Not required:**
- Runner
- Other platform agents

### Proxmox On-Premises VM Deployment

**Required:**
- Control Plane
- proxmox-agent
- VS Code Extension
- Documentation

**Not required:**
- Runner
- Other platform agents

### TrueNAS SCALE On-Premises Container Deployment

**Required:**
- Control Plane
- truenas-agent
- Runner (execution service)
- VS Code Extension
- Documentation

**Not required:**
- Other platform agents

## Repository Dependencies

The following diagram shows the dependency relationships:

- **Control Plane** – the central coordinator
  - communicates with every agent
  - stores all state
- **Each Agent** – platform-specific orchestrator
  - receives instructions from Control Plane
  - calls platform APIs
  - reports state back to Control Plane
  - communicates with Runner only if needed (TrueNAS case)
- **Runner** – execution service (TrueNAS only)
  - executes commands from agent
  - manages containers and workspaces
- **VS Code Extension** – user interface
  - communicates with Control Plane
  - provides UX for workspace management

## Adding a New Platform

To add support for a new platform:

1. Create a new agent repository following the pattern of existing agents
2. Implement the platform-specific API client
3. Define the orchestration logic to translate control-plane intent into platform operations
4. If the platform lacks native execution capability, create or adapt a runner
5. Document the new platform in deployment and architecture guides
6. Add integration tests and examples

The Control Plane itself requires no changes; it remains platform-agnostic.
