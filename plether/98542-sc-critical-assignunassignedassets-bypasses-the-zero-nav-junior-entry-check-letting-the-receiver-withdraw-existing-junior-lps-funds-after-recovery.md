# \#98542 [SC-Critical] assignUnassignedAssets bypasses the zero-NAV Junior entry check, letting the receiver withdraw existing Junior LPs' funds after recovery

**Submitted on Nov 8th 2026 at 15:21:59 UTC by @Mukymuk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98542
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A mark-to-market wipe can leave Junior shares outstanding while the underlying cash remains in the pool. In the demonstrated scenario, the trader later closes at its entry price, which lets the surviving Junior holders recover their principal. Every Junior entry path refuses to mint new shares while Junior is at zero NAV with shares outstanding, except one: `assignUnassignedAssets`.

The owner's documented recovery (recapitalize, then assign the leftover to a tranche) mints Junior shares at a near-zero price per share, so the receiver ends up with almost the whole Junior supply. When the price returns and LPs redeem, the receiver withdraws the USDG the existing LPs deposited.

With the release parameters, an existing LP who deposited 50,000 USDG withdraws 59,898.31 USDG without the assignment and 0.000006 USDG with it. The receiver, assigned 10,141.62 USDG, withdraws 70,040.88 USDG.

## Vulnerability Details

All references are under `packages/perps/` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.

**What the owner's power is not allowed to do.** `SECURITY.md` describes the owner's power to assign ownerless assets as one that does "not permit withdrawal of existing claimant backing" (L243–246). Here the assignment does exactly that: the cash backing the existing Junior claims stays in the pool, and the assignment hands the claim on it to the receiver, who later withdraws it.

**The rule everywhere else.** While Junior has shares outstanding and is worth zero, Plether does not let anyone create new Junior shares. Deposit admission (`src/HousePool.sol` L494–497), deferred deposit activation (L917, L932) and the share quote (`src/TrancheVault.sol` L1366) all refuse it, and the accounting spec states it (`ACCOUNTING_SPEC.md` L307–308). The tranche stays with its existing holders instead of being revived for a new one.

The spec also says why. Shares created at zero value "participate in a later recovery like ordinary transferable Junior shares" (L398–401). The only place it allows that is the maintenance fee, which is governance-finalized and rate-bounded (L376–379, L392).

**The function that skips it.** `assignUnassignedAssets(false, receiver)` (`src/HousePool.sol` L660–704) gives the leftover assets to Junior by minting shares through `quoteBootstrapDeposit` (`src/TrancheVault.sol` L1392–1396). That path has no zero-NAV check. Its conversion still accounts for the surviving supply, but it prices against zero NAV, so each share costs almost nothing and the receiver ends up with nearly all of the Junior supply.

The function's own comment says it exists to stop new LPs from "implicitly capturing previously ownerless value" (L653–659). Here it lets the receiver capture value that does have an owner: the existing LPs' claim on the pool.

**How the owner gets there.** The owner does nothing wrong:

1. Recapitalizing is the documented recovery step (`SECURITY.md` L155–162, runbook L1505–1511).
2. Whatever is left after Senior is restored goes into `unassignedAssets` (`ACCOUNTING_SPEC.md` L639–643).
3. While that bucket is not empty, new deposits and activations are blocked (L324–325), so leaving it there has a cost.
4. Assigning it to Senior is not available in this state: it reverts with `HousePool__ExceedsSeniorDepositCapacity` (`src/HousePool.sol` L677–679; asserted in the PoC).

The program scope lists "assign ownerless pool assets" as a documented owner action, and says a "legitimate, documented action by a trusted role that triggers a bug harming users or the protocol" is in scope. The bug is the share price, not the owner's choice; the receiver can be an honest treasury or recapitalizer.

**Relationship to known issues.** The AI-audit finding LOW-72 covers the same zero-NAV state. Its patch (commit 899675594) suppresses the indicative deposit quote (`estimateDepositShares`) for a wiped tranche. It does not add the check to the bootstrap mint used by `assignUnassignedAssets`, which still executes and moves funds. LOW-65 protects ownerless value before a Junior seed is priced; that protection works here, and the problem is the assignment out of it.

## Impact Details

Selected impact: **Critical, Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

