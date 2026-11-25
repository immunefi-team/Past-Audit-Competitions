# \#97760 [SC-High] A leftover `pendingRecapitalizationAssets` residual reserves LP free cash and makes `settleLpEpoch` revert every time, so LP exits stay frozen after an honest recapitalization

**Submitted on Nov 3rd 2026 at 01:14:17 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97760
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

After a drawdown the owner recapitalizes the pool with `CfdEngine.recapitalizePool`, exactly as the runbook says (`SECURITY.md:1506-1510`). The whole recap amount is queued in `pendingRecapitalizationAssets`. A reconcile only settles the part that fits into `distributable`; the rest stays in the bucket as a residual. When the winning traders later close and take their profit as cash, the cash leaves the pool but the residual stays. That residual is reserved out of LP free cash, so `settleLpEpoch` has a budget of 0, makes no progress, and reverts with `HousePool__NoLpEpochProgress`. The revert also rolls back the reconcile that would have drained the residual. The pool cannot recover on its own and every LP redemption and deposit stays frozen until the owner donates extra cash.

It is a liveness gap in the recovery path the design relies on (see Vulnerability Details).

### Plain summary (how the bug fires)

Think of the pool as a cash box that LPs can withdraw from. The owner can top it up after a loss. The bug leaves a leftover "reserved" amount in the books that no longer has cash behind it. That leftover blocks all LP withdrawals.

1. **Loss.** Traders win big, the pool's cash drops and the protocol enters degraded mode.
2. **Top-up.** The owner follows the runbook: `recapitalizePool` with the minimum amount, then `clearDegradedMode`. The whole top-up (57,226 USDG in the PoC) is written down as "pending recap".
3. **Part of it is used, part is not.** Settlement books only the part that fits. The rest stays listed as "pending recap" in `pendingRecapitalizationAssets`.
4. **Cash leaves.** The winning trader closes and takes the profit. The cash goes out of the pool, but the leftover "pending recap" entry stays. Only 13,603 USDG real cash is left, and 57,226 USDG is marked as reserved.
5. **LPs see zero.** Reserved is more than twice the cash, so LP free cash is 0 and `settleLpEpoch` has nothing to pay out.
6. **It cannot fix itself.** The call reverts with `HousePool__NoLpEpochProgress`. The revert also cancels the reconcile that would have cleared the leftover. So every retry fails the same way (168 of 168 hourly attempts in the PoC).
7. **Only the owner can unlock it.** The owner has to send about 15,010 USDG of extra cash to the pool. After that, only 3,562 USDG becomes withdrawable.

### Vulnerability Details

1. The recap is queued whole:

```solidity
// CfdEngine.sol:632-653 (onlyOwner) -> HousePool.sol:640-647
@>  pool.recordClaimantInflow(amountAssets, IHousePool.ClaimantInflowKind.Recapitalization, ...CashArrived);
// HousePool.sol:1794-1799
@>  pendingRecapitalizationAssets += amount;
```

2. A reconcile settles only `min(pending, distributable)`; the remainder stays in the bucket:

