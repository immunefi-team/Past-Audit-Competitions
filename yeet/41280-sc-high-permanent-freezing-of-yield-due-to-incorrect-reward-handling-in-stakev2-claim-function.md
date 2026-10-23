# #41280 \[SC-High] Permanent freezing of yield due to incorrect reward handling in \`StakeV2\` claim functions

**Submitted on Mar 13th 2025 at 09:39:03 UTC by @merlinboii for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41280
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Users claiming rewards through `StakeV2`'s claim functions: `claimRewardsInNative()` , `claimRewardsInToken0()`, `claimRewardsInToken1()` and `claimRewardsInToken()`, which are designed to handle reward claims in a single output token (either native, token0, token1, or a whitelisted token) can **permanently lose access to a portion of their rewards**.

This occurs because the `Zapper` contract incorrectly sends any remaining token debt to `StakeV2` instead of the user who initiated the claim. Since `StakeV2` has no mechanism to recover or redistribute these tokens, the lost funds become permanently inaccessible to the user.

## Vulnerability Details

This vulnerability stems from **a flawed interaction between StakeV2’s claim functions and Zapper’s token return logic** when **users opt to swap their rewards into a single output token**:

1. In `claimRewardsInNative() -> zapOutNative() -> _swapToWBERA()`: The remaining `token0Debt` and `token1Debt` are transferred to `_msgSender()`, which is now refer to `msg.sender` (`StakeV2`)

```solidity
function _swapToWBERA(
...
) internal returns (uint256 wBeraDebt) {
    if (address(token0) == address(wbera)) {
        wBeraDebt += token0Debt;
        token0Debt = 0;
    } else {
        wBeraDebt += _verifyTokenAndSwap(swapData0, address(token0), address(wbera), address(this));
        token0Debt -= swapData0.inputAmount;
    }

    if (address(token1) == address(wbera)) {
        wBeraDebt += token1Debt;
        token1Debt = 0;
    } else {
        wBeraDebt += _verifyTokenAndSwap(swapData1, address(token1), address(wbera), address(this));
        token1Debt -= swapData1.inputAmount;
    }
    // log yeetBalance
@>  _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());  
}
```

2. In `claimRewardsInToken0() -> zapOutToToken0()`: The remaining `token1Debt` is transferred to `_msgSender()`, which is now refer to `msg.sender` (`StakeV2`)

```solidity
function zapOutToToken0(
    ...
) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken0Out) {
    --- SNIPPED ---
    token1Debt -= swapData.inputAmount;
    token0Debt += _verifyTokenAndSwap(swapData, address(token1), address(token0), address(this));
    _sendERC20Token(token0, receiver, token0Debt);
@>  _sendERC20Token(token1, _msgSender(), token1Debt);
    return (token0Debt);
}
```

3. In `claimRewardsInToken1() -> zapOutToToken1()`: The remaining `token0Debt` is transferred to `_msgSender()`, which is now refer to `msg.sender` (`StakeV2`)

```solidity
function zapOutToToken1(
    ...
) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken1Out) {
    --- SNIPPED ---
    token0Debt -= swapData.inputAmount;
    token1Debt += _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this));
@>  _sendERC20Token(token0, _msgSender(), token0Debt);
    _sendERC20Token(token1, receiver, token1Debt);
    return (token1Debt);
}
```

4. In `claimRewardsInToken() -> zapOut()`: The remaining `token0Debt` and `token1Debt` are transferred to `_msgSender()`, which is now refer to `msg.sender` (`StakeV2`)

```solidity
function zapOut(    
    ...
)
public
override
nonReentrant
onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault)
returns (uint256 totalAmountOut)
{
    --- SNIPPED ---
    } else {
        // sent directly to receiver. What is receiver is zapper?
        totalAmountOut += _verifyTokenAndSwap(swap0, address(token0), outputToken, receiver);
        token0Debt -= swap0.inputAmount;
        totalAmountOut += _verifyTokenAndSwap(swap1, address(token1), outputToken, receiver);
        token1Debt -= swap1.inputAmount;
    }
@>  _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
}
```

5. **In the `StakeV2`, there is no logic to handle the remaining token debt, whether after calling the Zapper function or an external function to handle this case.**

## Impact Details

Users lose access to their full earned rewards when claiming through `StakeV2` if they opt to receive a single output token (`Native, Token0, Token1, or a whitelisted token`) and do not perfectly specify the swap amount.

1. If the user specifies `swapDataX.inputAmount < tokenXDebt` (the amount received after removing liquidity from Kodiak), the excess `tokenXDebt - swapDataX.inputAmount` is sent to `StakeV2` instead of the user and users permanently lose access to a portion of their earned rewards.
2. It is **not feasible for users to perfectly predict `swapDataX.inputAmount`** as the remove liquidity function only guarantees slippage protection but cannot provide an exact output amount so **users cannot reliably specify an inputAmount that exactly matches `tokenXDebt`, making some amount of loss inevitable**.

