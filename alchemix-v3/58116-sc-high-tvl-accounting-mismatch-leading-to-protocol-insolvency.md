# 58116 sc high tvl accounting mismatch leading to protocol insolvency

**Submitted on Oct 30th 2025 at 18:37:22 UTC by @vah\_13 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58116
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

AlchemistV3 fails to decrement the internal `_mytSharesDeposited` accounting variable when transferring MYT tokens out during forced repayments (`_forceRepay`) and liquidations (`_doLiquidation`). This causes `getTotalUnderlyingValue()` to return overstated TVL values, leading to incorrect collateralization calculations, under-liquidations, bad debt accumulation, and potential protocol insolvency.

## Vulnerability Details

### Root Cause

The contract maintains an internal accounting variable `_mytSharesDeposited` to track total MYT shares deposited:

```solidity
// AlchemistV3.sol
uint256 private _mytSharesDeposited;

function _getTotalUnderlyingValue() internal view returns (uint256) {
    return IVaultV2(myt).convertToAssets(_mytSharesDeposited);
}
```

This variable is correctly incremented in `deposit()` (line 368) and decremented in `withdraw()` (line 403). However, when MYT tokens are forcibly transferred out in two critical paths, the accounting is **not updated**:

**Location 1: `_forceRepay()` at line 779**

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256 repaidInYield) {
    // ... calculations ...
    TokenUtils.safeTransfer(myt, transmuter, amount);  // ❌ Transfers MYT out
    // Missing: _mytSharesDeposited -= amount;
    _subDebt(accountId, debtToRepay);
    account.earmarked = account.earmarked > amount ? account.earmarked - amount : 0;
    return amount;
}
```

**Location 2: `_doLiquidation()` at lines 875, 879**

```solidity
function _doLiquidation(...) internal returns (...) {
    // ... calculations ...
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);  // ❌ Line 875

    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // ❌ Line 879
    }
    // Missing: _mytSharesDeposited -= (amountLiquidated);
}
```

## Proof of Concept

## Proof of Concept

```
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
import {TestERC20} from "../src/test/mocks/TestERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

/// @notice Foundry Script PoC: Demonstrates TVL/oracle mismatch after forceRepay/liquidation
/// @dev Bug #006: AlchemistV3 does not decrement `_mytSharesDeposited` when transferring MYT out
///      in `_forceRepay` and `_doLiquidation`, causing TVL to be overstated.
contract PoC_Bug006_TVL_Mismatch is Script {
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
        console.log("========================================");
        console.log("PoC: Critical TVL Mismatch Bug #006");
        console.log("========================================\n");

        vm.startBroadcast();

        deployer = msg.sender;

        // 1) Deploy underlying and MYT vault
        console.log("Step 1: Deploying underlying token and MYT vault...");
        underlying = new TestERC20(1_000_000e18, 18);
        myt = new VaultV2(deployer, address(underlying));
        myt.setName("MYT");
        myt.setSymbol("MYT");
        IERC20(address(underlying)).approve(address(myt), type(uint256).max);
        uint256 assets = 100_000e18;
        uint256 mintedShares = myt.deposit(assets, deployer);
        console.log("  Vault shares minted:", mintedShares);

        // 2) Deploy debtToken and Transmuter
        console.log("\nStep 2: Deploying debt token and Transmuter...");
        debtToken = new AlchemicTokenV3("alUSD", "alUSD", 0);
        ITransmuter.TransmuterInitializationParams memory tparams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(debtToken),
            feeReceiver: protocolFeeReceiver,
            timeToTransmute: 100,
            transmutationFee: 0,
            exitFee: 0,
            graphSize: 1000 // Allow deposits
        });
        transmuter = new Transmuter(tparams);

        // 3) Deploy AlchemistV3 with proxy
        console.log("\nStep 3: Deploying AlchemistV3...");
        AlchemistV3 alchemistLogic = new AlchemistV3();
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: deployer,
            debtToken: address(debtToken),
            underlyingToken: address(underlying),
            depositCap: type(uint256).max,
            minimumCollateralization: 2e18, // 200%
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

        // 4) Setup position NFT
        positionNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(positionNFT));

        // 5) Whitelist alchemist
        debtToken.setWhitelist(address(alchemist), true);

        // 6) Create position
        console.log("\nStep 4: Creating position and depositing MYT shares...");
        IERC20(address(myt)).approve(address(alchemist), type(uint256).max);
        uint256 sharesToDeposit = IERC20(address(myt)).balanceOf(deployer) / 2;
        uint256 debtValue = alchemist.deposit(sharesToDeposit, deployer, 0);
        console.log("  Debt value from deposit:", debtValue);

        uint256 tokenId = 1;

        // 7) Mint debt to capacity
        console.log("\nStep 5: Minting debt to maximum capacity...");
        uint256 capacity = alchemist.getMaxBorrowable(tokenId);
        console.log("  Max borrowable (capacity):", capacity);
        alchemist.mint(tokenId, capacity, deployer);

        // 8) Create redemption in transmuter
        console.log("\nStep 6: Creating transmuter redemption to generate earmarks...");
        IERC20(address(debtToken)).approve(address(transmuter), capacity);
        transmuter.setAlchemist(address(alchemist));
        transmuter.setDepositCap(uint256(type(int256).max)); // Max allowed by Transmuter validation
        uint256 redemptionAmount = capacity / 10;
        transmuter.createRedemption(redemptionAmount);
        console.log("  Redemption created:", redemptionAmount);

        // Advance block twice: once to make earmarks available, once more for liquidation timing
        vm.roll(block.number + 2);

        // 9) Measure TVL before liquidation
        console.log("\n========================================");
        console.log("BEFORE LIQUIDATION:");
        console.log("========================================");
        uint256 alchemistSharesBefore = IERC20(address(myt)).balanceOf(address(alchemist));
        uint256 actualUnderlyingBefore = IVaultV2(address(myt)).convertToAssets(alchemistSharesBefore);
        uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
        console.log("  Alchemist MYT shares:", alchemistSharesBefore);
        console.log("  Actual underlying (from vault):", actualUnderlyingBefore);
        console.log("  TVL (from _mytSharesDeposited):", tvlBefore);
        console.log("  Match?", actualUnderlyingBefore == tvlBefore ? "YES" : "NO");

        // 10) Call liquidation
        console.log("\n========================================");
        console.log("TRIGGERING LIQUIDATION...");
        console.log("========================================");
        (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        console.log("  Amount liquidated (yield):", amountLiquidated);
        console.log("  Fee in yield:", feeInYield);
        console.log("  Fee in underlying:", feeInUnderlying);

        // 11) Measure TVL after liquidation
        console.log("\n========================================");
        console.log("AFTER LIQUIDATION:");
        console.log("========================================");
        uint256 alchemistSharesAfter = IERC20(address(myt)).balanceOf(address(alchemist));
        uint256 actualUnderlyingAfter = IVaultV2(address(myt)).convertToAssets(alchemistSharesAfter);
        uint256 tvlAfter = alchemist.getTotalUnderlyingValue();
        console.log("  Alchemist MYT shares:", alchemistSharesAfter);
        console.log("  Actual underlying (from vault):", actualUnderlyingAfter);
        console.log("  TVL (from _mytSharesDeposited):", tvlAfter);
        console.log("  Match?", actualUnderlyingAfter == tvlAfter ? "YES" : "NO");

        // 12) Show the bug
        console.log("\n========================================");
        console.log("BUG DEMONSTRATION:");
        console.log("========================================");
        if (tvlAfter > actualUnderlyingAfter) {
            uint256 discrepancy = tvlAfter - actualUnderlyingAfter;
            console.log("  CRITICAL: TVL is OVERSTATED!");
            console.log("  Discrepancy:", discrepancy);
            console.log("  Percentage overstated (bps):", (discrepancy * 10000) / actualUnderlyingAfter);
            console.log("\n  Root cause: _mytSharesDeposited not decremented");
            console.log("  when MYT transferred out in _forceRepay/_doLiquidation");
        } else {
            console.log("  No discrepancy found");
        }
        console.log("========================================\n");

        vm.stopBroadcast();
    }
}

