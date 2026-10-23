# #42732 \[SC-High] Incomplete token return whena user claim his rewards leads to rewards fund loss

**Submitted on Mar 25th 2025 at 13:54:49 UTC by @Le\_Rems for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42732
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Theft of unclaimed royalties

## Description

## Description

If users claim rewards using `claimRewardsInToken0` or `claimRewardsInToken1`, the Zapper contract may return unused tokens to the `StakeV2` contract instead of to the user who initiated the claim.

The root cause is in the Zapper contract's token handling logic. When redeeming vault shares, the Zapper receives both token0 and token1. It then swaps a portion of one token for the other based on parameters provided by the user. However, any remaining tokens that aren't swapped are sent back to the calling contract (`StakeV2`) rather than to the user who initiated the claim.

This behavior is highly likely, as market conditions will change between the time the user calculates the swap inputs off-chain and when the transaction is executed.

This results in users receiving only part of their earned rewards, with the remainder being trapped in the `StakeV2` contract where they effectively become part of the pool's general rewards, benefiting all stakers rather than the specific user who earned them.

## Recommendations

### Approach 1: Modify Zapper to send all tokens to the receiver

Update the `zapOutToToken0` and `zapOutToToken1` functions in the Zapper contract to send all tokens to the specified receiver rather than sending some back to the caller:

```solidity
// In Zapper::zapOutToToken0
function zapOutToToken0(
    address receiver,
    SingleTokenSwap calldata swapData,
    KodiakVaultUnstakingParams calldata unstakeParams,
    VaultRedeemParams calldata redeemParams
) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken0Out) {
    (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
    if (token0Debt == 0 && token1Debt == 0) {
        return (0);
    }

    token0Debt -= swapData.inputAmount;
    token1Debt += _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this));
    
    // Send all tokens to the receiver instead of sending token0 back to _msgSender()
    _sendERC20Token(token0, receiver, token0Debt);
    _sendERC20Token(token1, receiver, token1Debt);
    return (token1Debt);
}
```

## Proof of Concept

## Proof of Concept

1. A user has earned rewards and calls `StakeV2::claimRewardsInToken0` to claim them
2. `StakeV2` calls `Zapper::zapOutToToken0` which:
   * Redeems vault shares, receiving both token0 and token1
   * Swaps a portion of token1 for token0 based on the `swapData` parameter
   * Sends token0 to the user (receiver)
   * Sends any remaining token1 back to `StakeV2` (the `_msgSender()`)
3. The relevant code in `Zapper::zapOutToToken0`:

```solidity
token0Debt -= swapData.inputAmount;
token1Debt += _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this));
// @audit-issue token remainder will be sent back to StakerV2 --> inflating other users shares and not sending them back to the legitmate user
_sendERC20Token(token0, _msgSender(), token0Debt);
_sendERC20Token(token1, receiver, token1Debt);
```

4. The issue is that `token0Debt` (remaining token0) is sent to `_msgSender()` (the `StakeV2` contract) instead of to the user who initiated the claim.
5. These tokens become "stuck" in the `StakeV2` contract and are counted as part of `accumulatedDeptRewardsYeet()`, which distributes them to all stakers rather than returning them to the original user.
