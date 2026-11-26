# \#98056 [SC-Critical] Refundable VPI open charge booked as realized LP revenue in CfdEngineSettlementSidecar lets a dual-role LP+trader steal remaining LPs' principal by redeeming around the charge

**Submitted on Nov 4th 2026 at 17:06:54 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98056
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader opens a skew-increasing position, the virtual-price-impact (VPI) charge it pays is refundable: the matching close pays part of it back out of pool cash. The settlement path nonetheless books that charge into the liquidity pool's accounted assets as realized revenue, so junior net asset value (NAV) rises the moment the charge is paid, as if it were permanent earnings. An actor who is both a junior LP and the trader opens the skew-increasing position, redeems its LP shares at the lifted share price inside the same hourly epoch, then closes so that the later refund is drawn from the cash the remaining LPs still hold. In the headline test the attacker commits 260,000 USDG and finishes with 260,119.28 USDG, a net gain of 119.28 USDG with no market move, while a co-LP who deposited 50,000 USDG redeems only 49,767.07 USDG. The result is a direct, repeatable transfer of LP principal to the attacker, and the caller needs no privileged role.

## Vulnerability Details

On a skew-increasing open, `CfdEngineSettlementSidecar.sol:454-463` records the trade cost as a claimant inflow of kind `Revenue` (`recordClaimantInflow(poolCashInflowAssets, ClaimantInflowKind.Revenue, ClaimantInflowCashMode.CashArrived)`). That trade cost includes the refundable VPI charge. `HousePool.sol:842-888` routes the inflow into `juniorPrincipal` at the settlement pass (`HousePool.sol:624-651`), so junior NAV rises as if the charge were permanent earnings. It is not permanent: the matching close pays the VPI rebate back later out of pool cash at `CfdEnginePlanLib.sol:973-977`.

The attack follows from that one accounting step. A junior LP who is also the trader opens the skew-increasing position, which lifts junior NAV; redeems its junior shares at the inflated price inside the same hourly epoch; then closes. Because the redemption has already paid the actor out at the lifted price, the close rebate is funded from the cash the remaining LPs still hold. NAV was overstated by exactly the refundable amount for the window between the charge and its refund, and the actor exited inside that window.

The project may point to `SECURITY.md:872`, which states that neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV. That sentence governs anticipated VPI held in the reserve. Here the charge is realized and still refundable, and it is booked as revenue with no offsetting liability, so the documented exclusion does not cover it. The share price therefore reflects value the pool cannot keep.

**Recommended fix.** Do not book the refundable portion of a VPI open charge into LP NAV as realized revenue. Hold it in the VPI rebate reserve as a liability until the matching close resolves, so the share price reflects only value the pool can keep.

## Impact Details

Direct theft of LP principal (Immunefi impact: direct theft of any user funds, other than unclaimed yield). In the headline test `test_attackerNetsPositiveWithDepth` the attacker commits 260,000 USDG (200,000 as a junior LP and 60,000 as trader margin, the two components visible in the test body and summing to the logged commitment) and finishes with 260,119.28 USDG, a net gain of 119.28 USDG with no price move. The co-LP who deposited 50,000 USDG redeems 49,767.07 USDG, a loss of 232.93 USDG (0.47%). A separate clean-twin test, `test_vpiRoundTripDilutesRemainingJuniorLp`, isolates the dilution: a remaining junior LP that would redeem 99,999.99 USDG in the untouched run redeems 99,940.85 USDG after the round trip, a loss of 59.14 USDG. The extraction repeats each hourly epoch; the exit-side variant `test_H20b_threeCyclesAgainstTwin` drains about 2,252.53 USDG from a roughly 1,000,000 USDG incumbent over three cycles (owed 1,000,999.94 USDG in the twin, held 998,747.41 USDG after). USDG is the dollar-denominated settlement token with 6 decimals, so one USDG is about one dollar. Funds at risk are junior LP principal held in the pool, bounded by the available LP principal; the figures are the minimum demonstrated at the release parameters.

### Before / after state delta (headline test, USDG, 6 decimals)

| Account or state | Before | After | Delta |
|---|---|---|---|
| attacker (junior LP 200,000 plus trader margin 60,000) | 260,000.00 committed | 260,119.28 | +119.28 |
| co-LP who deposited 50,000 | 50,000.00 | 49,767.07 | -232.93 |
| remaining junior LP, clean twin (`test_vpiRoundTripDilutesRemainingJuniorLp`) | 99,999.99 | 99,940.85 | -59.14 |

