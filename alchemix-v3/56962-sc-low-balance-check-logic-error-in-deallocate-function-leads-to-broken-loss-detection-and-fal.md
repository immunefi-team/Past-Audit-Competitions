# 56962 sc low balance check logic error in deallocate function leads to broken loss detection and false event emissions

**Submitted on Oct 22nd 2025 at 08:55:07 UTC by @HandsomeEarthworm6 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56962
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_deallocate() function in MorphoYearnOGWETHStrategy.sol contains a logic error where both balance checks (wethBalanceBefore and wethBalanceAfter) are performed after the vault withdrawal operation, rather than before and after. This causes the calculated wethRedeemed value to always equal zero, completely breaking the loss detection mechanism and causing false event emissions.

## Vulnerability Details

Since both wethBalanceBefore and wethBalanceAfter are measured at the same point in time (after the withdrawal), they will always contain the same value, making wethRedeemed = 0 in all cases.

function \_deallocate(uint256 amount) internal override returns (uint256) {

vault.withdraw(amount, address(this), address(this));

uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));@audit,both check happen after

uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this)); @audit ,check also happens after

uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore; if (wethRedeemed < amount) {

emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed); } require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed"); require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed"); TokenUtils.safeApprove(address(weth), msg.sender, amount); return amount; }

## Impact Details

1.Broken Loss Detection: The contract cannot detect if the vault returns less WETH than requested due to slippage, fees, or vault issues. This is a critical safety mechanism that is completely non-functional.

2.Redundant Safety Check: The require(wethRedeemed + wethBalanceBefore >= amount) check on line 54-55 becomes meaningless because it essentially checks require(0 + currentBalance >= amount), which is then immediately followed by the redundant check

## Reccomendation

uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this)); // Check BEFORE vault.withdraw(amount, address(this), address(this)); // Then withdraw uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this)); // Check AFTER

## Proof of Concept

## Proof of Concept

function test\_ProveWethRedeemedIsAlwaysZero() public { // Setup: Give strategy 100 WETH in vault weth.mint(address(strategy), 100 ether); strategy.allocate(100 ether);

```
    // Deallocate 50 WETH - vault returns exactly 50 WETH (no loss)
    vm.recordLogs();
    strategy.deallocate(50 ether);
    
    // Verify the bug: event shows 0 actual when 50 was redeemed
    Vm.Log[] memory logs = vm.getRecordedLogs();
    
    // Decode the StrategyDeallocationLoss event
    (string memory message, uint256 requested, uint256 actual) = 
        abi.decode(logs[0].data, (string, uint256, uint256));
    
    // BUG PROOF: actual is 0, not 50!
    assertEq(actual, 0, "BUG: wethRedeemed is 0 instead of 50");
    assertEq(requested, 50 ether, "Requested amount is correct");
    
    // The vault DID return 50 WETH (verify by checking balance)
    assertEq(weth.balanceOf(address(strategy)), 50 ether, "Strategy has 50 WETH");
}
```
