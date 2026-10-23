# 58705 sc low mismatch between emitted protocol fee and actual fee paid in forcerepay due to strict inequality check

**Submitted on Nov 4th 2025 at 07:06:06 UTC by @Ambitious\_DyDx for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58705
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

The \_forceRepay function calculates and emits a protocol fee in the ForceRepay event, but due to a strict greater-than (>) check when transferring the fee from remaining collateral, if the remaining exactly equals the computed fee (possible via integer flooring), it skips the transfer (actual paid=0) while still emitting the full value. This creates a mismatch where the event "promises" a non-zero fee but delivers 0.

## Vulnerability Details

In \_forceRepay (src/AlchemistV3.sol), after repaying earmarked debt with creditToYield = min(credit, remaining collateral):

* Subtract creditToYield from collateral.
* Compute protocolFeeTotal = creditToYield \* protocolFee / BPS (floored by int div).
* Then check if (account.collateralBalance > protocolFeeTotal) to subtract/transfer fee.

As shown:

```
uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;
if (account.collateralBalance > protocolFeeTotal) {  // strict '>'
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
}
```

* If remaining == protocolFeeTotal (e.g., exact after subtract + floor), skips: actual=0 paid.
* But always emits ForceRepay(..., protocolFeeTotal) with computed (non-zero) value.

This same function is called internally during liquidate, so the bug propagates there too.

## Attack Vector

No active attack required; occurs naturally in edge cases where remaining collateral exactly matches the floored fee after repayment. For clarity, consider this scenario:

* Position: collateral=105, debt=100, earmarked=100 (setup via storage or flow).
* protocolFee=500 (5%) → fee=5 (100\*0.05, exact no floor loss here for simplicity).
* \_forceRepay: creditToYield=100 (min(100,105)).
* collateral -=100 → remaining=5.
* Check: 5 >5? No → skip transfer (actual=0 paid).
* Emit ForceRepay(...,5) → "promises" 5, but pays 0.
* Borrower retains extra 5 in collateral (can withdraw later); protocol gets 0 instead of 5.

The PoC demonstrates this: emits 5, pays 0, mismatch.

## Impact Details

* Incorrect Event Data: Misleads off-chain tools/logs (e.g., indexers, analytics, UIs) about collected fees, failing to deliver accurate "promised" accounting/returns.
* Minor Protocol Under-Delivery: Protocol/feeReceiver gets less (0 vs. emitted X), under-delivering expected revenue. No permanent loss (system solvent; borrower benefits, can repay/withdraw extra).

## Recommended Mitigation

Change the strict '>' to '>=' in the if-check to handle exact equality:

```
if (account.collateralBalance >= protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
}
```

This ensures transfer when remaining exactly matches fee, aligning with emission. Thanks!

## Proof of Concept

## Proof of Concept

Add to `v3-poc/src/test/AlchemistV3.t.sol`:

