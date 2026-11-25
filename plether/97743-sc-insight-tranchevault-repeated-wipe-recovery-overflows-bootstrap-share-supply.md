# \#97743 [SC-Insight] TrancheVault: repeated wipe recovery overflows bootstrap share supply

**Submitted on Nov 2nd 2026 at 23:13:06 UTC by @zaaf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97743
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

# TrancheVault: repeated wipe recovery overflows bootstrap share supply

## Brief/Intro

When Junior principal is wiped to zero while Junior shares survive, `assignUnassignedAssets` prices the recovery against zero assets and the existing accrued/effective pricing supply. The quote is then minted without rebasing or retiring the old shares. Repeating that documented recovery sequence makes the share supply grow geometrically until either ERC-20 `_mint` or the next `pricingSupply + VIRTUAL_SHARES` calculation overflows.

The runnable test at the frozen competition commit reproduces the boundary after six wipe/recovery cycles. The same deposit and exit settle after five cycles; after the sixth recovery, bootstrap quoting and normal LP epoch settlement panic with Solidity error `0x11`. The paired counter-test also exercises the documented `pool.pause()` path: it bypasses deposit activation and funds part of the queued redemption, but the deposit remains pending.

This report is submitted as an Insight. The test uses mock Pyth fixture updates to reach six extreme adverse moves; it does not establish that an external user can force those prices on the intended deployment.

## Vulnerability Details

### Affected code

Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

- `TrancheVault.sol:1392-1395`: `quoteBootstrapDeposit` prices a bootstrap mint using the current accrued supply.
- `TrancheVault.sol:1853-1859`: `_convertToSharesUsingAssets` computes `assets * (accruedTotalSupply() + 1000) / (totalAssets + 1)`.
- `HousePool.sol:660-703`: `assignUnassignedAssets` obtains this quote and mints the resulting shares.
- `TrancheVault.sol:1549-1555`: `bootstrapMint` adds the quoted shares to the surviving ERC-20 supply.

### Mechanism

After a total Junior wipe, the pricing state used by the bootstrap quote is:

```text
pricingAssets = 0
pricingSupply = accrued/effective surviving Junior share supply
denominator = pricingAssets + 1 = 1
newShares = recoveryAssets * (pricingSupply + 1000)
```

The old shares are not burned or rebased before `newShares` are minted. Therefore a later wipe followed by another owner recovery uses the already amplified supply as the next multiplier. The documentation intentionally allows zero-NAV Junior holders to participate in a later recovery; the issue reported here is narrower: repeated use of that representation has no normalization bound and can reach the `uint256` liveness limit.

In the measured run, the supply after five recoveries is:

```text
976562500996093750019687500000156875000000626250000001251000000000000
```

At the sixth recovery:

- assigning `118571099` asset atoms produces a valid full-precision quote, but adding the quoted shares to the existing ERC-20 supply panics in `_mint`;
- assigning `118571098` asset atoms succeeds and establishes the exact positive boundary;
- attempting to assign the remaining assets then panics while evaluating `pricingSupply + 1000` in the next bootstrap quote.

This shows that using `Math.mulDiv` is not sufficient: the full-precision multiplication can return a value that cannot be added to the existing supply, and the following quote can fail before multiplication.

## Impact Details

### Measured result

The PoC contains a same-state paired comparison:

```text
five-recovery control:
  deposit claim = 1000000000 asset atoms
  incumbent exit claim = 125000000000 asset atoms

six-recovery target:
  deposit remains pending = 1000000000 asset atoms
  deposit recipient external cash = 0
  normal settlement funds no exit assets before reverting
  pool.pause() recovery then funds exit = 18461021 asset atoms
```

The emergency pause therefore refutes a terminal-lock claim for redemptions. The measured liveness defect is the overflowing deposit/bootstrap path and the failure of normal atomic settlement while that deposit remains at the queue head; redemptions can still be funded through the pause path.

The recovery-boundary test additionally records:

