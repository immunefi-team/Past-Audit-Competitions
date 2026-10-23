# #47034 \[SC-Medium] check minting cap function checks on incorrect amount in mintFromFreeUnderlying function

**Submitted on Jun 8th 2025 at 08:28:57 UTC by @swarun for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47034
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

check minting cap function checks on incorrect amount in mintFromFreeUnderlying function because it doesn't takes into account the amount that will be minted because of the pool fee.

## Vulnerability Details

In the mint from free underlying function there is a check minting cap call which ensures that the minted amount should be less than the minting cap set but in mint from free underlying function it doesn't take into account the amount that will be minted because of the pool fee. in the proof of concept i have clearly shown where the pool fees has not been included.

## Impact Details

It causes an important invariant to fail as it will allow more than minting cap to be minted.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L129>

## Proof of Concept

## Proof of Concept

1. The agent vault owner calls the self mint function and following is mint from free underlying function

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

2. We can see there is a checkMintingCap(valueAMG) and valueAMG = \_lots\*Globals.getSettings().lotSizeAMG;
3. Then we can see pool fee being calculated as follows poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);

Then in perform minting function we can see that the poolfeeuba is also minted to the pool \_performMinting(agent, MintingType.SELF\_MINT, 0, msg.sender, valueAMG, 0, poolFeeUBA);

```solidity
function _performMinting(
        Agent.State storage _agent,
        MintingType _mintingType,
        uint64 _crtId,
        address _minter,
        uint64 _mintValueAMG,
        uint256 _receivedAmountUBA,
        uint256 _poolFeeUBA
    )
        private
    {
        uint64 poolFeeAMG = Conversion.convertUBAToAmg(_poolFeeUBA);
        Agents.createNewMinting(_agent, _mintValueAMG + poolFeeAMG);
        // update agent balance with deposited amount (received amount is 0 in mintFromFreeUnderlying)
        UnderlyingBalance.increaseBalance(_agent, _receivedAmountUBA);
        // perform minting
        uint256 mintValueUBA = Conversion.convertAmgToUBA(_mintValueAMG);
        Globals.getFAsset().mint(_minter, mintValueUBA);
        Globals.getFAsset().mint(address(_agent.collateralPool), _poolFeeUBA);
        _agent.collateralPool.fAssetFeeDeposited(_poolFeeUBA);
        // notify
        if (_mintingType == MintingType.PUBLIC) {
            uint256 agentFeeUBA = _receivedAmountUBA - mintValueUBA - _poolFeeUBA;
            emit IAssetManagerEvents.MintingExecuted(_agent.vaultAddress(), _crtId,
                mintValueUBA, agentFeeUBA, _poolFeeUBA);
        } else {
            bool fromFreeUnderlying = _mintingType == MintingType.FROM_FREE_UNDERLYING;
            emit IAssetManagerEvents.SelfMint(_agent.vaultAddress(), fromFreeUnderlying,
                mintValueUBA, _receivedAmountUBA, _poolFeeUBA);
        }
    }
```