In the demonstrated scenario the existing LPs' USDG never leaves the pool: the markdown to zero is unrealized, and the trader's position is later closed at its entry price. What the assignment changes is who owns that USDG. After it, the receiver holds almost all Junior shares and withdraws the existing LPs' deposit through the normal redemption path.

Results from the PoC (USDG actually received, measured with `balanceOf`):

| | Without the assignment | With the assignment |
|---|---|---|
| Senior and Junior redemption: existing Junior LP (deposited 50,000) | 59,898.31 | 0.000006 |
| Senior and Junior redemption: receiver (assigned 10,141.62) | 0 | 70,040.88 |
| Junior-only exit (80% Senior cap binds): existing Junior LP | 22,398.57 | 0.000003 |
| Junior-only exit: receiver | 0 | 32,540.19 |

In the Senior and Junior redemption case, Senior withdraws the same amount (150,002.74 USDG) in both branches, so the whole loss falls on the existing Junior LPs.

Junior's total assets in the control branch (59,900.87 USDG) break down as: 50,001 USDG contributed (the 50,000 deposit plus the 1 USDG seed), restored when the mark reverts; 9,858.38 USDG of the owner's recapitalization, which restored Senior at the wipe mark and then flowed to Junior under the waterfall when Senior's loss reversed; 42.87 USDG of net trading revenue (VPI and carry); less 1.37 USDG of Senior coupon. The existing LP withdraws 59,898.31 USDG of that total; the rest belongs to the seed shares and rounding. With the assignment, the receiver takes it, including the existing LP's deposit.

Who is affected: every Junior LP holding through a mark-to-market wipe that is followed by the documented recapitalization and assignment. The LPs take no action in the sequence; redeeming earlier would return nothing, because the tranche is marked at zero.

## Recommended fix

Apply the rule the rest of the code already uses. In `assignUnassignedAssets`, refuse a Junior assignment while Junior supply is non-zero and Junior principal is zero:

```solidity
error HousePool__JuniorZeroNavWithSupply();

// in assignUnassignedAssets, before quoteBootstrapDeposit:
if (!toSenior && juniorPrincipal == 0 && _juniorShareSupply() != 0) {
    revert HousePool__JuniorZeroNavWithSupply();
}
```

With this change the assignment reverts with that error, the bucket stays unassigned, and the existing LP withdraws exactly the control amount. The four neighbouring test files (`HousePoolBootstrap`, `HousePoolSeedLifecycle`, `HousePoolDeposits`, `TrancheRecoveryBehavior`; 15 suites, 70 tests) give the same results with and without the change.

## Proof of Concept

## Step-by-step

All values come from a local Foundry test at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. No public network was used.

Configuration: the release parameters from `DEPLOYMENT.md` and the deploy script (maintenance/initial margin 10/20 bps, execution fee 4 bps, Senior share cap 80% and 40M USDG, 1 USDG seeds, router and oracle release values). The test asserts each of them. Orders are placed through the repository's `processOrderTyped` test path, as in `BasePerpTest`; the assignment and the redemptions use the production functions.

1. A Junior LP deposits 50,000 USDG, then a Senior LP deposits 150,000 USDG. (Junior goes first because the 80% Senior cap requires it.)

2. A trader opens a 75,000-size short at 1.00 with 5,000 USDG margin.

3. The mark moves to 1.80. The trader is up 60,000 USDG unrealized. Junior absorbs the loss first and its value goes to exactly 0; Senior falls to 140,142.62 USDG. Junior shares are still outstanding. (Both asserted.)

4. The owner recapitalizes with 20,000 USDG. 9,858.38 USDG restores Senior to its high-water mark; the remaining 10,141.62 USDG goes to `unassignedAssets`.

5. Assigning the bucket to Senior reverts with `HousePool__ExceedsSeniorDepositCapacity` (asserted by selector).

6. The owner assigns it to Junior: `assignUnassignedAssets(false, receiver)`. The test requires this call to succeed. The shares are priced by `quoteBootstrapDeposit`, which has no zero-NAV check:

```
shares = quoteBootstrapDeposit(amount)   // prices against zero NAV despite surviving Junior shares
// missing: if juniorPrincipal == 0 && juniorShares > 0 -> revert
mint(receiver, shares)
```

   At a near-zero price per share, the receiver gets 507,095,846,099,731,764,944,400 Junior shares and now owns almost the whole tranche.