```text
maxRecoverableAssets = 118571098
pending assets immediately after the failed remainder = 249881428902
unassigned assets after later reconciliation = 249881428902
overflow-attributable attacker gain = 0
```

The quoted asset amounts are fixture values, not a claim about currently deployed funds at risk.

### Preconditions and limits

- The fixture uses six mock price moves from `100000000` to `30000000` (70% adverse moves) followed by six owner-funded recovery attempts. A separate boundary probe measures `62.500001%` as the minimum adverse move that wipes Junior in this fixture.
- The owner calls the documented `recapitalizePool` and `assignUnassignedAssets` recovery paths; the test does not require the owner to act maliciously.
- Mock Pyth update data is used only in the repository test environment, as permitted for an out-of-scope component. It proves the contract arithmetic and resulting state transition, not the real-world probability of those price moves.
- No external attacker profit attributable to the overflow is claimed.
- The owner can avoid reaching the terminal state by not repeating this recovery pattern. The value of the Insight is that the recovery mechanism itself has no supply normalization or overflow-safe terminal handling.

### Public known-issue comparison

The closest published audit items describe different state transitions:

- Cecuro LOW-72 concerns an indicative deposit estimate after a wiped tranche; it does not cover repeated bootstrap mints or supply overflow.
- Cecuro MED-5 concerns refund dust when bootstrap shares exist without a seed receiver; it does not cover geometric supply amplification.

### Remediation

Define a bounded post-wipe share normalization before minting recovery shares. For example, retire or rebase the stale share supply and re-establish a fixed bootstrap exchange ratio, while preserving each holder's proportional entitlement under an explicit migration rule.

Before mutating pool accounting, `assignUnassignedAssets` should validate that `accruedTotalSupply() + VIRTUAL_SHARES` and `totalSupply() + quotedShares` are both representable.

A checked revert alone would make the failure explicit but would not restore deposit liveness. The recovery path needs a defined way to normalize or migrate an already amplified supply so unassigned assets can be assigned and pending deposits can be activated.

## References

- `packages/perps/src/TrancheVault.sol`, lines 1392-1395, 1549-1555 and 1853-1859 at the frozen commit.
- `packages/perps/src/HousePool.sol`, lines 660-703 at the frozen commit.
- `SECURITY.md`, zero-NAV Junior recovery behavior and paused redemption-funding guarantees.
- Plether competition scope and PoC requirements: `https://immunefi.com/audit-competition/plether/scope/`


## Proof of Concept

## Proof of Concept

