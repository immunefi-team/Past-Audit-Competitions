# 52915 sc low yield are transferred before eligibility check potentially leading to freezing of funds

**Submitted on Aug 14th 2025 at 09:13:02 UTC by @spongebob for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52915
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Permanent freezing of funds

## Description

The `distributeYield` and `distributeYieldWithLimit` functions in the `ArcToken` contract contain a flaw where yield tokens are transferred from the caller to the contract before checking if any token holders are eligible to receive yield.

* Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L408>

Behavior:

* When all token holders are restricted from receiving yield (resulting in `effectiveTotalSupply == 0`), the functions still execute the `safeTransferFrom` call but then `emit YieldDistributed(0, ...)` and return without distributing any tokens.
  * Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L425-L428>
* The transferred yield tokens become permanently trapped in the contract since there are no sweep or recovery functions implemented.

Root cause summary:

* `safeTransferFrom` executes first, pulling tokens from the caller.
* `effectiveTotalSupply` is calculated afterward by checking yield restrictions.
  * Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L417-L423>
* If no holders are eligible (`effectiveTotalSupply == 0`), the function returns early without distribution.
  * Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L417-L423>
* No recovery mechanism exists to retrieve the trapped tokens.

The same pattern exists in `distributeYieldWithLimit` where tokens are transferred on the first batch (`startIndex == 0`) before eligibility checks.

* Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L526-L528>

## Impact

Yield tokens transferred to the contract cannot be recovered through any on-chain mechanism, leading to permanent loss of yield tokens when all token holders become ineligible for yield distribution.

## Proof of Concept

{% stepper %}
{% step %}

### Deploy ArcToken with yield restrictions

* Deploy an ArcToken contract with yield blacklist restrictions enabled.
* Set up a yield token (e.g., USDC) for distribution.
  {% endstep %}

{% step %}

### Create token holders and restrict them

* Mint ArcTokens to multiple addresses (e.g., Alice, Bob, Charlie).
* Add all token holders to the yield blacklist using `YieldBlacklistRestrictions.addToBlacklist()`.
  {% endstep %}

{% step %}

### Verify all holders are restricted

* Confirm `effectiveTotalSupply == 0` by checking that `_isYieldAllowed()` returns false for all holders.
* All holders should be ineligible for yield distribution.
  {% endstep %}

{% step %}

### Attempt yield distribution

* As a `YIELD_DISTRIBUTOR_ROLE` account, approve yield tokens to the ArcToken contract.
* Call `distributeYield(amount)` with a significant amount of yield tokens.
  {% endstep %}

{% step %}

### Observe fund loss

* The `safeTransferFrom` executes first, transferring yield tokens from caller to contract.
* Function calculates `effectiveTotalSupply == 0` and emits `YieldDistributed(0, yieldTokenAddr)`.
* Yield tokens are now permanently trapped in the `ArcToken` contract.
  {% endstep %}

{% step %}

### Confirm permanent loss

* Check that yield tokens are stuck in the contract balance.
* Verify no sweep/recovery functions exist in the contract to retrieve the tokens.
* Funds remain inaccessible to all parties, including contract administrators.

Note: The same scenario applies to `distributeYieldWithLimit()` when called with `startIndex == 0`.
{% endstep %}
{% endstepper %}

## References

* ArcToken.sol at lines referenced above:
  * <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L408>
  * <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L417-L423>
  * <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L425-L428>
  * <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L526-L528>
