# \#98906 [SC-High] Trader-triggered degradedMode latch freezes matured Junior deposits and LP epoch settlement until the owner acts

**Submitted on Nov 10th 2026 at 21:09:54 UTC by @arcomus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98906
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

A trader with two accounts (one position per account is allowed) fills a hedged book to the solvency limit for new opens, waits for a 0.8% basket move in the smaller leg's favor and closes that leg. That latches `degradedMode` with zero terminal deficit, and every `settleLpEpoch` reverts: matured Junior deposits can't activate or be cancelled. While he holds the other leg, the owner can't clear without paying in the shortfall; after he exits, it still takes an owner transaction.

## Vulnerability Details

Closes and liquidations end in `_enterDegradedModeIfInsolvent` (CfdEngine.sol:1320, :1344, :1351-1369), which sets the sticky `degradedMode` flag when E (pool assets minus trader claims, :865-871) falls below L (the larger side's max profit, :1044-1050). That's the protocol's raw-solvency test, but new opens only need E >= L + B, with B = 25 bps of L (CfdEnginePlanLib.sol:755-772). On a hedged book at that limit, closing the smaller, winning leg pays out more than E-L while the larger leg keeps L unchanged, so the trader picks when E < L happens. Then `HousePool._settleLpEpoch` reverts (HousePool.sol:832, :1864-1870), a matured Junior deposit has no cancel path (TrancheVault.sol:851-860 only opens one on rejection, projected wipe or Senior impairment), and only the owner can reset the flag:

```solidity
// CfdEngine.sol:1364-1368, after every close and liquidation
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
// CfdEngine.sol:658-667, the only place that sets it back to false
function clearDegradedMode() external onlyOwner {
    // ... NotDegraded check
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

No listed audit has a trader setting the latch on purpose (nearest: Cecuro LOW-22, LOW-73), and the Known Limitations (SECURITY.md:1328-1457) never mention degraded mode. The spec lets closes complete into the latch and keeps it until the owner clears it (ACCOUNTING_SPEC.md:620, :1294-1295, WHITEPAPER.md:1098-1099); the repo's close-driven latch test uses an 80% crash. That is containment for a close revealing a loss. Here the book passed `E >= L + B` at every rung, 0.8% is enough, nothing is written off, the attacker picks the moment and repeats it after recovery.

## Impact Details

All numbers are PoC measurements at release parameters on an unfrozen weekday (both asserted). On a 10M pool, carol's 1,000,000 Junior deposit and bob's ~1M redeem mature at boundary M. Using two 400k accounts and his own keeper, the attacker builds a LONG 9,940,000 / SHORT 9,974,800 book (E-L-B 270.80). After a -1% basket move (LONG profits when the basket falls, CfdTypes.sol:14-19), closing the long in M's first block latches `degradedMode` at E-L -21,444.21 with terminal deficit 0 (-21,444.22 if closed at M-1s, before any settlement). A 0.80% move already latches.

At M, M+1h+1s and M+24h, settlement reverts, carol can't cancel, liquidating the short reverts `PositionIsSolvent`, and the owner's clear reverts `StillInsolvent`. Activating carol took three owner txs (21,444.21 recap, clear, `assignUnassignedAssets`).

Once the short closes, L is 0, free assets are 10,055,086.10 and realized Junior principal is 8,055,063.79 (was 8,000,000.99). Yet 1h+1s and 24h later settlement still reverts, carol's 1,000,000 stays locked, bob's redeem unfunded, and her own `clearDegradedMode` call reverts `OwnableUnauthorizedAccount`.

The attack cost him 67,000.04. After the same recovery, 300k more capital and a second -1% move latched it again (E-L -23,586.53; total cost 124,692.95 marked to market).

Deposits queued for M (new requests revert while latched) and matured redemptions stay frozen until the owner acts. Clearing is an immediate owner power, but the page assumes no response deadline, and for an unexpected `EngineDegraded` the guide prescribes containment and recovery only after root-cause review (EMERGENCY_RESPONSE_GUIDE.md:178, :221-223).

## Recommendation

Let anyone call `clearDegradedMode` under its existing `E >= L` check, so settlement reopens once raw solvency is back without waiting on the owner. With the diff, carol's own clear works after the exit and one keeper settlement activates her and funds bob; while the short is open it still reverts `StillInsolvent`, and the 33 related repo tests pass.

## References

- Latch after close / liquidation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369 (called at https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1320 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1344)
- Owner-only clear: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667
- E and L: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L865-L871 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1050
- Side convention (LONG profits when the basket falls): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTypes.sol#L12-L19
- Solvency check for new opens, E >= L + B: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L755-L772
- Settlement gate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L832 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870
- Matured Junior deposit cancel: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860
- One position per account (line 38): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/README.md#L38
- Latch is sticky and owner-cleared; opens need E >= L + B while closes may complete into it: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L619-L620, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1294-L1295 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1098-L1099
- Repo close-driven latch test (80% crash): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/close/DegradedSettlementBehavior.t.sol#L33-L46
- Known Limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1328-L1457
- Emergency guide, EngineDegraded response and governance recovery: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/EMERGENCY_RESPONSE_GUIDE.md#L178 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/EMERGENCY_RESPONSE_GUIDE.md#L219-L228
- Settlement buffer 25 bps (release parameter): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L143

## Link to Proof of Concept

https://gist.github.com/antonio-rosendo/ea9d93d5d738c23da7a9bfd260d8f0f6

## Proof of Concept

## Proof of Concept

The secret gist has the test file, a README and the fix diff: https://gist.github.com/antonio-rosendo/ea9d93d5d738c23da7a9bfd260d8f0f6

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
export GIST=/absolute/path/to/this/gist
mkdir -p packages/perps/test/perps/poc/round3 && cp "$GIST"/DegradedLatchFreeze.t.sol packages/perps/test/perps/poc/round3/
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/round3/DegradedLatchFreeze.t.sol -vv
```

The run should print `Suite result: ok. 8 passed; 0 failed; 0 skipped`. Two controls don't latch (no move; a smaller close that keeps E >= L), and in both carol's deposit activates at M.

Key test, lines 529-568 of `DegradedLatchFreeze.t.sol` (helpers in the same file):

```solidity
    function test_R3_1_Persistence_AfterFullExit_StillFrozenUntilOwner() public {
        console2.log("=== R3-1 persistence: attacker exits, cash restored, settlement still frozen ===");
        _r3Book();
        _moveAndCloseAtBoundary(bk.sizeA, R3_MOVED_PRICE);
        _assertLatched();
        _checkpoint("M (latch block)", true);

        // The attacker releases the losing leg ten minutes later (closes are allowed while degraded).
        _warpTo(bk.mat + 600);
        _trade(EVE_B, CfdTypes.Side.SHORT, bk.sizeB, 0, true);
        (uint256 sizeB,,,,,,) = engine.positions(EVE_B);
        assertEq(sizeB, 0, "B fully closed");
        assertEq(_maxLiability(), 0, "no open interest left: L == 0");
        assertTrue(engine.degradedMode(), "latch survives the attacker's full exit");
        uint256 free = pool.getFreeAssets();
        uint256 bobValue = juniorVault.estimateRedeemAssets(bk.bobShares);
        console2.log(string.concat("pool free assets after full exit          : ", _usd(int256(free))));
        console2.log(string.concat("bob matured redeem value (estimate)       : ", _usd(int256(bobValue))));
        assertGe(free, bobValue, "free cash already covers bob's matured redemption");
        _logBook("after attacker full exit");
        _assertRealizedLpNavNotReduced();

        uint256 release = bk.mat + 600;
        _warpTo(release + 1 hours + 1);
        _checkpoint("release + 1h + 1s (attacker fully out)", false);
        _warpTo(release + 24 hours);
        _checkpoint("release + 24h (attacker fully out)", false);

        // OWNER (documented recovery action): clearDegradedMode needs no recap once L == 0.
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "OWNER cleared");
        bytes[] memory data = _markUpdateData();
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
        uint256 carolClaimable = juniorVault.claimableDepositRequest(bk.carolReq, CAROL);
        uint256 bobClaimable = juniorVault.claimableRedeemRequest(bk.bobReq, BOB);
        console2.log(string.concat("after OWNER clear + settle: carol claimable assets ", _usd(int256(carolClaimable))));
        console2.log(string.concat("after OWNER clear + settle: bob claimable shares  ", vm.toString(bobClaimable)));
        assertEq(carolClaimable, R3_CAROL_DEPOSIT, "carol's deposit activates only after the owner tx");
        assertEq(bobClaimable, bk.bobShares, "bob's redemption is funded only after the owner tx");
```

