---
layout: default
title: Saxon Automate
---

# Saxon Automate

**The automation and operations layer for a Canton Network validator.** Saxon Automate scans your participant's installed DARs, auto-discovers what to automate, and drives the recurring on-chain work a live validator needs — settling trades, managing expiries, billing agreements, buying traffic, onboarding users, serving ledger reads — on triggers, not manual intervention. Zero config to start; YAML when you want control.

## Why You Need It

Canton's privacy model means no third party can observe your contracts or act on your behalf. Unlike public blockchains where services like Chainlink Keepers automate transactions for anyone, Canton requires automation to run on your own node with your own credentials. Every validator that wants automated settlement, expiry management, recurring billing, or traffic management needs it running on its own participant.

Saxon Automate follows the same automation patterns used internally by [Splice](https://github.com/hyperledger-labs/splice) — polling jitter to prevent thundering herd across validators, silent retries for transient failures, and graceful reconnection on stream interruptions — and extends them across the operational surface a validator actually hits in production.

## What It Does

Five capabilities, each on its own page:

1. **Contract-lifecycle automation** — auto-discover installed apps and fire choices on triggers (`deadline`, `interval`, `exists`, `match`): settle multi-leg trades, execute accepted mints and transfers, cancel expired proposals, bill agreements at intervals, clean up audit records. Override or extend with YAML or [imported actions](imported-actions); on-chain payment via the [Splice CIP-56 transfer factory](imported-actions#settlement-via-the-cip-56-token-standard) is supported.
2. **Traffic & liveness** — keep the node able to submit. Per-node [auto-top-up](traffic-topup) buys CIP-0104 synchronizer traffic before the balance runs out; the [fleet-watchdog](fleet-watchdog) adds a cross-member rescue layer so one validator can fund another before it stalls.
3. **Revenue-grade settlement** — automated periodic on-chain settlement of fees or revenue share, with a revenue-assurance forecast that checks the paying wallet can cover the next settle before it comes due.
4. **External-party onboarding** — [stand up self-custody (external) parties](external-party-onboarding) for your end users at signup via a simple API; the user's key never leaves your custody.
5. **Ledger reads** — two complementary read layers: the [Query API](query-api) over PQS (indexed SQL/HTTP, full history) and the [Ledger Follower](ledger-follower) (in-memory ACS mirror, single-digit-millisecond live reads).

See [Services](services) for how these divide up and which you need.

**Financial benefits:**
- **Earn Canton Coin rewards** — for a registered Featured App, the transactions Saxon Automate submits earn Canton Coin rewards from the network reward pool. Saxon Automate keeps that rewarded volume flowing automatically.
- **Maximize transaction volume** — automated choices fire immediately when conditions are met, generating more rewarded transactions than manual operation.
- **Reduce operational cost** — no manual monitoring or intervention needed for routine contract lifecycle operations.
- **Featured App program** — the reward pool is front-loaded toward active applications in the network's early years. Featured App status isn't automatic, though: it requires a `FeaturedAppRight` and, under CIP-0116, locking a Canton Coin stake (App-provider tier) to activate and maintain reward eligibility. Saxon helps you set this up.

How large those rewards actually are is set by the network, not by your app, and traffic is a real cost against them — [Rewards](rewards) covers what governs the size, including the per-round minimum below which a round pays nothing.

## Example Output

```
INFO  [automate] starting — host=participant:5001 party=mynode-validator-1::1220abcd...
INFO  [automate] loaded 0 manual job(s)
INFO  [discover] scanning 87 package(s) against 4 catalog app(s)
INFO  [discover] found "utility-dars" (detected module: Utility.Settlement.App.V1.Model.Dvp)
INFO  [discover] discovered 6 job(s) from 1 app(s)
INFO  [automate] auto-discovered 1 app(s): utility-dars
INFO  [automate] total 6 job(s): settle-dvp, execute-accepted-mints, ...
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 000000000000001a47
...
INFO  [settle-dvp] deadline passed for 00234042dbb64c32..., exercising Dvp_Settle
INFO  [settle-dvp] Dvp_Settle submitted for 00234042dbb64c32...
```

## Quick Start

Just provide your credentials. Saxon Automate auto-discovers everything else:

```bash
docker run -d \
  --name saxon-automate \
  --restart unless-stopped \
  --network host \
  -v saxon-automate-data:/data \
  -e AUTH0_TOKEN_URL="https://mynode.uk.auth0.com/oauth/token" \
  -e AUTH0_CLIENT_ID="your-client-id" \
  -e AUTH0_CLIENT_SECRET="your-client-secret" \
  -e LEDGER_API_AUDIENCE="https://ledger-api.canton.mynode.example.com" \
  -e VENUE_PARTY="mynode-validator-1::1220abcd..." \
  -e LEDGER_API_USER="your-client-id@clients" \
  -e LEDGER_API_HOST="localhost:5001" \
  ghcr.io/saxon-xyz/saxon-automate:latest
```

No config file needed. Saxon Automate scans your participant for installed apps and generates jobs automatically. Add a `saxon-automate.yaml` mount only if you need to override or extend.

See the full [Installation Guide](install) for details.

## Pages

- [Installation Guide](install) — Step-by-step setup for Docker and Kubernetes
- [Services](services) — The service surfaces at a glance, and which one fits your need
- [Configuration Reference](config) — Trigger types, field paths, and argument expressions
- [Imported Actions](imported-actions) — Plug in custom JS/TS functions for workloads that don't fit a single choice exercise (multi-step orchestration, ledger-derived choice args, CIP-56 settlement)
- [Traffic Top-Up](traffic-topup) — Auto-purchase CIP-0104 synchronizer traffic when the operator's balance runs low
- [Fleet-Watchdog](fleet-watchdog) — Fleet-level liveness: cross-member traffic rescue and a Canton Coin balance alarm
- [External-Party Onboarding](external-party-onboarding) — Onboard self-custody (external) end-user parties at signup via a simple API
- [Query API (PQS)](query-api) — Query your validator's ledger state in SQL/HTTP, incrementally indexed (history and reporting)
- [Ledger Follower](ledger-follower) — Fast, in-memory current-state reads with runtime template/schema discovery; the live counterpart to the PQS Query API
- [Example Configs](examples) — Ready-made configs for DA Utility DARs, Cantara, BitSafe CBTC, and traffic top-up
- [Canton Coin Rewards](rewards) — How Saxon Automate earns rewards, and how the CIP-0104 model changes them
- [Operator Tips](operator-tips) — Canton/Splice platform quirks worth knowing
- [Security](security) — Credential handling, blast radius, and what the daemon can and cannot do
- [Roadmap](roadmap) — Shipped, active, planned

## Support

Contact Saxon Nodes for support.
