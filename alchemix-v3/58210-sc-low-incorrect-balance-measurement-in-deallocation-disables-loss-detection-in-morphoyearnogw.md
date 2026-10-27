# 58210 sc low incorrect balance measurement in deallocation disables loss detection in morphoyearnogweth&#x20;

**Submitted on Oct 31st 2025 at 12:14:34 UTC by @godwinudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58210
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `MorphoYearnOGWETH` strategy has a logic error in its `deallocation` function, where both balance measurements are taken after the vault withdrawal completes, rather than before and after. This causes the loss detection mechanism to always report zero loss regardless of actual losses incurred, allowing the strategy to attempt approving more tokens than it actually holds. When the Morpho Yearn vault experiences any losses due to fees, slippage, or market conditions during withdrawal, the strategy will fail to detect these losses and subsequently revert when trying to transfer more tokens than available, breaking the deallocation process entirely.

## Vulnerability Details

When funds need to be withdrawn from a yield-generating protocol, the strategy's `deallocate` function is called. This function must accomplish several important goals: withdraw the requested amount from the underlying protocol, detect if any losses occurred during the withdrawal, emit appropriate events if losses are detected, and approve the exact amount that was actually received back to the vault.

The loss detection mechanism is particularly important because yield-generating protocols like Morpho Yearn vaults do not always return exactly what you request. Due to the vault's exchange rate mechanics, protocol fees, rounding differences, or market conditions, you might deposit ten tokens but only be able to withdraw 9.85 tokens later.

Every other strategy in the codebase correctly implements this loss detection by measuring the balance before the withdrawal operation and then measuring it again after the operation completes.

Here is an example from the `EulerARBWETHStrategy`:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

Notice the careful sequencing here. The balance is read before the withdrawal happens, then the vault withdrawal executes, and then the balance is read again after the withdrawal completes. The difference between these two measurements tells us exactly how many tokens the vault actually returned. If the vault returns less than requested, the strategy emits a loss event to alert the system.

Now, let us look at the broken implementation in `MorphoYearnOGWETH`:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

Here, both balance readings happen after the vault withdrawal completes. When you read the balance at line two and then read it again at line three, you are reading the exact same value twice because nothing has happened between these two operations. The mathematical result of subtracting the same number from itself is always zero, which means `wethRedeemed` will always equal zero regardless of what the vault actually returned.

If wethRedeemed`is zero, the conditional check`if (wethRedeemed < amount)\` will always evaluate to true, but more importantly, the require statement assumes the strategy has the full requested amount available. The function then attempts to approve the full requested amount to the vault caller. However, if the vault experienced any loss during withdrawal, the strategy will not actually have enough tokens to fulfill this approval, causing the transaction to revert with an insufficient balance error.

The Morpho Yearn vault uses an exchange rate system where shares are converted to underlying assets. This exchange rate can fluctuate based on the vault's performance, accrued interest, and other factors. When you deposit tokens, you receive shares based on the current exchange rate. When you withdraw, your shares are converted back to tokens at the potentially different current exchange rate. This is normal vault behavior and why loss detection is essential.

## Impact Details

This issue completely breaks the deallocation functionality of the `MorphoYearnOGWETH` strategy whenever the Morpho Yearn vault returns less than the requested amount. The deallocation process will revert with an ERC20 insufficient balance error, preventing any withdrawals from this strategy. This makes the strategy unable to rebalance funds or fulfill user withdrawal requests that depend on accessing capital from this strategy. The loss detection mechanism provides no warning to operators about vault performance issues, as the `StrategyDeallocationLoss` event never emits with meaningful data. All funds allocated to this strategy become effectively locked until the vault's exchange rate improves enough to allow full withdrawals without losses

## Proof of Concept

## Proof of Concept

Need to set your MAINNET\_RPC\_URL

Then, Add this to the `MorphoYearnOGWETHStrategy.t.sol` test suite and run

```solidity
function test_deallocate_loss_detection_broken() public {
    vm.startPrank(vault);
    uint256 amount = 10e18;
    
    // Allocate
    deal(WETH, strategy, amount);
    bytes memory prevAlloc = abi.encode(0);
    IMYTStrategy(strategy).allocate(prevAlloc, amount, "", address(vault));
    
    // Deallocate - will revert due to bug
    // Vault returns ~9.858e18 but bug makes wethRedeemed=0
    // so no loss detected, tries to approve 10e18 with only 9.858e18
    bytes memory prevAlloc2 = abi.encode(amount);
    vm.expectRevert(); // Expecting insufficient balance revert
    IMYTStrategy(strategy).deallocate(prevAlloc2, amount, "", address(vault));
    
    vm.stopPrank();
}
```
