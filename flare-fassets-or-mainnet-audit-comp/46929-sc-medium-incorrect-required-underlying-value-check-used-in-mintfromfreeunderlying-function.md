# #46929 \[SC-Medium] Incorrect required underlying value check used in mintFromFreeUnderlying function

**Submitted on Jun 6th 2025 at 11:34:00 UTC by @swarun for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46929
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Incorrect required underlying value check used in mintFromFreeUnderlying function which causes less amount to be minted than the available amount.

## Vulnerability Details

In the minFromFreeUnderlying function the vault owner can mint FAsset using the underlying free balance the agent has. So for minting a certain amount of FAsset tokens, it is required that the agent must hold a minimum underlying balance otherwise it will get liquidated. For calculating how much amount of underlying the agent must have which is calculating by multiplying the amount of FAsset which will be minted with minUnderlyingBackingBIPS and the value of minUnderlyingBackingBIPS is less than 100% so less than minted amount is required to available in the agent underlying balance.Now the issue is in the mint from free underlying function minted amount is not multiplied with minUnderlyingBackingBIPS due to which incorrect amount is used when checking that the underlying balance of the agent is sufficient.

## Impact Details

This will prevent the agent from minting using the underlying balance even when they have sufficient balance.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L132>

## Proof of Concept

## Proof of Concept

1. Lets suppose that the available free lots are 2 lots and the agent vault owner calls the mint from free underlying function which is as follows

```solidity

    function mintFromFreeUnderlying(
        address _agentVault,
        uint64 _lots
    )
        internal
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Agent.State storage agent = Agent.get(_agentVault);
        Agents.requireAgentVaultOwner(agent);
        Agents.requireWhitelistedAgentVaultOwner(agent);
        Collateral.CombinedData memory collateralData = AgentCollateral.combinedData(agent);
        require(state.mintingPausedAt == 0, "minting paused");
        require(_lots > 0, "cannot mint 0 lots");
        require(agent.status == Agent.Status.NORMAL, "self-mint invalid agent status");
        require(collateralData.freeCollateralLots(agent) >= _lots, "not enough free collateral");
        uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
        checkMintingCap(valueAMG);
        uint256 mintValueUBA = Conversion.convertAmgToUBA(valueAMG);
        uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
        uint256 requiredUnderlyingAfter = UnderlyingBalance.requiredUnderlyingUBA(agent) + mintValueUBA + poolFeeUBA;
        require(requiredUnderlyingAfter.toInt256() <= agent.underlyingBalanceUBA, "free underlying balance to small");
        _performMinting(agent, MintingType.FROM_FREE_UNDERLYING, 0, msg.sender, valueAMG, 0, poolFeeUBA);
    }
```

2. So the value amg will be 2*lotSize. After that the pool fee will be calculated using 2*lotSize amg.

3 . Total amount of FAsset minted will be mintValyeUBA + The poolFeeUBA

4. So for minting mintValueUBA + poolFeeUBA to check that the agent should have free underlying balance = (mintValueUBA + poolFeeUBA).mulBIPS(minUnderlyingBackingBIPS )
5. But in the current implementation it is checked that the whole minting value UBA + the pool fee UBA balance must be there which is incorrect because more than the required balance is used.

```solidity
 uint256 requiredUnderlyingAfter = UnderlyingBalance.requiredUnderlyingUBA(agent) + mintValueUBA + poolFeeUBA;
        require(requiredUnderlyingAfter.toInt256() <= agent.underlyingBalanceUBA, "free underlying balance to small");
      
```
