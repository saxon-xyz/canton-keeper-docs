---
layout: default
title: Fleet-Watchdog
---

# Fleet-Watchdog

[Traffic auto-top-up](traffic-topup) keeps a single node's synchronizer traffic funded. The fleet-watchdog is the layer above it: it watches a *set* of validators and steps in when one is heading for trouble its own top-up loop can't fix alone.

(For steady-state per-node funding we recommend leaving the validator-app's own in-process loop enabled — see [Traffic Top-Up](traffic-topup). The fleet-watchdog is not a replacement for it; it covers what no per-node loop can see or do for itself.)

Two problems it covers.

## Cross-member traffic rescue

A validator that exhausts its traffic balance can't submit commands — including the command to buy more traffic. Under load, or after a burst, a node's own top-up loop can fall behind the burn curve and paint itself into that corner.

The fleet-watchdog monitors every member's traffic headroom and, when one is about to run dry, has a **healthy member buy CIP-0104 traffic on its behalf**. On Canton, buying traffic for another member is a first-class operation: the paying validator exercises the buy against the *recipient's* member id, and the sequencer credits the recipient's allowance. The watchdog automates the decision — whose headroom is low, how much to buy, which member pays — so one node's traffic stall doesn't cascade into a stuck fleet.

## Canton Coin balance alarm

Buying traffic and settling on-chain both spend Canton Coin. A wallet that quietly drains toward zero starts failing purchases and settlements with no warning. The fleet-watchdog tracks each payer wallet's Canton Coin balance and raises an alert while there's still time to act — before a low balance turns into a failed transaction.

## How it relates to per-node top-up

|                  | [Traffic auto-top-up](traffic-topup) | Fleet-watchdog                              |
| ---------------- | ------------------------------------ | ------------------------------------------- |
| Scope            | one node, its own balance            | a set of validators                         |
| Buys traffic for | itself                               | itself **or another member**                |
| Reacts to        | its own balance threshold            | any member's headroom, plus payer CC balance |
| Role             | steady-state funding                 | rescue + early warning                      |

Run per-node top-up on every validator for steady-state funding; run the fleet-watchdog once across the set as the safety net that catches what a single node's loop can't.
