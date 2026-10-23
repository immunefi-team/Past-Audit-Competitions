# 60527 sc insight delegationexitrequested event emits inconsistent exit period values

**Submitted on Nov 23rd 2025 at 19:30:17 UTC by @danvinci\_20 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60527
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**

## Description

The protocol emits the `DelegationExitRequested` event from two separate code paths, but the final parameter (exit block / exit period) is derived from two different units, leading to inconsistent and misleading event data for indexers and off-chain analytics.

In the first code path, the event is emitted with:

```
emit DelegationExitRequested(
    _tokenId,
    delegation.validator,
    delegation.delegationId,
    Clock.clock()
);
```

Here, the final argument uses `Clock.clock()`, representing the current block clock value (block-based counter /number).

However, in another part of the system, the same event is emitted again with:

```
@>> (, uint32 exitBlock) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

// ...
emit DelegationExitRequested(
    _tokenId,
    delegation.validator,
    delegationId,
@>>    exitBlock
);
```

The `exitBlock` value here is not a block clock. It is a delegation period index derived from `getDelegationPeriodDetails()`, which returns `startPeriod` and `endPeriod` obtained from the underlying native delegation structure:

```
return []any{ //note this is the golang return statement from the node impleementation.
    delegation.Validation,
    staker.ToWei(delegation.Stake),
    delegation.Multiplier,
    isLocked,
    delegation.FirstIteration,
    lastPeriod,
}
```

These values represent iteration periods, not timestamps or block numbers. As a result, the same event field represents fundamentally different meanings depending on where it was emitted.

This inconsistency causes indexers, subgraphs, explorers, and analytics pipelines to record incorrect or misleading exit period data, making it impossible to reliably reconstruct exit requests across the protocol.

## Note

the event was defined such that the exit block was supposed to be a block number, and not a period number, this is the code implementation

```
    /// @notice Emitted when a delegation exit is requested
    /// @dev When the delegation is pending, exit is immediate and this event is emitted
    /// @param tokenId The ID of the token that was delegated
    /// @param validator The validator that was delegated to
    /// @param delegationId The ID of the delegation
    /// @param exitBlock The block at which the delegation exit will be processed,
    /// in case the delegation is pending it will be the current block
    event DelegationExitRequested(
        uint256 indexed tokenId,
        address indexed validator,
        uint256 indexed delegationId,
        uint48 exitBlock
    );
```

## Recommendation

We need to express `exitBlock` to be in the form of specific block at which the period will end and not the period number itself.

## Proof of Concept

## Proof of Concept

Consider a delegation with ID delegationId. When an exit is requested in through `unstake`, the event is emitted with: `DelegationExitRequested(..., Clock.clock());` Assume the clock is currently `1_234_567`. Indexers will store the exit period as `1_234_567`.

Later, if another exit is requested through `requestDelegatioExit`

```
DelegationExitRequested(..., exitBlock);
```

Where `exitBlock` (actually `endPeriod`) may be something like `42`.

Now, indexers will record `42`
