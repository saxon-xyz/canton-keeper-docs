---
layout: default
title: Installation Guide
---

# Installation Guide

## Prerequisites

- A running Canton validator node with participant API on port 5001
- Auth0 or Keycloak credentials for the Ledger API
- Docker installed on your validator node

## Step 1: Get Your Credentials

You need these values from your validator setup:

| Value | Where to find it | Example |
|-------|-----------------|---------|
| Token URL | Auth0: domain + `/oauth/token`. Keycloak: realm + `/protocol/openid-connect/token` | `https://mynode.uk.auth0.com/oauth/token` or `https://keycloak.example.com/realms/canton/protocol/openid-connect/token` |
| Client ID | Your M2M application / service account settings | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u` |
| Client Secret | Your M2M application / service account settings | `xY9wV8uT7sR6qP5oN4mL3kJ2iH1gF0e...` |
| Ledger API audience | Your participant's API identifier | `https://ledger-api.canton.mynode.example.com` |
| Party ID | Your validator party ID | `mynode-validator-1::1220abcd1234...` |
| Ledger API user | Usually `<client-id>@clients` | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u@clients` |

## Step 2: Choose Your Jobs

Download an example config:

```bash
# For Canton Swap automation
curl -o canton-keeper.yaml {{ site.examples_url }}/canton-swap.yaml

# For DA Utility DAR automation
curl -o canton-keeper.yaml {{ site.examples_url }}/utility-dars.yaml
```

Or write your own — see the [Configuration Reference](config).

## Step 3: Run

```bash
docker run -d \
  --name canton-keeper \
  --restart unless-stopped \
  --network host \
  -v $(pwd)/canton-keeper.yaml:/app/canton-keeper.yaml \
  -v canton-keeper-data:/data \
  -e AUTH0_TOKEN_URL="https://mynode.uk.auth0.com/oauth/token" \
  -e AUTH0_CLIENT_ID="your-client-id" \
  -e AUTH0_CLIENT_SECRET="your-client-secret" \
  -e LEDGER_API_AUDIENCE="https://ledger-api.canton.mynode.example.com" \
  -e VENUE_PARTY="mynode-validator-1::1220abcd..." \
  -e LEDGER_API_USER="your-client-id@clients" \
  -e LEDGER_API_HOST="localhost:5001" \
  ghcr.io/saxon-xyz/canton-keeper:latest
