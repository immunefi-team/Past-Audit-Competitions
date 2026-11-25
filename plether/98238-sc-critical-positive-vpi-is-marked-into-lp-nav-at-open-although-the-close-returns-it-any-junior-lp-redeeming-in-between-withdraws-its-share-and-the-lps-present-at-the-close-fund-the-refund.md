# \#98238 [SC-Critical] Positive VPI is marked into LP NAV at open although the close returns it: any Junior LP redeeming in between withdraws its share, and the LPs present at the close fund the refund

**Submitted on Nov 6th 2026 at 07:38:23 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98238
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
Positive VPI paid on a skew-increasing open is booked as LP revenue at once, so it raises the Junior tranche's NAV and redemption price immediately. But the protocol is built to give it back:
- VPI is a difference of a state potential, so a reversal at unchanged depth "telescopes" (WHITEPAPER).
- A close that reverses the skew pays a rebate up to the position's accrued VPI (the lifetime clamp).

**The refund obligation is neither reserved nor excluded from LP pricing.** Between the open and the close, the collected VPI is withdrawable LP equity:
- any Junior LP that redeems at an epoch in between receives its pro-rata share, s·VPI;
- the LPs in Junior at the close, whether they stayed or entered later, fund the full refund.

For negative VPI the protocol does exactly the opposite: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (WHITEPAPER).

**PoC results** (release parameters, public entry points only):
- **An unrelated LP exits:** a 3.9M LONG pays 7,604.998 USDG of VPI. An unrelated LP holding 74% of Junior redeems at the next epoch and receives **+5,627.70 USDG** more than without the trade, or +4,463.84 more than if it had stayed. The Junior LP that stayed loses **−4,059.20 USDG**. The trader's VPI round trip is exactly 0.
- **One actor does both:** an actor that is both the trader and the 74% Junior LP nets **+2,507.09** and **+2,512.98 USDG** in two consecutive 3-hour cycles, while the staying LP loses −4,067.29 and −4,073.18. The trader's price exposure is about 2.5 minutes per cycle.

## Vulnerability Details
**Root cause: collected VPI is booked as revenue, its refund is paid later, and nothing in between reserves it**
1. **Collected at open as LP revenue.** `CfdEngineSettlementSidecar.executeOpen` (`:446-462`) passes the trade cost, including the positive VPI and minus the treasury fee, to `HousePool.recordClaimantInflow(..., ClaimantInflowKind.Revenue, CashArrived)`. WHITEPAPER:411 says: "Positive VPI increases physical pool assets". From that moment it is part of the pool assets that price LP deposits and redemptions.
2. **Refunded at close.** `CloseAccountingLib` (`:112-118`) clamps the close VPI so that `proportionalAccrualAssets + vpiDeltaAssets >= 0`. So a skew-reducing close gets a rebate up to exactly the VPI this position paid. The rebate is paid from pool cash: `CfdEngineSettlementSidecar` `:621-624`, `pool.payOut(...)`.
3. **Nothing reserves it in between.** For the opposite sign, the protocol keeps `vpiRebateReserveAssets = max(-vpiAccrued, 0)` as a dedicated sub-balance outside the price curve (ACCOUNTING_SPEC:798, WHITEPAPER:873-888). There is no counterpart for `max(vpiAccrued, 0)`, the amount that the same clamp makes refundable. Redemptions settled by `router.settleLpEpoch` therefore price Junior shares on assets that include VPI the pool will hand back.

**Spec vs code**
- WHITEPAPER:887-888 states the rule for the negative sign: "LPs cannot withdraw it as marked equity before the matching position obligation settles". A positive-VPI refund is exactly such a "matching position obligation", but LPs can withdraw the marked VPI before it settles.
- WHITEPAPER:852 ("a sequence at unchanged D telescopes") and WHITEPAPER:903-904 ("The VPI charge is $20,000. Reversing the move at unchanged depth produces a $20,000 rebate before the lifetime clamp") show that a positive VPI charge is, by design, returned when the skew reverses. It is economically a refundable deposit, not final revenue.
- **The protocol already treats this attack class as a threat, and its defence is incomplete.** The lifetime clamp exists to stop an LP/trader from extracting an LP-funded rebate. `CloseAccountingLib:112-115`: "Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate."
  - The clamp closes the depth variant. It does not close the NAV variant: the rebate it still allows (up to the accrued VPI) is exactly the amount that was distributable as Junior NAV in between.
  - This is not a design choice the team accepted. It is a gap in a defence the team already judged necessary.