```solidity
// HousePool.sol:1829-1834
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

3. The remaining bucket is reserved out of LP free cash, and the settle budget is computed from it:

```solidity
// HousePool.sol:842-848
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;
...
@>  budget: _min(freeAssets, seniorPrincipal),
// HousePool.sol:1727-1735: reserveAssets(snapshot, reservedPendingClaimantAssets)
```

4. With `budget == 0` the redemption loop stops at `fundedAssets >= budget` (`:988`) and deposits stay closed while `unassigned | pending > 0` (`:1108`). No epoch is processed, so settlement reverts, and the revert undoes the reconcile at `:835` that was the only thing that could shrink the bucket:

```solidity
// HousePool.sol:1183-1184
@>  if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
@>      revert HousePool__NoLpEpochProgress();
```

Freeze condition for a flat book: post-reconcile free assets = `max(0, 2X - P)`, where `P` is the stored pending recap and `X` the LP equity left in the pool. So the pool freezes when `P >= 2X` and a matured senior redeem exists.

Why this is not "residual stays reserved by design": the upstream test `test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177-220` asserts `freeAssets == 0` while the residual exists, and recovers by "claims go to 0, then the vault calls `pool.reconcile()`". In production that recovery does not exist: (a) the backing is paid out to traders, it never comes back; (b) `HousePool.reconcile()` (`:1471`, `onlyVault`) has no caller in `packages/perps/src`; (c) the only permissionless reconcile, inside `settleLpEpoch`, rolls itself back at `:1183-1184`.

No attacker is needed. The trigger is a market sequence plus the owner doing the documented recapitalization.

### Impact Details

PoC numbers (local Foundry suite, real router flows):

| | USDG |
|---|---|
| Pending recap `P` | 57,226.70 |
| LP equity left in pool `X` (all of it) | 13,603.85 |
| Settle attempts over 7 days (hourly) | 168 / 168 revert |
| Extra owner donation needed to unlock (PoC value, includes 1 USDG margin) | 15,010.50 |
| LP claimable after that donation | 3,562.28 |

- 100% of the remaining LP equity is frozen. There is no permissionless unlock and no time-based unlock: a new junior deposit is blocked (`TerminallyWiped`), a matured senior redeem cannot be cancelled, and trader round trips do not add pool cash. Only owner actions work: donate cash and call `accountExcess` (`HousePool.sol:571`), or a 72h-timelocked rate change through `finalizePoolConfig` (`:336-341`).
- The freeze is temporary, not permanent, because the owner can cure it. It is not theft and no value is destroyed. Trader backing is unaffected (liability, claims and bounty reserves are reserved separately).

### Mitigation

Do not let an already paid-out recap amount stay reserved. Any one of these closes it:

- When a recap is recorded, queue only the part that is not used to fill the existing deficit, or cap the reserve used at `:843` to `min(pendingRecapitalizationAssets, distributable)`.
- Do not revert in `_emitLpEpochSettled` (`:1183-1184`) when the reconcile inside the same call consumed claimant buckets; let that state change commit.
- Or give the vault/keepers a real caller for `reconcile()` so the residual can be cleared once its backing is gone.

### References

- `packages/perps/src/CfdEngine.sol:632-653`: `recapitalizePool` (onlyOwner); `:658-667`: `clearDegradedMode`
- `packages/perps/src/HousePool.sol:640-647`, `:1794-1799`: recap queued whole in `pendingRecapitalizationAssets`
- `packages/perps/src/HousePool.sol:1829-1834`: `capClaimantBuckets` leaves the residual
- `packages/perps/src/HousePool.sol:835`, `:842-848`, `:1727-1735`: reconcile, free-asset reserve, settle budget
- `packages/perps/src/HousePool.sol:988`, `:1108`, `:1183-1184`: loop break, deposit gate, no-progress revert
- `packages/perps/src/HousePool.sol:1471`: `reconcile()` (onlyVault, no caller in `src/`)
- `packages/perps/src/HousePool.sol:571`, `:336-341`: owner-only cures
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177-220`: upstream test that expects the residual to be released by a vault `reconcile()`
- `packages/perps/SECURITY.md:1506-1510`: recap runbook
- Related known items: MED-12 (fix at `HousePool.sol:1577-1578`), LOW-22 (`HousePool.sol:601`); different root causes

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/poc/RecapResidualFreeze.t.sol` and run `forge test --root packages/perps --match-path test/poc/RecapResidualFreeze.t.sol -vv`. It reuses the repo's `BasePerpTest` fixture (full stack: engine, HousePool, Senior/Junior vaults, router, mock Pyth for the out-of-scope oracle feed).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract PoCRecapResidualFreeze is BasePerpTest {

    address internal constant LONG_TRADER = address(0xA11CE);
    address internal constant SHORT_TRADER = address(0xB0B);
    address internal constant SENIOR_LP = address(0x1B0001);
    address internal constant KEEPER = address(0xE0EC);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.orderSettlementWindow = config.maxExecutionWindowSeconds;
        _setRouterConfig(config);
        router.updateMarkPrice(_px(1e8, 0));
        _fundSenior(SENIOR_LP, 50_000e6);
    }

    function _px(uint256 price, uint64 prev) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), prev);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _trade(address acct, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 price) internal {
        vm.prank(acct);
        uint64 oid = router.commitOrder(side, size, margin, 0, isClose);
        (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
        bytes[] memory d = _px(price, pv.commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(oid, d);
        require(r.terminalReason == OrderV3Types.TerminalReason.Executed, "trade not executed");
    }

    function test_PoC_recapResidualFreezesLpRedemptions() public {
        // 1. Two-sided book near pool capacity (healthy, well-collateralized traders).
        _fundTrader(LONG_TRADER, 30_000e6);
        _fundTrader(SHORT_TRADER, 100_000e6);
        for (uint256 i; i < 3; ++i) {
            _trade(LONG_TRADER, CfdTypes.Side.LONG, 45_000e18, 6_000e6, false, 1e8);
            _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 45_000e18, 27_000e6, false, 1e8);
        }
        console2.log("pool assets / L after book", pool.totalAssets(), _maxLiability());

        // 2. Market falls 55%: LONG closes and is paid from pool cash (pool drawdown, degraded latches).
        router.updateMarkPrice(_px(0.45e8, 0));
        (uint256 lsz,,,,,,) = engine.positions(LONG_TRADER);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, lsz, 0, true, 0.45e8);
        // 3. Market reverses to 1.9: SHORT is deep in profit (LP equity negative at the mark).
        router.updateMarkPrice(_px(1.9e8, 0));
        console2.log("degraded / pool assets", engine.degradedMode(), pool.totalAssets());

        // 4. Owner follows the insolvency runbook: recapitalize until raw solvency E >= L, then clear degraded mode.
        uint256 recap = _maxLiability() - pool.totalAssets() + 1e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();
        console2.log("recap", recap);

        // 5. Senior LP asks to exit (after the 1h LP transfer cooldown); then the SHORT realizes its profit (cash).
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        router.updateMarkPrice(_px(1.9e8, 0));
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_LP);
        assertGt(shares, 0, "LP cannot request");
        vm.prank(SENIOR_LP);
        uint256 rid = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        (uint256 ssz,,,,,,) = engine.positions(SHORT_TRADER);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, ssz, 0, true, 1.9e8);

        // State: book flat, no claims, not degraded, pool holds real cash -- but the recap bucket outlived its purpose.
        assertEq(engine.terminalNavSnapshot().hasOpenPositions, false, "book not flat");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "claims");
        assertFalse(engine.degradedMode(), "degraded");
        uint256 cash = pool.totalAssets();
        uint256 residual = pool.pendingRecapitalizationAssets();
        console2.log("pool cash (all remaining LP equity)", cash);
        console2.log("pendingRecapitalizationAssets", residual);
        assertGt(cash, 0, "no LP equity left");
        assertGe(residual, 2 * cash, "residual not >= 2x equity");
        assertEq(pool.getFreeAssets(), 0, "free cash should be fully reserved by the stale bucket");

        // 6. Keepers try every hour for 7 days: every settlement reverts and rolls back its own reconcile.
        uint256 reverts;
        for (uint256 h; h < 168; ++h) {
            vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
            vm.prank(KEEPER);
            try pool.settleLpEpoch(0, 0) {}
            catch (bytes memory err) {
                assertEq(bytes4(err), bytes4(0x86cca6b8), "unexpected revert (expected HousePool__NoLpEpochProgress)");
                reverts++;
            }
        }
        console2.log("settlement attempts reverted (of 168)", reverts);
        assertEq(reverts, 168, "some settlement made progress");
        assertEq(seniorVault.claimableRedeemRequest(rid, SENIOR_LP), 0, "LP redemption was funded");
        assertEq(pool.pendingRecapitalizationAssets(), residual, "bucket changed");

        // 7. Only an extra owner cash gift (donation + accountExcess) of ~(residual - 2*cash)/2 unlocks settlement.
        uint256 gift = (residual - 2 * cash) / 2 + 1e6;
        settlementToken.mint(address(pool), gift);
        pool.accountExcess();
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        console2.log("owner gift needed to unlock", gift);
        console2.log("LP claimable after gift", seniorVault.claimableRedeemRequest(rid, SENIOR_LP));
        assertGt(seniorVault.claimableRedeemRequest(rid, SENIOR_LP), 0, "still frozen");
    }

}
```

Result (actually run, `forge test --root packages/perps --match-path test/break_campaign1_assetflow/PoC_RecapResidualFreeze.t.sol -vv`, same test body, re-run on a clean tree during verification with identical numbers):

```
Ran 1 test for test/break_campaign1_assetflow/PoC_RecapResidualFreeze.t.sol:PoCRecapResidualFreeze
[PASS] test_PoC_recapResidualFreezesLpRedemptions() (gas: 34267186)
Logs:
  pool assets / L after book 152000000188 135000000000
  degraded / pool assets true 77774300617
  recap 57226699383
  pool cash (all remaining LP equity) 13603848432
  pendingRecapitalizationAssets 57226699383
  settlement attempts reverted (of 168) 168
  owner gift needed to unlock 15010501259
  LP claimable after gift 3562284879

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 53.80ms (45.65ms CPU time)
```

