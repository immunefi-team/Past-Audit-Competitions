# 57530 sc high stale tvl accounting in liquidations leads to protocol insolvency

**Submitted on Oct 27th 2025 at 00:26:52 UTC by @legion for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57530
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `AlchemistV3` contract contains a critical accounting flaw where liquidations transfer yield token shares out of the protocol but fail to update the global `_mytSharesDeposited` counter. This creates a permanent divergence between the protocol's reported Total Value Locked (TVL) and the actual on-chain balance. Because critical protocol operations—including deposit caps, liquidation calculations, and health checks—rely on this inflated TVL number, the system gradually becomes insolvent as liquidations accumulate bad debt while appearing healthy. Users are unable to redeem their synthetic tokens once the transmuter exhausts its real collateral reserves.

## Vulnerability Details

The `AlchemistV3` contract accepts deposits of yield-bearing tokens (e.g., Morpho vault shares) and allows users to borrow synthetic debt tokens against this collateral. The protocol maintains a single global state variable `_mytSharesDeposited` to track the total amount of yield token shares held by the contract:

```solidity
uint256 private _mytSharesDeposited;
```

This counter is intended to represent the actual on-chain balance of yield tokens that the contract controls. It serves as the foundation for several critical protocol functions:

1. **TVL Reporting**: `getTotalUnderlyingValue()` converts `_mytSharesDeposited` to underlying token value
2. **Deposit Cap Enforcement**: Checks `_mytSharesDeposited + amount <= depositCap` before accepting deposits
3. **Global Health Metrics**: Feeds into `calculateLiquidation()` to determine protocol-wide collateralization
4. **Risk Assessment**: Used by operators and dashboards to monitor protocol solvency

The bug occurs in the way it was used during liquidation, during liquidations; tokens are transferred OUT of the contract but `_mytSharesDeposited` is never decremented:

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal
    returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    Account storage account = _accounts[accountId];
    
    // Calculate how much to seize
    (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = 
        calculateLiquidation(...);
    
    amountLiquidated = convertDebtTokensToYield(liquidationAmount);
    feeInYield = convertDebtTokensToYield(baseFee);
    
    // Update user's individual balance
    account.collateralBalance -= amountLiquidated;
    _subDebt(accountId, debtToBurn);
    
    // Transfer tokens OUT to transmuter (net liquidation amount)
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
    
    // Transfer tokens OUT to liquidator (fee)
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    }
    
    // MISSING: _mytSharesDeposited -= amountLiquidated;
    // The global counter is never updated!
    
    emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
}
```

\*State after liquidation:

* Contract's actual balance: `-amountLiquidated` (tokens sent to transmuter + liquidator)
* `_mytSharesDeposited`: unchanged (still includes the liquidated tokens)
* Accounting is now desynchronized by `amountLiquidated`.

Since the protocol's TVL calculation fully trusts `_mytSharesDeposited`:

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    // Converts the stale counter to underlying token value
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    return yieldTokenTVLInUnderlying;
}

function getTotalUnderlyingValue() external view returns (uint256) {
    return _getTotalUnderlyingValue();
}
```

After liquidations occur, this function returns a value higher than the actual contract balance:

Reported TVL = convertYieldTokensToUnderlying(\_mytSharesDeposited) Includes tokens already sent to transmuter/liquidators, while Actual TVL = Only counts tokens still in the contract.

Example with Numbers

**Initial State:**

* `_mytSharesDeposited = 1000e18` shares
* Actual contract balance = `1000e18` shares
* Share price = 1:1 for simplicity
* Reported TVL = 1000 underlying tokens

**User deposits and borrows:**

* User deposits `100e18` more shares
* `_mytSharesDeposited = 1100e18`
* User borrows max amount (assuming 200% collateralization)
* User debt = `50 debt tokens` (worth 50 underlying)

**Price drops, liquidation triggered:**

* Liquidator calls `liquidate(tokenId)`
* `amountLiquidated = 60e18` shares calculated
* `feeInYield = 5e18` shares
* Total removed from contract = `65e18` shares

