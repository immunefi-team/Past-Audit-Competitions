# #45665 \[SC-Medium] \[H-02] Minting Cap Bypass via Pool Fee Exclusion during Self Mint

**Submitted on May 18th 2025 at 20:06:07 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45665
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Description

The protocol’s minting system enforces a cap on the total amount of FAssets that can be minted. However, the `selfMint()` function omits pool fee accounting from the cap enforcement, allowing agents to bypass the cap by exploiting this inconsistency.

In the normal minting flow (`reserveCollateral()` in `CollateralReservations.sol`), the cap check includes both the minted amount and the pool fee:

```solidity
uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
@>> _reserveCollateral(agent, valueAMG + _currentPoolFeeAMG(agent, valueAMG));
```

```solidity
 function _reserveCollateral(
        Agent.State storage _agent,
        uint64 _reservationAMG
    )
        private
    {
        AssetManagerState.State storage state = AssetManagerState.get();
    @>>    Minting.checkMintingCap(_reservationAMG);
   // @>> _reservationAMG includes both the valueAMG and poolFee
        _agent.reservedAMG += _reservationAMG;
        state.totalReservedCollateralAMG += _reservationAMG;
    }

```

```solidity

    function checkMintingCap(
        uint64 _increaseAMG
    )
        internal view
    {
@audit this performs check against the settings/configurations of the assetmanager
        AssetManagerState.State storage state = AssetManagerState.get();
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        uint256 mintingCapAMG = settings.mintingCapAMG;
        if (mintingCapAMG == 0) return;     // minting cap disabled
        uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
        uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
        require(totalAMG + _increaseAMG <= mintingCapAMG, "minting cap exceeded");
    }
```

In contrast, the `selfMint()` function in `Minting.sol` performs the cap check based only on the base mint amount:

```solidity
uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
checkMintingCap(valueAMG);
```

This discrepancy allows agents to self-mint repeatedly, with the pool fee portion (which is also minted FAssets) excluded from cap enforcement.

This leads to inflation beyond the minting cap. For example, with a 1% pool fee and a mint cap of 10,000 AMG, an agent can self-mint 1,000 AMG ten times. Each time, an additional 10 AMG (1%) is minted to the pool, resulting in a total of 10,100 AMG minted, bypassing the cap by 100 AMG.

According to protocol documentation:

> Pool share This share is minted as FAssets and sent to the collateral pool. The percentage of this share is defined by the agent and can be changed by the agent after a delay that provides time for minters to notice the change.

## Impact Details

This can easily lead to the following consequences if not fixed:

1. Bypass of critical system constraint (minting cap)
2. Uncontrolled inflation of the FAsset supply
3. Undermining of collateralization and protocol integrity
4. Potential for economic imbalance and unfair advantage for agents

## References

<https://dev.flare.network/fassets/minting#minting-fee>

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/MintingFacet.sol#L45-L55>

## Recommendation

Update the `selfMint()` implementation to include the pool fee when enforcing the minting cap. The fix should ensure the total amount of FAssets minted (including those minted to the pool) is counted toward the cap.

```solidity
  uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
        uint256 mintValueUBA = Conversion.convertAmgToUBA(valueAMG);
        uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
checkMintingCap(valueAMG + _currentPoolFeeAMG(agent, valueAMG));
```

## Proof of Concept

## Proof of Concept

Assume the protocol has a minting cap of `10,000` AMG.

1. The agent sets a pool fee of `1%`, which means for every `1,000` AMG minted, an additional `10` AMG is minted and sent to the collateral pool.
2. The agent then performs 10 self-mint operations, each for 1,000 AMG. In each operation, `1,000` AMG is minted to the minter, and `10` AMG is minted to the pool as a fee.
3. However, the pool fee is not counted against the minting cap. So after 10 operations, the agent has minted exactly `10,000` AMG to themselves, which reaches the cap.
4. But in addition to that,`100` AMG has been minted to the pool as fees. This brings the total FAsset supply to `10,100` AMG, even though the cap is `10,000` AMG.

This means the cap has effectively been bypassed due to the fee exclusion. Over time, this could lead to more assets in circulation than allowed, breaking protocol assumptions and creating risk of economic imbalance.
