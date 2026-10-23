# #42538 \[SC-Insight] Incorrect value in events emitted in StakeV2

**Submitted on Mar 24th 2025 at 15:07:29 UTC by @dobrevaleri for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42538
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief

All `StakeV2` claim functions emit misleading events when claiming rewards, as it only accounts for tokens received by the Zapper contract rather than the actual recipient.

## Vulnerability Details

The issue occurs in All `StakeV2` claim functions where the `Claimed` event is emitted with the return value from the Zapper's withdrawal functions:

```solidity
function claimRewardsInToken1(
    uint256 amountToWithdraw,
    IZapper.SingleTokenSwap calldata swapData,
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
    IZapper.VaultRedeemParams calldata redeemParams
) external nonReentrant {
    _updateRewards(msg.sender);
    IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);
    
    IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
    uint256 receivedAmount = zapper.zapOutToToken1(msg.sender, swapData, unstakeParams, updatedRedeemParams);
    
    emit Claimed(msg.sender, receivedAmount);
}
```

The problem is that the Zapper's `_yeetOut()` function returns 0 as claimed value if the recipient in either `KodiakVaultUnstakingParams` or `VaultRedeemParams` is different from the Zapper's address, which doesn't correctly represent the actual claimed assets. Also because of this the `zapOutToToken1()` will also return 0, which is used for emitting the `Claimed` event:

```solidity
function _yeetOut(
    IZapper.VaultRedeemParams calldata redeemParams,
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams
) internal returns (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) {
    uint256 islandTokensReceived = _withdrawFromVault(redeemParams);
    if (redeemParams.receiver == address(this)) {
        (token0, token1, token0Debt, token1Debt) = 
            _approveAndUnstakeFromKodiakVault(unstakeParams, islandTokensReceived);
        if (unstakeParams.receiver != address(this)) {
            return (IERC20(address(0)), IERC20(address(0)), 0, 0);
        }
    }
}

function zapOutToToken1(
        address receiver,
        SingleTokenSwap calldata swapData,
        KodiakVaultUnstakingParams calldata unstakeParams,
        VaultRedeemParams calldata redeemParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken1Out) {
        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
        if (token0Debt == 0 && token1Debt == 0) {
@>          return (0);
        }
        token0Debt -= swapData.inputAmount;
        token1Debt += _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this));
        _sendERC20Token(token0, _msgSender(), token0Debt);
        _sendERC20Token(token1, receiver, token1Debt);
        return (token1Debt);
    }
```

## Impact

* Events emit incorrect reward claim amounts
* Off-chain systems tracking rewards through events will have incorrect data
* Users may appear to have claimed 0 rewards when they actually received tokens

## Proof of Concept

## Proof of Concept

1. User has earned rewards in `StakeV2`
2. User calls `claimRewardsInToken1()` and sets the recipient in either `unstakeParams` or `redeemParams` to their address
3. The withdrawal executes successfully and user receives tokens
4. However, the `Claimed` event emits 0 as the claimed amount since the recipient wasn't the Zapper
5. Off-chain systems tracking rewards see a claim for 0 tokens despite actual tokens being transferred
