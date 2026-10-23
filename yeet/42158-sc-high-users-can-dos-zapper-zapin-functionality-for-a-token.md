# #42158 \[SC-High] Users can DoS \`Zapper::zapIn\` functionality for a token

**Submitted on Mar 21st 2025 at 11:29:11 UTC by @kmm for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42158
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In certain edge cases involving Kodiak Island's liquidity pools, specifically when `totalSupply` is 0 or when there is only one-sided liquidity, a user can exploit the `safeIncreaseAllowance` mechanism used by the zapper. This allows the user to perform a Denial-of-Service (DoS) attack on the router's approval functionality, effectively blocking any future deposit attempts via the Kodiak router for that specific token.

## Vulnerability Details

Kodiak Island calculates mint share rates using the following logic:

```solidity
function getMintAmounts(uint256 amount0Max, uint256 amount1Max) external view returns (uint256 amount0, uint256 amount1, uint256 mintAmount) {
    uint256 totalSupply = totalSupply();
    if (totalSupply > 0) {
        (amount0, amount1, mintAmount) = _computeMintAmounts(totalSupply, amount0Max, amount1Max);
    } else {
        (uint160 sqrtRatioX96,,,,,,) = pool.slot0();
        uint128 newLiquidity = LiquidityAmounts.getLiquidityForAmounts(sqrtRatioX96, lowerTick.getSqrtRatioAtTick(), upperTick.getSqrtRatioAtTick(), amount0Max, amount1Max);
        mintAmount = uint256(newLiquidity);
        (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(sqrtRatioX96, lowerTick.getSqrtRatioAtTick(), upperTick.getSqrtRatioAtTick(), newLiquidity);
    }
}

function _computeMintAmounts(uint256 totalSupply, uint256 amount0Max, uint256 amount1Max) private view returns (uint256 amount0, uint256 amount1, uint256 mintAmount) {
    (uint256 amount0Current, uint256 amount1Current) = getUnderlyingBalances();

    if (amount0Current == 0 && amount1Current > 0) {
        mintAmount = FullMath.mulDiv(amount1Max, totalSupply, amount1Current);
    } else if (amount1Current == 0 && amount0Current > 0) {
        mintAmount = FullMath.mulDiv(amount0Max, totalSupply, amount0Current);
    } else if (amount0Current == 0 && amount1Current == 0) {
        revert("");
    } else {
        uint256 amount0Mint = FullMath.mulDiv(amount0Max, totalSupply, amount0Current);
        uint256 amount1Mint = FullMath.mulDiv(amount1Max, totalSupply, amount1Current);
        require(amount0Mint > 0 && amount1Mint > 0, "mint 0");

        mintAmount = amount0Mint < amount1Mint ? amount0Mint : amount1Mint;
    }

    amount0 = FullMath.mulDivRoundingUp(mintAmount, amount0Current, totalSupply);
    amount1 = FullMath.mulDivRoundingUp(mintAmount, amount1Current, totalSupply);
}
```

When `totalSupply == 0`, the mint amount directly correlates to the liquidity minted from the provided token amounts. In the case of one-sided liquidity, the vault only pulls one of the token amounts.

The zapper's `_yeetIn` function sets token allowances to the router using `safeIncreaseAllowance` for both `amount0Max` and `amount1Max`, and then invokes `addLiquidity`. However, the `zapInNative` function does **not** validate `amount0Max` or `amount1Max`. This opens the door for an attacker to set a massive allowance, such as `type(uint256).max`, without actually spending the tokens.

In zero-supply situations, an attacker can manipulate the pool state (e.g., by shifting it outside of `tickLower` or `tickUpper`) to ensure one of the token amounts calculated by `getAmountsForLiquidity` returns zero. This causes the vault to pull only the other token, allowing the attacker to set an inflated approval for the unused token.

For pools with one-sided liquidity, this same effect occurs without the need to push the pool outside any tick range, as one of the amounts will inherently be zero.

## Impact Details

This vulnerability enables an attacker to push the router's token allowance to near `type(uint256).max` without spending it, leveraging the behavior of `safeIncreaseAllowance`. As a result, future attempts to increase the allowance for that token will fail, effectively blocking zapper deposits for that token. This creates a persistent DoS condition for the affected token within the Kodiak Island ecosystem.

## References

* [Zapper.sol - Line 507-508](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L507-L508)
* [Zapper.sol - Line 132-159](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L132-L159)
* [SafeERC20 - Line 70](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6015d7ef79172976b949be6e4b6c495593983c99/contracts/token/ERC20/utils/SafeERC20.sol#L70C9-L70C21)

## Proof of Concept

## Proof of Concept

**totalSupply=0**

1. The vault is added with `totalSupply=0`
2. The user sees this and in one transaction
   * Pushes the price outside of the range in the island
   * Calls `zapInNative`, and passes `amount1Max` as `type(uint256).max`
   * The island only pulls `amount0` leaving the approval at `type(uint256).max`
3. Any further attempt at depositing this token, will revert due to overflow in `safeIncreaseAllowance`.
