---
layout: default
title: Ledger Follower
---

# Ledger Follower

The [Query API](query-api) serves ledger state from PQS — a durable, indexed Postgres projection, ideal for history, analytics, and heavy queries. The Ledger Follower is the complementary layer for **fast live reads**: an in-memory mirror of the participant's active contract set, served as a read-only HTTP API.

## Why a separate read path

Reading directly from a busy participant is slow and fragile. The JSON Ledger API caps active-contract responses — a dense party trips a maximum-element-count error past a couple hundred contracts — and bulk reads pile up in-flight connections against the participant. The follower sidesteps both: it holds a single streamed projection of the ACS in memory and answers lookups from that. Reads are single-digit-millisecond and never queue against the participant.

## What it serves

- **Current state** — active contracts by template (`/v1/active`, paged, with top-level field filters), a contract by id (`/v1/contract`), or a contract by Daml key (`/v1/key`). Backed by the full current ACS: anything active *now* is present, however long ago it was created.
- **Runtime template discovery** — `/v1/templates` returns the live menu of projected templates with counts and metadata. Package ids move as DARs are upgraded, so resolving them at runtime means an upgrade doesn't break your client. Don't hard-code template names. (The same principle as auto-discovery in [the daemon](index).)
- **Authoritative payload schemas** — `/v1/schema` returns the schema for a template, so typed clients can be generated rather than hand-maintained against a package version that will move underneath them.
- **Event history** — create/archive events for history-enabled templates, from the follower's start offset forward.
- **Offset-aware sync helpers** — `await` blocks until the projection has caught up to a given ledger offset; `verify` confirms a specific contract is visible at or after an offset. Pair these with the offset a write returns to **confirm your write is projected before you act on it**.
- **Materialized wallet views**, where enabled — `/v1/wallet/balance`, `/v1/wallet/positions` and `/v1/wallet/feed` serve pre-computed per-party balances, positions and activity feeds, so common wallet questions don't mean scanning holdings on every request.
- **Health & metrics** — liveness, readiness (is the projection caught up), and Prometheus metrics.

Reads authenticate with `Authorization: Bearer <key>`; health and metrics are open.

> Note the contrast with the [Query API](query-api), which uses an API-key header. Two services, two schemes — worth getting right in your client, since the symptom of mixing them up is a bare `401`.

## Read-after-write

A projection necessarily trails the participant by a small offset, so reading immediately after a write is a race — and the fix is not a sleep:

```text
1. Submit the command (gRPC Ledger API).
2. Capture the completion offset and any created contract ids from the response.
3. Call /v1/await?offset=<offset>  (or /v1/verify?offset=<offset>&id=<cid>).
4. Now read current state.
```

That turns an unbounded guess into a bounded wait, and it's the most important pattern for anything that writes and then displays the result.

## What it is not

The follower is a live mirror, not an archive. It holds current state plus a forward-only event window from when it started; it does not persist. A restart re-seeds the full active state — reads are unaffected — but resets the history window to the new start offset. For durable, long-range history (auditing past settlements, replaying weeks of events), use the [Query API](query-api) over PQS.

Two more limits worth designing around:

- **Large templates must be paged.** Responses are guarded against unbounded size.
- **The projection covers a configured set of templates.** If something you need is absent from `/v1/templates`, it needs adding to the projection — it will not appear on its own.

## Query API vs Follower — pick by need

|          | [Query API (PQS)](query-api)   | Ledger Follower                    |
| -------- | ------------------------------ | ---------------------------------- |
| Backing  | Postgres, indexed views        | in-memory ACS projection           |
| Best for | history, audit, analytics, SQL | fast live current-state reads      |
| History  | full, durable                  | forward-only window, resets on restart |
| Latency  | query-dependent                | single-digit ms                    |
| Extra    | offset-pinned snapshots        | `await` / `verify` offset helpers  |
| Auth     | API-key header                 | `Authorization: Bearer`            |

Run both beside the participant and route each read to whichever fits. Use HTTP keep-alive and a bounded connection pool; many workers issuing unbounded concurrent requests is the usual cause of self-inflicted latency.

> **Access:** base URL, key, and the enabled route set are per deployment. Contact Saxon Nodes.
