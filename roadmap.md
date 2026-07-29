---
layout: default
title: Roadmap
---

# Roadmap

What's shipped, what's actively being built, what's planned.

## Shipped

- **Auto-discovery of installed apps.** Scan participant DARs, match against built-in catalog entries, generate jobs automatically. Zero YAML required for supported apps.
- **YAML jobs with four trigger types:** `deadline`, `interval`, `exists`, `match`. See [Configuration Reference](config).
- **Imported actions:** custom JavaScript/TypeScript functions registered in the catalog, dispatched on triggers with the daemon's shared OAuth2 token provider + ledger client + process-lifetime cache. See [Imported Actions](imported-actions).
- **On-chain settlement via Splice CIP-56 transfer factory.** `RegistryClient` in core wraps the scan registry call; imported actions can submit `SettleX` choices that exercise the CIP-56 transfer flow without daml needing to know the registry shape.
- **OAuth2 with refresh-ahead-of-expiry.** Token provider mints once, refreshes proactively before expiry, deduplicates concurrent callers, invalidates and retries on 401. 24h JWTs are minted once per daemon lifetime.
- **Canton 3.5+ JSON Ledger API support.** `filtersByParty.<party>.cumulative[].templateFilter` request shape; suffix-match on hex-vs-`#`-prefixed templateId in responses; activeAtOffset anchored to ledger-end for consistent ACS snapshots.
- **Adaptive backoff on participant element-count caps.** Catches HTTP 413 `JSON_API_MAXIMUM_LIST_ELEMENTS_NUMBER_REACHED` mid-scan, halves the window from the same cursor, retries; sticks at the post-shrink size for the rest of the scan so dense ledger regions don't repeatedly re-hit the cap.
- **Traffic auto-top-up (CIP-0104).** New imported action `runTrafficTopup` monitors the operator's traffic balance via scan and submits `WalletAppInstall_CreateBuyTrafficRequest` when the balance falls below a configured threshold. The validator-app's internal wallet automation completes the buy on-ledger. Paused-by-default in the catalog; `TRAFFIC_TOPUP_ALLOW_LIVE=true` in env to leave shadow mode. See [Traffic Top-Up](traffic-topup).
- **External-party onboarding service.** A stateless API to onboard self-custody (external) end-user parties at signup: a two-call generate/submit handshake where the user's key signs in the middle, so the private key never leaves the operator's custody and never reaches the participant. Deterministic party id up front, idempotent, async-friendly. See [External-Party Onboarding](external-party-onboarding).
- **Query API over PQS.** An authenticated read/query layer over the Daml Participant Query Store: ledger state (active contracts, balances, history) projected into typed, indexed views and served over SQL/HTTP, with offset-pinned snapshots for consistent reads. Read-only, runs beside the participant. See [Query API](query-api).
- **Ledger Follower.** A read-only HTTP API backed by an in-memory mirror of the participant's active contract set — single-digit-millisecond current-state lookups that don't queue against the participant or trip its JSON-ACS element-count cap. Paged active reads, contract and Daml-key lookup, runtime template discovery with authoritative payload schemas (so clients survive package upgrades instead of breaking on a stale package id), and offset-aware `await`/`verify` helpers to confirm a write is projected before acting on it. Optional materialized wallet views serve per-party balances, positions and activity feeds without scanning holdings per request. The fast/live counterpart to the PQS Query API. See [Ledger Follower](ledger-follower).
- **Fleet-watchdog.** A fleet-level liveness layer above per-node traffic top-up: monitors every member's traffic headroom and has a healthy validator buy CIP-0104 traffic on behalf of one about to run dry (cross-member rescue), plus a Canton Coin balance alarm that warns before a payer wallet drains. See [Fleet-Watchdog](fleet-watchdog).

## Active

- **Per-validator deployment template** (Helm values templating). The daemon already supports running one instance per tenant via env config; the Helm chart is being parameterized so spinning up a new instance is a values-file diff.

## Planned

- **Prometheus alerting rule pack.** The daemon already exposes Prometheus metrics — `saxon_automation_runs_total`, `saxon_automation_submissions_total`, `saxon_automation_skips_total`, `saxon_automation_provider_failures_total` and the `saxon_automation_run_duration_seconds` histogram; the alerting rules to consume them are next.

## Open questions

These shape later work:

- **CIP-0104 rewards semantics.** Traffic-based app rewards are approved but **not yet live** — the mechanism is in preview, and it lands in increments rather than all at once. Free confirmations arrive early in that sequence; the switch that makes app rewards traffic-derived, replacing today's explicit activity markers and introducing a new reward-coupon contract, comes later. Until that switch, app rewards flow through the `FeaturedAppRight` activity-marker mechanism. Two accounting details materially affect what automation is worth, and both are set by network governance rather than by an app: a **per-round minimum below which an app's reward for that round is not paid at all**, and a **per-app activity weight**. See [Rewards](rewards). Saxon tracks these and keeps the automation aligned; expect this section to move as the increments ship.
- **DAR vetting workflow on mainnet.** Mainnet DAR uploads require the Splice SV super-validator vote to vet a new package — `MissingVettedPackages` errors until the vote passes. Faster than waiting, but no automation hook yet.

## Versioning

The daemon's published image is tagged with both `latest` and the commit SHA. There's no semantic-version release cadence yet — every push to main publishes. Pin to a SHA tag in production if you want change control.