```

## Step 4: Verify

```bash
docker logs canton-keeper
```

You should see CK start, resolve packages, snapshot contracts, and begin streaming:

```
INFO  [keeper] starting — host=localhost:5001 party=mynode-validator-1::1220abcd...
INFO  [keeper] loaded 2 job(s): cancel-expired-proposals, settle-trades
INFO  [packages] scanning 87 package(s) for module "Your.App.Module"
INFO  [packages] found "Your.App.Module" in package 3f8a2b...
INFO  [keeper] resolved Your.App.Module → 3f8a2b...
INFO  [keeper] FeaturedAppRight discovered: 00a1b2c3d4e5f6...  (pkg=7804375f...)
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] ledger end offset: 000000000000001a47
INFO  [streamer] snapshot complete at offset 000000000000001a47, store size: 12
INFO  [streamer] streaming live from offset 000000000000001a47
```

If no FeaturedAppRight is registered for your validator, CK still runs normally:

```
INFO  [keeper] no FeaturedAppRight found — transactions will not earn rewards
```

To see per-poll detail, set `LOG_LEVEL=DEBUG`:

```bash
docker stop canton-keeper
# Re-run the docker run command from Step 3 with -e LOG_LEVEL=DEBUG
```

```
DEBUG [cancel-expired-proposals] tick: 3 contract(s)
DEBUG [streamer] transaction at offset 000000000000001a48, 2 event(s)
```

When a contract matches a trigger, CK exercises the choice:

```
INFO  [cancel-expired-proposals] deadline passed for 007bc94ffd..., exercising Proposal_Cancel
INFO  [cancel-expired-proposals] Proposal_Cancel submitted for 007bc94ffd...
INFO  [store] archived Your.App.Module:Proposal 007bc94ffd...
```

## Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `AUTH0_TOKEN_URL` | Yes | OAuth2 token endpoint | `https://mynode.uk.auth0.com/oauth/token` |
| `AUTH0_CLIENT_ID` | Yes | Service account client ID | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u` |
| `AUTH0_CLIENT_SECRET` | Yes | Service account client secret | `xY9wV8uT7sR6qP5oN4mL3kJ2iH1gF0e...` |
| `LEDGER_API_AUDIENCE` | Yes | Ledger API resource identifier | `https://ledger-api.canton.mynode.example.com` |
| `VENUE_PARTY` | Yes | Operator party ID (Canton format) | `mynode-validator-1::1220abcd1234...` |
| `LEDGER_API_USER` | Yes | Ledger API user (JWT subject) | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u@clients` |
| `LEDGER_API_HOST` | No | Participant address (default: `participant:5001`) | `participant:5001` |
| `KEEPER_CONFIG` | No | Config file path (default: `canton-keeper.yaml`) | `/app/canton-keeper.yaml` |
| `KEEPER_DB_PATH` | No | SQLite state DB path (default: `./keeper-state.db`) | `/data/keeper-state.db` |
| `LOG_LEVEL` | No | `DEBUG`, `INFO`, `WARN`, `ERROR` (default: `INFO`) | `INFO` |

## Updating

```bash
docker pull ghcr.io/saxon-xyz/canton-keeper:latest
docker restart canton-keeper
```

## Updating Your Config

```bash
# Edit canton-keeper.yaml, then restart
docker restart canton-keeper
```

## Health Check

CK exposes an HTTP health endpoint on port 8080:

```bash
curl http://localhost:8080/health
```

```json
{"status":"healthy","streamActive":true,"contractCount":12,"offset":"000000000000001a47"}
```

| Field | Meaning |
|-------|---------|
| `streamActive` | `true` when CK is connected to the participant and receiving updates |
| `contractCount` | Number of contracts CK is tracking across all subscribed templates |
| `offset` | Current ledger offset (increases with each transaction) |

If using Kubernetes, the example manifest includes liveness and readiness probes pointed at this endpoint.

## After a Network Upgrade

When the Canton Network upgrades (e.g. Splice 0.5.17 to 0.5.18), package IDs change because new DAR versions are deployed. CK handles this automatically — it resolves all package IDs at startup by scanning the participant's package store.

**What to do: restart CK.**

```bash
# Docker
docker restart canton-keeper

# Kubernetes
kubectl -n canton rollout restart deployment canton-keeper
```

After restart, check the logs for successful package resolution:

```
INFO  [packages] scanning 94 package(s) for module "Your.App.Module"
INFO  [packages] found "Your.App.Module" in package 9c4d7e...
INFO  [keeper] resolved Your.App.Module → 9c4d7e...
INFO  [streamer] snapshot complete at offset 000000000000002b13, store size: 8
INFO  [streamer] streaming live from offset 000000000000002b13
```

The new package ID (`9c4d7e...` instead of the previous `3f8a2b...`) confirms CK picked up the upgraded packages. No config changes required.

**If a module is not found** after an upgrade, the DAR may not be installed on your participant yet. Check with your network operator.

## Kubernetes

For Kubernetes deployments, download the example manifest:

```bash
curl -o canton-keeper-k8s.yaml {{ site.examples_url }}/k8s-deployment.yaml
```

Then edit the placeholder values and apply:

```bash
kubectl -n canton create configmap canton-keeper-config \
  --from-file=canton-keeper.yaml=canton-keeper.yaml
kubectl -n canton apply -f canton-keeper-k8s.yaml
```

## Troubleshooting

**No jobs fire:** Set `-e LOG_LEVEL=DEBUG` to see poll ticks and contract counts.

**Authentication errors:** Verify your credentials work:
```bash
# Auth0
curl -s -X POST https://mynode.uk.auth0.com/oauth/token \
  -H "content-type: application/json" \
  -d '{"client_id":"...","client_secret":"...","audience":"...","grant_type":"client_credentials"}'

# Keycloak
curl -s -X POST https://keycloak.example.com/realms/canton/protocol/openid-connect/token \
  -d "client_id=...&client_secret=...&grant_type=client_credentials"
```

**Module not found after upgrade:** The DAR hasn't been installed on your participant yet. CK will exit with:
```
fatal: No package found containing module "My.Module.V1"
```
Wait for the DAR to be deployed, then restart CK.

**Stream disconnects:** Normal during participant restarts — CK automatically reconnects within 5 seconds:
```
ERROR [streamer] cycle failed (code=14): 14 UNAVAILABLE: Connection dropped
WARN  [streamer] reconnecting in 5000ms
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 000000000000002b15
```

## Support

Contact Saxon Nodes for support.