7. The mark returns to 1.00 and the trader closes the position at its entry price.

8. The LPs redeem through the normal path (`requestRedeem` → `settleLpEpoch` → `claimRedeem`). USDG received, measured with `balanceOf`, compared with the same state without step 6:

| | Without the assignment | With the assignment |
|---|---|---|
| Senior and Junior redemption: existing Junior LP | 59,898.31 | 0.000006 |
| Senior and Junior redemption: receiver | 0 | 70,040.88 |
| Senior and Junior redemption: Senior LP | 150,002.74 | 150,002.74 |
| Junior-only exit: existing Junior LP | 22,398.57 | 0.000003 |
| Junior-only exit: receiver | 0 | 32,540.19 |

In the Junior-only case Senior stays in the pool, so the 80% Senior cap limits how much Junior can withdraw in the epoch. In the Senior and Junior redemption case, Senior requests redemption in the same epoch and is funded first. In both cases each LP requests all of its shares; the test measures what is received in the settled epochs and does not require every queue to end empty.

Forge output on unmodified code:

```
[FAIL: existing junior holder withdrew less USDG than control: 1000006 < 59898305836] test_AssignAtZeroJuniorNav_FullExit_PreservesHoldersUsdg()
    existing junior LP USDG received: 59898305836      (control)
    existing junior LP USDG received: 6                (assignment)
    receiver USDG received: 70040876209                (assignment)
[FAIL: existing junior holder withdrew less USDG than control: 1000003 < 22398569038] test_AssignAtZeroJuniorNav_JuniorExit_PreservesHoldersUsdg()
[PASS] test_SeniorAssignmentRevertsAtZeroJuniorNav()
```

Amounts are 6-decimal token units. The left side of each assertion adds a 1 USDG rounding tolerance (`1000006` = 6 + 1,000,000).

## With the fix

With the one-line check from the Description applied, all three tests pass. The assignment reverts with `HousePool__JuniorZeroNavWithSupply()`, the bucket stays unassigned, the existing LP withdraws exactly the control amount, and the receiver receives 0.

## Runnable test

Save the test below as `packages/perps/test/perps/spec/lp/AssignUnassignedZeroNav.t.sol` (the path matters, because it imports `../../BasePerpTest.sol`) and run from the repository root:

```
forge test --offline --root packages/perps --match-path test/perps/spec/lp/AssignUnassignedZeroNav.t.sol -vvv
```

