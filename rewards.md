---
layout: default
title: Rewards
---

# Rewards

Saxon Automate increases the volume of **rewarded operations** your validator performs. It fires contract-lifecycle choices the moment conditions are met, so work that would otherwise sit undone — and unrewarded — gets done, 24/7.

## How rewards work on Canton

Two reward streams matter here:

- **Validator rewards** — earned by the validator operator for participating in the network. You earn these as a validator; the more activity that happens on your node, the more of this work is captured.
- **App rewards** — earned by **Featured Apps** for the transactions they drive. This is the stream automation amplifies: every automated settle, execute, cancel, and bill is a transaction your node wouldn't have submitted manually.

## App rewards aren't automatic

To earn app rewards, a party must be a registered **Featured App**, which takes two things:

1. **A `FeaturedAppRight`** — the on-ledger grant that makes your app's transactions reward-eligible. **Today, app rewards flow through this mechanism**: the featured app marks its activity and earns from the app reward pool.
2. **A Canton Coin stake (CIP-0116)** — to activate and maintain Featured-App (App-provider) status, you lock a stake in Canton Coin. Without the lock, the status doesn't take effect.

> **The model is changing.** CIP-0104 (traffic-based app rewards) is **approved but not yet live**, and it arrives in increments rather than in one step. When the reward switch lands, app rewards will be computed from each transaction's synchronizer traffic rather than from explicit activity markers, carried by a new reward-coupon contract. Until then, the `FeaturedAppRight` mechanism above is what applies. Saxon tracks these changes and keeps the automation aligned.

## What governs the size of the reward

Three things determine what a rewarded transaction is actually worth, and **none of them are set by your app**:

- **A per-round minimum.** If an app's reward for a given round falls below a threshold — a network-set parameter, defaulted on the order of $0.50 per round — that round pays **nothing**, rather than paying a small amount. For a low-velocity application this is the dominant effect: most rounds can fall below the line, so the stream is not merely small but frequently zero. Automation helps here precisely because concentrating activity into rounds that clear the threshold is worth more than the same activity spread thinly.
- **A per-app activity weight.** Set by network governance, not by the application.
- **Sharing across confirming apps.** Where more than one Featured App confirms the same transaction, reward credit for it is divided between them. An asset issuer that must confirm every transfer of its own asset is therefore a co-earner on those transfers by construction.

## Be realistic about traffic

Rewards are gross, not net. Every transaction consumes synchronizer traffic, which must be bought with Canton Coin, so the honest measure of an automation's value is **reward earned minus traffic consumed**.

For low-traffic workloads — issuance, lifecycle management, periodic settlement — that difference is comfortably positive, which is why issuer and registry applications suit this model well. For high-frequency, transfer-heavy workloads it can invert: traffic spend can approach or exceed reward earned, and busy applications have run net-negative on the network. The CIP-0104 increments improve this, free confirmations most directly, but the arithmetic still has to be done per workload rather than assumed.

Saxon will model this with you against your expected volume before you build around it.

## Why it matters now

The Featured App reward pool is **front-loaded toward active applications** in the network's early years — while the pool is large and competition is low, the rewarded activity your node drives translates into a larger share. Saxon Automate's role is to make sure you're capturing that activity automatically, not leaving it on the table.

## Keep your traffic topped up

Committing transactions consumes synchronizer traffic, so a node that earns through activity also has to keep its traffic balance funded. See [Traffic Top-Up](traffic-topup) for the automated top-up flow.
