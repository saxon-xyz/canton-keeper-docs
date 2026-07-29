---
layout: default
title: Traffic Top-Up
---

# Traffic Top-Up (CIP-0104)

Every transaction consumes synchronizer traffic, and a validator that exhausts its balance cannot submit anything at all until it buys more. So top-up is not an optimisation; it is the difference between a working node and a stalled one.

**Start with the built-in loop.** Splice's validator-app ships its own in-process top-up loop (`ADDITIONAL_CONFIG_TOPUPS`, helm `topup.targetThroughput` + `minTopupInterval`). For keeping a single node's tank full it is the right mechanism and the one to use — it is in the same process as the wallet, it needs no extra moving parts, and in our own operation it does the job.

What an in-process loop structurally *cannot* do is see past its own node. It cannot tell you that one validator in a fleet has a mis-set target, or that its top-up has been silently failing, or that a burn curve has changed shape since the target was chosen. That blind spot is where Saxon Automate's traffic action earns its place: as a **fleet-level watchdog** across many validators, and — where a static target genuinely doesn't fit a bursty burn curve — as a dynamic override on a specific node.

The rest of this page documents the mechanism, so you can judge which of those two roles you actually need.

## How It Works

The buy is split across two actors:

1. **Saxon Automate** exercises `Splice.Wallet.Install:WalletAppInstall_CreateBuyTrafficRequest` via the JSON Ledger API. The choice creates a `BuyTrafficRequest` contract — Saxon Automate is the **scheduler**, not the executor.
2. **The validator-app's internal wallet automation** picks up the `BuyTrafficRequest`, runs coin selection through its `TreasuryService`, exercises `BuyTrafficRequest_Complete` → `AmuletRules_BuyMemberTraffic`. The sequencer applies the new allowance.

This means the validator-app must be running for completion to happen. The wallet's existing automation handles coin selection, disclosed-contracts, and the round-rollover edge cases; Saxon Automate just decides when.

## Why JSON Ledger API, not the wallet HTTP endpoint

The validator-app exposes `POST /api/validator/v0/wallet/buy-traffic-requests` that does the same thing — internally just exercising `WalletAppInstall_CreateBuyTrafficRequest` after a participant-topology lookup. Going JSON Ledger API direct keeps Saxon Automate's submitter pattern unified: one client, one auth model, same path billing uses. The cost is replacing the endpoint's tracking-id dedup with catalog-level interval gating + a deterministic `commandId`, and configuring the receiving participant ID as static env (`MEMBER_ID`) — fine because it rarely changes.

## Decision Logic

Each tick fetches the operator's traffic-status from scan and computes:

```
remaining = total_limit - total_consumed
inFlight  = total_purchased - total_limit   (pending buy not yet applied)
```

- If `inFlight > maxInFlight` → skip (a buy is already in progress).
- If `remaining >= threshold` → skip (no action needed).
- Otherwise → fire (in live mode) or shadow-log (otherwise).

A per-`(validator, member, migration)` dedup window (`minIntervalMs`) prevents multiple ticks from firing the same buy in rapid succession.

## Safety: Two Opt-In Gates

The job is **paused by default in the built-in catalog** so first-deploy doesn't fire surprise purchases. Two explicit gates must both be flipped to actually start firing in production:

1. **Unpause the job** in `saxon-automate.yaml`:
   ```yaml
   jobs:
     topup-validator-traffic:
       paused: false
       trigger: interval
       watch:
         module: Splice.AmuletRules
         entity: AmuletRules
       every: 300000   # 5 min
       action:
         type: imported
         package: "@saxon-xyz/saxon-automate-daemon"
         function: runTrafficTopup
         args:
           threshold: 5000000      # fire when remaining < 5 MB
           topupAmount: 20000000   # buy 20 MB per fire
           maxInFlight: 1000000    # skip if pending buy > 1 MB
           minIntervalMs: 60000    # at least 1 min between fires
           expiresInSec: 600       # request auto-expires if uncompleted
   ```

2. **Set `TRAFFIC_TOPUP_ALLOW_LIVE=true`** in the daemon's env. Without it, the job ticks in **shadow mode** — logs the decision it would make but never submits. Useful for verifying threshold/topup tuning is sane before committing to real spends.

Recommended workflow: unpause first, watch the shadow-mode logs for a few hours/days against your validator's real burn curve, then flip live.

## Required Env

In addition to the standard Saxon Automate credentials:

