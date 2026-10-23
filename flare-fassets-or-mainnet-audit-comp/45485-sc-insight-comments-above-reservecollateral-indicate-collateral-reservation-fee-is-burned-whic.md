# #45485 \[SC-Insight] Comments above \`reserveCollateral\` indicate collateral reservation fee is burned, which is not the case

**Submitted on May 15th 2025 at 11:56:14 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45485
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CollateralReservationsFacet.sol>
* **Impacts:**

## Description

## Brief/Intro

As the title suggests, comments above `reserveCollateral` indicate collateral reservation fee is burnt, which is not the case.

## Vulnerability Details

Look at the comments above the `reserveCollateral` function in `CollateralReservationsFacet`:

```solidity
     * If the minter pays the underlying amount, the collateral reservation fee is burned and minter obtains
     * f-assets. Otherwise the agent collects the collateral reservation fee.
```

But this is not the case. The collateral reservation fee is not burned. The collateral reservation fee is instead distributed to the vault pool and the collateral pool. This can be seen when `executeMinting` is called:

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L66-L70>

```solidity
        CollateralReservations.distributeCollateralReservationFee(agent,
            crt.reservationFeeNatWei + unclaimedExecutorFee);
        // cleanup
        CollateralReservations.releaseCollateralReservation(crt, _crtId);   // crt can't be used after this
```

```solidity
     function distributeCollateralReservationFee(
        Agent.State storage _agent,
        uint256 _fee
    )
        internal
    {
        if (_fee == 0) return;
        uint256 poolFeeShare = _fee.mulBips(_agent.poolFeeShareBIPS);
        _agent.collateralPool.depositNat{value: poolFeeShare}();
        IIAgentVault(_agent.vaultAddress()).depositNat{value: _fee - poolFeeShare}(Globals.getWNat());
    }
```

Fees are not burned here, instead sent to the collateral pool and vault pool using the `distributeCollateralReservationFee` function.

## Impact Details

Misleading explanation of the functionality of the code in the comments/docs

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/CollateralReservationsFacet.sol#L23-L24>

## Proof of Concept

## Proof of Concept

We suggest that the protocol change the comment to say that the fees are not actually burnt, but instead sent to the collateral and vault pools when the minting is successful. The current comments do not suggest the same and hence are misleading.
