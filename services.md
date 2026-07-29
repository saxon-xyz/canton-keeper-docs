---
layout: default
title: Services
---

# Services

Saxon operates a small family of services that run beside a Canton Network validator. They're separable — take one, or several — and they divide along what you're actually trying to do: **act** on contracts, **read** current state, **read** history, or **onboard** end users.

| Service | Job |
|---|---|
| **[Automation daemon](index)** | Act on contracts. Auto-discovers installed apps and exercises choices on triggers — deadlines, settlement matching, contract existence, intervals. |
| **[External-Party Onboarding](external-party-onboarding)** | Onboard self-custody end-user parties at signup. The user's key signs in the middle of a two-call handshake, so it never reaches the participant. |
| **[Ledger Follower](ledger-follower)** | Read *current* state on a hot path. Active contracts, contract/key lookup, runtime template and schema discovery, and correct read-after-write. |
| **[Query API (PQS)](query-api)** | Read *history*. A durable, SQL-queryable projection for reporting and long-range queries. |

## Which do I need?

- **"My contracts need something exercised when a condition is met."** The automation daemon. Start with [Install](install).
- **"My users each need their own party, and I must not hold their keys."** [External-Party Onboarding](external-party-onboarding).
- **"My app needs to show live balances and positions without hammering the Ledger API."** [Ledger Follower](ledger-follower).
- **"I need to report over weeks or months of ledger activity."** [Query API](query-api).
- **"Both of the last two."** Common, and they're designed to sit side by side — see the [comparison](query-api#where-it-fits).

## Two things worth reading before you commit to a design

- **[Canton Coin Rewards](rewards)** — what determines whether automated activity actually pays, including the per-round minimum below which a round pays nothing, and the honest net-of-traffic picture.
- **[Traffic Top-Up](traffic-topup)** — traffic is a real operating cost and a stalled validator is the failure mode to avoid. Includes our recommendation on the built-in top-up loop.

## Operating model

These run on your validator, on your credentials. Canton's privacy model means no third party can observe your contracts or act for you, so automation and read layers have to run where the data is — see [Why You Need It](index#why-you-need-it). Deployment specifics, API surfaces, and keys are provided per deployment; contact Saxon Nodes.
