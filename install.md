---
layout: default
title: Installation Guide
---

# Installation Guide

## Prerequisites

- A running Canton validator node with participant API on port 5001
- Auth0 M2M credentials for the Ledger API
- Docker installed on your validator node

## Step 1: Get Your Credentials

You need four values from your validator setup:

| Value | Where to find it |
|-------|-----------------|
| Auth0 token URL | `https://<your-domain>.auth0.com/oauth/token` |
| Client ID + Secret | Your Auth0 M2M application settings |
| Ledger API audience | Your participant's Auth0 API identifier |
| Party ID | Your validator party (e.g. `mynode-various-1::1220...`) |
| Ledger API user | Usually `<client-id>@clients` |

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
  -e VENUE_PARTY="mynode-various-1::1220abcd..." \
  -e LEDGER_API_USER="your-client-id@clients" \
  -e LEDGER_API_HOST="localhost:5001" \
  -e DEVELOPER_PARTY="saxon-various-1::1220abcd..." \
  ghcr.io/saxon-xyz/canton-keeper:latest
```

## Step 4: Verify

```bash
docker logs canton-keeper
```

You should see:

```
INFO  [keeper] starting — host=localhost:5001 party=mynode-various-1::1220...
INFO  [keeper] loaded 2 job(s): cancel-expired-proposals, settle-trades
INFO  [keeper] FeaturedAppRight discovered: ...
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 0
```

## Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `AUTH0_TOKEN_URL` | Yes | OAuth2 token endpoint | `https://mynode.uk.auth0.com/oauth/token` |
| `AUTH0_CLIENT_ID` | Yes | Service account client ID | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u` |
| `AUTH0_CLIENT_SECRET` | Yes | Service account client secret | `xY9wV8uT7sR6qP5oN4mL3kJ2iH1gF0e...` |
| `LEDGER_API_AUDIENCE` | Yes | Ledger API resource identifier | `https://ledger-api.canton.mynode.example.com` |
| `VENUE_PARTY` | Yes | Operator party ID (Canton format) | `mynode-various-1::1220abcd1234...` |
| `LEDGER_API_USER` | Yes | Ledger API user (JWT subject) | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u@clients` |
| `LEDGER_API_HOST` | No | Participant address (default: `participant:5001`) | `participant:5001` |
| `KEEPER_CONFIG` | No | Config file path (default: `canton-keeper.yaml`) | `/app/canton-keeper.yaml` |
| `KEEPER_DB_PATH` | No | SQLite state DB path (default: `./keeper-state.db`) | `/data/keeper-state.db` |
| `LOG_LEVEL` | No | `DEBUG`, `INFO`, `WARN`, `ERROR` (default: `INFO`) | `INFO` |
| `DEVELOPER_PARTY` | No | Saxon party ID for reward split (see [Rewards](rewards)) | `saxon-various-1::1220abcd1234...` |

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

## Test It

To verify CK is working end-to-end, generate test traffic. This creates a short-lived TradeProposal that CK automatically cancels after 30 seconds — producing a real ledger transaction.

```bash
docker run --rm --network host \
  -e AUTH0_TOKEN_URL="https://mynode.uk.auth0.com/oauth/token" \
  -e AUTH0_CLIENT_ID="your-client-id" \
  -e AUTH0_CLIENT_SECRET="your-client-secret" \
  -e LEDGER_API_AUDIENCE="https://ledger-api.canton.mynode.example.com" \
  -e VENUE_PARTY="mynode-various-1::1220abcd..." \
  -e LEDGER_API_USER="your-client-id@clients" \
  -e LEDGER_API_HOST="localhost:5001" \
  ghcr.io/saxon-xyz/canton-keeper:latest \
  node dist/scripts/generate-traffic-k8s.js
```

Then check CK's logs:

```bash
docker logs canton-keeper --since 2m
```

You should see:

```
INFO  [store] created Obsidian.CantonSwap.V1:TradeProposal 0076f7e5...
INFO  [cancel-expired-proposals] deadline passed for 0076f7e5..., exercising TradeProposal_Cancel
INFO  [cancel-expired-proposals] TradeProposal_Cancel submitted for 0076f7e5...
INFO  [store] archived Obsidian.CantonSwap.V1:TradeProposal 0076f7e5...
```

Requires the canton-swap DAR deployed on your node and the `canton-swap.yaml` example config.

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

**Authentication errors:** Verify your Auth0 credentials work:
```bash
curl -s -X POST https://mynode.uk.auth0.com/oauth/token \
  -H "content-type: application/json" \
  -d '{"client_id":"...","client_secret":"...","audience":"...","grant_type":"client_credentials"}'
```

**Stream disconnects:** Normal — CK automatically reconnects within 5 seconds.

## Support

Contact Saxon Nodes for support.