**Transfers executed:**

* `55e18` shares → Transmuter
* `5e18` shares → Liquidator
* User's `collateralBalance` reduced by `60e18`
* Actual contract balance = `1100e18 - 65e18 = 1035e18` shares

**BUT:**

* `_mytSharesDeposited` still = `1100e18`
* Reported TVL = `1100 underlying` (inflated by 65)
* Actual TVL = `1035 underlying`
* **Discrepancy = 65 underlying tokens**

After just one liquidation, the protocol believes it has 6.3% more collateral than it actually controls. This gap compounds with every subsequent liquidation. since the `calculateLiquidation` logic checks if the protocol's global collateralization ratio is healthy:

```solidity
if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    // Fully liquidate because protocol is unhealthy globally
    outsourcedFee = (debt * feeBps) / BPS;
    return (debt, debt, 0, outsourcedFee);
}
```

With inflated TVL:

* Protocol appears healthier than reality
* Liquidations seize **less collateral** than needed
* Under-liquidation leaves bad debt in the system
* Global collateralization continues degrading

## Impact Details

1. Protocol Insolvency **Mechanism:**

* Liquidations appear to restore health but actually accumulate bad debt
* Global collateralization calculation uses inflated TVL
* System believes it's overcollateralized when actually undercollateralized
* No mechanism exists to correct the accounting error retroactively

**Consequences:**

* Bad debt accrues silently across multiple liquidation events
* Eventually, total debt > actual collateral value
* Transmuter cannot fulfill redemption requests
* Synthetic token loses peg permanently
* Users lose funds with no recovery mechanism

2. Permanent Deposit Denial-of-Service (High)

**Mechanism:**

```solidity
_checkState(_mytSharesDeposited + amount <= depositCap);
```

Since `_mytSharesDeposited` never decreases after liquidations:

**Example:**

* Deposit cap set to `10,000e18`
* Current `_mytSharesDeposited = 9,800e18`
* Actual balance after liquidations = `8,500e18`
* Real capacity available = `1,500e18`
* Reported capacity = `200e18`
* New deposit of `500e18` rejected despite ample real space

**Consequences:**

* Protocol cannot accept new capital even when undercollateralized
* Emergency capital injections during crisis are blocked
* Strategy migrations fail (require new deposits)
* System enters "frozen" state where it can only bleed value
* Recovery becomes impossible without contract upgrade

3. Under-Liquidation and Cascading Failures (High)

**Mechanism:** The `calculateLiquidation` function includes this logic:

```solidity
if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    // FULL liquidation: seize all debt worth of collateral
    outsourcedFee = (debt * feeBps) / BPS;
    return (debt, debt, 0, outsourcedFee);
}

// Otherwise, PARTIAL liquidation: only restore to target ratio
// ... partial liquidation math ...
```

With inflated TVL:

* Global ratio appears: `(inflated TVL) / (total debt)` = looks healthy
* Triggers partial liquidation when full liquidation needed
* Leaves underwater positions in the system
* Creates a death spiral: a. Under-liquidation → bad debt remains b. Next liquidation also under-liquidates (TVL still inflated) c. Bad debt compounds d. Actual collateralization degrades e. More positions become liquidatable f. Cycle repeats

**Consequences:**

* Liquidators seize less collateral than required to restore health
* Individual accounts left with debt > collateral value
* Protocol accumulates systemic bad debt
* Even proper liquidations can't fix earlier under-liquidations
* Point of no return reached quickly

4. False Risk Metrics and Failed Monitoring (Medium)

**Mechanism:**

* Operators and automated systems call `getTotalUnderlyingValue()`
* Dashboards show healthy collateralization ratios
* Alert thresholds never trigger
* Governance believes system is functioning normally

**Consequences:**

* No early warning of insolvency
* Risk management decisions based on false data
* Emergency response delayed until user funds already lost
* Reputation damage when "healthy" system suddenly fails
* Regulatory and legal exposure from misleading metrics