To check the fix, apply the change from the Description and run the same command with `LOW72_EXPECT_FIX=true`. In that mode the test requires the assignment to revert with the exact error.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Property under test (ACCOUNTING_SPEC: "Shares minted at zero NAV ... participate in a later recovery
// like ordinary transferable Junior shares"; assignUnassignedAssets NatSpec: "Prevents later LPs from
// implicitly capturing previously ownerless value"):
//
//   Assigning ownerless pool assets to the Junior tranche while Junior NAV is exactly zero and Junior supply
//   is non-zero must not transfer the existing Junior holders' later recovery to the assignment receiver.
//
// Configuration: the release parameters of packages/perps/DEPLOYMENT.md "Initial release parameters" and
// script/DeployPerpsArbitrumSepolia.s.sol (risk params, 80% / $40M Senior limits, 1 USDG seeds, Router and
// Oracle release values). Junior is funded before Senior, as the 80% Senior share cap requires.
//
// The trader position is closed through the Engine, and every Junior holder then exits through the
// asynchronous redemption path (requestRedeem -> settleLpEpoch -> claimRedeem), so the Junior withdrawal
// cap (HousePool._juniorSettlementCapacity) applies. Outcomes are settlement-token balances (balanceOf).
//
// Mode: by default the test expects the reference code, where the assignment must execute. With
// LOW72_EXPECT_FIX=true it expects the patched code, where the assignment must revert with
// HousePool__JuniorZeroNavWithSupply().
import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

contract AssignUnassignedZeroNavTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0xA11CE);
    address internal constant JUNIOR_LP = address(0xB0B);
    address internal constant RECEIVER = address(0x7EC7);
    address internal constant TRADER = address(0xAAA);
    address internal constant JUNIOR_SEED = address(0x5EED1);
    address internal constant SENIOR_SEED = address(0x5EED2);

    bytes4 internal constant FIX_SELECTOR = bytes4(keccak256("HousePool__JuniorZeroNavWithSupply()"));

    // Scenario (6-decimal USDG; size in 18-decimal synthetic units).
    uint256 internal constant JUNIOR_DEPOSIT = 50_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 150_000e6;
    uint256 internal constant TRADER_MARGIN = 5_000e6;
    uint256 internal constant SIZE = 75_000e18; // multiple of CfdTypes.SIZE_QUANTUM (1e20)
    uint256 internal constant WIPE_MARK = 1.8e8;
    uint256 internal constant RECAP = 20_000e6;
    uint256 internal constant MAX_EPOCHS = 4;

    // Rounding tolerance (virtual-share offset): 1 USDG.
    uint256 internal constant TOLERANCE = 1e6;

    struct Outcome {
        uint256 holderUsdg;
        uint256 receiverUsdg;
        uint256 seniorUsdg;
        uint256 holderSharesLeft;
        uint256 receiverSharesLeft;
        uint256 juniorTotalAssets;
        uint256 juniorPrincipalBeforeExit;
        uint256 seniorPrincipalBeforeExit;
        uint256 maxJuniorWithdrawBeforeExit;
        uint256 unassignedAfter;
    }

    // ---- Release configuration -------------------------------------------------------------------------

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _juniorSeedReceiver() internal pure override returns (address) { return JUNIOR_SEED; }
    function _seniorSeedReceiver() internal pure override returns (address) { return SENIOR_SEED; }

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
        // Release HousePool limits (ArbitrumSepoliaReleaseHousePool), through the governed 72h path.
        _setSeniorCapacity(40_000_000e6, 8000);
        // Release Router/Oracle values (ArbitrumSepoliaReleaseRouter / ArbitrumSepoliaReleaseOracle).
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.maxPendingOrders = 5;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(rc);
        // The governed paths warp 72h each (into a weekend, oracle frozen); return to the fixture's Monday.
        vm.warp(SETUP_TIMESTAMP);
        _assertReleaseConfig();
    }

    function _assertReleaseConfig() internal view {
        CfdTypes.RiskParams memory rp = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(rp.vpiFactor, 0.01e18, "vpiFactor");
        assertEq(rp.maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(rp.maintMarginBps, 10, "maintMarginBps");
        assertEq(rp.initMarginBps, 20, "initMarginBps");
        assertEq(rp.fadMarginBps, 300, "fadMarginBps");
        assertEq(rp.baseCarryBps, 500, "baseCarryBps");
        assertEq(rp.minBountyAssets, 1e6, "minBountyAssets");
        assertEq(rp.bountyBps, 10, "bountyBps");
        assertEq(rp.keeperShareBps, 5000, "keeperShareBps");
        assertEq(rp.protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "fadRunwaySeconds");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "basketMaxConfidenceRatioBps");
        assertEq(router.maxPendingOrders(), 5, "maxPendingOrders");
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "positionProtectionTriggerBountyAssets");
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "juniorMaintenanceFeeAprBps");
        assertEq(SIZE % CfdTypes.SIZE_QUANTUM, 0, "size quantum");
    }

    // ---- Helpers -----------------------------------------------------------------------------------------

    function _mark(uint256 p) internal {
        _ensureLaterOracleTime();
        vm.prank(address(router));
        engine.updateMarkPrice(p, uint64(vm.getBlockTimestamp()));
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// Junior NAV wiped to exactly zero by mark-to-market (supply > 0), then the documented owner
    /// recapitalization restores Senior and leaves the remainder in `unassignedAssets`.
    function _setupZeroNavWithUnassigned() internal returns (uint256 unassigned) {
        // Junior first: with the 80% Senior share cap a Senior-first deposit exceeds capacity.
        _fundJunior(JUNIOR_LP, JUNIOR_DEPOSIT);
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);
        _fundTrader(TRADER, TRADER_MARGIN);
        _ensureLaterOracleTime();
        _open(TRADER, CfdTypes.Side.SHORT, SIZE, TRADER_MARGIN, 1e8);
        _mark(WIPE_MARK);
        emit log_named_uint("setup: juniorPrincipal at wipe mark", pool.juniorPrincipal());
        emit log_named_uint("setup: seniorPrincipal at wipe mark", pool.seniorPrincipal());
        emit log_named_uint("setup: seniorHighWaterMark", pool.seniorHighWaterMark());
        assertEq(juniorVault.totalAssets(), 0, "precondition: junior NAV is exactly zero");
        assertGt(juniorVault.totalSupply(), 0, "precondition: junior supply is non-zero");

        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        _mark(WIPE_MARK);
        unassigned = pool.unassignedAssets();
        emit log_named_uint("setup: unassignedAssets after recapitalization", unassigned);
        emit log_named_uint("setup: seniorPrincipal after recapitalization", pool.seniorPrincipal());
        assertEq(pool.juniorPrincipal(), 0, "precondition: junior principal still zero");
        assertGt(unassigned, 0, "precondition: recapitalization remainder is unassigned");
    }

    /// Mark returns to entry and the trader closes; then the listed Junior holders (and optionally Senior)
    /// request full redemption in the same LP epoch and claim what settlement funds.
    function _recoverCloseAndExit(bool withReceiver, bool seniorExits) internal returns (Outcome memory o) {
        _mark(1e8);
        _close(TRADER, CfdTypes.Side.SHORT, SIZE, 1e8);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "trader position closed");
        vm.prank(address(juniorVault));
        pool.reconcile();

        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1); // receiver cooldown after bootstrap mint
        _mark(1e8);
        o.juniorTotalAssets = juniorVault.totalAssets();
        o.juniorPrincipalBeforeExit = pool.juniorPrincipal();
        o.seniorPrincipalBeforeExit = pool.seniorPrincipal();
        o.maxJuniorWithdrawBeforeExit = pool.getMaxJuniorWithdraw();

        uint256 hBefore = settlementToken.balanceOf(JUNIOR_LP);
        uint256 rBefore = settlementToken.balanceOf(RECEIVER);
        uint256 sBefore = settlementToken.balanceOf(SENIOR_LP);

        uint256 hReq = _requestFull(JUNIOR_LP, false);
        uint256 rReq = withReceiver ? _requestFull(RECEIVER, false) : 0;
        uint256 sReq = seniorExits ? _requestFull(SENIOR_LP, true) : 0;

        uint256 epochStart = pool.lpEpochStart(hReq);
        if (vm.getBlockTimestamp() < epochStart) vm.warp(epochStart);
        // Settle successive LP epochs. The first settlement must make progress; later ones either fund more or
        // revert with exactly HousePool__NoLpEpochProgress (the Junior withdrawal cap still binds).
        for (uint256 i; i < MAX_EPOCHS; ++i) {
            _mark(1e8);
            (bool ok, bytes memory ret) = address(pool).call(abi.encodeCall(pool.settleLpEpoch, (0, 0)));
            if (!ok) {
                assertGt(i, 0, "first LP epoch settlement must make progress");
                assertEq(_revertSelector(ret), IHousePool.HousePool__NoLpEpochProgress.selector, "unexpected revert");
                emit log_named_uint("  no further LP epoch progress at epoch index", i);
                break;
            }
            IHousePool.LpEpochSettlementResult memory r = abi.decode(ret, (IHousePool.LpEpochSettlementResult));
            emit log_named_uint("  epoch index", i);
            emit log_named_uint("    senior funded assets", r.seniorFundedAssets);
            emit log_named_uint("    junior funded assets", r.juniorFundedAssets);
            _claimAll(JUNIOR_LP, hReq, false);
            if (withReceiver) _claimAll(RECEIVER, rReq, false);
            if (seniorExits) _claimAll(SENIOR_LP, sReq, true);
            vm.warp(vm.getBlockTimestamp() + pool.LP_EPOCH_DURATION());
        }

        o.holderUsdg = settlementToken.balanceOf(JUNIOR_LP) - hBefore;
        o.receiverUsdg = settlementToken.balanceOf(RECEIVER) - rBefore;
        o.seniorUsdg = settlementToken.balanceOf(SENIOR_LP) - sBefore;
        o.holderSharesLeft = juniorVault.pendingRedeemRequest(hReq, JUNIOR_LP);
        o.receiverSharesLeft = withReceiver ? juniorVault.pendingRedeemRequest(rReq, RECEIVER) : 0;
        o.unassignedAfter = pool.unassignedAssets();
    }

    function _requestFull(address lp, bool senior) internal returns (uint256 requestId) {
        uint256 shares = senior ? seniorVault.maxRequestRedeem(lp) : juniorVault.maxRequestRedeem(lp);
        assertGt(shares, 0, "redeemable shares");
        vm.prank(lp);
        requestId = senior ? seniorVault.requestRedeem(shares, lp, lp) : juniorVault.requestRedeem(shares, lp, lp);
    }

    function _claimAll(address lp, uint256 requestId, bool senior) internal {
        uint256 shares = senior
            ? seniorVault.claimableRedeemRequest(requestId, lp)
            : juniorVault.claimableRedeemRequest(requestId, lp);
        if (shares == 0) return;
        vm.prank(lp);
        if (senior) seniorVault.claimRedeem(requestId, shares, lp, lp);
        else juniorVault.claimRedeem(requestId, shares, lp, lp);
    }

    function _log(string memory tag, Outcome memory o) internal {
        emit log_string(tag);
        emit log_named_uint("  junior totalAssets after close", o.juniorTotalAssets);
        emit log_named_uint("  juniorPrincipal before exit", o.juniorPrincipalBeforeExit);
        emit log_named_uint("  seniorPrincipal before exit", o.seniorPrincipalBeforeExit);
        emit log_named_uint("  getMaxJuniorWithdraw before exit", o.maxJuniorWithdrawBeforeExit);
        emit log_named_uint("  existing junior LP USDG received", o.holderUsdg);
        emit log_named_uint("  receiver USDG received", o.receiverUsdg);
        emit log_named_uint("  senior LP USDG received", o.seniorUsdg);
        emit log_named_uint("  existing junior LP shares still queued", o.holderSharesLeft);
        emit log_named_uint("  receiver shares still queued", o.receiverSharesLeft);
        emit log_named_uint("  unassignedAssets after exit", o.unassignedAfter);
    }

    function _runProperty(bool seniorExits) internal {
        uint256 unassigned = _setupZeroNavWithUnassigned();
        uint256 snap = vm.snapshotState();

        // Control: no assignment.
        Outcome memory c = _recoverCloseAndExit(false, seniorExits);
        _log("CONTROL (no assignment)", c);
        assertGt(c.holderUsdg, 0, "control: existing junior LP withdraws USDG");

        vm.revertToState(snap);

        // Assignment branch: owner assigns the ownerless bucket to Junior at exactly zero Junior NAV.
        if (vm.envOr("LOW72_EXPECT_FIX", false)) {
            vm.expectRevert(FIX_SELECTOR);
            pool.assignUnassignedAssets(false, RECEIVER);
            assertEq(pool.unassignedAssets(), unassigned, "fix: bucket stays unassigned");
            emit log_string("assignment reverted with HousePool__JuniorZeroNavWithSupply()");
        } else {
            pool.assignUnassignedAssets(false, RECEIVER); // must execute on the reference code
            assertEq(pool.unassignedAssets(), 0, "reference: bucket assigned");
            assertGt(juniorVault.balanceOf(RECEIVER), 0, "reference: receiver holds junior shares");
            emit log_named_uint("assignment executed; receiver junior shares", juniorVault.balanceOf(RECEIVER));
        }
        bool hasReceiverShares = juniorVault.balanceOf(RECEIVER) > 0;
        Outcome memory a = _recoverCloseAndExit(hasReceiverShares, seniorExits);
        emit log_named_uint("ownerless assets", unassigned);
        _log("ASSIGNMENT BRANCH", a);

        assertGe(a.holderUsdg + TOLERANCE, c.holderUsdg, "existing junior holder withdrew less USDG than control");
        assertLe(a.receiverUsdg, unassigned + TOLERANCE, "receiver withdrew more USDG than the ownerless assets");
    }

    // ---- Tests -------------------------------------------------------------------------------------------

    /// Senior stays in the pool; the Junior withdrawal cap (protected-senior covenant at 80%) binds.
    function test_AssignAtZeroJuniorNav_JuniorExit_PreservesHoldersUsdg() public {
        _runProperty(false);
    }

    /// Senior requests full redemption in the same epoch (funded first), so the covenant relaxes.
    function test_AssignAtZeroJuniorNav_FullExit_PreservesHoldersUsdg() public {
        _runProperty(true);
    }

    /// The other target available to the owner in the same state.
    function test_SeniorAssignmentRevertsAtZeroJuniorNav() public {
        _setupZeroNavWithUnassigned();
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, RECEIVER);
    }
}
```