1. The reviewer saves the following complete source as `poc.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract RepeatedWipeBootstrapSupplyPoCTest is BasePerpTest {
    address private constant INCUMBENT = address(0xA5520001);
    address private constant KEEPER = address(0xA5520002);
    address private constant RECOVERY_RECEIVER = address(0xA5520003);
    address private constant FRESH_RECEIVER = address(0xA5520004);
    address private constant DEPOSIT_VICTIM = address(0xA5520005);
    uint256 private constant RECOVERY_ASSETS = 250_000e6;
    uint256 private constant MAX_RECOVERABLE = 118_571_098;
    uint256 private constant REMAINING_ASSETS = RECOVERY_ASSETS - MAX_RECOVERABLE;
    uint256 private constant SIZE = 400_000e18;
    uint256 private constant FUND = 150_000e6;
    uint256 private constant MARGIN = 100_000e6;
    uint256 private constant ORDINARY_DEPOSIT = 1000e6;
    uint256 private ownerCapital;
    uint256 private traderGain;
    uint256 private lpLoss;

    struct ConsumerArm {
        bool quoteSuccess;
        bytes quoteData;
        bool depositSettleSuccess;
        bytes depositSettleData;
        uint256 depositClaimAssets;
        bool depositCancelSuccess;
        bytes depositCancelData;
        bool depositRetrySuccess;
        bool exitRequestSuccess;
        bool exitSettleSuccess;
        uint256 exitClaimAssets;
        uint256 exitPendingShares;
        uint256 victimPendingAssets;
        uint256 victimExternalCash;
        uint256 holderExternalCash;
        uint256 holderExternalCashBeforePause;
        uint256 depositQueueHead;
        bool pausedRecoverySettleSuccess;
        uint256 pausedRecoveryClaimAssets;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = 0;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        _refresh(1e8);
        pool.finalizePoolConfig();
        _fundSeniorPublic(INCUMBENT, 1_000_000e6);
        vm.warp(seniorVault.lastDepositTime(INCUMBENT) + seniorVault.DEPOSIT_COOLDOWN());
        _refresh(1e8);
    }

    function test_SixJuniorRecoveriesReachBootstrapSupplyOverflow() public {
        uint256 cycle5Supply;
        uint256 finalGap;
        for (uint256 cycle = 1; cycle <= 6; ++cycle) {
            if (cycle == 2) {
                uint256 priceSnapshot = vm.snapshotState();
                assertTrue(_probeWipe(37_499_999), "37.499999 must wipe Junior");
                assertTrue(vm.revertToState(priceSnapshot));
                assertFalse(_probeWipe(37_500_000), "37.500000 must preserve Junior");
                assertTrue(vm.revertToState(priceSnapshot));
                vm.deleteStateSnapshot(priceSnapshot);
            }
            (uint256 gain, uint256 loss) = _lossCycle(cycle, 30_000_000);
            traderGain += gain;
            lpLoss += loss;
            (uint256 projectedSenior, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
            assertEq(projectedJunior, 0, "cycle did not wipe Junior");
            uint256 gap = pool.seniorHighWaterMark() - projectedSenior;
            ownerCapital += gap + RECOVERY_ASSETS;
            if (cycle <= 5) {
                _recap(gap + RECOVERY_ASSETS);
                (bool ok,,) = _tryAssignJunior(RECOVERY_RECEIVER);
                assertTrue(ok, "cycles 1-5 must assign full recovery");
                assertEq(pool.unassignedAssets(), 0);
                if (cycle == 5) cycle5Supply = juniorVault.totalSupply();
            } else finalGap = gap;
        }
        assertEq(cycle5Supply, 976_562_500_996_093_750_019_687_500_000_156_875_000_000_626_250_000_001_251_000_000_000_000);
        assertEq(ownerCapital, 1_929_000_656_824);

        uint256 boundary = vm.snapshotState();
        _recap(finalGap + RECOVERY_ASSETS);
        (bool fullOk, bytes4 fullSelector, uint256 fullCode) = _tryAssignJunior(RECOVERY_RECEIVER);
        assertFalse(fullOk); assertEq(fullSelector, bytes4(0x4e487b71)); assertEq(fullCode, 0x11);
        assertTrue(vm.revertToState(boundary));
        _recap(finalGap + MAX_RECOVERABLE + 1);
        (bool plusOneOk, bytes4 plusOneSelector, uint256 plusOneCode) = _tryAssignJunior(RECOVERY_RECEIVER);
        assertFalse(plusOneOk); assertEq(plusOneSelector, bytes4(0x4e487b71)); assertEq(plusOneCode, 0x11);
        assertTrue(vm.revertToState(boundary));
        _recap(finalGap + MAX_RECOVERABLE);
        (bool maxOk,,) = _tryAssignJunior(RECOVERY_RECEIVER);
        assertTrue(maxOk, "MAX_RECOVERABLE positive control failed");
        _recap(REMAINING_ASSETS);
        (bool remainderOk, bytes4 remainderSelector, uint256 remainderCode) = _tryAssignJunior(RECOVERY_RECEIVER);
        assertFalse(remainderOk); assertEq(remainderSelector, bytes4(0x4e487b71)); assertEq(remainderCode, 0x11);
        assertEq(pool.pendingRecapitalizationAssets(), REMAINING_ASSETS);

        vm.warp(juniorVault.lastDepositTime(RECOVERY_RECEIVER) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 juniorShares = juniorVault.maxRequestRedeem(RECOVERY_RECEIVER);
        vm.prank(RECOVERY_RECEIVER);
        uint256 juniorRequest = juniorVault.requestRedeem(juniorShares, RECOVERY_RECEIVER, RECOVERY_RECEIVER);
        vm.warp(pool.lpEpochStart(juniorRequest)); _refresh(1e8); vm.prank(KEEPER);
        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(first.juniorFundedAssets, 18_461_021); _claimJunior(juniorRequest);

        uint256 seniorShares = seniorVault.maxRequestRedeem(INCUMBENT);
        vm.prank(INCUMBENT);
        uint256 seniorRequest = seniorVault.requestRedeem(seniorShares, INCUMBENT, INCUMBENT);
        vm.warp(pool.lpEpochStart(seniorRequest)); _refresh(1e8); vm.prank(KEEPER); pool.settleLpEpoch(0, 0);
        _claimJunior(juniorRequest);
        uint256 seniorClaimable = seniorVault.claimableRedeemRequest(seniorRequest, INCUMBENT);
        vm.prank(INCUMBENT); seniorVault.claimRedeem(seniorRequest, seniorClaimable, INCUMBENT, INCUMBENT);
        _refresh(1e8); vm.prank(KEEPER); vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);

        assertEq(settlementToken.balanceOf(RECOVERY_RECEIVER), 118_471_022);
        (, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
        assertEq(projectedJunior, 100_076);
        assertGt(juniorVault.pendingRedeemRequest(juniorRequest, RECOVERY_RECEIVER), 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0); assertEq(pool.unassignedAssets(), REMAINING_ASSETS);
        (bool existingOk, bytes4 existingSelector, uint256 existingCode) = _tryAssignJunior(RECOVERY_RECEIVER);
        assertFalse(existingOk); assertEq(existingSelector, bytes4(0x4e487b71)); assertEq(existingCode, 0x11);
        (bool freshOk, bytes4 freshSelector, uint256 freshCode) = _tryAssignJunior(FRESH_RECEIVER);
        assertFalse(freshOk); assertEq(freshSelector, bytes4(0x4e487b71)); assertEq(freshCode, 0x11);
        (bool seniorOk, bytes4 seniorSelector,) = _tryAssignSenior(FRESH_RECEIVER);
        assertFalse(seniorOk); assertEq(seniorSelector, IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);

        assertFalse(juniorVault.redeemRefundPending(juniorRequest, RECOVERY_RECEIVER));
        vm.startPrank(RECOVERY_RECEIVER);
        vm.expectRevert(bytes4(keccak256("TrancheVault__RedeemCancellationUnavailable()")));
        juniorVault.cancelRedeemRequest(juniorRequest, RECOVERY_RECEIVER);
        vm.expectRevert(bytes4(keccak256("TrancheVault__RedeemRefundUnavailable()")));
        juniorVault.claimRedeemRefund(juniorRequest, RECOVERY_RECEIVER, RECOVERY_RECEIVER);
        vm.stopPrank();
        pool.pauseLpEpochSettlement(); pool.unpauseLpEpochSettlement(); _refresh(1e8); vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector); pool.settleLpEpoch(0, 0);
        assertEq(pool.rawAssets(), pool.accountedAssets());
        assertEq(pool.rawAssets(), settlementToken.balanceOf(address(pool)));
        assertEq(_conserved(), settlementToken.totalSupply());

        console2.log("minimumWipingClosePrice", uint256(37_499_999));
        console2.log("minimumNonWipingClosePrice", uint256(37_500_000));
        console2.log("minimumAdverseMovePriceUnits", uint256(62_500_001));
        console2.log("cycle5JuniorSupply", cycle5Supply);
        console2.log("maxRecoverableAssets", MAX_RECOVERABLE);
        console2.log("pendingAssetsAfterFailedRemainder", REMAINING_ASSETS);
        console2.log("holderRecoveredAssets", settlementToken.balanceOf(RECOVERY_RECEIVER));
        console2.log("terminalJuniorPrincipal", projectedJunior);
        console2.log("terminalUnassignedAssets", pool.unassignedAssets());
        console2.log("overflowAttributableAttackerGain", uint256(0));
        console2.log("totalOwnerCapital", ownerCapital);
        console2.log("ordinaryMarketTraderGain", traderGain);
        console2.log("ordinaryMarketLpLoss", lpLoss);
        vm.deleteStateSnapshot(boundary);
    }

    function test_SixthRecoveryBlocksDepositButPauseFundsExit() public {
        for (uint256 cycle = 1; cycle <= 5; ++cycle) {
            _lossCycle(cycle, 30_000_000);
            (uint256 senior, uint256 junior) = pool.getPendingDepositTrancheState();
            assertEq(junior, 0); _recap(pool.seniorHighWaterMark() - senior + RECOVERY_ASSETS);
            (bool assigned,,) = _tryAssignJunior(RECOVERY_RECEIVER); assertTrue(assigned);
        }
        assertEq(juniorVault.totalSupply(), 976_562_500_996_093_750_019_687_500_000_156_875_000_000_626_250_000_001_251_000_000_000_000);
        uint256 victimRequest = _requestJuniorDeposit(DEPOSIT_VICTIM, ORDINARY_DEPOSIT);
        uint256 common = vm.snapshotState();
        ConsumerArm memory control = _consumerArm(victimRequest, false);
        assertTrue(vm.revertToState(common));
        ConsumerArm memory target = _consumerArm(victimRequest, true);
        assertTrue(control.quoteSuccess); assertTrue(control.depositSettleSuccess);
        assertEq(control.depositClaimAssets, ORDINARY_DEPOSIT);
        assertTrue(control.exitRequestSuccess); assertTrue(control.exitSettleSuccess);
        assertEq(control.exitClaimAssets, 125_000e6);
        assertFalse(target.quoteSuccess); assertEq(_selector(target.quoteData), bytes4(0x4e487b71));
        assertFalse(target.depositSettleSuccess); assertEq(_selector(target.depositSettleData), bytes4(0x4e487b71));
        assertFalse(target.depositCancelSuccess); assertEq(_selector(target.depositCancelData), bytes4(0x1ad6d803));
        assertFalse(target.depositRetrySuccess); assertTrue(target.exitRequestSuccess); assertFalse(target.exitSettleSuccess);
        assertGt(target.exitPendingShares, 0); assertEq(target.exitClaimAssets, 0);
        assertTrue(target.pausedRecoverySettleSuccess);
        assertGt(target.pausedRecoveryClaimAssets, 0);
        assertEq(target.victimPendingAssets, ORDINARY_DEPOSIT); assertEq(target.victimExternalCash, 0);
        assertEq(target.holderExternalCashBeforePause, 0);
        assertEq(target.holderExternalCash, target.pausedRecoveryClaimAssets);
        assertEq(target.depositQueueHead, victimRequest);
        console2.log("controlVictimClaimAssets", control.depositClaimAssets);
        console2.log("controlHolderExitAssets", control.exitClaimAssets);
        console2.log("targetVictimPendingAssets", target.victimPendingAssets);
        console2.log("targetVictimExternalCash", target.victimExternalCash);
        console2.log("targetHolderPendingShares", target.exitPendingShares);
        console2.log("targetHolderExternalCash", target.holderExternalCash);
        console2.log("pausedRecoveryHolderClaimAssets", target.pausedRecoveryClaimAssets);
        vm.deleteStateSnapshot(common);
    }

    function _consumerArm(uint256 victimRequest, bool sixthWipe) private returns (ConsumerArm memory out) {
        if (sixthWipe) {
            _lossCycle(6, 30_000_000);
            (uint256 senior, uint256 junior) = pool.getPendingDepositTrancheState();
            assertEq(junior, 0); _recap(pool.seniorHighWaterMark() - senior + MAX_RECOVERABLE);
            (bool assigned,,) = _tryAssignJunior(RECOVERY_RECEIVER); assertTrue(assigned);
        }
        (out.quoteSuccess, out.quoteData) = address(juniorVault).staticcall(abi.encodeWithSignature(
            "quoteDepositFromState(uint256,uint256,uint256,uint256)", ORDINARY_DEPOSIT,
            pool.juniorPrincipal(), juniorVault.accruedTotalSupply(), 0));
        _mature(victimRequest); (out.depositSettleSuccess, out.depositSettleData) = _trySettle();
        if (out.depositSettleSuccess) {
            out.depositClaimAssets = juniorVault.claimableDepositRequest(victimRequest, DEPOSIT_VICTIM);
            vm.prank(DEPOSIT_VICTIM);
            juniorVault.claimDeposit(victimRequest, out.depositClaimAssets, DEPOSIT_VICTIM, DEPOSIT_VICTIM);
        } else {
            vm.prank(DEPOSIT_VICTIM);
            (out.depositCancelSuccess, out.depositCancelData) = address(juniorVault).call(abi.encodeWithSignature(
                "cancelPendingDeposit(uint256,address,address)", victimRequest, DEPOSIT_VICTIM, DEPOSIT_VICTIM));
            (out.depositRetrySuccess,) = _trySettle();
        }
        uint256 unlock = juniorVault.lastDepositTime(RECOVERY_RECEIVER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);
        uint256 shares = juniorVault.maxRequestRedeem(RECOVERY_RECEIVER) / 2;
        vm.prank(RECOVERY_RECEIVER); bytes memory requestData;
        (out.exitRequestSuccess, requestData) = address(juniorVault).call(abi.encodeWithSignature(
            "requestRedeem(uint256,address,address)", shares, RECOVERY_RECEIVER, RECOVERY_RECEIVER));
        assertTrue(out.exitRequestSuccess); uint256 exitId = abi.decode(requestData, (uint256));
        _mature(exitId); (out.exitSettleSuccess,) = _trySettle();
        if (out.exitSettleSuccess) {
            uint256 claimable = juniorVault.claimableRedeemRequest(exitId, RECOVERY_RECEIVER);
            out.exitClaimAssets = juniorVault.claimableRedeemAssets(exitId, RECOVERY_RECEIVER);
            vm.prank(RECOVERY_RECEIVER);
            juniorVault.claimRedeem(exitId, claimable, RECOVERY_RECEIVER, RECOVERY_RECEIVER);
        }
        out.holderExternalCashBeforePause = settlementToken.balanceOf(RECOVERY_RECEIVER);
        if (sixthWipe && !out.exitSettleSuccess) {
            pool.pause();
            (out.pausedRecoverySettleSuccess,) = _trySettle();
            if (out.pausedRecoverySettleSuccess) {
                uint256 pausedClaimable = juniorVault.claimableRedeemRequest(exitId, RECOVERY_RECEIVER);
                out.pausedRecoveryClaimAssets = juniorVault.claimableRedeemAssets(exitId, RECOVERY_RECEIVER);
                if (pausedClaimable != 0) {
                    vm.prank(RECOVERY_RECEIVER);
                    juniorVault.claimRedeem(
                        exitId, pausedClaimable, RECOVERY_RECEIVER, RECOVERY_RECEIVER
                    );
                }
            }
            pool.unpause();
        }
        out.exitPendingShares = juniorVault.pendingRedeemRequest(exitId, RECOVERY_RECEIVER);
        out.victimPendingAssets = juniorVault.pendingDepositRequest(victimRequest, DEPOSIT_VICTIM);
        out.victimExternalCash = settlementToken.balanceOf(DEPOSIT_VICTIM);
        out.holderExternalCash = settlementToken.balanceOf(RECOVERY_RECEIVER);
        out.depositQueueHead = juniorVault.depositQueueHead();
    }

    function _requestJuniorDeposit(address lp, uint256 amount) private returns (uint256 id) {
        settlementToken.mint(lp, amount); vm.startPrank(lp); settlementToken.approve(address(juniorVault), amount);
        id = juniorVault.requestDeposit(amount, lp, lp); vm.stopPrank();
    }
    function _mature(uint256 id) private {
        uint256 maturity = pool.lpEpochStart(id); if (block.timestamp < maturity) vm.warp(maturity); _refresh(1e8);
    }
    function _trySettle() private returns (bool ok, bytes memory data) {
        vm.prank(KEEPER); (ok, data) = address(pool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
    }
    function _selector(bytes memory data) private pure returns (bytes4 selector) {
        if (data.length >= 4) { assembly ("memory-safe") { selector := mload(add(data, 32)) } }
    }
    function _lossCycle(uint256 cycle, uint256 closePrice) private returns (uint256 gain, uint256 loss) {
        address trader = address(uint160(0xA5521000 + cycle));
        uint256 beforePrincipal = pool.seniorPrincipal() + pool.juniorPrincipal();
        _fundTrader(trader, FUND); _trade(trader, 100_000_000, false); _trade(trader, closePrice, true);
        _withdrawAll(trader); _withdrawAll(KEEPER); _withdrawAll(PROTOCOL_TREASURY_ACCOUNT); _refresh(1e8);
        (uint256 senior, uint256 junior) = pool.getPendingDepositTrancheState();
        loss = beforePrincipal - senior - junior; gain = settlementToken.balanceOf(trader) - FUND;
    }
    function _probeWipe(uint256 closePrice) private returns (bool) {
        address trader = address(0xA5522000); _fundTrader(trader, FUND);
        _trade(trader, 100_000_000, false); _trade(trader, closePrice, true); _refresh(1e8);
        (, uint256 junior) = pool.getPendingDepositTrancheState(); return junior == 0;
    }
    function _trade(address trader, uint256 price, bool close) private {
        vm.prank(trader); uint64 id = router.commitOrder(CfdTypes.Side.LONG, SIZE, close ? 0 : MARGIN, 0, close);
        bytes[] memory data = _mockPythUpdateData(price); vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }
    function _fundSeniorPublic(address lp, uint256 amount) private {
        settlementToken.mint(lp, amount); vm.startPrank(lp); settlementToken.approve(address(seniorVault), amount);
        uint256 id = seniorVault.requestDeposit(amount, lp, lp); vm.stopPrank();
        vm.warp(seniorVault.depositEpochStart(id)); _refresh(1e8); vm.prank(KEEPER); pool.settleLpEpoch(0, 0);
        vm.prank(lp); seniorVault.claimDeposit(id, amount, lp, lp);
    }
    function _refresh(uint256 price) private {
        _ensureLaterOracleTime(); bytes[] memory data = _mockPythUpdateData(price); vm.prank(KEEPER); router.updateMarkPrice(data);
    }
    function _recap(uint256 amount) private {
        settlementToken.mint(address(this), amount); settlementToken.approve(address(engine), amount); engine.recapitalizePool(amount);
    }
    function _tryAssignJunior(address receiver) private returns (bool ok, bytes4 selector, uint256 code) {
        bytes memory data; (ok, data) = address(pool).call(abi.encodeCall(pool.assignUnassignedAssets, (false, receiver)));
        if (!ok && data.length >= 4) selector = bytes4(data);
        if (selector == bytes4(0x4e487b71) && data.length >= 36) { assembly ("memory-safe") { code := mload(add(data, 36)) } }
    }
    function _tryAssignSenior(address receiver) private returns (bool ok, bytes4 selector, uint256 code) {
        bytes memory data; (ok, data) = address(pool).call(abi.encodeCall(pool.assignUnassignedAssets, (true, receiver)));
        if (!ok && data.length >= 4) selector = bytes4(data);
        if (selector == bytes4(0x4e487b71) && data.length >= 36) { assembly ("memory-safe") { code := mload(add(data, 36)) } }
    }
    function _claimJunior(uint256 id) private {
        uint256 shares = juniorVault.claimableRedeemRequest(id, RECOVERY_RECEIVER);
        if (shares != 0) { vm.prank(RECOVERY_RECEIVER); juniorVault.claimRedeem(id, shares, RECOVERY_RECEIVER, RECOVERY_RECEIVER); }
    }
    function _withdrawAll(address account) private {
        uint256 amount = clearinghouse.balanceAssets(account); if (amount != 0) { vm.prank(account); clearinghouse.withdrawMargin(amount); }
    }
    function _conserved() private view returns (uint256 total) {
        total = settlementToken.balanceOf(address(this)) + settlementToken.balanceOf(INCUMBENT)
            + settlementToken.balanceOf(KEEPER) + settlementToken.balanceOf(PROTOCOL_TREASURY_ACCOUNT)
            + settlementToken.balanceOf(RECOVERY_RECEIVER) + settlementToken.balanceOf(FRESH_RECEIVER)
            + settlementToken.balanceOf(address(pool)) + settlementToken.balanceOf(address(seniorVault))
            + settlementToken.balanceOf(address(juniorVault)) + settlementToken.balanceOf(address(clearinghouse));
        for (uint256 cycle = 1; cycle <= 6; ++cycle) total += settlementToken.balanceOf(address(uint160(0xA5521000 + cycle)));
    }
}
```