| Variable | Description | Example |
|---|---|---|
| `LEDGER_API_URL` | JSON Ledger API base | `http://participant:7575` |
| `SCAN_URL` | Scan API base for traffic-status | `https://scan.sv-1.example.com` |
| `MEMBER_ID` | Receiving participant's member ID | `PAR::mynode-validator-1::1220abcd...` |
| `SYNCHRONIZER_ID` | Daml synchronizer ID | `global-domain::1220abcd...` |
| `MIGRATION_ID` | Domain migration ID (often `0` or `1`) | `1` |
| `TRAFFIC_TOPUP_ALLOW_LIVE` | `true` to leave shadow mode | `true` |

The action reuses the daemon's existing `AUTH0_*` / `LEDGER_API_AUDIENCE` credentials.

### Finding the cluster-specific values

- **`MEMBER_ID`** — the participant's member ID on the sequencer. Format: `PAR::<participant-name>::<namespace-fingerprint>`. Find via the participant admin API (`GetParticipantId`) or — quicker — by inspecting your own validator's traffic-status response from scan: any member-id that returns data is yours.
- **`SYNCHRONIZER_ID`** — from your validator-app's config or via the JSON Ledger API at `/v2/state/connected-synchronizers`.
- **`MIGRATION_ID`** — from your validator-app's helm/config (`canton.validator-apps.validator_backend.domain-migration-id` or `ADDITIONAL_CONFIG_MIGRATION_ID`). It increments with each synchronizer migration, so it is **not** a small fixed number — MainNet has already migrated several times. Read it from your own config rather than assuming; a stale migration id makes the buy target the wrong synchronizer.

## Tuning the Gating Params

The catalog defaults (5 MB threshold, 20 MB topup) are conservative — they suit a quiet validator that occasionally needs to keep its tank from running dry. Hot validators burning megabytes per minute should run with much larger values:

| Validator profile | `threshold` | `topupAmount` | `every` |
|---|---|---|---|
| Quiet (catalog default) | 5 MB | 20 MB | 5 min |
| Moderate volume | 20 MB | 100 MB | 2 min |
| Hot validator | 50 MB | 200 MB | 1 min |

The right pattern is small-but-frequent rather than rare-and-huge: the network's recommended topup config keeps purchase bounds tight to avoid wasting CC on unused allocations.

## Coexistence With the Built-In Topup Loop

**Our recommendation is to keep the built-in loop enabled.** It is the mechanism Splice ships, it lives in the same process as the wallet automation that completes the buy, and for per-node top-up it is sufficient. We run it.

Two caveats are worth knowing, because they explain when an external actor helps:

- Under heavy, bursty load, in-process block top-ups can contend on the same state rows and produce serialization-retry log noise.
- `target-throughput` is a static target. If a node's burn curve is spiky rather than steady, a single figure is either too low during peaks or wasteful the rest of the time.

Where either bites on a specific node, Saxon Automate's action can run alongside as a dynamic layer — the gates below (`minIntervalMs`, `maxInFlight`) exist so that two actors requesting buys don't stampede. Its more valuable role, though, is **fleet oversight**: one watcher across many validators that notices the node whose top-up has stopped working, which no per-node loop can do for itself.

**Do not disable the built-in loop simply because Saxon Automate is running.** An earlier version of this page suggested that as a long-term goal; our own operational experience since does not support it. Removing the in-process safety net leaves a node dependent on an external daemon being healthy, which trades a small inefficiency for a real single point of failure.

## Gotchas

These come up rarely but cost time when they do:

- **Daml `Int` must be JSON-string-encoded in choice args.** `migrationId` and `trafficAmount` sent as raw JSON numbers fail with `HTTP 500 "Expected ujson.Str (data: …)"`. Saxon Automate's submitter handles this internally; mentioned here for anyone reading the wire format or writing a similar integration.
- **Scan's `target.total_purchased` lags `actual.total_limit` by ~60-90s after a buy completes.** The `inFlight = purchased - limit` formula can be briefly *negative* right after a successful topup completes (sequencer applies the new allowance before scan's purchased counter refreshes). Saxon Automate's decision gate (`inFlight > maxInFlight`) handles this correctly; code that asserts `inFlight >= 0` will spuriously fire.

## Cost, honestly

A `WalletAppInstall_CreateBuyTrafficRequest` submission is itself a Daml transaction, so for a registered Featured App it is reward-eligible like any other. **That does not make buying traffic profitable, and it should not be presented as though it does.**

The purchase spends Canton Coin to acquire traffic; the reward earned on the single submission that requests it is small by comparison. Traffic is a real cost of operating, frequently the largest one for a busy validator — on transfer-heavy workloads it can consume the majority of gross rewards. The case for automating top-up is **avoiding the outage** that occurs when a validator's traffic runs dry and it can submit nothing at all, plus buying in small, frequent, well-sized amounts rather than over-provisioning allowances that go unused.

Budget for traffic as a cost line. See [Rewards](rewards) for how reward size is actually determined, including the per-round minimum below which a round pays nothing.