- **What we do not dispute:** SECURITY:872 ("neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV") and ACCOUNTING_SPEC:1064 ("anticipated carry, VPI, fees, spreads ... stay outside terminal `K`"). These rules concern ANTICIPATED VPI. The problem here is COLLECTED VPI that sits inside NAV while its return is anticipated.

**Why this is LP-side instability, not a trick of one actor**
- **Any LP benefits.** Any Junior LP that redeems while a VPI-paying position is open receives s·VPI, where s is its Junior share. The PoC's main case uses an unrelated LP and a trader who only pays the two execution fees.
- **The loss follows whoever holds Junior at the close.** With a refill deposit, the LPs present at the close (the stayer plus the new entrant) lose −4,059.21 in total. That is the same as the stayer alone without the refill (−4,059.20). Entrants who buy in while the refundable VPI sits in NAV are overcharged too.
- **This is not Junior's first-loss role.** First loss covers traders' price PnL. It does not cover returning a deposit that was booked to the same Junior as revenue. Entrants pay a NAV that includes VPI the pool is about to hand back.
- **One actor can sit on both sides** and harvest it repeatedly. That is the profit case below.

**Not a known issue or duplicate**
- **AI-scan "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (info), and the depth sandwich the clamp comment names:** that issue is a depth effect, where withdrawing liquidity before the close inflates the rebate. Here the gain survives at constant depth.
  - The PoC refills Junior with an unrelated deposit in the same epoch, so pool depth at the close is within 33.27 USDG (0.0003%) of the depth at the open.
  - The rebate is 7,604.97 USDG, against 7,605.00 in the no-refill branch.
  - The exiting LP's gain is identical in both branches (+5,627.70).
- **SECURITY:1443** ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"): here the trader's own VPI round trip is exactly 0. The value comes from the LP leg.
- **Searched in every audit text and SECURITY:** VPI together with LP exit, redemption, NAV, epoch and tranche, and "price impact" with the same terms. No item covers positive VPI being withdrawable as LP equity before its refund. No public commit touches it.

## Impact Details
**Impact: High, "Theft of unclaimed yield".**
- **What the attacker takes:** pool VPI revenue, i.e. yield marked into Junior NAV, before the obligation that should net against it (the refund) settles.
- **Who loses:** the LPs present at the close, pro rata.
- **The victim's loss exceeds its own yield share.** In the PoC the staying LP loses 4,059.20 USDG, while its 26% share of the VPI is about 1,977. It loses that share AND funds the rest of the refund from principal. We map by the attacker's take (yield) and leave it to the judges whether the principal component warrants a higher category.

**Why this mapping:** We selected High "Theft of unclaimed yield" because it is the only listed impact in which an attacker profits and users lose value. Critical "Direct theft" overstates the mechanism (an LP-to-LP transfer through unmarked liability, not a direct withdrawal of user funds). Medium "Griefing" does not apply because the attacker's net is positive. We leave the final mapping to triage.

**Scale and constraints** (all measured; release parameters: vpiFactor 0.01, fee 4 bps, maxSkewRatio 0.4)
- **Third-party LP:** no threshold. Every exit while a VPI-paying position is open receives s·VPI.
- **Same actor (trader + LP):** the profit per cycle is s·VPI − 2·fee·N, with VPI = k·N·(2·S0 + N)/(2D). It is positive iff s·(2·S0 + N)/D > 4·fee/k = 0.16.
  - From zero skew, N ≤ 0.4·D gives s > 40% of Junior.
  - With same-side skew S0 already open, (2·S0 + N)/D can approach 0.8, so the requirement drops toward s > 20%.
