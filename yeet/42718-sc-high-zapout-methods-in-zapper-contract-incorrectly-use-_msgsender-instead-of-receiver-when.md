# #42718 \[SC-High] zapOut methods in zapper contract incorrectly use \_msgSender() instead of receiver when sending back remainder tokens

**Submitted on Mar 25th 2025 at 13:31:35 UTC by @valy001 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42718
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

zapOut methods in Zapper contract, including zapOutToToken0(), zapOutToToken1(), ZapOut() incorrectly use \_msgsender() instead of receiver when clearing remainder tokens.

## Vulnerability Details

StakeV2 contract is used for yeet tokens staking and rewards distribuion. The rewards including Yeet token and native token can be deposited into Kodiak vault and then again deposite into compounding vault with minted Kodiak island tokens to earn compounding yeilds. Users of Yeet protocol can get a portion of rewards propotional to their staking shares in StakV2 contract.\
As users claim their rewards via StakeV2.claimRewardsInToken0(), StakeV2.claimRewardsInToken1() or StakeV2.claimRewardsInToken(). Stake contract calls zapper.zapOutToToken0/zapOutToToken1/zapOutToToken to withdraw from vault and swapp all the rewards into specified tokens.\
However, when zapper clearing the remainder tokens after withdrawing and swapping, \_msg.send() is passed as receiver of these remainder tokens, which is StakerV2 contract address, not users address. This will effectively cause users loss some amount of tokens.\
As in zaoOutToToken0():

```solidity
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
        token1Debt -= swapData.inputAmount;
        token0Debt += _verifyTokenAndSwap(swapData, address(token1), address(token0), address(this));
        _sendERC20Token(token0, receiver, token0Debt);
        _sendERC20Token(token1, _msgSender(), token1Debt); //@audit <-
        return (token0Debt);
    }
```

Same as in zapOutToToken1() and zapOut()

## Impact Details

Users may loss some amount of tokens when claiming rewards

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

1. Users stake Yeet tokens through StakeV2 contract.
2. After some time, user call claim function to claim rewards.
3. While user may get most of their rewards, some amount of tokens are lost into StakeV2 contract.
