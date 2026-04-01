---
layout: default
title: Canton Coin Rewards
---

# Canton Coin Rewards

Canton Keeper automatically discovers `FeaturedAppRight` contracts at startup. When found, every transaction includes a `FeaturedAppRight_CreateActivityMarker` command, tagging it for Canton Coin reward eligibility under the Featured App program.

## How It Works

1. CK scans your active contracts at startup for a `FeaturedAppRight` where `provider` matches your `VENUE_PARTY`
2. If found, every choice CK exercises includes an activity marker in the same transaction
3. The Canton Network rewards pool distributes Canton Coins to marked transactions

No manual configuration of contract IDs is needed.

## Reward Split

| Recipient | Share | Description |
|-----------|-------|-------------|
| Validator operator | 80% | The party running CK (`VENUE_PARTY`) |
| Saxon Nodes | 20% | CK developer fee, consistent with NAAS pricing |

On Saxon's own nodes, 100% goes to Saxon.

The split is controlled by the `DEVELOPER_PARTY` environment variable:
- **Third-party validators:** Set to Saxon's party ID for your network — your node gets 80%, Saxon gets 20%
- **Saxon's own nodes:** Leave unset or set to the same as `VENUE_PARTY` — 100% to Saxon

## Enabling Featured App Status

### DevNet

Self-feature through the Canton wallet UI:
1. Open your validator's wallet
2. Navigate to Featured Apps
3. Enable Featured App status

CK will detect the `FeaturedAppRight` contract on its next restart.

### Mainnet

Featured App status requires approval from the Canton Foundation's Tokenomics committee. Contact Saxon Nodes for guidance.

## Without Featured App Status

CK runs normally without a `FeaturedAppRight` contract — it just won't tag transactions for rewards. You'll see this at startup:

```
INFO  [keeper] no FeaturedAppRight found — transactions will not earn rewards
```