Honest bounds. The net-positive case needs senior-provided free-cash depth and the attacker holding only a minority of the junior tranche. A dominant-share attacker that owns most of the junior tranche bears its own dilution and nets a loss: `test_attackerNetsVictimValue` commits 2,100,000 USDG and returns 1,420,901.67 USDG. Test settings: the cited tests use the release VPI factor and skew cap (`vpiFactor` 0.01e18, max skew ratio 0.4e18) with maintenance and initial margin of 100 and 150 bps and base carry 0, not the release 10 and 20 bps and 500 bps carry. The headline test opens and closes the position at one block timestamp, so no carry accrues over its hold. Re-running the headline test with the full release configuration (`_enableVpi` set to margins 10 and 20, carry 500) gave the same 260,119.28 USDG result and 49,767.07 USDG co-LP redemption. A longer hold pays carry, which this PoC does not measure, and carries market risk on the open position. The attack uses no oracle manipulation, no flash loan, and no user interaction. A guardian pause of LP settlement would contain it, but the extraction completes inside a single hourly epoch before any such containment.

## References

- Vulnerable code at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:
  - `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` (refundable VPI open charge recorded as `Revenue`).
  - `packages/perps/src/HousePool.sol:842-888` and `packages/perps/src/HousePool.sol:624-651` (the inflow routed into `juniorPrincipal` at the settlement pass).
  - `packages/perps/src/libraries/CfdEnginePlanLib.sol:973-977` (the matching close rebate paid from pool cash).
- Design note the project may cite: `SECURITY.md:872` (anticipated VPI and its reserve excluded from NAV), which does not cover a realized, still-refundable charge.
- Known-issue check: no item in the project's prior audits (Cecuro, pre-audit, collaborative, Cantina) or scanner results describes booking a refundable VPI open charge as realized LP revenue. The nearest scanner item (temporary pool liquidity reducing a skew-increasing close charge) is depth manipulation of a close charge, a different mechanism; items on a negative `vpiAccrued` term in the IMR projection and on carry and redemption rounding are unrelated. A duplicate is valid until a fix is public.


## Link to Proof of Concept

https://gist.github.com/tdev00/05f7b54abe5e1d273530fa8e338c5819

## Proof of Concept

## Proof of Concept

**Attack, step by step (as the test executes it, at the release VPI factor with base carry off).**

1. The test sets the risk parameters to the release VPI factor (`vpiFactor` 0.01e18, max skew ratio 0.4e18) with base carry switched off and margins of 100 and 150 bps (`_enableVpi`). The position is opened and closed at one block timestamp, so no time passes while it is held.
2. A senior LP deposits 500,000 USDG into the senior tranche; the attacker deposits 200,000 USDG into the junior tranche; a second junior LP deposits 50,000 USDG. Each deposit settles at its epoch boundary. The senior capital provides free-cash depth so the attacker's later redemption can fund in full.
3. After the redemption cooldown passes, the attacker funds a trader account with 60,000 USDG of margin (the harness mints mock USDG and deposits it into the clearinghouse). Its total commitment is 260,000 USDG.
4. The attacker queues its full junior redemption (`requestRedeem` of all its junior shares) and the clock advances to that redemption's epoch.
5. The attacker opens a skew-increasing long (250,000e18 notional, 50,000 USDG margin, mark price held flat at 1e8, so there is no market move). In the test the open order carries the router's authority: the harness pranks as the router and calls `engine.processOrderTyped`, as any order routed through the OrderRouter does in production. The skew-increasing open pays the refundable VPI charge, which the settlement path books into the liquidity pool as realized revenue, lifting junior NAV.
6. Settlement runs: the harness refreshes the mark price and calls `pool.settleLpEpoch`, both under the router's authority in the test, as a permissionless keeper call through the router does in production. The attacker's queued redemption now funds in full at the lifted NAV; the test asserts it funds entirely from the senior-provided free cash. The attacker claims its full redemption and exits the junior tranche.
7. The attacker closes the long. The matching close pays the VPI rebate back out of pool cash; because the attacker has already exited the junior tranche, that rebate is drawn from the cash the remaining LPs still hold.
8. The attacker withdraws its whole clearinghouse balance. It committed 260,000 USDG and holds 260,119.28 USDG, a net gain of 119.28 USDG with no market move. The remaining junior LP redeems 49,767.07 USDG against its 50,000 USDG deposit. The test asserts that the remaining LP redeems less than its deposit and that the attacker ends above its commitment.

