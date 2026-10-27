# 57771 sc medium fee not collected in forcerepay when should

**Submitted on Oct 28th 2025 at 19:44:26 UTC by @dldLambda for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57771
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In \_forceRepay, after subtracting the repayment amount from collateral, the protocol calculates protocolFeeTotal and checks if (account.collateralBalance > protocolFeeTotal) before deducting and transferring the fee to protocolFeeReceiver. Because of the strict > comparison, the fee is not deducted when account.collateralBalance == protocolFeeTotal. Other parts of the code (e.g., repay) allow equality — this leads to inconsistent behavior and potential loss of protocol revenue in edge cases.

## Vulnerability Details

Let's look at the code:

1. \_forceRepay:

```
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }
```

2. repay: (correct)

```
        // Debt is subject to protocol fee similar to redemptions
        uint256 feeAmount = creditToYield * protocolFee / BPS;
        if (feeAmount > account.collateralBalance) {
            revert("Not enough collateral to pay for debt fee");
        } else {
            account.collateralBalance -= creditToYield * protocolFee / BPS;
        }
```

3. \_doLiquidation: (correct)

```
        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
```

As you can see, in functions repay and \_doLiquidation fee is written off if `fee<=account.collateralBalance`.

However, function \_forceRepay, when `fee = account.collateralBalance`, considers that the fee cannot be written off.

This is an unjustified inconsistency and may result in loss of fee.

## Impact Details

The protocol does not receive the expected protocol fee in edge cases => economic damage (lost commission)

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852> - \_doLiquidation

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L498> - repay

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738> - \_forceRepay

## Proof of Concept

## Proof of Concept

Run this simple script-simulation:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Script.sol";
import "forge-std/console.sol";

contract ForceRepaySimulation is Script {

    struct ForceRepayAccount {
        uint256 collateralBalance;
        uint256 protocolFeeTotal;
    }

    function simulateForceRepay(ForceRepayAccount memory account) internal pure returns (bool commissionPaid) {
        if (account.collateralBalance > account.protocolFeeTotal) {
            commissionPaid = true; 
        } else {
            commissionPaid = false; 
        }
    }

    function run() external {
        console.log("=== ForceRepay Bug Simulation ===");

        ForceRepayAccount memory a1 = ForceRepayAccount({collateralBalance: 100, protocolFeeTotal: 100});
        ForceRepayAccount memory a2 = ForceRepayAccount({collateralBalance: 101, protocolFeeTotal: 100});
        ForceRepayAccount memory a3 = ForceRepayAccount({collateralBalance: 50, protocolFeeTotal: 100});

        console.log("a1 commissionPaid:", simulateForceRepay(a1)); // false
        console.log("a2 commissionPaid:", simulateForceRepay(a2)); // true
        console.log("a3 commissionPaid:", simulateForceRepay(a3)); // false
    }
}

```

commands:

1. forge init poc
2. cd poc
3. add this script to script/ folder
4. forge build
5. forge script script/ForceRepaySimulation.s.sol:ForceRepaySimulation --broadcast

And you will see:

```
== Logs ==
  === ForceRepay Bug Simulation ===
  a1 commissionPaid: false
  a2 commissionPaid: true
  a3 commissionPaid: false
```