```solidity
function _approveAndUnstakeFromKodiakVault(
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
    uint256 islandTokenDebt
) internal returns (IERC20, IERC20, uint256, uint256) {
    --- SNIPPED ---
    (uint256 _amount0, uint256 _amount1,) = kodiakStakingRouter.removeLiquidity(
        IKodiakVaultV1(unstakeParams.kodiakVault),
        islandTokenDebt,
@>      unstakeParams.amount0Min,
@>      unstakeParams.amount1Min,
        unstakeParams.receiver
    );

    // require(islandTokenDebt == _liqBurned, "Invalid island token burn amount");
    return (_token0, _token1, _amount0, _amount1);
}
```

### The severity assessment

This issue leads to `Permanent Freezing of Unclaimed Yield` under the intended functionality of `StakeV2`’s claiming functions: `claimRewardsInNative()` , `claimRewardsInToken0()`, `claimRewardsInToken1()` and `claimRewardsInToken()`. **The intended functionality of these functions is to return rewards in a single output token, but due to this bug, users permanently lose access to a portion of their rewards**.

**Although there is a workaround** (users can set `unstakeParams.receiver != address(this)`, bypassing the swap and receiving `token0` and `token1` directly), **users may not be aware** of this workaround and **the default intention behavior encourages using the claim functions, which exposes users to this bug**.

A case could be made for `Low` Severity (`Contract fails to deliver promised returns but doesn’t lose value`), as the lost yield remains within the contract rather than being drained or stolen, **However, because the logic directly impacts user funds (yield)** and **leads to irreversible loss in most cases**, I believe a `Medium` severity rating is more appropriate.

## References

* The claim reward functions of `StakeV2` : <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L327-L394>
* The related Zapper's functions: <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L249-L353>
* The related vulnerable lines:
  * <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L262>
  * <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L284>
  * <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L310>
  * <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L622>
  * <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L352>

## Proof of Concept

## Proof of Concept

Use the case `claimRewardsInToken0()` for example, here's how users can permanently lose their remaining debt equivalent to a portion of their rewards:

### Prerequisites

* User has staked YEET in `StakeV2` and earned `500` trifecta vault shares as rewards, which can be redeemed into **YEET-WBERA LP tokens**
* Current YEET-WBERA Kodiak Island ratio: `1 YEET = 0.00167 BERA` (`token0 = YEET`, `token1 = WBERA`)
* Expected rewards after removing LP: `2,000 YEET + 3.34 WBERA`
* **What the user can estimate (but not precisely predict at execution time)**:
  * The **amount of vault shares** available for redemption (`calculateRewardsEarned()`).
  * The **amount of `YEET` and `WBERA` received from LP removal** (from current of Kodiak Island vault state).

### Vulnerability Flow

1. The user intends to claim all rewards as `YEET` by swapping `WBERA` to `YEET`.

```solidity
StakeV2.claimRewardsInToken0(
    500 shares,  // Amount of vault shares to redeem
    IZapper.SingleTokenSwap({
        inputAmount: 3.30 * 1e18,  // User estimates to swap all output WBERA to YEET
        outputQuote: 2,000 * 1e18,  // Quote amount for YEET
        outputMin: 1,995 * 1e18,  // Slippage for YEET
        executor: address(executor),
        path: swapPathData
    }),
    IZapper.KodiakVaultUnstakingParams({
        kodiakVault: address(YEET_BERA_KODIAK_ISLAND),
        amount0Min: 1,995 * 1e18,  // Slippage for YEET
        amount1Min: 3.30 * 1e18,  // Slippage for WBERA
        receiver: address(zapper),  // Opt in swap process
    }),
    redeemParams // the amount to claim will be prepared in `StakeV2._verifyAndPrepareClaim()`
);
```

2. The actual amount from Kodiak remove liquidity:

* YEET received: 2,010 YEET
* WBERA received: 3.34 WBERA
* These amounts include `LP fees` and potential price impact variations.

3. The swap processes a fixed `inputAmount` for swapping `WBERA` to `YEET` via `OBRouter` and the given `executor`:

* Swap `inputAmount`: 3.30 WBERA
* Swap `output`: 1,995 YEET
* Balance Zapper after swap:
  * YEET: 2,010 (from LP) + 1,995 (swap output) = 4,005 YEET
  * WBERA: 3.34 (from LP) - 3.30 (swapped) = 0.04 WBERA

4. Zapper processes to transfer remaining Debt:

* YEET: 4,005 YEET -> sent to user (`receiver`)
* WBERA: 0.04 WBERA -> sent to `StakeV2` (`_msgSender()`)

### Final State: User's loss 0.04 WBERA and permanently locked in the `StakeV2`

**Additional Note**: If the lost amount is in `YEET`, it will be incorrectly redistributed as extra rewards via: `StakeV2.executeRewardDistributionYeet()` so this creates unfair rewards distribution, benefiting other claimers.
