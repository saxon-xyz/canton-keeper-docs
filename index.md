---
layout: default
title: Canton Keeper
---

# Canton Keeper

Zero-config automation daemon for Canton Network validators. Scans your participant's installed DARs, auto-discovers what to automate, and exercises choices based on triggers — deadlines, settlement matching, contract existence, and periodic intervals. No YAML required to get started.

## Why You Need It

Canton's privacy model means no third party can observe your contracts or act on your behalf. Unlike public blockchains where services like Chainlink Keepers can automate transactions for anyone, Canton requires automation to run on your own node with your own credentials. Every validator that wants automated settlement, expiry management, or recurring billing needs its own keeper daemon.

Canton Keeper solves this: one daemon that auto-discovers installed apps and handles any Daml contract lifecycle automation. Zero config to get started, YAML overrides when you need control. No custom code, no Daml expertise required.

CK follows the same automation patterns used internally by [Splice](https://github.com/hyperledger-labs/splice) — polling jitter to prevent thundering herd across validators, silent retries for transient failures, and graceful reconnection on stream interruptions.

**Financial benefits:**
- **Earn Canton Coin rewards** — every transaction CK submits is tagged as a Featured App activity, earning rewards from the Canton Network reward pool
- **Maximize transaction volume** — automated choices fire immediately when conditions are met, generating more rewarded transactions than manual operation
- **Reduce operational cost** — no manual monitoring or intervention needed for routine contract lifecycle operations
- **Featured App reward pool** — significantly favours active applications until mid-2029, making early adoption especially valuable

## What It Does

Canton Keeper runs alongside a Canton participant node and automates contract lifecycle operations that would otherwise require manual intervention or custom code. CK auto-discovers installed apps from a built-in catalog of known Daml templates. Operators can override or extend with YAML config.

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
INFO  [keeper] loaded 0 manual job(s)
INFO  [discover] scanning 87 package(s) against 4 catalog app(s)
INFO  [discover] found "utility-dars" (detected module: Utility.Settlement.App.V1.Model.Dvp)
INFO  [discover] discovered 6 job(s) from 1 app(s)
INFO  [keeper] auto-discovered 1 app(s): utility-dars
INFO  [keeper] total 6 job(s): settle-dvp, execute-accepted-mints, ...
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 000000000000001a47
...
INFO  [settle-dvp] deadline passed for 00234042dbb64c32..., exercising Dvp_Settle
INFO  [settle-dvp] Dvp_Settle submitted for 00234042dbb64c32...
```

## Quick Start

Just provide your credentials. CK auto-discovers everything else:

```bash
docker run -d \
  --name canton-keeper \
  --restart unless-stopped \
  --network host \
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

No config file needed. CK scans your participant for installed apps and generates jobs automatically. Add a `canton-keeper.yaml` mount only if you need to override or extend.

See the full [Installation Guide](install) for details.

## Pages

- [Installation Guide](install) — Step-by-step setup for Docker and Kubernetes
- [Configuration Reference](config) — Trigger types, field paths, and argument expressions
- [Example Configs](examples) — Ready-made configs for Canton Swap, DA Utility DARs, and Cantara
- [Canton Coin Rewards](rewards) — How CK earns rewards and the 80/20 split

## Support

Contact Saxon Nodes for support.
