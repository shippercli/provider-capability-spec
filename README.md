# Core Provider Capability Spec

## Goal

Define the minimum shared provider contract for Shipper so providers can remain independent packages while presenting a consistent user experience.

## Capability states

Every provider should declare each capability as one of:

- `supported`
- `partial`
- `unsupported`

Optional metadata:

- `notes`
- `requirements`
- `limitations`

## Core provider contract

### 1. Application deployment

Provider can create or update a deployed application target.

Expected concerns:

- application identifier
- deploy target
- release trigger
- runtime status

### 2. Domain management

Provider can attach, update, or validate domains.

Expected concerns:

- primary domain
- additional domains
- staging domains
- DNS assumptions

### 3. SSL management

Provider can provision or manage certificates.

Expected concerns:

- automated certificate issuance
- renewal behavior
- provider limitations

### 4. Environment configuration

Provider can manage runtime configuration values.

Expected concerns:

- environment variables
- secrets
- per-environment overrides

### 5. Database lifecycle

Provider can provision, link, or reference databases.

Expected concerns:

- create database
- link existing database
- credential management
- per-profile database mapping

### 6. Deployment profiles

Provider can distinguish production, staging, preview, or custom deployment contexts.

Expected concerns:

- branch mapping
- domain overrides
- database overrides
- provider-specific profile settings

### 7. Background workloads

Provider can manage workers, daemons, jobs, or similar non-web workloads.

Expected concerns:

- queue workers
- long-running processes
- scheduled tasks

### 8. Observability

Provider exposes enough status for Shipper to report useful deployment feedback.

Expected concerns:

- deployment status
- logs access
- health or readiness signals

### 9. Rollback and recovery

Provider supports reversing or recovering from a failed deployment.

Expected concerns:

- rollback support
- release history
- partial recovery workflow

### 10. Preview environments

Provider supports ephemeral or branch-specific environments when possible.

Expected concerns:

- naming strategy
- temporary domains
- cleanup behavior

## Suggested manifest shape

```yaml
provider:
  slug: "example"
  capabilities:
    app_deploy:
      state: supported
    domain_management:
      state: supported
    ssl:
      state: partial
      notes: "Manual attachment required for wildcard domains."
    env:
      state: supported
    databases:
      state: supported
    profiles:
      state: supported
    background_workloads:
      state: partial
    observability:
      state: partial
    rollback:
      state: unsupported
    previews:
      state: partial
```

## Provider implementation rules

1. A provider package must declare capability states explicitly.
2. Partial support must explain limitations.
3. Provider-specific config belongs in the provider package and provider docs.
4. Core docs should describe generic Shipper workflow, not provider quirks.
5. A missing capability should fail clearly rather than behaving implicitly.

## Documentation split

### Core docs should cover

- installation
- general configuration structure
- project and profile concepts
- deployment workflow
- planning and applying changes

### Provider docs should cover

- credentials
- provider-specific fields
- capability support matrix
- provider limitations
- example configurations

## Why this matters

Without a shared capability spec:

- providers drift in behavior
- docs become provider-specific too early
- users cannot predict which workflows work where
- “provider-agnostic” becomes marketing rather than a system property