- **Repeatability:** one cycle is 3 epochs, i.e. 3 hours. The binding limit is `TrancheVault.DEPOSIT_COOLDOWN` (1 hour, `TrancheVault.sol:38`, checked at `:617`). That means at most 8 cycles per day per LP account. Two consecutive cycles are demonstrated with no decay: +2,507.09, then +2,512.98.
- **Size:** about 2.5 bps of pool depth per cycle in the PoC (10M pool, 39% skew, 74% Junior share). With at most 8 cycles per day, that is about 20 bps of pool depth per day, roughly 20,000 USDG/day on a 10M pool, moved from the remaining LPs to one account in this geometry. The remaining LPs will eventually exit too, which is exactly the instability this report describes.
- **Price risk:** the trader leg is open for about 2.5 minutes per cycle (opened 2 minutes before the epoch, closed 30 s after), and until the epoch it is partly offset by the actor's own Junior stake.
- **Interaction with the close fee:** today the close execution fee stays in the pool (separately reported, #98232), which partly offsets the stayers. Once that is fixed, the stayers' loss is the full s·VPI.

**Suggested fix:** mirror the negative-VPI reserve.
- Keep the refundable positive VPI, the sum of `max(vpiAccrued, 0)` over open positions (bounded by what the clamp can return), outside the LP price curve used for deposits and redemptions until the position closes or is liquidated.
- Equivalently, recognise positive VPI as LP revenue only once it can no longer be rebated.

## References
- Code (frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):
  - `packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L462` (open: trade cost booked as pool Revenue)
  - `packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624` (close: action rebate paid from the pool)
  - `packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118` (lifetime clamp)
  - `packages/perps/src/TrancheVault.sol#L38`, `#L617` (deposit cooldown)
  - `packages/perps/src/libraries/CfdEnginePlanLib.sol#L473-L475` (maxSkewRatio)
- Spec:
  - `packages/perps/WHITEPAPER.md#L411`, `#L811`, `#L852`, `#L873-L888`, `#L903-L904`
  - `packages/perps/ACCOUNTING_SPEC.md#L798`, `#L1064`
  - `packages/perps/SECURITY.md#L872`, `#L1443`


## Proof of Concept

## Proof of Concept
This is one runnable Foundry test file in the repository's own perps test suite, at the frozen commit, with no change to any in-scope contract.
- **Fixture:** the repo's `BasePerpTest`. The only mocks are MockPyth and the USDG mock, both out of scope.
- **Parameters:** release values from DEPLOYMENT.md (vpiFactor 0.01, maxSkewRatio 0.4, MM 10 / IM 20 / FAD 300 bps, carry 500 bps, execution fee 4 bps).
- **Entry points:** public only.
  - LPs use `TrancheVault.requestDeposit/claimDeposit/requestRedeem/claimRedeem`.
  - An untrusted keeper calls `OrderRouter.settleLpEpoch`, `updateMarkPrice` and `executeOrder`.
  - Traders call `router.commitOrder`.
- **No protocol pranks and no `vm.store`.** The price is constant, so price PnL is zero everywhere and every delta comes from VPI, fees and carry.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test code below as packages/perps/test/PocS3_RefundableVpi.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/PocS3_RefundableVpi.t.sol -vv   # Foundry 1.5.1
```

No env or RPC is needed. Expected result: `3 passed`. `FOUNDRY_SPARSE_MODE=true` compiles only this test and its dependencies. Without it, forge first builds the whole via-IR suite, which takes far longer and needs about 16 GB RAM.

**Book:** Junior 5M (STAYER 1.3M = 26%, X 3.7M = 74%), Senior 5M. The trader opens a 3.9M LONG, which is 39% of depth (the cap is 40%). All amounts are in USDG atoms (6 decimals).

**Test 1, `test_S3_1_AnyLpExitCapturesRefundableVpi`: an unrelated LP exits.** It runs four snapshot branches: trade or no trade, and X exits or stays.
1. At 13:00, trader T opens and pays VPI of 7,604.998479 USDG. Junior NAV goes from 5,000,000 to 5,007,560.31 at once.
2. At 13:01, X, a Junior LP unrelated to T, requests redemption of all its shares. The keeper settles the 14:00 epoch; X receives 3,705,560.07.
3. At 14:00:30, T closes. The lifetime clamp gives a rebate of exactly its accrued VPI, 7,604.998479, paid from the pool.
4. **Results:**
   - X gains **+5,627.70** vs the no-trade branch (= 74% × VPI), or +4,463.84 vs staying.
   - The stayer loses **−4,059.20**.
   - If nobody exits, the stayer instead gains +409.30.
   - T's own VPI round trip is **0**. T's net is −3,128.69, which is the two 4 bps fees plus carry.

**Test 2, `test_S3_2_ConstantDepthSameGain`: the gain is not a depth effect.**
- An unrelated depositor, REFILL, puts 3,697,988.35 into Junior in the same epoch pass, so the pool depth at the close (10,000,035.27) equals the depth at the open (10,000,002.00) within 0.0003%.
- The close rebate is 7,604.97, against 7,605.00 without the refill.
- X's gain is identical in both branches: **+5,627.70**.
- The LPs present at the close lose −4,059.21 in total, against −4,059.20 without the refill.

**Test 3, `test_S3_3_SameActorTwoCycles`: one actor sits on both sides, twice in a row.**
- X is both the 74% Junior LP and the trader. Each cycle runs as follows:
  1. redeem request at h0+10s;
  2. open at h0+58m;
  3. the keeper settles the epoch at h0+1h+5s, and X claims;
  4. close at h0+1h+30s;
  5. re-deposit, activated at h0+2h+5s;
  6. at h0+2h+30m, `requestRedeem` reverts with `TrancheVault__DepositCooldown` (asserted);
  7. the next cycle starts at h0+3h.
- **Profit vs the same LP moves without trading:**
  - cycle 1: **+2,507.09**, with the stayer at −4,067.29;
  - cycle 2: **+2,512.98**, with the stayer at −4,073.18.

Output of the run:
```
Ran 3 tests for test/PocS3_RefundableVpi.t.sol:PocS3_RefundableVpiTest
[PASS] test_S3_1_AnyLpExitCapturesRefundableVpi() (gas: 11919416)
Logs:
  Branch [trade, X exits], step by step:
  1. pool depth before open 10000002000000
  1. T opens 3.9M LONG; VPI paid 7604998479
  1. Junior NAV after the open (5M + VPI) 5007560309414
  2. X redeems all its Junior shares at the 14:00 epoch; assets 3705560070905
  3. pool depth before close 6302046927574
  3. T closes; close VPI rebate (clamped to accrued) 7604998479
  X gain vs no-trade branch 5627697751
  X gain vs staying (trade, X stays) 4463837093
  expected s*VPI (s = 74%) 5627698874
  Stayer delta vs no-trade (X exits) -4059199636
  Stayer delta if nobody exits 409303264
  T net (2 x 4 bps fees = 3,120 USDG, plus carry) -3128694998
  T VPI round trip (open VPI - close rebate) 0

[PASS] test_S3_2_ConstantDepthSameGain() (gas: 14776829)
Logs:
  refill deposit (same epoch as X's redemption) 3697988346145
  refill:    depth at open 10000002000000
  refill:    depth at close 10000035273719
  no refill: depth at open 10000002000000
  no refill: depth at close 6302046927574
  close rebate, refill 7604966745
  close rebate, no refill 7604998479
  X gain, refill 5627697751
  X gain, no refill 5627697751
  LPs at close total delta, refill (STAYER + REFILL) -4059208266
  LPs at close total delta, no refill (STAYER) -4059199636
  refill: depth close - open 33273719

[PASS] test_S3_3_SameActorTwoCycles() (gas: 14143053)
Logs:
  cycle 1 h0 (UTC seconds) 1710162000
    58m: open, VPI paid 7604998479
    1h+5s: Junior redemption settled, assets 3705560070905
    1h+30s: closed; trading account balance 1996879413509
    2h+5s: re-deposit activated, Junior shares 3723268214874691
    2h+30m: requestRedeem reverts TrancheVault__DepositCooldown (ends 3h+5s)
  cycle 2 h0 (UTC seconds) 1710172800
    58m: open, VPI paid 7603811990
    1h+5s: Junior redemption settled, assets 3711126078422
    1h+30s: closed; trading account balance 1993758827027
    2h+5s: re-deposit activated, Junior shares 3740819452884180
    2h+30m: requestRedeem reverts TrancheVault__DepositCooldown (ends 3h+5s)
  cycle length: 3 epochs = 3 hours (redemptions settle at 14:00 and 17:00 UTC)
  cycle 1 profit vs same LP moves without trading 2507090898
  cycle 1 stayer delta -4067287774
  cycle 2 profit vs same LP moves without trading 2512983033
  cycle 2 stayer delta -4073179905

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 58.49ms (115.77ms CPU time)

Ran 1 test suite in 61.07ms (58.49ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### Full test code (`packages/perps/test/PocS3_RefundableVpi.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC S3: refundable positive VPI is booked as distributable Junior NAV.
//
// A trader who increases skew pays a positive VPI charge. The cash goes into HousePool and immediately raises Junior
// NAV. The same position keeps the right to get that VPI back on close: the lifetime clamp (CloseAccountingLib
// l.112-118) caps the close rebate at exactly the accrued VPI it paid. Nothing reserves that refund obligation in NAV
// (the negative-VPI side IS protected, WHITEPAPER l.887: "LPs cannot withdraw it as marked equity before the matching
// position obligation settles"). Any Junior LP that redeems at the next epoch while such a position is open walks away
// with its share of the VPI; the LPs present at the close pay the rebate. Test 1: an unrelated LP gains s*VPI, the
// stayer loses, the trader only pays fees. Test 2: the gain is unchanged when pool depth at close equals depth at open
// (not the AI-scan "temporary depth" item). Test 3: one actor on both sides profits in two consecutive 3-hour cycles.
//
// Run (from the audit-comp-plether repo root, after copying this file to packages/perps/test/):
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/PocS3_RefundableVpi.t.sol -vv
// No env vars, no RPC. Fixture: the repo's BasePerpTest (MockPyth and MockSettlementToken = out-of-scope mocks).
// Release parameters (DEPLOYMENT.md). Public entry points only: TrancheVault request/claim, HousePool.settleLpEpoch
// (no open positions), OrderRouter.settleLpEpoch / updateMarkPrice / executeOrder (keeper), router.commitOrder (the
// fixture's scalar adapter over the production V3 commit). No protocol pranks, no vm.store. Amounts in USDG atoms.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract PocS3_RefundableVpiTest is BasePerpTest {

    address internal constant KEEPER = address(0x0E0E1); // any untrusted caller
    address internal constant T = address(0x7101); // trader (tests 1-2)
    address internal constant X = address(0xB0B1); // Junior LP, 3.7M = 74% of Junior (exits)
    address internal constant STAYER = address(0xA0A1); // Junior LP, 1.3M = 26% of Junior (stays)
    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant REFILL = address(0xD1D1); // unrelated Junior depositor (test 2)

    uint256 internal constant MON = 1_710_151_200; // Mon 2024-03-11 10:00 UTC, live market
    uint256 internal constant H13 = MON + 3 hours;
    uint256 internal constant P0 = 1e8; // constant price: price PnL is zero everywhere
    uint256 internal constant FUND = 2_000_000e6;
    uint256 internal constant SIZE = 39_000e20; // 3.9M tokens = 3.9M USDG notional = 39% of the 10M pool (cap 40%)
    uint256 internal constant MARGIN = 78_000e6; // 2% (IM 0.2%)

    struct Out {
        uint256 depthOpen;
        int256 vpiOpen;
        uint256 juniorNavAfterOpen;
        uint256 xAssets;
        uint256 depthClose;
        int256 rebate;
        int256 tNet;
        uint256 xWorth;
        uint256 stayWorth;
        uint256 refillWorth;
        uint256 refillAmt;
    }

    // ---------------- release configuration ----------------

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DEPLOYMENT.md release values (vpi 0.01, skew cap 0.4, MM 10 / IM 20 / FAD 300 bps, carry 500 bps).
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    // LPs are funded through the public async path below; seeds are DEPLOYMENT.md's 1e6 each (owner bootstrap).
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    /// @dev LP book through public async requests: Junior 1.3M (STAYER) + 3.7M (X), Senior 5M.
    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000); // release Senior limits through the owner's 72 h timelock
        _warpTo(MON);
        _pushMark();
        uint256 a = _requestDeposit(STAYER, juniorVault, 1_300_000e6, true);
        uint256 b = _requestDeposit(X, juniorVault, 3_700_000e6, true);
        _warpTo(pool.lpEpochStart(a));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0); // no open positions: permissionless
        _claimDeposit(STAYER, juniorVault, a);
        _claimDeposit(X, juniorVault, b);
        _warpTo(_now() + 60);
        uint256 s = _requestDeposit(SENIOR_LP, seniorVault, 5_000_000e6, true);
        _warpTo(pool.lpEpochStart(s));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(SENIOR_LP, seniorVault, s);
        _fundTrader(T, FUND);
    }

    // ---------------- public-path helpers ----------------

    function _now() internal view returns (uint256) { return vm.getBlockTimestamp(); }

    function _warpTo(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _liveData() internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(P0)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
            );
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _pushMark() internal {
        vm.prank(KEEPER);
        router.updateMarkPrice(_liveData());
    }

    function _settle() internal {
        vm.prank(KEEPER);
        router.settleLpEpoch(_liveData());
    }

    function _requestDeposit(address lp, TrancheVault v, uint256 amt, bool mint) internal returns (uint256 id) {
        if (mint) settlementToken.mint(lp, amt); // USDG mock
        vm.startPrank(lp);
        settlementToken.approve(address(v), amt);
        id = v.requestDeposit(amt, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(address lp, TrancheVault v, uint256 id) internal {
        uint256 a = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        v.claimDeposit(id, a, lp, lp);
    }

    function _redeemAll(address lp) internal returns (uint256 id, uint256 shares) {
        shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        id = juniorVault.requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(address lp, uint256 id, uint256 shares) internal returns (uint256) {
        assertEq(juniorVault.claimableRedeemRequest(id, lp), shares, "redemption fully funded");
        vm.prank(lp);
        return juniorVault.claimRedeem(id, shares, lp, lp);
    }

    /// @dev Router commit by the trader, execution by an untrusted keeper (LONG; close when isClose).
    function _trade(address who, bool isClose) internal {
        _warpTo(_now() + 1);
        _pushMark();
        vm.prank(who);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, SIZE, isClose ? 0 : MARGIN, isClose ? 0 : P0, isClose);
        bytes[] memory upd = _mockPythUpdateData(P0);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _vpi(address a) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _worth(address a) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + juniorVault.convertToAssets(juniorVault.balanceOf(a));
    }

    function _d(uint256 a, uint256 b) internal pure returns (int256) {
        return int256(a) - int256(b);
    }

    // ---------------- one branch: open -> (X exits at the epoch) -> close ----------------

    /// @param refill 0 = none; max = "depth-neutral" amount (X's redemption minus the VPI); else that amount.
    function _branch(bool trade, bool exits, uint256 refill, bool log) internal returns (Out memory o) {
        _warpTo(H13);
        o.depthOpen = pool.totalAssets();
        if (trade) {
            _trade(T, false);
            o.vpiOpen = _vpi(T);
        }
        o.juniorNavAfterOpen = juniorVault.totalAssets();
        if (log) console.log("1. pool depth before open", o.depthOpen);
        if (log) console.log("1. T opens 3.9M LONG; VPI paid", uint256(o.vpiOpen));
        if (log) console.log("1. Junior NAV after the open (5M + VPI)", o.juniorNavAfterOpen);
        _warpTo(H13 + 60);
        uint256 id;
        uint256 shares;
        if (exits) {
            uint256 est = juniorVault.estimateRedeemAssets(juniorVault.balanceOf(X));
            (id, shares) = _redeemAll(X);
            if (refill != 0) {
                o.refillAmt = refill == type(uint256).max ? est - uint256(o.vpiOpen) : refill;
                _pushMark();
                _requestDeposit(REFILL, juniorVault, o.refillAmt, true);
            }
        }
        _warpTo(H13 + 1 hours + 5);
        if (exits) {
            _settle(); // Router: post-boundary mark + LP settlement in one call
            o.xAssets = _claimRedeem(X, id, shares);
            if (o.refillAmt != 0) _claimDeposit(REFILL, juniorVault, id);
        }
        if (log) console.log("2. X redeems all its Junior shares at the 14:00 epoch; assets", o.xAssets);
        _warpTo(H13 + 1 hours + 30);
        o.depthClose = pool.totalAssets();
        if (trade) {
            o.rebate = -engineLens.previewClose(T, SIZE, P0).vpiDeltaAssets;
            _trade(T, true);
        }
        if (log) console.log("3. pool depth before close", o.depthClose);
        if (log) console.log("3. T closes; close VPI rebate (clamped to accrued)", uint256(o.rebate));
        _warpTo(H13 + 1 hours + 120);
        _pushMark(); // fresh mark before measuring NAV views
        o.tNet = int256(clearinghouse.balanceAssets(T)) - int256(FUND);
        o.xWorth = _worth(X);
        o.stayWorth = _worth(STAYER);
        o.refillWorth = _worth(REFILL);
    }

    // ---------------- tests ----------------

    function test_S3_1_AnyLpExitCapturesRefundableVpi() public {
        uint256 snap = vm.snapshotState();
        console.log("Branch [trade, X exits], step by step:");
        Out memory te = _branch(true, true, 0, true);
        vm.revertToState(snap);
        Out memory ts = _branch(true, false, 0, false);
        vm.revertToState(snap);
        Out memory ne = _branch(false, true, 0, false);
        vm.revertToState(snap);
        Out memory ns = _branch(false, false, 0, false);

        int256 gainNoTrade = _d(te.xWorth, ne.xWorth);
        console.log("X gain vs no-trade branch", gainNoTrade);
        console.log("X gain vs staying (trade, X stays)", _d(te.xWorth, ts.xWorth));
        console.log("expected s*VPI (s = 74%)", (te.vpiOpen * 74) / 100);
        console.log("Stayer delta vs no-trade (X exits)", _d(te.stayWorth, ne.stayWorth));
        console.log("Stayer delta if nobody exits", _d(ts.stayWorth, ns.stayWorth));
        console.log("T net (2 x 4 bps fees = 3,120 USDG, plus carry)", te.tNet);
        console.log("T VPI round trip (open VPI - close rebate)", te.vpiOpen - te.rebate);

        assertGt(gainNoTrade, 0, "exiting LP gains");
        assertGt(_d(te.xWorth, ts.xWorth), 0, "exiting beats staying");
        assertLt(_d(te.stayWorth, ne.stayWorth), 0, "stayer pays");
        assertEq(te.vpiOpen - te.rebate, 0, "trader VPI round trip is zero");
    }

    function test_S3_2_ConstantDepthSameGain() public {
        uint256 snap = vm.snapshotState();
        Out memory wr = _branch(true, true, type(uint256).max, false); // REFILL deposits in the same epoch pass
        vm.revertToState(snap);
        Out memory wn = _branch(true, true, 0, false);
        vm.revertToState(snap);
        Out memory nr = _branch(false, true, wr.refillAmt, false);
        vm.revertToState(snap);
        Out memory nn = _branch(false, true, 0, false);

        console.log("refill deposit (same epoch as X's redemption)", wr.refillAmt);
        console.log("refill:    depth at open", wr.depthOpen);
        console.log("refill:    depth at close", wr.depthClose);
        console.log("no refill: depth at open", wn.depthOpen);
        console.log("no refill: depth at close", wn.depthClose);
        console.log("close rebate, refill", uint256(wr.rebate));
        console.log("close rebate, no refill", uint256(wn.rebate));
        console.log("X gain, refill", _d(wr.xWorth, nr.xWorth));
        console.log("X gain, no refill", _d(wn.xWorth, nn.xWorth));
        int256 lpsAtClose = _d(wr.stayWorth, nr.stayWorth) + _d(wr.refillWorth, nr.refillWorth);
        console.log("LPs at close total delta, refill (STAYER + REFILL)", lpsAtClose);
        console.log("LPs at close total delta, no refill (STAYER)", _d(wn.stayWorth, nn.stayWorth));

        int256 depthDiff = _d(wr.depthClose, wr.depthOpen);
        console.log("refill: depth close - open", depthDiff);
        assertLe(depthDiff < 0 ? -depthDiff : depthDiff, int256(wr.depthOpen / 100_000), "depth equal (<0.001%)");
        assertGt(_d(wr.xWorth, nr.xWorth), 0, "gain at constant depth");
        assertEq(_d(wr.xWorth, nr.xWorth), _d(wn.xWorth, nn.xWorth), "same gain with or without the depth change");
    }

    /// @dev h0+10s redeem request -> h0+58m open -> h0+1h settle + close -> re-deposit -> h0+2h activation ->
    ///      1 h cooldown from that settlement block -> next cycle at h0+3h.
    function _cycle(uint256 h0, bool trade, bool log) internal {
        _warpTo(h0 + 10);
        (uint256 rid, uint256 shares) = _redeemAll(X);
        if (trade) {
            _warpTo(h0 + 58 minutes);
            _trade(X, false);
            if (log) console.log("  58m: open, VPI paid", uint256(_vpi(X)));
        }
        _warpTo(h0 + 1 hours + 5);
        _settle();
        uint256 assets = _claimRedeem(X, rid, shares);
        if (log) console.log("  1h+5s: Junior redemption settled, assets", assets);
        if (trade) {
            _warpTo(h0 + 1 hours + 30);
            _trade(X, true);
            if (log) console.log("  1h+30s: closed; trading account balance", clearinghouse.balanceAssets(X));
        }
        _warpTo(h0 + 1 hours + 60);
        uint256 did = _requestDeposit(X, juniorVault, settlementToken.balanceOf(X), false);
        _warpTo(h0 + 2 hours + 5);
        _settle();
        _claimDeposit(X, juniorVault, did);
        if (log) console.log("  2h+5s: re-deposit activated, Junior shares", juniorVault.balanceOf(X));
        _warpTo(h0 + 2 hours + 30 minutes);
        vm.prank(X);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(1e9, X, X);
        if (log) console.log("  2h+30m: requestRedeem reverts TrancheVault__DepositCooldown (ends 3h+5s)");
    }

    function _value() internal view returns (uint256) {
        return _worth(X) + clearinghouse.balanceAssets(X);
    }

    function test_S3_3_SameActorTwoCycles() public {
        _fundTrader(X, FUND); // X is both the 74% Junior LP and the trader
        uint256 snap = vm.snapshotState();
        uint256[3] memory w;
        uint256[3] memory n;
        uint256[3] memory sw;
        uint256[3] memory sn;
        for (uint256 b; b < 2; ++b) {
            vm.revertToState(snap);
            bool trade = b == 0;
            uint256[3] memory v = trade ? w : n;
            uint256[3] memory s = trade ? sw : sn;
            (v[0], s[0]) = (_value(), _worth(STAYER));
            for (uint256 c; c < 2; ++c) {
                if (trade) console.log("cycle", c + 1, "h0 (UTC seconds)", H13 + c * 3 hours);
                _cycle(H13 + c * 3 hours, trade, trade);
                (v[c + 1], s[c + 1]) = (_value(), _worth(STAYER));
            }
        }
        int256 p1 = _d(w[1], w[0]) - _d(n[1], n[0]);
        int256 p2 = _d(w[2], w[1]) - _d(n[2], n[1]);
        console.log("cycle length: 3 epochs = 3 hours (redemptions settle at 14:00 and 17:00 UTC)");
        console.log("cycle 1 profit vs same LP moves without trading", p1);
        console.log("cycle 1 stayer delta", _d(sw[1], sw[0]) - _d(sn[1], sn[0]));
        console.log("cycle 2 profit vs same LP moves without trading", p2);
        console.log("cycle 2 stayer delta", _d(sw[2], sw[1]) - _d(sn[2], sn[1]));
        assertGt(p1, 0, "cycle 1 profitable");
        assertGt(p2, 0, "cycle 2 profitable");
    }

}
```

