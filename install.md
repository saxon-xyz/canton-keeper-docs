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

You need these values from your validator setup:

| Value | Where to find it | Example |
|-------|-----------------|---------|
| Auth0 token URL | Your Auth0 domain + `/oauth/token` | `https://mynode.uk.auth0.com/oauth/token` |
| Client ID | Your Auth0 M2M application settings | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u` |
| Client Secret | Your Auth0 M2M application settings | `xY9wV8uT7sR6qP5oN4mL3kJ2iH1gF0e...` |
| Ledger API audience | Your participant's Auth0 API identifier | `https://ledger-api.canton.mynode.example.com` |
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

You should see:

```
INFO  [keeper] starting — host=localhost:5001 party=mynode-validator-1::1220...
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

## Test It

Check CK is running and connected:

```bash
docker logs canton-keeper
```

You should see:
```
INFO  [keeper] starting — host=localhost:5001 party=mynode-validator-1::1220...
INFO  [keeper] loaded N job(s): ...
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 0
```

To see CK actively processing contracts, set `LOG_LEVEL=DEBUG`:

```bash
docker stop canton-keeper
# Re-run the docker run command from Step 3 with -e LOG_LEVEL=DEBUG
```

You'll see tick logs showing how many contracts CK is watching:
```
DEBUG [my-job] tick: 3 contract(s)
```

When a contract matches a trigger condition, CK will exercise the configured choice and you'll see:
```
INFO  [my-job] deadline passed for 00abcd..., exercising MyChoice
INFO  [my-job] MyChoice submitted for 00abcd...
INFO  [store] archived My.Module:MyTemplate 00abcd...
```

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
