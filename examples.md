---
layout: default
title: Example Configs
---

# Example Configs

Download a config to get started, or combine jobs from multiple examples.

## Canton Swap

Automates OTC trade settlement and expired proposal cleanup for the Canton Swap application.

```yaml
defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  cancel-expired-proposals:
    trigger: deadline
    watch:
      module: Obsidian.CantonSwap.V1
      entity: TradeProposal
    when:
      field: validUntil
      condition: past
    exercise:
      choice: TradeProposal_Cancel
      args:
        cancelor: "$party"

  settle-otc-trades:
    trigger: match
    watch:
      module: Obsidian.CantonSwap
      entity: OTCTrade
    match:
      module: Splice.AmuletAllocation
      entity: AmuletAllocation
      ref: allocation.settlement.settlementRef.cid??
      leg: allocation.transferLegId
      legs: transferLegs
    deadline: settleBefore
    exercise:
      choice: OTCTrade_Settle
      args:
        allocationsWithContext:
          $eachLeg:
            _1: "$match.contractId"
            _2:
              context:
                values: {}
              meta:
                values: {}
```

## Digital Asset Utility DARs

Automates operations for Digital Asset's reference utility packages: DVP settlement, registry mint/transfer execution, collateral transfers, commercial fee billing, and audit record cleanup.

```yaml
defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  settle-dvp:
    trigger: match
    watch:
      module: Utility.Settlement.App.V1.Model.Dvp
      entity: Dvp
    match:
      module: Splice.AmuletAllocation
      entity: AmuletAllocation
      ref: allocation.settlement.settlementRef.cid??
      leg: allocation.transferLegId
      legs: transferLegs
    deadline: settleBefore
    exercise:
      choice: Dvp_Settle
      args:
        allocationsWithContext:
          $eachLeg:
            _1: "$match.contractId"
            _2:
              context:
                values: {}
              meta:
                values: {}

  execute-accepted-mints:
    trigger: exists
    watch:
      module: Utility.Registry.V0.Holding.Mint
      entity: AcceptedMint
    exercise:
      choice: AcceptedMint_Execute
      args: {}

  execute-accepted-transfers:
    trigger: exists
    watch:
      module: Utility.Registry.V0.Holding.Transfer
      entity: AcceptedTransfer
    exercise:
      choice: AcceptedTransfer_Execute
      args: {}

  execute-collateral-transfers:
    trigger: exists
    watch:
      module: Utility.Collateral.App.Model.Collateral
      entity: InstructedCollateral
    exercise:
      choice: InstructedCollateral_ExecuteTransfer
      args: {}

  bill-base-fees:
    trigger: interval
    watch:
      module: Utility.Commercials.V0.Model.CommercialAgreement
      entity: CommercialAgreement
    every: 86400000
    exercise:
      choice: CommercialAgreement_BillBaseFee
      args:
        currentTime: "$now"

  cleanup-settled-dvps:
    trigger: exists
    watch:
      module: Utility.Settlement.App.V1.Model.Dvp
      entity: SettledDvp
    exercise:
      choice: SettledDvp_Delete
      args: {}
```

## Cantara Subscriptions

Automates subscription lifecycle for the Cantara billing platform. Based on user feedback from the Cantara Slack community:
- Subscriptions with payments older than 30 days have expired locks needing manual unlock
- Cancellations older than 30 days get stuck because locked amulets expired

CK automates all of these operations.

```yaml
lookups:
  - module: Splice.Round
    entity: OpenMiningRound
  - module: Splice.Amulet
    entity: FeaturedAppRight

defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  # Auto-accept user service requests
  accept-user-service-requests:
    trigger: exists
    watch:
      module: Cantara
      entity: UserServiceRequest
    exercise:
      choice: UserServiceRequest_Accept
      args: {}

  # Process recurring payments on time (prevents 30-day lock expiry)
  bill-subscriptions:
    trigger: deadline
    watch:
      module: Cantara
      entity: Subscription
    when:
      field: nextPaymentTime
      condition: past
    exercise:
      choice: Subscription_MakePayment
      args:
        issuerTransferContext:
          openMiningRound: "$lookup.Splice.Round:OpenMiningRound"
          featuredAppRight: "$lookup.Splice.Amulet:FeaturedAppRight"
        operatorTransferContext:
          openMiningRound: "$lookup.Splice.Round:OpenMiningRound"
          featuredAppRight: "$lookup.Splice.Amulet:FeaturedAppRight"
        issuerRewardShare: null
        subscriberRewardShare: null
        operatorRewardShare: null

  # Unlock expired locked amulets automatically
  # (no more manual clicking on Reporting > Expired Locks page)
  unlock-expired-locks:
    trigger: exists
    watch:
      module: Splice.Amulet
      entity: LockedAmulet
    exercise:
      choice: LockedAmulet_OwnerExpireLock
      args: {}
```
