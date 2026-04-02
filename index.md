---
layout: default
title: Canton Keeper
---

# Canton Keeper

Config-driven automation daemon for Canton Network validators. Watches Daml contracts on the ledger and exercises choices based on YAML-defined triggers — deadlines, multi-leg settlement matching, contract existence, and periodic intervals.

## Why You Need It

Canton's privacy model means no third party can observe your contracts or act on your behalf. Unlike public blockchains where services like Chainlink Keepers can automate transactions for anyone, Canton requires automation to run on your own node with your own credentials. Every validator that wants automated settlement, expiry management, or recurring billing needs its own keeper daemon.

Canton Keeper solves this: one daemon, configurable via YAML, that handles any Daml contract lifecycle automation. No custom code per workflow, no Daml expertise required to operate.

CK follows the same automation patterns used internally by [Splice](https://github.com/hyperledger-labs/splice) — polling jitter to prevent thundering herd across validators, silent retries for transient failures, and graceful reconnection on stream interruptions.

**Financial benefits:**
- **Earn Canton Coin rewards** — every transaction CK submits is tagged as a Featured App activity, earning rewards from the Canton Network reward pool
- **Maximize transaction volume** — automated choices fire immediately when conditions are met, generating more rewarded transactions than manual operation
- **Reduce operational cost** — no manual monitoring or intervention needed for routine contract lifecycle operations
- **Featured App reward pool** — significantly favours active applications until mid-2029, making early adoption especially valuable

## What It Does

Canton Keeper runs alongside a Canton participant node and automates contract lifecycle operations that would otherwise require manual intervention or custom code. Operators define automation rules in a YAML config file.

**Example automations:**
- Cancel expired trade proposals when their deadline passes
- Settle multi-leg trades when all allocations are present
- Execute accepted mints and transfers automatically
- Process recurring subscription payments
- Bill fees on commercial agreements at regular intervals
- Clean up audit records (settled DVPs, failed transfers)

Every transaction CK submits earns Canton Coin rewards under the Featured App program — 80% to the validator, 20% to Saxon Nodes.

## Example Output

```
INFO  [keeper] starting — host=participant:5001 party=mynode-validator-1::1220abcd...
INFO  [keeper] loaded 2 job(s): cancel-expired-proposals, settle-trades
INFO  [streamer] snapshotting active contracts...
INFO  [cancel-expired-proposals] job started (trigger=deadline), polling every 10000ms
INFO  [settle-trades] job started (trigger=match), polling every 10000ms
INFO  [streamer] streaming live from offset 0
...
INFO  [store] created Your.App.Module:Proposal 00234042dbb64c32...
INFO  [cancel-expired-proposals] deadline passed for 00234042dbb64c32..., exercising Proposal_Cancel
INFO  [cancel-expired-proposals] Proposal_Cancel submitted for 00234042dbb64c32...
INFO  [store] archived Your.App.Module:Proposal 00234042dbb64c32...
```

## Quick Start

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

See the full [Installation Guide](install) for details.

## Pages

- [Installation Guide](install) — Step-by-step setup for Docker and Kubernetes
- [Configuration Reference](config) — Trigger types, field paths, and argument expressions
- [Example Configs](examples) — Ready-made configs for Canton Swap, DA Utility DARs, and Cantara
- [Canton Coin Rewards](rewards) — How CK earns rewards and the 80/20 split

## Support

Contact Saxon Nodes for support.