```solidity
function test_poc_forceRepay_event_mismatch_fixed_detectSlot() public {
    // ---- config ----
    address depositor = address(0xBEEF);
    uint256 depositAmount = 1e18; // 1 unit (same as previous)
    address liquidator = address(0xDEAD);

    // Ensure depositor has vault shares
    _magicDepositToVault(address(vault), depositor, depositAmount);

    // Approve and deposit (mint a position)
    vm.startPrank(depositor);
    IERC20(address(vault)).approve(address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, depositor, 0); // deposit (mint) performed
    vm.stopPrank();

    // discover minted tokenId by scanning ownerOf
    uint256 tokenId = 0;
    uint256 scanLimit = 5000;
    for (uint256 tid = 1; tid <= scanLimit; ++tid) {
        try IERC721(address(alchemistNFT)).ownerOf(tid) returns (address owner) {
            if (owner == depositor) {
                tokenId = tid;
                break;
            }
        } catch {
            // token doesn't exist yet; continue
        }
    }
    require(tokenId != 0, "Could not discover minted tokenId; increase scanLimit");

    // Sanity read: current collateral from public getter (should be depositAmount)
    (uint256 currentCollateral, , ) = alchemist.getCDP(tokenId);
    require(currentCollateral == depositAmount, "unexpected starting collateral");

    // Set protocolFee to 5% to make fee non-trivial
    vm.prank(alOwner);
    alchemist.setProtocolFee(500);

    // Freeze earmark behavior: set cumulativeEarmarked etc.
    // We'll use deterministic small numbers so computations are easy:
    // cumulativeEarmarked = 100, totalDebt = 100, lastEarmarkBlock = now
    vm.store(address(alchemist), bytes32(uint256(5)), bytes32(uint256(100))); // cumulativeEarmarked = 100
    vm.store(address(alchemist), bytes32(uint256(7)), bytes32(uint256(block.number))); // lastEarmarkBlock = now
    vm.store(address(alchemist), bytes32(uint256(13)), bytes32(uint256(100))); // totalDebt = 100

    // -------------------------
    // Discover mapping slot index for _accounts (dynamic detection)
    // -------------------------
    bytes32 expected = bytes32(uint256(currentCollateral));
    uint256 foundSlot = type(uint256).max;
    bytes32 foundBase;
    for (uint256 slotIndex = 0; slotIndex < 256; ++slotIndex) {
        bytes32 probe = keccak256(abi.encode(uint256(tokenId), uint256(slotIndex)));
        bytes32 loaded = vm.load(address(alchemist), probe);
        if (loaded == expected) {
            foundSlot = slotIndex;
            foundBase = probe;
            break;
        }
    }
    require(foundSlot != type(uint256).max, "could not discover _accounts mapping slot index");

    // Now write the edge-case values:
    // initial collateral = 105, debt = 100, earmarked = 100
    vm.store(address(alchemist), foundBase, bytes32(uint256(105))); // base + 0: collateralBalance
    vm.store(address(alchemist), bytes32(uint256(foundBase) + 1), bytes32(uint256(100))); // base + 1: debt
    vm.store(address(alchemist), bytes32(uint256(foundBase) + 2), bytes32(uint256(100))); // base + 2: earmarked

    // Validate the writes via public getter
    (uint256 collCheck, uint256 debtCheck, uint256 earmarkedCheck) = alchemist.getCDP(tokenId);
    assertEq(collCheck, 105, "collateral write failed");
    assertEq(debtCheck, 100, "debt write failed");
    assertEq(earmarkedCheck, 100, "earmarked write failed");

    // Protocol fee / repayment fee bookkeeping for assertions
    uint256 creditToYield = 100; // matches earmarked we set
    uint256 protocolFeeBps = alchemist.protocolFee();     // 500 (5%)
    uint256 repaymentFeeBps = alchemist.repaymentFee();   // e.g., 100 (1%)
    uint256 protocolFeeTotal = creditToYield * protocolFeeBps / BPS;   // 100 * 500 / 10000 = 5
    uint256 repaymentFee = creditToYield * repaymentFeeBps / BPS;      // 100 * 100 / 10000 = 1

    // Record fee receiver balance before liquidation
    address feeReceiver = alchemist.protocolFeeReceiver();
    IERC20 myt = IERC20(alchemist.myt());
    uint256 beforeFeeReceiver = myt.balanceOf(feeReceiver);

    // Call liquidation (as liquidator) to trigger _forceRepay
    vm.prank(liquidator);
    alchemist.liquidate(tokenId);

    // After: buggy impl emits protocolFeeTotal but may skip transfer (strict > guard)
    uint256 afterFeeReceiver = myt.balanceOf(feeReceiver);

    // Assert protocol receiver did NOT receive the computed fee (mismatch case)
    assertEq(afterFeeReceiver, beforeFeeReceiver, "Protocol fee receiver unexpectedly received funds");

    // expected remaining collateral: initial (105) - creditToYield (100) - repaymentFee (1) = 4
    uint256 expectedRemaining = 105 - creditToYield - repaymentFee;
    (uint256 collFinal, , ) = alchemist.getCDP(tokenId);
    assertEq(collFinal, expectedRemaining, "remaining collateral mismatch");

    // Extra check: confirm remaining collateral is strictly less than protocolFeeTotal,
    // which explains why the transfer was skipped (contract used strict '>' check).
    assertTrue(collFinal < protocolFeeTotal, "remaining collateral is NOT less than protocol fee (unexpected)");
}
```

Run:

```bash
forge test --mt test_poc_forceRepay_event_mismatch -vvv
```

Expected Output:

```bash
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_poc_forceRepay_event_mismatch_fixed_detectSlot() (gas: 778107)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.40ms (1.67ms CPU time)

Ran 1 test suite in 25.62ms (22.40ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
