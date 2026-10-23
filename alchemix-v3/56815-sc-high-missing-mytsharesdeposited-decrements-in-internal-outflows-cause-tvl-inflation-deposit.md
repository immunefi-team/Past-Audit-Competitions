# 56815 sc high missing mytsharesdeposited decrements in internal outflows cause tvl inflation deposit dos

**Submitted on Oct 20th 2025 at 21:37:36 UTC by @ihtishamsudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56815
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Alchemist V3 broken internal accounting fails to decrement `_mytSharesDeposited` when MYT (yield token) is transferred out during force-repay and liquidation operations. it causes the (TVL) calculation to become inflated relative to the actual MYT balance held by the contract (demonstrated in poc below)

## Vulnerability Details

The AlchemistV3 contract maintains a private state variable `_mytSharesDeposited` to track the total MYT shares deposited into the protocol. This value is used in two functions:

1. `_getTotalUnderlyingValue()` - Converts `_mytSharesDeposited` to underlying value for TVL calculations
2. `deposit()` - Enforces the deposit cap: `_checkState(_mytSharesDeposited + amount <= depositCap)`

`_mytSharesDeposited` is correctly incremented on deposits and correctly decremented on user-initiated withdrawals, repayments with fees, and redemptions. However, it is nott decremented during internal MYT outflows in two paths:

```solidity
function _forceRepay(uint256 id) internal {
    // ... calculations ...
    
    // Transfer MYT to transmuter
    SafeERC20.safeTransfer(myt, transmuter, creditToYield);
    
    // Transfer protocol fee to fee receiver
    SafeERC20.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    
    // ❌ Missing: _mytSharesDeposited decrement
    
    // ... rest of function ...
}
```

```solidity
function _doLiquidation(
    uint256 id,
    uint256 amountLiquidated,
    uint256 debtToBurn,
    uint256 feeInYield,
    uint256 feeInUnderlying
) internal returns (uint256, uint256, uint256) {
    // ... calculations ...
    
    // Transfer liquidated amount to transmuter
    uint256 amountToTransmuter = amountLiquidated - feeInYield;
    SafeERC20.safeTransfer(myt, transmuter, amountToTransmuter);
    
    // Transfer fee to liquidator (if fee in yield)
    if (feeInYield > 0) {
        SafeERC20.safeTransfer(myt, msg.sender, feeInYield);
    }
    
    // ❌ Missing: _mytSharesDeposited decrement
    
    // ... rest of function ...
}
```

## Impact Details

Since the `deposit()` function enforces a cap check using the inflated `_mytSharesDeposited` value, legitimate users are permanently blocked from depositing once the discrepancy grows large enough, even when the actual token balance is well below the configured `depositCap`.

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L738> <https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L852>

## Proof of Concept

## Proof of Concept

Here's what i cooked:

```solidity
 /// PoC: Internal MYT outflows in liquidation don't decrement _mytSharesDeposited
    /// This inflates TVL (getTotalUnderlyingValue) and falsely DoSes new deposits via depositCap check
    function test_PoC_DepositCap_DoS_After_Liquidation() external {
        console.log("=== PoC: DepositCap DoS after liquidation ===");
        // 1) User deposits MYT into Alchemist, minting a new position
        uint256 depositAmt = minimumDeposit; // use existing test constant (1,000e18)
        console.log("[setup] externalUser depositing: ", depositAmt);
        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(depositAmt, externalUser, 0);
        vm.stopPrank();

        // Get the newly minted NFT tokenId owned by externalUser
        uint256 tokenId;
        for (uint256 id = 1; id <= 10; id++) {
            try IERC721(address(alchemistNFT)).ownerOf(id) returns (address owner) {
                if (owner == externalUser) { tokenId = id; break; }
            } catch {}
        }
        assertGt(tokenId, 0, "tokenId not found for externalUser");
        console.log("[info] position tokenId: ", tokenId);

        // Record current MYT balance held by Alchemist and set depositCap to this balance
        uint256 balBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        console.log("[state] alchemist MYT balance before: ", balBefore);
        vm.prank(alOwner);
        alchemist.setDepositCap(balBefore);
        console.log("[config] depositCap set to: ", alchemist.depositCap());

    // 2) Borrow maximum to sit at the minimum collateralization boundary
    uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
    vm.prank(externalUser);
    alchemist.mint(tokenId, maxBorrow, externalUser);
        console.log("[action] minted debt (maxBorrowable): ", maxBorrow);

        // Manipulate yield token price down so the position becomes liquidatable
    _manipulateYieldTokenPrice(590); // similar drop used in other liquidation tests (~5.9%)
    console.log("[manipulation] applied MYT price drop (bps): ", uint256(590));

        // 3) Liquidate: this transfers MYT from the contract to transmuter/liquidator
        vm.prank(someWhale);
        (uint256 liquidated, uint256 feeInYield, ) = alchemist.liquidate(tokenId);
        assertGt(liquidated, 0, "expected liquidation to move tokens");
        console.log("[liquidation] liquidated shares (yield): ", liquidated);
        console.log("[liquidation] fee paid in yield shares: ", feeInYield);

        // Contract's actual MYT balance decreased after liquidation
        uint256 balAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        assertLt(balAfter, balBefore, "MYT balance should decrease after liquidation");
        console.log("[state] alchemist MYT balance after: ", balAfter);

        // 4) TVL is inflated because _mytSharesDeposited wasn't decremented by outflows
        uint256 tvlUnderlying = alchemist.getTotalUnderlyingValue();
        uint256 actualUnderlying = IVaultV2(address(vault)).convertToAssets(balAfter);
        assertGt(tvlUnderlying, actualUnderlying, "TVL inflated due to missing _mytSharesDeposited decrement");
        console.log("[check] tvlUnderlying (from _mytSharesDeposited): ", tvlUnderlying);
        console.log("[check] actualUnderlying (from MYT balance): ", actualUnderlying);

        // 5) False deposit-cap DoS: even a 1 wei deposit reverts since the cap check uses inflated _mytSharesDeposited
        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), 1);
        console.log("[expect] next deposit of 1 wei should revert due to inflated TVL vs depositCap");
        vm.expectRevert(IllegalState.selector); // _checkState(_mytSharesDeposited + amount <= depositCap)
        alchemist.deposit(1, externalUser, 0);
        vm.stopPrank();
    }
```

* `Test Logs`

```bash
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_PoC_DepositCap_DoS_After_Liquidation() (gas: 1121814)
Logs:
  === PoC: DepositCap DoS after liquidation ===
  [setup] externalUser depositing:  1000000000000000000000
  [info] position tokenId:  1
  [state] alchemist MYT balance before:  1000000000000000000000
  [config] depositCap set to:  1000000000000000000000
  [action] minted debt (maxBorrowable):  900000000000000000090
  [manipulation] applied MYT price drop (bps):  590
  [liquidation] liquidated shares (yield):  953100000000000001008
  [liquidation] fee paid in yield shares:  0
  [state] alchemist MYT balance after:  46899999999999998992
  [check] tvlUnderlying (from _mytSharesDeposited):  944287063267233238000
  [check] actualUnderlying (from MYT balance):  44287063267233237910
  [expect] next deposit of 1 wei should revert due to inflated TVL vs depositCap

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 15.91ms (2.82ms CPU time)

Ran 1 test suite in 18.79ms (15.91ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
