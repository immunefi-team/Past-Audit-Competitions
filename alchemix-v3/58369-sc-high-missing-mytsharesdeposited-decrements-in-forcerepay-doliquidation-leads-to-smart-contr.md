# 58369 sc high missing mytsharesdeposited decrements in forcerepay doliquidation leads to smart contract unable to operate due to lack of token funds

**Submitted on Nov 1st 2025 at 17:27:23 UTC by @gor97 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58369
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

The AlchemistV3 protocol contains a critical accounting vulnerability where the internal `_mytSharesDeposited` variable fails to be decremented when MYT tokens are transferred out during forced repayments (`_forceRepay()`) and liquidations (`_doLiquidation()`). This accounting mismatch causes the `_getTotalUnderlyingValue()` function to return progressively overstated Total Value Locked (TVL) calculations, which compound with each liquidation event. As the discrepancy between reported and actual token balances grows, the smart contract becomes unable to operate properly due to insufficient token funds relative to what the protocol believes it holds, ultimately leading to operational failure and potential system-wide dysfunction.

### Vulnerability Details

The vulnerability stems from incomplete accounting maintenance in two critical functions within the AlchemistV3 contract. The contract uses an internal variable `_mytSharesDeposited` (line 134) to track the total MYT shares deposited:

```solidity
uint256 private _mytSharesDeposited;
```

This variable is used by the `_getTotalUnderlyingValue()` function (line 1238-1241) to calculate the protocol's TVL:

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

The contract correctly maintains this accounting variable in most operations:

* **Incremented** during `deposit()` at line 383: `_mytSharesDeposited += amount;`
* **Decremented** during `withdraw()` at line 410: `_mytSharesDeposited -= amount;`
* **Decremented** for protocol fees in multiple locations

However, two critical functions transfer MYT tokens out without updating the accounting:

#### Location 1: `_forceRepay()` Function (Line 779)

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ... calculation logic ...
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); // Transfers MYT out
    // Missing: _mytSharesDeposited -= creditToYield;
    return creditToYield;
}
```

#### Location 2: `_doLiquidation()` Function (Lines 875, 879)

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    // ... calculation logic ...
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);  // Line 875
    
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // Line 879
    }
    // Missing: _mytSharesDeposited -= amountLiquidated;
}
```

#### Technical Impact Chain

1. **Accounting Discrepancy**: Each liquidation/forced repayment creates a growing gap between `_mytSharesDeposited` and actual MYT token balance
2. **TVL Overstatement**: `_getTotalUnderlyingValue()` returns inflated values based on the incorrect `_mytSharesDeposited`
3. **Dependent Function Failures**: Functions like `totalValue()` (line 1073) and collateralization calculations rely on accurate TVL data
4. **Operational Breakdown**: As the discrepancy grows, operations requiring actual token transfers fail when the contract believes it has more tokens than it actually possesses

#### Vulnerability Reproduction Path

1. User deposits 50,000 MYT shares → `_mytSharesDeposited = 50,000`
2. User borrows against collateral and creates transmuter redemption
3. Liquidation occurs → `_forceRepay()` transfers 50 MYT shares to Transmuter
4. **Actual MYT balance: 49,950 shares** (correctly decreased)
5. **`_mytSharesDeposited`: 50,000 shares** (incorrectly unchanged)
6. **TVL calculation overstated by 50 shares**
7. Process repeats with each liquidation, compounding the discrepancy

### Impact Details

The vulnerability manifests as progressive operational degradation with compounding effects:

#### Primary Impact: Smart Contract Operational Failure

* **Token Fund Shortage**: The contract believes it holds more MYT tokens than it actually possesses
* **Operation Failures**: Functions requiring token transfers fail when attempting to move non-existent tokens
* **Progressive Deterioration**: Each liquidation/forced repayment event worsens the discrepancy

#### Financial Quantification

* **Per-Event Impact**: Each liquidation creates a permanent accounting gap equivalent to the liquidated amount
* **Cumulative Effect**: The discrepancy compounds with protocol usage - active protocols with frequent liquidations will accumulate larger discrepancies faster
* **System-Wide Risk**: Affects all users as the core TVL calculation becomes increasingly unreliable

#### Operational Consequences

1. **Withdrawal Failures**: Users may be unable to withdraw funds when the contract's perceived balance exceeds actual balance
2. **Liquidation System Breakdown**: Incorrect TVL calculations affect liquidation triggers and calculations
3. **Risk Management Failure**: Collateralization ratios become unreliable, preventing proper risk assessment
4. **Protocol Dysfunction**: Core protocol operations that depend on accurate token accounting begin to fail

#### Severity Factors

* **Likelihood**: HIGH - Occurs during normal protocol operations (liquidations and forced repayments)
* **Impact Scope**: ALL protocol users affected as TVL is a global calculation
* **Persistence**: Bug effects are permanent and accumulate over time
* **Recovery Difficulty**: Requires protocol-level intervention to correct accumulated discrepancies

#### Code Locations

* **Primary Contract**: `/src/AlchemistV3.sol`
* **Vulnerable Functions**:
  * `_forceRepay()` - Line 779
  * `_doLiquidation()` - Lines 875, 879
* **Affected Calculations**:
  * `_getTotalUnderlyingValue()` - Line 1238-1241
  * `totalValue()` - Line 1073 (depends on accurate TVL)

#### Recommended Fixes

1. **For `_forceRepay()`**: Add `_mytSharesDeposited -= creditToYield;` after the token transfer
2. **For `_doLiquidation()`**: Add `_mytSharesDeposited -= amountLiquidated;` after both token transfers
3. **Additional Safeguards**: Implement assertion checks to verify `_mytSharesDeposited == IERC20(myt).balanceOf(address(this))` in critical functions

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import "forge-std/Script.sol";
import "forge-std/console.sol";

