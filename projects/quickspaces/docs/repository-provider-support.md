# Repository Provider Support: Architectural Intent

## Problem Statement

Repository providers are the entrypoint for external identity and repository access. The control plane does not own provider identity as a domain concept because provider accounts are not the source of truth for internal authorization decisions. Instead, the control plane consumes provider identity and repository access as external assertions.

This matters because the control plane must accept identity from multiple providers without baking provider-specific identity rules into the core domain. Provider support is therefore a boundary concern: it is necessary for onboarding users and repositories, but it should not pollute reconciliation logic or long-lived state.

## Current Scope

### First-class providers today

- GitHub
- GitLab
- Azure DevOps

These providers are supported because they are the current operational targets and because they expose sufficiently similar OAuth-based identity and repository access models for the control plane to integrate without introducing multiple incompatible patterns.

### Why these providers were chosen

- They are widely used by the initial audience for Quickspaces
- They support the OAuth flows already used by the control plane
- They expose repository and membership metadata in a way that fits a reconciliation-driven model

### What support means in practice

Support today is intentionally narrow:

- OAuth-based identity acquisition
- Authorization of repository access for control plane-managed workspaces
- Mapping external repository references into normalized internal references

Support does not mean:

- full provider feature parity
- provider-specific workflow automation
- provider-hosted secrets, policies, or build pipelines

The practical contract is: if a user or service has authority to a repository through one of the supported providers, the control plane can recognize that authority, persist the necessary metadata, and use it to drive workspaces and reconciliations.

## Provider-Agnostic Design Principle

### Avoiding provider-specific assumptions in the core domain

The core domain is designed around internal concepts such as workspaces, repository references, and ownership claims, not around GitHub, GitLab, or Azure DevOps abstractions. This means:

- provider-specific metadata is kept at the edge
- internal reconciliation loops operate on normalized repository identifiers and internal auth claims
- provider behavior is not encoded into long-lived state transitions

### Why normalization matters

Normalizing identity and repo references is critical because provider APIs use different identifiers and data shapes. Without normalization:

- the control plane would be forced to special-case provider identity semantics
- repository reconciliation would become provider-dependent
- restart-safe state would be harder to reason about

Normalized references allow the control plane to treat 
all supported providers with the same plumbing for reconciliation and authorization checks.

### Why OAuth is isolated to the edge

OAuth is an integration surface, not a domain primitive. The control plane uses OAuth to establish a trusted external identity and to obtain enough information to map that identity into an internal principal.

By isolating OAuth to the edge:

- the core remains stateless except for its database
- provider token lifetimes and refresh semantics do not leak into reconciliation logic
- the control plane can be restarted safely without losing its domain model

## Intentional Non-Goals

### Providers explicitly not supported today

- Bitbucket
- AWS CodeCommit
- Google Cloud Source Repositories
- any provider requiring non-OAuth authentication flows

These providers are excluded because they are not part of the current target audience and because they would introduce additional integration complexity without improving the current product fit.

### Why supporting "everything" is harmful

A broad provider strategy would increase operational cost and dilute focus. Key risks include:

- more provider-specific edge cases in onboarding and reconciliation
- higher test surface area for provider compatibility
- more integration points where restart-safety and state consistency can fail

### Problems intentionally deferred

- provider-specific repository policies or permission models beyond basic repo access
- support for non-repository source systems
- provider-hosted feature flags or CI integrations
- deep provider-specific identity federation beyond OAuth

These are deferred to keep the current design simple and aligned with the control plane’s primary goal: managing workspace access and repository binding reliably.

## Forward Compatibility

### Realistic future providers

The most natural next additions are providers that:

- support OAuth for identity
- offer repository access metadata in a restful API
- expose repository references that can be normalized

That makes providers such as Bitbucket Cloud and potentially GitHub Enterprise Server realistic, pending further evaluation.

### What the design already anticipates

The architecture anticipates future providers by:

- separating provider-specific identity information from internal principals
- storing provider references in a normalized, provider-agnostic form
- keeping OAuth and provider API integration isolated at the edge

### What requires code changes vs no changes

No changes are expected for:

- core reconciliation logic for workspace and repository state
- internal authorization checks once identity is normalized
- restart-safe persistence of provider metadata

Code changes would be expected for:

- provider-specific OAuth integration flows
- provider API clients and repository metadata mapping
- onboarding flows that translate provider-specific claims into internal principals

## Cost and Complexity Guardrails

### Why cost is prioritized

The control plane is intentionally optimized for low operational cost and simplicity. That means minimizing dependencies and reducing recurring provider API usage.

### How provider API usage is minimized

- provider data is fetched only when needed for onboarding and periodic reconciliation
- runtime decisions are made against normalized internal state rather than live provider queries
- short-lived internal tokens avoid long-lived provider sessions

### Why correctness and simplicity outweigh feature breadth

A broader provider surface would increase the chance of subtle bugs and restart-safety issues. The current design prefers a smaller, well-understood set of supported workflows over a larger set of partially supported providers.

## Decision Stability

### Stable long-term choices

- the control plane remains stateless aside from the database
- provider identity remains external to the core domain
- OAuth continues to be an edge integration mechanism
- repository access support remains keyed to basic auth and repo metadata, not provider-specific features

### Allowed changes if constraints shift

- the set of supported providers may expand when the operational audience grows
- provider onboarding and API client implementations may evolve
- provider-specific claims and metadata mapping may be refined as new providers are added

The stability boundary is the core domain model: it should continue to operate on normalized principals and repository references, even as new provider integrations are introduced at the edge.
