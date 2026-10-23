# #45450 \[SC-Insight] Outdated underlying chain data lead to shortened minting windows or DoS when minting fAssets

**Submitted on May 14th 2025 at 22:36:56 UTC by @holydevoti0n for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45450
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the protocol relies on potentially stale underlying chain data when setting time constraints for minting operations, which can force users to execute minting within an unfairly reduced timeframe or even make it impossible to complete transactions if the window has already passed.

## Vulnerability Details

The vulnerability exists in the collateral reservation process in `CollateralReservations.sol` where the protocol assigns underlying chain block/timestamp parameters that determine when a user can execute minting operations: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol#L88-L91>

```solidity
// @audit-issue  can be outdated, will impact both lastUnderlyingBlock and lastUnderlyingTimestamp.
(uint64 lastUnderlyingBlock, uint64 lastUnderlyingTimestamp) = _lastPaymentBlock();
// @audit-issue will force user to have a shorter redemption period.
cr.firstUnderlyingBlock = state.currentUnderlyingBlock;
cr.lastUnderlyingBlock = lastUnderlyingBlock;
cr.lastUnderlyingTimestamp = lastUnderlyingTimestamp;
```

The issue stems from the fact that `state.currentUnderlyingBlock` may not be up-to-date when a collateral reservation is created. The protocol has a separate function `updateCurrentBlock()` in `UnderlyingTimekeepingFacet`.sol that should be called to keep this value updated:

```solidity
/**
 * Prove that a block with given number and timestamp exists and
 * update the current underlying block info if the provided data higher.
 * This method should be called by minters before minting and by agent's regularly
 * to prevent current block being too outdated, which gives too short time for
 * minting or redemption payment.
 * NOTE: anybody can call.
 * @param _proof proof that a block with given number and timestamp exists
 */
function updateCurrentBlock(
    IConfirmedBlockHeightExists.Proof calldata _proof
)
    external
{
    StateUpdater.updateCurrentBlock(_proof);
}
```

However, this function is not automatically called before creating a collateral reservation, and the documentation only states that it "should be called" rather than enforcing this requirement. When a user later attempts to execute minting with `executeMinting()` in `Minting.sol`, the following check is performed: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/Minting.sol#L52-L53>

```solidity
require(_payment.data.responseBody.blockNumber >= crt.firstUnderlyingBlock,
    "minting payment too old");
```

If `firstUnderlyingBlock` was set with outdated data, this creates two significant problems:

**Shortened Redemption Period**: Users have less time than they should to execute minting operations, as the window between `firstUnderlyingBlock` and `lastUnderlyingBlock` is artificially compressed.

**Potential DoS**: In extreme cases, if the underlying blockchain has progressed significantly since the last update, a user's payment might shortly be considered "too old" at the moment of collateral reservation, making it impossible to complete the minting process.

The issue is exacerbated by the fact that the `_lastPaymentBlock()` function calculates `lastUnderlyingBlock` and `lastUnderlyingTimestamp` based on the current value of `state.currentUnderlyingBlock`, meaning both the start and end of the redemption window can be affected by outdated data.

## Impact Details

The vulnerability directly impacts users attempting to mint `fAssets` through the protocol by:

* Reducing the time window available for completing minting operations, potentially forcing users to rush transactions or pay higher gas fees.
* Creating situations where minting becomes impossible if the underlying blockchain has progressed too far beyond the outdated `firstUnderlyingBlock` value.

Introducing unpredictability in the minting process, as the actual time window depends on when `updateCurrentBlock()` was last called, which is not under the user's direct control.

## Recommendation

Make sure to update the underlying chain data(block, timestamp, etc) with `updateCurrentBlock` in every function that needs to store and use the current state of the underlying chain for later comparison.

## Proof of Concept

## Proof of Concept

* Protocol's `currentUnderlyingBlock` is outdated (4800) while the current is at block 5000
* User creates collateral reservation, gets assigned outdated `firstUnderlyingBlock = 4800`
* User makes payment at block 5001, which is valid (5001 > 4800) Result: User starts 200 blocks behind the schedule for executing the minting on Flare Network. Lets say the delay time is 500 blocks, if user calls the execute minting only 301 blocks later, transaction will revert as his payment is too old.
