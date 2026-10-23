# 56328 sc insight redundant require statement in eulerusdcstrategy deallocate function leads to unnecessary gas consumption

**Submitted on Oct 14th 2025 at 16:38:58 UTC by @Snuggle for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56328
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/EulerUSDCStrategy.sol>
* **Impacts:**

## Description

## Brief/Intro

The `_deallocate` function in `src/strategies/mainnet/EulerUSDCStrategy.sol` contains a redundant require statement that performs the same balance check twice. This leads to unnecessary gas consumption and code bloat without providing additional security benefits. The same issue exists in `EulerWETHStrategy.sol` contract.

## Vulnerability Details

* The `_deallocate` function contains two identical require statements checking the same condition:

```34:46:src/strategies/mainnet/eulerusdcstrategy.sol
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
        vault.withdraw(amount, address(this), address(this));
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
        uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
        uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
        if (usdcRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        return amount;
    }
```

* The first require statement on line 37 checks the balance after withdrawal
* The second require statement on line 44 performs the identical check before return
* Between these two checks, no USDC is spent or transferred out of the contract
* The `TokenUtils.safeApprove` call on line 38 does not reduce the contract's USDC balance

## Impact Details

* **Gas inefficiency**: Unnecessary gas consumption due to redundant balance check and external call
* **Code maintainability**: Redundant code makes the function harder to understand and maintain
* **No security benefit**: The second check provides no additional security since no state changes occur between the checks
* **Pattern repetition**: The same redundant pattern exists in `EulerWETHStrategy.sol` (lines 37, 44)
* **Severity**: Insight (code quality issue, not a security vulnerability)

## References

* `src/strategies/mainnet/EulerUSDCStrategy.sol`:
  * First redundant require statement

```37:37:src/strategies/mainnet/eulerusdcstrategy.sol
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

* Second redundant require statement

```44:44:src/strategies/mainnet/eulerusdcstrategy.sol
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

* `src/strategies/mainnet/EulerWETHStrategy.sol`:
  * Same redundant pattern exists (lines 37, 44)

```37:37:src/strategies/mainnet/eulerwethstrategy.sol
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

```44:44:src/strategies/mainnet/eulerwethstrategy.sol
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

## Recommendation

Remove the redundant require statement on line 44:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    vault.withdraw(amount, address(this), address(this));
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
    }
    return amount;
}
```

This maintains the same security guarantees while eliminating redundant gas consumption and improving code clarity.

## Proof of Concept

## Proof of Concept

**Steps to Reproduce:**

1. Call `_deallocate` function with any valid amount
2. Observe that both require statements execute the same balance check
3. Verify that no USDC is spent between the two checks (lines 37-44)

**Expected vs Actual:**

* Expected: Single balance check after withdrawal is sufficient
* Actual: Two identical balance checks are performed unnecessarily

**Gas Analysis:**

* Each `TokenUtils.safeBalanceOf` call consumes gas for external call
* Redundant check adds unnecessary gas cost to every deallocation
* No functional difference in behavior
