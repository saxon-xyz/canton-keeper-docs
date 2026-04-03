---
layout: default
title: Security
---

# Security Model

Canton Keeper runs with full `actAs` authority on the validator's operator party. This section explains what that means and how to operate CK securely.

## What CK Can Do

CK authenticates to the Ledger API using the same credentials as your validator. It can exercise any choice that your operator party is authorised to exercise. This is by design — it needs this access to automate contract operations on your behalf.

## Credential Security

CK requires Auth0 or Keycloak M2M credentials. Protect these as you would any service account with ledger access:

- **Docker**: pass credentials via `-e` flags or Docker secrets. Do not bake them into images.
- **Kubernetes**: use Kubernetes secrets (referenced in the example manifest). Do not store in ConfigMaps.
- **Never commit credentials** to version control.

## Configuration Security

The YAML config file determines which choices CK exercises. Anyone who can modify the config controls what CK does on the ledger.

- Restrict write access to `canton-keeper.yaml` to trusted operators.
- On Kubernetes, the ConfigMap should only be editable by cluster admins.
- Review config changes before applying — each job entry is a standing instruction to exercise a choice.

## Image Integrity

CK is distributed as a public Docker image from `ghcr.io/saxon-xyz/canton-keeper`. The image is built automatically by GitHub Actions from the private source repository.

- Verify the image digest matches the CI build output before deploying to production.
- Pin to a specific SHA tag rather than `:latest` for production deployments:
  ```
  ghcr.io/saxon-xyz/canton-keeper@sha256:abc123...
  ```

## Network Access

CK connects to the participant's Ledger API (default port 5001) via gRPC. It also makes outbound HTTPS calls to the Auth0/Keycloak token endpoint.

- CK does not expose any inbound ports to the network (the health endpoint on 8080 is optional and internal).
- CK does not send contract data or ledger content to any external service.
- The only outbound calls are token requests to your configured auth provider.

## Rate Limiting

CK is polling-based with configurable intervals (default 10 seconds) and jitter. It does not react to events in real-time, providing a natural rate limit on ledger interactions.

## Reporting Issues

If you discover a security issue, contact Saxon Nodes directly rather than opening a public issue.