Only Pyth (the harness MockPyth) and the settlement token USDG (a mock ERC20) are mocked, which the competition rules permit. The `vm.prank` calls stand in for the router and keeper authority that the OrderRouter carries in production; `vm.warp` advances the hourly epoch clock; no price move is used.

**Runnable test(s).**

The full test file (TriageVault.t.sol) and the captured forge log are in the secret Gist linked below. The test runs through the `BasePerpTest` harness at the frozen commit.

1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included) and install solc 0.8.35 (the repository's pinned compiler).
2. Create `poc/` at the repository root and copy TriageVault.t.sol into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`; the file imports the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run, from the repository root, with the test directory pointed at `poc/`:
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/TriageVault.t.sol --match-test test_attackerNetsPositiveWithDepth -vv
   ```
   The supporting tests run the same way, changing only the test name:
   ```
   forge test --match-path poc/TriageVault.t.sol --match-test test_vpiRoundTripDilutesRemainingJuniorLp -vv
   forge test --match-path poc/TriageVault.t.sol --match-test test_attackerNetsVictimValue -vv
   forge test --match-path poc/TriageVault.t.sol --match-test test_H20b_threeCyclesAgainstTwin -vv
   ```

Headline test (`contract O13VpiTest`), verbatim:

```solidity
    function test_attackerNetsPositiveWithDepth() public {
        _enableVpi();
        _activateSenior(CAROL, 500_000e6); // depth / free cash, takes no junior revenue at HWM
        uint256 attackerShares = _activateJunior(ATTACKER, 200_000e6);
        _activateJunior(BOB, 50_000e6);
        vm.warp(block.timestamp + 1 hours + 1);

        uint256 traderMargin = 60_000e6;
        _fundTrader(ATTACKER, traderMargin);
        uint256 committed = 200_000e6 + traderMargin;

        vm.prank(ATTACKER);
        uint256 rid = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        _warpToEpoch(rid);

        uint256 depth = pool.totalAssets();
        _open(ATTACKER, CfdTypes.Side.LONG, 250_000e18, 50_000e6, 100_000_000, depth);
        _routerBoundSettle();
        uint256 s = juniorVault.claimableRedeemRequest(rid, ATTACKER);
        assertEq(s, attackerShares, "attacker redemption must fully fund from senior-provided free cash");
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(rid, s, ATTACKER, ATTACKER);
        _close(ATTACKER, CfdTypes.Side.LONG, 250_000e18, 100_000_000, pool.totalAssets());

        uint256 chBal = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, chBal);
        uint256 attackerFinal = settlementToken.balanceOf(ATTACKER);

        emit log_named_uint("attacker committed", committed);
        emit log_named_uint("attacker final USDG", attackerFinal);
        uint256 bobBack = _juniorRedeemAll(BOB);
        emit log_named_uint("BOB redemption (50k deposit)", bobBack);
        assertLt(bobBack, 50_000e6, "remaining junior LP BOB diluted");
        assertGt(attackerFinal, committed, "attacker nets positive: direct theft of LP principal");
    }
```

Captured output (TriageVault.log in the Gist):

```
[PASS] test_attackerNetsPositiveWithDepth() (gas: 8826573)
Logs:
  attacker committed: 260000000000
  attacker final USDG: 260119283342
  BOB redemption (50k deposit): 49767071576

[PASS] test_vpiRoundTripDilutesRemainingJuniorLp() (gas: 12125274)
Logs:
  BOB clean redemption: 99999989293
  BOB redemption after attack: 99940851967

[PASS] test_attackerNetsVictimValue() (gas: 7439250)
Logs:
  attacker committed: 2100000000000
  attacker final USDG: 1420901671612
  BOB redemption (20k deposit): 19985383707

[PASS] test_H20b_threeCyclesAgainstTwin() (gas: 22213965)
Logs:
  incumbent owed after 3 cycles (twin): 1000999941541
  incumbent held after 3 cycles: 998747414810
```

The headline test proves the theft on its own: the attacker ends above its 260,000 USDG commitment and the remaining junior LP redeems less than its 50,000 USDG deposit. The clean-twin, dominant-share, and three-cycle exit-side tests are the dilution isolation, the honest loss case, and the repeatability measurement for the same refundable-charge mechanism.

