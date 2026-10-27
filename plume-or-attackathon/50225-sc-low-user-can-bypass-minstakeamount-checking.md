# 50225 sc low user can bypass minstakeamount checking&#x20;

**Submitted on Jul 22nd 2025 at 18:24:34 UTC by @New5paceXyz for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50225
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief/Intro\
User can `unstake` partially which allows the user to end up with a remaining staked balance that is less than `minStakeAmount` because the code does not check that the remaining staked amount stays >= `minStakeAmount`.

## Vulnerability Details

* On staking, the protocol validates the stake amount against `minStakeAmount` via:
  * User -> `stake` -> `_performStakeSetup` -> `_validateStaking` -> `_validateStakeAmount`
  * Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L112-L114>
* Users can `unstake` a specific amount. The unstake flow updates `$.stakeInfo[user].staked` by decreasing the requested `amount` in `_updateUnstakeAmounts`:
  * References:
    * <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L353-L358>
    * <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L366-L390>
* The unstake logic reduces the staked amount but does not validate that the remaining staked amount remains >= `minStakeAmount`.
* As a result, a user can have a staked balance that is less than `minStakeAmount`, breaking the intended invariant that all staked balances must be at least `minStakeAmount`.
* Mitigation suggestion (high level): When processing an unstake, check the resulting remaining staked amount; if it would be below `minStakeAmount`, require the user to unstake the full remaining amount (or disallow the partial unstake), so no account remains with a staked amount < `minStakeAmount`.

## Impact Details

User can end up with a staked balance below `minStakeAmount`, which breaks the design invariant that every active stake be >= `minStakeAmount`. This affects protocol assumptions about minimum effective stakes but does not directly drain funds.

## References

<details>

<summary>Relevant source lines</summary>

* Stake validation:\
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L112-L114>
* Unstake and update logic:\
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L353-L358\\>
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L366-L390>

</details>

## Proof of Concept

{% stepper %}
{% step %}

1. User stakes an amount greater than `minStakeAmount`, e.g. `minStakeAmount + 1`.
   {% endstep %}

{% step %}
2\. User partially unstakes `minStakeAmount`.
{% endstep %}

{% step %}
Result: The user now has `1` remaining staked (or generally a remainder < `minStakeAmount`), demonstrating the protocol allows a staked balance below `minStakeAmount`.
{% endstep %}
{% endstepper %}
