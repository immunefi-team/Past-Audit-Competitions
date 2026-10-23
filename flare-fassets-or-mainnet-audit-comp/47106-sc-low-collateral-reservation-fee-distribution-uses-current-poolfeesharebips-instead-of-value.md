# #47106 \[SC-Low] Collateral Reservation Fee distribution uses current poolFeeShareBips instead of value stored during during time of collateral reservation

**Submitted on Jun 9th 2025 at 01:05:07 UTC by @rilwan99 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47106
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

During minting operations in the FAssets protocol, when `distributeCollateralReservationFee()` is called (either after a successful mint via `executeMinting()` or after a payment default via `mintingPaymentDefault()`), the function incorrectly uses the agent's current `poolFeeShareBIPS` instead of the value that was stored in the collateral reservation at the time of reservation. This leads to inconsistent fee distribution between the agent and collateral pool compared to what was agreed upon during the initial collateral reservation.

## Vulnerability Details

The issue occurs in the `distributeCollateralReservationFee()` function in CollateralReservations.sol:

```solidity
function distributeCollateralReservationFee(
    Agent.State storage _agent,
    uint256 _fee // reservationFeeNatWei + executorFeeNatGWei
)
    internal
{
    if (_fee == 0) return;
    // @audit BUG this uses agent.poolFeeShareBIPS state var, which might have been updated
    uint256 poolFeeShare = _fee.mulBips(_agent.poolFeeShareBIPS);
    _agent.collateralPool.depositNat{value: poolFeeShare}();
    IIAgentVault(_agent.vaultAddress()).depositNat{value: _fee - poolFeeShare}(Globals.getWNat());
}
```

When a collateral reservation is created in `reserveCollateral()`, the `poolFeeShareBIPS` value is stored in the reservation: `cr.poolFeeShareBIPS` = `agent.poolFeeShareBIPS` + 1;

However, `distributeCollateralReservationFee()` ignores this stored value and uses the agent's current `poolFeeShareBIPS` setting instead. This is inconsistent with how pool fees are calculated in minting operations, where the stored value is correctly used:

```solidity
function calculatePoolFeeUBA(
    Agent.State storage _agent,
    CollateralReservation.Data storage _crt
)
    internal view
    returns (uint256)
{
    // After an upgrade, poolFeeShareBIPS is stored in the collateral reservation.
    // To allow for backward compatibility, value 0 in this field indicates use of old _agent.poolFeeShareBIPS.
    uint16 storedPoolFeeShareBIPS = _crt.poolFeeShareBIPS;
    uint16 poolFeeShareBIPS = storedPoolFeeShareBIPS > 0 ? storedPoolFeeShareBIPS - 1 : _agent.poolFeeShareBIPS;
    return _calculatePoolFeeUBA(_crt.underlyingFeeUBA, poolFeeShareBIPS);
}
```

## Impact Details

Impact Details This vulnerability leads to incorrect fee distribution in the following scenarios:

1. Agent reduces `poolFeeShareBIPS` after collateral reservation: The collateral pool receives less fees than expected, while the agent vault receives more.
2. Agent increases `poolFeeShareBIPS` after collateral reservation: The collateral pool receives more fees than expected, while the agent vault receives less.

The impact affects:

* Collateral Pool Token (CPT) holders: May receive incorrect fee distributions based on changed agent settings rather than the agreed terms at reservation time
* Agent vault: May receive more or less fees than the originally agreed split
* Protocol integrity: Breaks the immutability principle that reservation terms should remain fixed once established

The financial impact depends on the difference between the original and current poolFeeShareBIPS values and the total collateral reservation fees involved. While individual impacts may be small, this could affect multiple minting operations over time, potentially leading to systematic over/under-compensation of either party.

## References

<https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>

## Proof of Concept

## Proof of Concept

Assume

* Agent's poolFeeShareBIPS is set to 1,000 (10%)
* Collateral reservation fee is 100 NAT
* Expected fee distribution: Agent vault gets 90 NAT, Collateral pool gets 10 NAT

1. User creates collateral reservation via reserveCollateral()

* Agent's current poolFeeShareBIPS is 1,000 (10%)
* Collateral reservation stores poolFeeShareBIPS = 1,000 + 1 = 1,001 for backward compatibility
* User pays 100 NAT as collateral reservation fee
* Expected fee split: 90 NAT to agent vault, 10 NAT to collateral pool

2. Agent updates their pool fee share

* Agent calls function to update poolFeeShareBIPS to 2,000 (20%)
* This change should not affect existing collateral reservations

3. User completes minting via executeMinting()

* distributeCollateralReservationFee() is called
* Function incorrectly uses current agent.poolFeeShareBIPS (2,000) instead of stored value (1,000)
* Actual fee split: 80 NAT to agent vault, 20 NAT to collateral pool

**Collateral pool receives 10 NAT more than expected, agent vault receives 10 NAT less**
