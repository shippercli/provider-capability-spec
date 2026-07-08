# Server Lifecycle Capability

## Goal

Define a provider-agnostic server lifecycle model so Shipper can either deploy to an existing server or create one on demand before deployment.

## Why this capability exists

This unlocks:

- preview environments
- ephemeral staging environments
- deploy-on-demand infrastructure
- cleanup of Shipper-created temporary servers

It also gives panel-based providers such as Ploi a clean way to expose server provisioning without hard-coding provider-specific behavior into Shipper core.

## Capability name

- `server_lifecycle`

## Capability states

- `supported`
- `partial`
- `unsupported`

## Core behaviors

### 1. Resolve target server

Shipper must be able to determine whether a deployment profile:

- uses an existing server
- creates a new server

### 2. Provision server

If configured for creation, the provider must be able to create a server from a provider-supported specification.

### 3. Mark ownership

If Shipper creates a server, it must mark that server as Shipper-managed using provider metadata, naming convention, or both.

### 4. Cleanup

If a server was created by Shipper, the provider may clean it up according to policy.

Cleanup must never apply to unmanaged or pre-existing infrastructure.

## Cleanup policies

- `retain`
- `destroy`
- `manual`
- `suspend`

Notes:

- `destroy` is the best default for previews
- `suspend` is optional and provider-dependent
- `manual` is useful for debugging temporary environments

## Suggested config shape

```yaml
projects:
  api:
    profiles:
      production:
        infrastructure:
          server:
            mode: existing
            id: "123456"

      preview:
        infrastructure:
          server:
            mode: create
            cleanup: destroy
            ttl: 72h
            spec:
              name: "api-pr-${PR_NUMBER}"
              size: "small"
              region: "eu-west"
```

## Provider contract

Providers that support `server_lifecycle` should expose:

- resolve existing server
- create server
- fetch server details
- delete server

Optional:

- restart server
- suspend server
- resume server
- monitoring/status access
- server logs

## Required safety rules

1. Shipper must only auto-delete servers it created itself.
2. Existing servers must never be deleted by cleanup logic.
3. A created server should carry enough metadata to link it to:
   - project
   - profile
   - environment type
   - PR number or preview identifier
   - creation timestamp
4. Providers must fail clearly if cleanup is requested but ownership cannot be proven.

For providers with limited metadata support, a deterministic managed naming convention is acceptable ownership proof for MVP.

## Best first implementation scope

### MVP

- `mode: existing`
- `mode: create`
- `cleanup: destroy`
- ownership tagging

### Phase 2

- `cleanup: manual`
- `cleanup: retain`
- TTL support
- restart/status support

### Phase 3

- suspend/resume
- richer provider-specific provisioning templates

## Fit with other core capabilities

This capability should work alongside:

- `app_deploy`
- `domain_management`
- `ssl`
- `databases`
- `background_workloads`
- `previews`

Server lifecycle should provision the target infrastructure first, then hand off to deployment capabilities.
