# 57272 sc medium silent failures on moonwell deposit are not catched by strategy

**Submitted on Oct 24th 2025 at 21:08:52 UTC by @Paludo0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57272
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Loss of funds for the protocol and users

## Description

## Brief/Intro

`MoonwellWETHStrategy:_allocate()` calls `mWETH.mint(amount)` without checking the returned status code.

When Moonwell’s `mint` fails without reverting, the underlying never leaves the strategy and remains idle.

## Vulnerability Details

In `MoonwellWETHStrategy:_allocate()` there's no check of returned value from `mWETH.mint(amount)`.

By looking into `mWETH.mint()` traces is evident that this call can fail silently without reverting for vaious reasons.

This is a snippet as non exhaustive example:

```
    function mintFresh(
        address minter,
        uint mintAmount
    ) internal returns (uint, uint) {
        /* Fail if mint not allowed */
        uint allowed = comptroller.mintAllowed(
            address(this),
            minter,
            mintAmount
        );
        if (allowed != 0) {
            return (
                failOpaque(
                    Error.COMPTROLLER_REJECTION,
                    FailureInfo.MINT_COMPTROLLER_REJECTION,
                    allowed
                ),
                0
            );
        }

        /* Verify market's block timestamp equals current block timestamp */
        if (accrualBlockTimestamp != getBlockTimestamp()) {
            return (
                fail(Error.MARKET_NOT_FRESH, FailureInfo.MINT_FRESHNESS_CHECK),
                0
            );
        }

        MintLocalVars memory vars;

        (
            vars.mathErr,
            vars.exchangeRateMantissa
        ) = exchangeRateStoredInternal();
        if (vars.mathErr != MathError.NO_ERROR) {
            return (
                failOpaque(
                    Error.MATH_ERROR,
                    FailureInfo.MINT_EXCHANGE_RATE_READ_FAILED,
                    uint(vars.mathErr)
                ),
                0
            );
   ...
```

Where `failOpaque` is as follows:

```
    /**
     * @dev use this when reporting an opaque error from an upgradeable collaborator contract
     */
    function failOpaque(
        Error err,
        FailureInfo info,
        uint opaqueError
    ) internal returns (uint) {
        emit Failure(uint(err), uint(info), opaqueError);

        return uint(err);
    }
```

Therefore it can happen that Moonwell call fails without reverting and underlying strategy funds remain inside strategy itself and Moonwell tokens are not minted.

This is the relevant snippet:

```
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(mWETH), amount);
        // Mint mWETH with underlying WETH
        mWETH.mint(amount);
        return amount;
    }
```

## Impact Details

VaultV2 uses `VaultV2::accrueInterestView()` to calculate shares or assets amounts in critical functions: `previewDeposit()`, `previewMint()`, `previewWithdraw()`, `previewRedeem()`.

`VaultV2::accrueInterestView()` sums realAssets in VaultV2 and in strategies adapters.

```
        uint256 realAssets = IERC20(asset).balanceOf(address(this));
        for (uint256 i = 0; i < adapters.length; i++) {
            realAssets += IAdapter(adapters[i]).realAssets();
        }
```

But `IAdapter(adapters[i]).realAssets()` omits strategy underlying balance, i.e. wETH in MoonwellWETHStrategy and USDC in MoonwellUSDCStrategy.

```
function realAssets() external view override returns (uint256) {
        // Use stored exchange rate and mToken balance to avoid state changes during static calls
        uint256 mTokenBalance = mWETH.balanceOf(address(this));
        if (mTokenBalance == 0) return 0;
        uint256 exchangeRate = mWETH.exchangeRateStored();
        // Exchange rate is scaled by 1e18, so we need to divide by 1e18
        return (mTokenBalance * exchangeRate) / 1e18;
    }
```

`VaultV2.totalAssets()` the price per share is too low, and newcomers can mint more shares then expected creating irreversible value extraction from existing LPs and potentially from the treasury via mischarged fees.

Funds are later recoverable during redeem for this failure mode, but the economic loss remains. This is the reason why I tagged this issue as critical.

The same issue applies to **MoonwellUSDCStrategy**.

## Proof of Concept

## Proof of Concept

The aim of this PoC is to demonstrate that if Moonwell minting function fails silently, the strategy call doesn't fail and the mTokens are not deposited

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "forge-std/StdCheats.sol";

// Inherit from your existing base test for MoonwellWETHStrategy
// NOTE: adjust the relative import path if needed.
import "./strategies/MoonwellWETHStrategy.t.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MoonwellWETHStrategy} from "../strategies/optimism/MoonwellWETHStrategy.sol";


interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function allowance(address, address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}

interface IMToken {
    function mint(uint256) external returns (uint256);
    function redeemUnderlying(uint256) external returns (uint256);
    function balanceOf(address) external view returns (uint256);
    function exchangeRateStored() external view returns (uint256);
}

/// @notice PoC that demonstrates:
/// 1) Silent-fail on Moonwell mint leaves underlying idle in the strategy;
/// 2) Strategy returns `amount` from allocate even though no mTokens were minted;
contract MoonwellWETHStrategy_SilentMint_PoC is MoonwellWETHStrategyTest {
    function _adapter() internal view returns (MoonwellWETHStrategy) {
        return MoonwellWETHStrategy(payable(strategy));
    }

    function _vault() internal view returns (address) {
        return vault;
    }

    function _weth() internal view returns (IERC20) {
        // Get WETH from the strategy's receiptToken
        return IERC20(_adapter().receiptToken());
    }

    function _mToken() internal view returns (IMToken) {
        // Get the mWETH token from the strategy
        return IMToken(address(_adapter().mWETH()));
    }

    // --- PoC 1: Silent mint failure leaves underlying idle; adapter returns `amount` ---
    function test_SilentMintFailure_NoRevert_UnderlyingRemainsIdle() public {
        // Arrange
        uint256 amount = 10 ether;

        // Ensure the strategy holds the underlying before allocate()
        // (Vaults commonly transfer underlying to the adapter before allocate. We emulate that.)
        deal(address(_weth()), address(_adapter()), amount);

        // Sanity check: strategy holds underlying, has 0 mTokens
        assertEq(_weth().balanceOf(address(_adapter())), amount, "pre: strategy WETH");
        assertEq(_mToken().balanceOf(address(_adapter())), 0, "pre: strategy mToken");

        // Mock Moonwell.mint(amount) to return non-zero (failure) without revert
        bytes memory callData = abi.encodeWithSelector(IMToken.mint.selector, amount);
        vm.mockCall(address(_mToken()), callData, abi.encode(uint256(1))); // 1 = failure code

        // Act: call allocate as if from the Vault with correct signature
        // allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        uint256 oldAllocation = 0; // Starting from 0
        bytes memory data = abi.encode(oldAllocation);

        vm.prank(_vault());
        (bytes32[] memory ids, int256 change) = _adapter().allocate(
            data,
            amount,
            bytes4(0), // selector - not critical for this test
            address(this) // sender
        );

        // Assert: strategy "reports" allocation = amount (unchecked return path)
        // change should equal amount since oldAllocation was 0
        assertEq(uint256(change), amount, "allocate() should report amount (unchecked path)");

        // Assert: no mTokens were minted; underlying never left the strategy
        assertEq(_mToken().balanceOf(address(_adapter())), 0, "no mTokens minted");
        assertEq(_weth().balanceOf(address(_adapter())), amount, "underlying still idle in strategy");
    }
}

```
