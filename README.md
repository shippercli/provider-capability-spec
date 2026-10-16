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
      limitations:
        - "Queue workers are not managed by the provider."
    observability:
      state: partial
      notes: "Only deployment logs are available."
    rollback:
      state: unsupported
    previews:
      state: partial
      limitations:
        - "Preview cleanup requires an explicit provider operation."
    server_lifecycle:
      state: supported
      requirements:
        - "Provider credentials can create and inspect servers."
```

## Provider implementation rules

1. A provider package must declare every canonical key: `app_deploy`,
   `domain_management`, `ssl`, `env`, `databases`, `profiles`,
   `background_workloads`, `observability`, `rollback`, `previews`, and
   `server_lifecycle`.
2. A provider package must validate its manifest with
   `ShipperCli\\Contracts\\CapabilityManifest::from()`.
3. Partial support must include non-empty `notes` or `limitations`.
4. Provider-specific config belongs in the provider package and provider docs.
5. Core docs should describe generic Shipper workflow, not provider quirks.
6. A missing capability should fail clearly rather than behaving implicitly.

## Ownership and credential acceptance criteria

Capability declarations describe supported behavior; they do not establish that a
particular deployment is safe to mutate. These criteria apply across application,
database, workload, preview, and server lifecycle operations. See the proposed
[contracts ownership and credential requirements](https://github.com/shippercli/contracts/pull/5).
Provider releases must document which criteria they satisfy and any remaining
limitations; this specification does not assert that existing releases satisfy them.

### Resource ownership

- Verify persisted Shipper ownership evidence for the exact project/profile before
  updating or deleting an existing resource. A matching name, domain, explicit UUID,
  or generic Shipper marker alone is insufficient.
- Encode scope boundaries unambiguously. Distinct project/profile pairs must not
  share ownership evidence because of delimiters, case folding, slug normalization,
  truncation, or a wildcard sentinel.
- Refuse unmanaged or differently scoped collisions before writes. Do not silently
  retag resources or create a duplicate to bypass an ownership failure. Plan output
  must identify blocked collisions.
- Apply the same scope checks to reconciliation, destroy, and orphan cleanup.
  Missing owned resources may be treated as already removed. Legacy ownership
  formats require an explicit, separately reviewed migration.
- Adoption is per-resource and requires operator authorization for the exact
  identity and scope. The plan must list adopted resources and planned changes;
  `--force` alone is insufficient. Apply rechecks identity/scope and persists
  `adopted` state without converting it to Shipper-created ownership. Destroy
  preserves adopted resources. A child resource under an adopted parent may be
  deleted only when Shipper separately recorded creating that exact child in
  scope. Deleting an adopted resource requires a separate, recorded ownership
  transfer or deletion authorization; parent adoption does not authorize
  recursive deletion. Providers unable to persist and verify this state refuse
  adoption.
- Read environment-based markers as effective assignments, including quoting and
  duplicate-key precedence. Marker-looking text inside multiline values does not
  prove ownership; malformed evidence must fail closed.
- Require a successful, structurally valid response for ownership evidence. Explicit
  failure/error fields override matching content. Missing content is unreadable,
  not an empty environment or proof of resource absence.
- Parse the complete ownership document. A valid marker followed by an unfinished
  quoted value must not authorize mutation, even when the parser returns the
  earlier assignments.

### Database connections

- Deliver host, database, username, and password through the intended application's
  environment or a supported secret reference before deployment.
- Map multiple databases to distinct connection variables or secret references.
  Reject ambiguous mappings and normalized resource-name collisions before writes.
- Generate an independent password for each new database unless shared credentials
  are explicitly configured. Preserve existing passwords and working connection
  settings on re-apply; unavailable credentials require an actionable failure.
- Prove required database/site associations before reuse or cleanup. Preserve
  resources and connection evidence when a prerequisite deletion fails.
- Exclude generated secrets from plans, logs, errors, and debug output.
- Verify persisted selected connection values through the supported read path;
  an acknowledged write alone does not establish delivery. Refuse unreadable,
  incomplete or mismatching readback before dependent operations.
- Deliver each confirmed connection before creating the next database. A fresh
  retry must retain earlier credentials and create only missing resources.
- Run configuration steps sequentially and stop at a failed prerequisite. Document
  ambiguous creation/storage, delayed persistence, partial cleanup and recovery
  procedures. Incremental writes alone provide neither atomicity nor durability.

### Evidence required for review

Provider changes should include regressions for unowned and cross-scope resources,
legacy markers, naming collisions, malformed evidence, re-apply credential
preservation, multiple databases, and exact preview cleanup scope. Tests must
assert the writes that occur and the writes that are refused. When generated tags
are used, exercise their actual format through public listing and cleanup paths.

Mocked API tests, dependency audits, builds, and live provider checks are separate
evidence. Record which were performed and the tested versions; a green manifest
validator or mocked suite alone does not prove a live deployment works.

### Minimum failure-path scenarios

Record the operation trace and resulting resource/connection state for each
applicable scenario. A refusal test must fail when its source guard is disabled,
not merely assert that an exception occurred somewhere in the flow.

| Scenario | Required observable result |
| --- | --- |
| Matching ownership marker plus unfinished trailing value | No update or deletion request; valid multiline control remains accepted. |
| Matching marker in an explicitly failed API response | No ownership-inferred mutation or deletion. |
| Missing environment content versus explicitly empty content | Missing is refused; explicit empty is readable but does not prove ownership. |
| Write acknowledged but selected values absent on fresh read | Failure before dependent configuration or deployment; no secret in diagnostics. |
| First database confirmed, second creation fails | First connection remains available; report the partial outcome. |
| Fresh process retries the partial operation | Reuse confirmed first resource without password rotation; create only missing resources. |
| Configuration prerequisite fails | Later configuration mutations and deployment are not invoked. |

A simulated API can verify serialization and request order; it cannot establish
live database authentication, provider persistence timing, or concurrency safety.
Acceptance evidence must name those limits and leave unverified requirements open.

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
