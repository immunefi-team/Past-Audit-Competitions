# #41895 \[SC-Medium] Potential loss of token0, token1 in the MoneyBrinter contract

**Submitted on Mar 19th 2025 at 08:46:59 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41895
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The function `MoneyBrinter::compound()` is expected to be called by manager to compound rewards by swapping harvested tokens, staking in Kodiak vault and depositing into Beradrome farm. However, the unused token0, token1 held by the contract is not handled

## Vulnerability Details

The function `MoneyBrinter::compound()` calls `Zapper::zapInWithMultipleTokens()` to handle zapping in with multiple input tokens.

It is expected behavior that the Zapper contract returns the unused token0, token1 to the vault in the flow of compounding. However, that unused tokens returned to vault contract are not explicitly handled. Besides, if the manager is about to use these tokens for the next compounding, there can be 2 scenarios:

* Token0 and token1 are supposed to directly added to the liquidity pool ==> This is not supported by the function `Zapper::zapInWithMultipleTokens()` because all input tokens are swapped to token0 and token1
* The manager resolves the scenario (1) by swap token0 to token1 and token1 to token0. This case, the swaps will have to suffer slippage/price impacts. ==> The amounts of token0, token1 after swap can be lower than the original amounts ==> The actual amounts to be added into liquidity pool is lower than expected

```solidity
    // Zapper contract
    function zapInWithMultipleTokens(
        MultiSwapParams calldata swapParams,
        KodiakVaultStakingParams calldata stakingParams,
        VaultDepositParams calldata vaultParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(stakingParams.kodiakVault) returns (uint256, uint256) {

        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        // loop and swap Input tokens with corresponding swapData
        (uint256 _token0Debt, uint256 token1Debt) = _performMultiSwaps(token0, token1, swapParams);

        return _yeetIn(token0, token1, _token0Debt, token1Debt, stakingParams, vaultParams);

    function _performMultiSwaps(IERC20 token0, IERC20 token1, MultiSwapParams calldata params)
    internal
    returns (uint256 token0Debt, uint256 token1Debt)
    {
        require(
            params.inputTokens.length == params.swapToToken0.length + params.swapToToken1.length,
            "Zapper: Swap Array disparity"
        );

        for (uint256 i = 0; i < params.swapToToken0.length; i++) {
            // fetch user token
            IERC20(params.inputTokens[i]).safeTransferFrom(
                _msgSender(), address(this), params.swapToToken0[i].inputAmount
            );

// @> input tokens are swapped even when the input token is token0 or token1
            token0Debt +=
                            _verifyTokenAndSwap(params.swapToToken0[i], params.inputTokens[i], address(token0), address(this));
        }
        
        for (uint256 i = params.swapToToken0.length; i < params.inputTokens.length; i++) {
            // fetch user token
            IERC20(params.inputTokens[i]).safeTransferFrom(
                _msgSender(), address(this), params.swapToToken1[i - params.swapToToken0.length].inputAmount
            );

// @> input tokens are swapped even when the input token is token0 or token1
            token1Debt += _verifyTokenAndSwap(
                params.swapToToken1[i - params.swapToToken0.length],
                params.inputTokens[i],
                address(token1),
                address(this)
            );
        }
    }
    }
```

## Impact Details

* Token0, token1 held by MoneyBrinter contract can be either stuck or suffer swapping's slippage/price impacts before being added into liquidity pool

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L230-L240>

## Proof of Concept

## Proof of Concept

1. Rewards are harvested
2. The manager calls `compound()` with all holding rewards
3. There are unused token0, token1 in the process of zapping in, let's say `x` and `y` are unsued amount of token0 and token1
4. The manager can not directly add `x` token0 and `y` token1 into liquidity pool OR\
   if the manager tries to swap `x` token0 to token1 and `y` token1 to token1, then the resulted amounts are `x'` token0 and `y'` token1. Most of the cases, `x' < x` and `y' < y` happens => Loss of `x - x'`token0 and `y - y'` token1 for adding liquidity