import {AlchemistV3} from "../src/AlchemistV3.sol";
import {IAlchemistV3, AlchemistInitializationParams} from "../src/interfaces/IAlchemistV3.sol";
import {AlchemistV3Position} from "../src/AlchemistV3Position.sol";
import {Transmuter} from "../src/Transmuter.sol";
import {ITransmuter} from "../src/interfaces/ITransmuter.sol";
import {VaultV2} from "vault-v2/VaultV2.sol";
import {IVaultV2} from "vault-v2/interfaces/IVaultV2.sol";
import {AlchemicTokenV3} from "../src/test/mocks/AlchemicTokenV3.sol";
import {TestERC20} from "./TestERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

contract PoC_Bug_TVL_Mismatch is Script {
    AlchemistV3 alchemist;
    AlchemistV3Position positionNFT;
    Transmuter transmuter;
    AlchemicTokenV3 debtToken;
    VaultV2 myt;
    TestERC20 underlying;

    address protocolFeeReceiver = address(0x10);
    address proxyOwner = address(0x99);
    address deployer;

    function run() external {
        console.log("PoC: TVL Mismatch Bug");
        console.log("========================================");

        vm.startBroadcast();
        deployer = msg.sender;

        // Deploy underlying token and MYT vault
        underlying = new TestERC20(1_000_000e18, 18);
        myt = new VaultV2(deployer, address(underlying));
        myt.setName("MYT");
        myt.setSymbol("MYT");
        IERC20(address(underlying)).approve(address(myt), type(uint256).max);
        uint256 assets = 100_000e18;
        myt.deposit(assets, deployer);

        // Deploy debt token and transmuter
        debtToken = new AlchemicTokenV3("alUSD", "alUSD", 0);
        ITransmuter.TransmuterInitializationParams memory tparams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(debtToken),
            feeReceiver: protocolFeeReceiver,
            timeToTransmute: 100,
            transmutationFee: 0,
            exitFee: 0,
            graphSize: 1000
        });
        transmuter = new Transmuter(tparams);

        // Deploy AlchemistV3 with proxy
        AlchemistV3 alchemistLogic = new AlchemistV3();
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: deployer,
            debtToken: address(debtToken),
            underlyingToken: address(underlying),
            depositCap: type(uint256).max,
            minimumCollateralization: 2e18,
            globalMinimumCollateralization: 2e18,
            collateralizationLowerBound: 2e18,
            transmuter: address(transmuter),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: 0,
            repaymentFee: 0,
            myt: address(myt)
        });
        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        TransparentUpgradeableProxy proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Setup position NFT and whitelist
        positionNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(positionNFT));
        debtToken.setWhitelist(address(alchemist), true);

        // Create position and deposit
        IERC20(address(myt)).approve(address(alchemist), type(uint256).max);
        uint256 sharesToDeposit = IERC20(address(myt)).balanceOf(deployer) / 2;
        alchemist.deposit(sharesToDeposit, deployer, 0);
        uint256 tokenId = 1;

        // Mint debt to maximum capacity
        uint256 capacity = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, capacity, deployer);

        // Create transmuter redemption
        IERC20(address(debtToken)).approve(address(transmuter), capacity);
        transmuter.setAlchemist(address(alchemist));
        transmuter.setDepositCap(uint256(type(int256).max));
        uint256 redemptionAmount = capacity / 10;
        transmuter.createRedemption(redemptionAmount);

        vm.roll(block.number + 2);

        // Measure TVL before liquidation
        console.log("BEFORE LIQUIDATION:");
        uint256 alchemistSharesBefore = IERC20(address(myt)).balanceOf(address(alchemist));
        uint256 actualUnderlyingBefore = IVaultV2(address(myt)).convertToAssets(alchemistSharesBefore);
        uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
        console.log("  Alchemist MYT shares:", alchemistSharesBefore);
        console.log("  Actual underlying:", actualUnderlyingBefore);
        console.log("  TVL reported:", tvlBefore);
        console.log("  Match?", actualUnderlyingBefore == tvlBefore ? "YES" : "NO");

        // Trigger liquidation
        console.log("TRIGGERING LIQUIDATION...");
        alchemist.liquidate(tokenId);

        // Measure TVL after liquidation
        console.log("AFTER LIQUIDATION:");
        uint256 alchemistSharesAfter = IERC20(address(myt)).balanceOf(address(alchemist));
        uint256 actualUnderlyingAfter = IVaultV2(address(myt)).convertToAssets(alchemistSharesAfter);
        uint256 tvlAfter = alchemist.getTotalUnderlyingValue();
        console.log("  Alchemist MYT shares:", alchemistSharesAfter);
        console.log("  Actual underlying:", actualUnderlyingAfter);
        console.log("  TVL reported:", tvlAfter);
        console.log("  Match?", actualUnderlyingAfter == tvlAfter ? "YES" : "NO");

        // Demonstrate the bug
        console.log("BUG RESULT:");
        if (tvlAfter > actualUnderlyingAfter) {
            uint256 discrepancy = tvlAfter - actualUnderlyingAfter;
            console.log("  TVL OVERSTATED by:", discrepancy);
            console.log("  Percentage (bps):", (discrepancy * 10000) / actualUnderlyingAfter);
            console.log("  Root cause: _mytSharesDeposited not decremented in liquidation");
        } else {
            console.log("  No discrepancy found");
        }
        console.log("========================================");

        vm.stopBroadcast();
    }
}
```