```

```
vah@vah-fuzz:~/Desktop/immunify/AlchemixV3/v3-poc$ forge script script/PoC_Bug006.s.sol --evm-version cancun
Warning: Found unknown `exclude` config for profile `default` defined in foundry.toml.
Warning: Found unknown `exclude` config for profile `lite` defined in foundry.toml.
Warning: Found unknown `fuzz_runs` config for profile `lite` defined in foundry.toml.
Warning: Found unknown `exclude` config for profile `test` defined in foundry.toml.
[⠰] Compiling...
No files changed, compilation skipped
Script ran successfully.
Gas used: 21263564

== Logs ==
  ========================================
  PoC: Critical TVL Mismatch Bug #006
  ========================================

  Step 1: Deploying underlying token and MYT vault...
    Vault shares minted: 100000000000000000000000
  
Step 2: Deploying debt token and Transmuter...
  
Step 3: Deploying AlchemistV3...
  
Step 4: Creating position and depositing MYT shares...
    Debt value from deposit: 50000000000000000000000
  
Step 5: Minting debt to maximum capacity...
    Max borrowable (capacity): 25000000000000000000000
  
Step 6: Creating transmuter redemption to generate earmarks...
    Redemption created: 2500000000000000000000
  
========================================
  BEFORE LIQUIDATION:
  ========================================
    Alchemist MYT shares: 50000000000000000000000
    Actual underlying (from vault): 50000000000000000000000
    TVL (from _mytSharesDeposited): 50000000000000000000000
    Match? YES
  
========================================
  TRIGGERING LIQUIDATION...
  ========================================
    Amount liquidated (yield): 50000000000000000000
    Fee in yield: 0
    Fee in underlying: 0
  
========================================
  AFTER LIQUIDATION:
  ========================================
    Alchemist MYT shares: 49950000000000000000000
    Actual underlying (from vault): 49950000000000000000000
    TVL (from _mytSharesDeposited): 50000000000000000000000
    Match? NO
  
========================================
  BUG DEMONSTRATION:
  ========================================
    CRITICAL: TVL is OVERSTATED!
    Discrepancy: 50000000000000000000
    Percentage overstated (bps): 10
  
  Root cause: _mytSharesDeposited not decremented
    when MYT transferred out in _forceRepay/_doLiquidation
  ========================================

```