2. Save `poc.t.sol` in the repository root, then run:

```
bash -lc "mkdir -p packages/perps/test/perps/bootstrap && cp poc.t.sol packages/perps/test/perps/bootstrap/RepeatedWipeBootstrapSupplyPoC.t.sol && cd packages/perps && forge test --match-path test/perps/bootstrap/RepeatedWipeBootstrapSupplyPoC.t.sol -vv"
```

Observed output from a fresh run at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` on 2026-11-03:

```
[PASS] test_SixJuniorRecoveriesReachBootstrapSupplyOverflow()
minimumWipingClosePrice 37499999
minimumNonWipingClosePrice 37500000
minimumAdverseMovePriceUnits 62500001
cycle5JuniorSupply 976562500996093750019687500000156875000000626250000001251000000000000
maxRecoverableAssets 118571098
pendingAssetsAfterFailedRemainder 249881428902
holderRecoveredAssets 118471022
terminalJuniorPrincipal 100076
terminalUnassignedAssets 249881428902
overflowAttributableAttackerGain 0
totalOwnerCapital 1929000656824
ordinaryMarketTraderGain 1678750799288
ordinaryMarketLpLoss 1679999999288
[PASS] test_SixthRecoveryBlocksDepositButPauseFundsExit()
controlVictimClaimAssets 1000000000
controlHolderExitAssets 125000000000
targetVictimPendingAssets 1000000000
targetVictimExternalCash 0
targetHolderPendingShares 39867702677383806690256857885162654347751588066360346926071483902040824527001
targetHolderExternalCash 18461021
pausedRecoveryHolderClaimAssets 18461021
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The paired control is executed inside `test_SixthRecoveryBlocksDepositButPauseFundsExit`: both branches start from the same snapshot, and the sixth wipe/recovery sequence is the changed variable. The target first shows that normal settlement reverts atomically, then pauses the pool and measures that redemption funding remains available while the deposit stays pending.

```
[PASS] test_SixthRecoveryBlocksDepositButPauseFundsExit()
controlVictimClaimAssets 1000000000
controlHolderExitAssets 125000000000
targetVictimPendingAssets 1000000000
targetVictimExternalCash 0
pausedRecoveryHolderClaimAssets 18461021
```

