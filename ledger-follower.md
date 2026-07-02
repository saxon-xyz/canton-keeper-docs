---
layout: default
title: Ledger Follower
---

# Ledger Follower

The [Query API](query-api) serves ledger state from PQS — a durable, indexed Postgres projection, ideal for history, analytics, and heavy queries. The Ledger Follower is the complementary layer for **fast live reads**: an in-memory mirror of the participant's active contract set, served as a read-only HTTP API.

## Why a separate read path

Reading directly from a busy participant is slow and fragile. The JSON Ledger API caps active-contract responses — a dense party trips a maximum-element-count error past a couple hundred contracts — and bulk reads pile up in-flight connections against the participant. The follower sidesteps both: it holds a single streamed projection of the ACS in memory and answers lookups from that. Reads are single-digit-millisecond and never queue against the participant.

## What it serves

- **Current state** — active contracts by template, a contract by id, or a contract by key. Backed by the full current ACS: anything active *now* is present, however long ago it was created.
- **Event history** — create/archive events for history-enabled templates, from the follower's start offset forward.
- **Offset-aware sync helpers** — `await` blocks until the projection has caught up to a given ledger offset; `verify` confirms a specific contract is visible at or after an offset. Pair these with the offset a write returns to **confirm your write is projected before you act on it**.
- **Health & metrics** — liveness, readiness (is the projection caught up), and Prometheus metrics.

## What it is not

The follower is a live mirror, not an archive. It holds current state plus a forward-only event window from when it started; it does not persist. A restart re-seeds the full active state — reads are unaffected — but resets the history window to the new start offset. For durable, long-range history (auditing past settlements, replaying weeks of events), use the [Query API](query-api) over PQS.

## Query API vs Follower — pick by need

|          | [Query API (PQS)](query-api)   | Ledger Follower                    |
| -------- | ------------------------------ | ---------------------------------- |
| Backing  | Postgres, indexed views        | in-memory ACS projection           |
| Best for | history, audit, analytics, SQL | fast live current-state reads      |
| History  | full, durable                  | forward-only window, resets on restart |
| Latency  | query-dependent                | single-digit ms                    |
| Extra    | offset-pinned snapshots        | `await` / `verify` offset helpers  |

Reads are authenticated with a bearer token; health and metrics are open. Run both beside the participant and route each read to whichever fits.
