# #42113 \[SC-High] yeetOut function in Zapper.sol sends tokens back to StakeV2 contract instead of user

**Submitted on Mar 20th 2025 at 21:49:47 UTC by @cryptostaker for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42113
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When a user wants to claim their staked rewards by calling `claimRewardsInToken()` in StakeV2.sol, the function calls `zapper.zapOut()`, which sends the excess Yeet & WBera tokens back to the StakeV2 contract instead of the user.

## Vulnerability Details

This is the flow of funds when `claimRewardsInToken()` is called. Shares in the vault contract will be burned, and the user will receive the outputToken noted in the parameter.

1. Tokens from the Beradrome farm will be withdrawn back to the Vault.
2. The KDK LP tokens from the Vault will be swapped to WBera and Yeet.
3. WBera and Yeet will be swapped to the outputToken

In the event that there is excess tokens to be returned back when swapping WBera / Yeet to the outputToken (can be set from the parameters, or the swap leaves some WBera due to low liquidity etc), these excess tokens is sent back to the `_msgSender()` which is the vault, instead of the user.

```
    function zapOut(
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
```

## Impact Details

User may not get their excess tokens back, leading to stuck funds.

## Recommendations

Ensure that the `_msgSender()` is the direct caller (if someone interacts with the Zapper directly). If StakeV2 contract is calling, ensure that `_msgSender()` is the receiver.

```
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
```

## References

[Zapping out in StakeV2](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L391)

[Transferring leftover debt to msgSender (which is StakeV2 contract) instead of user](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L352)

## Proof of Concept

## Proof of Concept

Function flow:

1. claimRewardsInToken() calls zapper.zapOut()

```
    function claimRewardsInToken(
    ...
        uint256 receivedAmount =
                            zapper.zapOut(outputToken, msg.sender, swap0, swap1, unstakeParams, updatedRedeemParams);
```

2. `Zapper.zapOut()` calls yeetOut which withdraws from BeradromeFarm to get KDK LP tokens, and swaps these LP tokens to WBera and Yeet (token0 and token1, can be other tokens depending on the vault).

```
    function zapOut(
        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
        if (token0Debt == 0 && token1Debt == 0) {
            return totalAmountOut;
```

3. token0 and token1 is converted to outputToken through `_verifyTokenAndSwap()` using the oogaBooga router:

```
 totalAmountOut += _verifyTokenAndSwap(swap0, address(token0), outputToken, receiver);
            token0Debt -= swap0.inputAmount;
            totalAmountOut += _verifyTokenAndSwap(swap1, address(token1), outputToken, receiver);
            token1Debt -= swap1.inputAmount;
```

4. The leftover tokens are sent back to the `_msgSender()`, which in this case is the StakeV2 contract.

```
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
```

Scenario:

Let's say a user earns \~1e18 of rewards through his staked Yeet tokens. He wants to claim these rewards and calls `claimRewardsInToken()`.

This 1e18 rewards gets him 1e18 KDK LP tokens, which is 1 WBera and 1 Yeet. Let's say the user doesn't want to swap all the token to outputToken and intends to get some WBera and Yeet back as well.

The user sets the parameter to ensure that he can get some WBera back, so swap0.inputAmount is \~0.5WBera.

Instead of getting 0.5WBera back, the 0.5WBera is left in the StakeV2.sol contract.