## References

**Primary Source Code:**

* [`src/AlchemistV3.sol#L134`](broken://pages/ea149fe9e53ca66466addfdfc8502ac9954d33cc#L134) — Declaration of `_mytSharesDeposited` state variable
* [`src/AlchemistV3.sol#L363-L389`](broken://pages/ea149fe9e53ca66466addfdfc8502ac9954d33cc#L363-L389) — `deposit()` function correctly incrementing counter
* [`src/AlchemistV3.sol#L391-L418`](broken://pages/ea149fe9e53ca66466addfdfc8502ac9954d33cc#L391-L418) — `withdraw()` function correctly decrementing counter
* [`src/AlchemistV3.sol#L851-L914`](broken://pages/ea149fe9e53ca66466addfdfc8502ac9954d33cc#L851-L914) — `_doLiquidation()` function with missing decrement
* [`src/AlchemistV3.sol#L1239-L1242`](broken://pages/ea149fe9e53ca66466addfdfc8502ac9954d33cc#L1239-L1242) — `_getTotalUnderlyingValue()` trusting stale counter
* [`src/AlchemistV3.sol#L869-L876`](broken://pages/ea149fe9e53ca66466addfdfc8502ac9954d33cc#L869-L876) — `calculateLiquidation()` call site using inflated TVL

## Proof of Concept

## Proof of Concept## Proof of Concept

// SPDX-License-Identifier: GPL-3.0-or-later pragma solidity 0.8.28;

import {AlchemistV3Test} from "./AlchemistV3.t.sol"; import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol"; import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol"; import {IMockYieldToken} from "./mocks/MockYieldToken.sol"; import {SafeERC20} from "../libraries/SafeERC20.sol"; import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

contract AlchemistV3TVLInflationPoC is AlchemistV3Test { /// @dev Demonstrates that `_mytSharesDeposited` is not decremented when liquidations move /// MYT shares out of the contract, causing the reported TVL to diverge from reality. function testReportedTVLInflatedAfterLiquidation() external { // Prime the mock yield token so the vault has ample supply. vm.startPrank(someWhale); IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale); vm.stopPrank();

```
    // Keep global collateralization healthy with an auxiliary deposit.
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Victim deposits, borrows to the max, and becomes eligible for liquidation after price move.
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, alchemist.getMaxBorrowable(tokenId), address(0xbeef));
    vm.stopPrank();

    // Decrease share price by inflating the mock vault supply.
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    uint256 reportedBefore = alchemist.getTotalUnderlyingValue();
    uint256 onChainBefore = IVaultV2(vault).convertToAssets(IERC20(address(vault)).balanceOf(address(alchemist)));
    assertApproxEqAbs(reportedBefore, onChainBefore, 1e9); // sanity: aligned before liquidation

    // Liquidator repays earmarked debt and seizes collateral.
    vm.prank(externalUser);
    (uint256 liquidatedAmount,,) = alchemist.liquidate(tokenId);
    assertGt(liquidatedAmount, 0);

    uint256 reportedAfter = alchemist.getTotalUnderlyingValue();
    uint256 onChainAfter = IVaultV2(vault).convertToAssets(IERC20(address(vault)).balanceOf(address(alchemist)));

    // PoC: TVL bookkeeping thinks the vault still holds all shares, but actual balance is lower.
    assertGt(reportedAfter, onChainAfter);
    uint256 delta = reportedAfter - onChainAfter;
    assertGt(delta, 1e21); // deviation is material (~1e3 underlying after scaling).
}
```

} //Deposit-cap DoS: once a few liquidations happen the cap stays “full,” so fresh deposits or strategy rotations just brick. //Under-liquidation & systemic insolvency: calculateLiquidation leans on the inflated TVL when deciding whether to escalate to full liquidations. The system ends up believing it’s healthy while the real collateral is gone, leaving bad debt to accumulate. //Operators flying blind: every TVL-derived metric is wrong, so risk dashboards and on-chain checks miss the insolvency until redemptions start failing.
