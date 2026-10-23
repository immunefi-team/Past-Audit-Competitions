# #46282 \[SC-High] Wrong implementation of \`payout\` would lead to loss of fee share of \`AgentVault\`

**Submitted on May 27th 2025 at 19:00:08 UTC by @farman1094 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46282
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Theft of unclaimed yield

## Description

## Brief/Intro

There is issue in `CollateralPool::payout` function, This function is called to in situation of liquidation, to pay the collateral from collateral pool to liquidator. But the way it implements, the fee share of the agent is lost which could be transferred to agent.

## Vulnerability Details

So the Liquidation mechanism of flare system is like this. When liquidation starts, any liquidator can send FAssets and get paid with a combination of vault collateral and pool collateral. When only vault collateral is not enough.

The way we are liquidating is

* Taking the amount from the collateral pool.
* Burning the share of the Agent correspondence to amount taken.

Check here: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L865>

```
// CollateralPool::payout
     @>   _transferWNat(_recipient, _amount);
            
           ....more code
     @>       token.burn(agentVault, toSlashToken, true);
           ....more code
```

But in the collateral pool the share of agent represent.

* The collateral they hold
* There fee share (FAsset) which earned from mint fee, etc.

At the time of burning the share of the agent we are not claiming the agent fee share and transfer back that to Agent which is lost now for agent and directly a financial loss for him.

```solidity
    // CollateralPool::payout
uint256 agentTokenBalance = token.balanceOf(agentVault);
uint256 toSlashTokenMax = assetData.poolNatBalance > 0 ?
assetData.poolTokenSupply.mulDiv(_agentResponsibilityWei, assetData.poolNatBalance) : agentTokenBalance;
uint256 toSlashToken = Math.min(toSlashTokenMax, agentTokenBalance);
if (toSlashToken > 0) {
    @> (uint256 debtFAssetFeeShare,) = _getDebtAndFreeFAssetFeesFromTokenShare(
            assetData, agentVault, toSlashToken, TokenExitType.KEEP_RATIO); // @audit freeFAssetFeeShare lost of toSlashToken
        _burnFAssetFeeDebt(agentVault, debtFAssetFeeShare);
token.burn(agentVault, toSlashToken, true);
```

However, a right way to do this is transfer back the `freeFAssetFeeShare` of AgentVault like it happened in function `_exitTo`

```
// CollateralPool::_exitTo
        (uint256 debtFAssetFeeShare, uint256 freeFAssetFeeShare) = _getDebtAndFreeFAssetFeesFromTokenShare(
            assetData, msg.sender, _tokenShare, _exitType);
        // transfer/burn assets
        if (freeFAssetFeeShare > 0) {
            _transferFAsset(address(this), _recipient, freeFAssetFeeShare);
        }
        if (debtFAssetFeeShare > 0) {
            _burnFAssetFeeDebt(msg.sender, debtFAssetFeeShare);
        }
```

## Impact Details

Financial loss to Agent, As all the fees which Agent earned from minting process in collateral pool and not claimed is lost. Which is lost now because of the wrong implementation As explained above.

## Proof of Concept

## Proof of Concept

This is implementation mistake not a attack path. This issue will affect every time, this situation occurs.

1. The Liquidation have start for the agent for XYZ reason.

* It will start from `LiquidationFacet::liquidate` which then called `Liquidation::liquidate`

2. The amount in AgentVault is not enough so the extra amount is taken from Collateral Pool.

* So the `LiquidationFacet::liquidate` underside call `Agents::payoutFromPool`

```
// LiquidationFacet::liquidate
... more code 
        if (payoutPoolWei > 0) {
            uint256 agentResponsibilityWei = _agentResponsibilityWei(agent, payoutPoolWei);
            _amountPaidPool = Agents.payoutFromPool(agent, msg.sender, payoutPoolWei, agentResponsibilityWei);
        }
... more code
```

* which then call `CollateralPool::payout`

```
// Agents::payoutFromPool
... more code 
   _amountPaid = Math.min(_amountWei, poolBalance);
        _agentResponsibilityWei = Math.min(_agentResponsibilityWei, _amountPaid);
        _agent.collateralPool.payout(_receiver, _amountPaid, _agentResponsibilityWei);
```

3. `CollateralPool::payout` is where the issue is.

```
// CollateralPool::payout
... more code
            (uint256 debtFAssetFeeShare,) = _getDebtAndFreeFAssetFeesFromTokenShare(
                assetData, agentVault, toSlashToken, TokenExitType.KEEP_RATIO); // @audit freeFAssetFeeShare lost of toSlashToken
            _burnFAssetFeeDebt(agentVault, debtFAssetFeeShare);
```

4. In this function we are calling `_getDebtAndFreeFAssetFeesFromTokenShare` this to get 2 values

* debtFAssetFeeShare: Fee Share Debt of user
* freeFAssetFeeShare: Which is the user share of fees earned thoughout the minting process.

5. In this function we are not retrieving `freeFAssetFeeShare` and transferring back to Agent which should be done before burning the share of agent.
6. But the share got burned without claiming the fee share. So this is the financial loss for the Agent.
