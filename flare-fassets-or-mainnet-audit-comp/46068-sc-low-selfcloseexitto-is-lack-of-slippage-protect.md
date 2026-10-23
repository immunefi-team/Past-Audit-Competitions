# #46068 \[SC-Low] selfCloseExitTo is lack of slippage protect

**Submitted on May 24th 2025 at 12:11:24 UTC by @ox9527 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46068
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a user invokes selfCloseExitTo, the calculation of requiredFAssets is non-trivial, as it depends on the current collateral ratio (CR). For simplicity, let’s assume the current CR is below the healthy threshold. In this case, the formula for requiredFAssets becomes:

> requiredFAssets = agentBackedFAsset \* \_natShare / poolNatBalance

The agentBackedFAsset is composed of reservedAMG, mintedAMG, and poolRedeemingAMG.

Importantly, any user can call CollateralReservationsFacet.sol::reserveCollateral() to increase the value of reservedAMG. Since agentBackedFAsset includes reservedAMG, this operation directly increases the agentBackedFAsset value.

As a result, the required amount of f-assets for selfCloseExitTo also increases, potentially making the operation more expensive and allowing malicious actors to exploit the mechanism by inflating the cost for other users.

## Vulnerability Details

Firstly CollateralReservationsFacet.sol::reserveCollateral() can increase the value of reservedAMG reserveCollateral()->\_reserveCollateral():

```solidity
    function _reserveCollateral(
        Agent.State storage _agent,
        uint64 _reservationAMG
    )
        private
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Minting.checkMintingCap(_reservationAMG); //@audit-info if minting cap is reached, revert
        _agent.reservedAMG += _reservationAMG;
        state.totalReservedCollateralAMG += _reservationAMG;
    }
```

And from the selfCloseExitTo()->\_selfCloseExitTo()->\_getFAssetRequiredToNotSpoilCR():

```solidity
        } else {
            // f-asset that preserves pool CR (assume poolNatBalance >= natShare > 0)
            // solve (N - n) / (F - f) = N / F get n = N f / F
            return _assetData.agentBackedFAsset.mulDiv(_natShare, _assetData.poolNatBalance);
        }
```

requiredFAssets is depends on \_assetData.agentBackedFAsset

Due to agentBackedFAsset is fetched from assetManager::getFAssetsBackedByPool(agent)->AgentsExternal.getFAssetsBackedByPool():

```solidity
    function getFAssetsBackedByPool(address _agentVault)
        internal view
        returns (uint256)
    {
        Agent.State storage agent = Agent.get(_agentVault);
        return Conversion.convertAmgToUBA(agent.reservedAMG + agent.mintedAMG + agent.poolRedeemingAMG);
    }
```

## Impact Details

If user's account can't afford the required fAssets , user have to transfer more fAssets to the protocol. If the max cost fAssets is not checked user may cost more fAssets than expected.

## References

```solidity
        require(assetData.poolNatBalance == natShare ||
            assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
            "collateral left after exit is too low and non-zero");
        uint256 maxAgentRedemption = assetManager.maxRedemptionFromAgent(agentVault);
        uint256 requiredFAssets = _getFAssetRequiredToNotSpoilCR(assetData, natShare); <@
```

## Proof of Concept

## Proof of Concept

Add the following test to file CollateralPool.ts

```ts
        it.only("Test selfCloseExit is lack of slippage protect", async () => {

            await collateralPool.enter(0, true, { value: ETH(10), from: accounts[1] });
            await collateralPool.enter(0, false, { value: ETH(3), from: accounts[0] });
            await givePoolFAssetFees(ETH(22));
            const exitTokens = ETH(2);
            const fBefore = await collateralPool.fAssetFeesOf(accounts[0]);

            //user front-run.
            // await fAsset.mint(accounts[6], ETH(2), { from: assetManager.address });
            await collateralPool.selfCloseExitTo(exitTokens, true, accounts[6], "underlying_1", ZERO_ADDRESS, { from: accounts[0] });

            const fAfter = await collateralPool.fAssetFeesOf(accounts[0]);

            console.log("diff:", fBefore.sub(fAfter).toString());

            //3692307692307692308
            //3384615384615384616
        });
```

Why do i use fAsset.mint ? Cuz from the AssetManagerMock.sol the getFAssetsBackedByPool is total supply :

```solidity
    function getFAssetsBackedByPool(address /* _backer */) external view returns (uint256) {
        return fasset.totalSupply();
    }
```

Add or comment fAsset.mint we can see the output difference is :

```shell
```
