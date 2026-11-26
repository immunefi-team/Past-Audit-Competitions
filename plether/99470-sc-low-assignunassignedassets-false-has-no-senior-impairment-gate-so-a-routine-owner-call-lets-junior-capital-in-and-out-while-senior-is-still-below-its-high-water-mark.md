# \#99470 [SC-Low] `assignUnassignedAssets(false, …)` has no senior-impairment gate, so a routine owner call lets junior capital in, and out, while senior is still below its high-water mark

**Submitted on Nov 13th 2026 at 19:51:46 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99470
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The tranche waterfall promises that senior impairment is restored before junior extracts value (SECURITY.md:394; WHITEPAPER.md:628-629). Every ordinary path that lets junior capital in enforces this. The deposit gate refuses activation while `seniorPrincipal < seniorHighWaterMark` (`HousePoolTrancheGateLib.sol:36-38`). `assignUnassignedAssets(false, receiver)` is the one junior-principal increase that skips the check.

The trigger is not a malicious or careless owner. It is the obvious call in the state where it matters. After a realized trader profit, junior is wiped (`J = 0`), senior is impaired, and `unassignedAssets` (U) is intact because U is excluded from loss absorption. An owner who follows the natspec ("Prevents later LPs from implicitly capturing previously ownerless value", `HousePool.sol:656`) and sees an empty junior tranche will naturally assign U to junior. Nothing in the function, its natspec or a revert warns that senior is impaired. The function already reconciles before assigning (`:671`), so the impairment is known on-chain at the moment of the call.

### Vulnerability Details

```solidity
// packages/perps/src/HousePool.sol:660-703
function assignUnassignedAssets(bool toSenior, address receiver) external onlyOwner {
    ...
    _reconcile(ctx.accountingSnapshot);              // :671, senior impairment is now known
    uint256 amount = unassignedAssets;
    if (toSenior) {
        _requireSeniorDepositCapacity(amount, _getWaterfallState());   // senior branch is checked
    }
    ...
    if (toSenior) {
        ...
    } else {
@>      juniorPrincipal += amount;                   // :699, no seniorPrincipal >= seniorHighWaterMark check
    }
    unassignedAssets = 0;
```

Compare the ordinary junior entry gate:

```solidity
// packages/perps/src/libraries/HousePoolTrancheGateLib.sol:36-38
if (projectedSeniorPrincipal < projectedSeniorHighWaterMark) {
    return false;
}
```

All other junior-principal increases restore senior first: epoch deposits (`HousePool.sol:908-912`), the revenue waterfall, and recap routing, which states that "recapitalization never accrues to junior principal" (`HousePoolPendingPreviewLib.sol:107`). Because U usually comes from recap overflow, the assign path also bypasses that rule. (`initializeSeedPosition(false, …)` lacks the gate too, but it runs once and only before `activateTrading`, so senior cannot be impaired at that point.)

The precondition `U > 0 && S < HWM` arises from documented flows only:
1. Owner recap while senior is whole. The whole recap goes to U (`HousePoolPendingPreviewLib.sol:124-135`).
2. A trader realizes a profit. Junior is wiped and senior is impaired. U is untouched, because it is excluded from loss absorption (`HousePoolReconcilePlanLib.sol:84-88`, ACCOUNTING_SPEC.md:336).
3. The owner assigns U to the empty junior tranche, which is the routine reading of the situation.

The result is `J > 0 && S < HWM`, a state no other path produces.

### Impact Details

PoC numbers: seeds 100k/100k, recap 60k → U = 60,000. Trader profit → J = 0, S = 86,000, HWM = 100,000. Owner assigns to junior → J = 60,000, S = 86,000.

- **Junior exits before senior is restored.** The assignee redeems, and `settleLpEpoch` funds **59,987.64 USDG** while senior stays at **86,002 / 100,000**. The 14,000 USDG that the waterfall reserves for senior restoration leaves with junior.
- **Mark round trips stop being neutral.** A favorable tick restores senior first, and the return tick charges junior first. One 1.00 → 1.05 → 1.00 reconcile moves **3,000.00 USDG** J→S compared with a control. Six round trips move the whole **14,000 USDG** gap. `settleLpEpoch` is permissionless (`OrderRouter.sol:479`), so the tranche split depends on who settles at which tick.

Bound: `min(U, HWM − S)` per occurrence. No value leaves the pool. The broken property is the tranche priority invariant.

Notes for triage:
- `infoRegardingComp.md:54`: "An in-scope impact from a logic error that a privileged address can trigger remains valid." This is a missing check that every sibling junior-entry path has, triggered by a legitimate call. It is not a request to trust a compromised owner.
- The PoC uses the repo's `BasePerpTest` config (`maxSeniorShareBps = 9999`). With a tighter production share cap, `juniorWithdrawalRatioCap` (`HousePoolSeniorCapacityLib.sol:64-90`) holds back part of J and shrinks the exit variant. The round-trip variant is not affected by the cap.

### Mitigation

In the junior branch, route `min(amount, seniorHighWaterMark − seniorPrincipal)` to `seniorPrincipal` first, as recap routing does, and add only the remainder to `juniorPrincipal`. Alternatively, revert while `seniorPrincipal < seniorHighWaterMark`, mirroring `HousePoolTrancheGateLib.sol:36-38`.

### References

- `packages/perps/src/HousePool.sol:653-659` — natspec, no impairment warning
- `packages/perps/src/HousePool.sol:671` — reconcile before assignment
- `packages/perps/src/HousePool.sol:699` — ungated `juniorPrincipal += amount`
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:36-38` — impairment gate on ordinary junior entry
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159` — senior restored before junior
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:124-135` — recap overflow → U
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:84-88` — U excluded from loss absorption
- `SECURITY.md:394`, `WHITEPAPER.md:628-629` — restore-before-junior promise

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/poc/AssignJuniorImpairedRatchet.t.sol`. It uses the repo's `BasePerpTest` fixture. Run:

```
forge test --root packages/perps --match-path test/perps/poc/AssignJuniorImpairedRatchet.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// assignUnassignedAssets(junior) while senior is impaired leaves J > 0 with S < HWM.
contract AssignJuniorImpairedRatchetTest is BasePerpTest {

    address internal traderA = address(0xA11CE);
    address internal traderB = address(0xB0B0);
    address internal assignee = address(0xA551);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _mark(uint256 price) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

    function _rec() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _log(string memory tag) internal view {
        console2.log(tag);
        console2.log("  senior   ", pool.seniorPrincipal());
        console2.log("  seniorHWM", pool.seniorHighWaterMark());
        console2.log("  junior   ", pool.juniorPrincipal());
        console2.log("  unassign ", pool.unassignedAssets());
    }

    /// U > 0 from an over-recap, then a realized trader profit impairs senior (J wiped), then owner assigns U to junior.
    function _setup() internal {
        _fundTrader(traderA, 100_000e6);
        _fundTrader(traderB, 100_000e6);
        // Owner recap while senior is whole: all of it becomes unassigned (no gap).
        settlementToken.mint(address(this), 60_000e6);
        settlementToken.approve(address(engine), 60_000e6);
        engine.recapitalizePool(60_000e6);
        _mark(1e8);
        _rec();
        assertEq(pool.unassignedAssets(), 60_000e6, "U from recap overflow");
        _open(traderA, CfdTypes.Side.LONG, 120_000e18, 20_000e6, 1e8);
        _open(traderB, CfdTypes.Side.LONG, 60_000e18, 20_000e6, 1e8);
        _rec();

        // Trader A realizes a large profit: junior wiped, senior impaired, U excluded from the loss.
        _mark(0.05e8);
        _close(traderA, CfdTypes.Side.LONG, 120_000e18, 0.05e8);
        _mark(1e8);
        _rec();
        _log("after realized loss");
        assertEq(pool.juniorPrincipal(), 0, "junior wiped");
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior impaired");

        // Routine owner action: refill the empty junior tranche with the ownerless assets. No impairment gate.
        pool.assignUnassignedAssets(false, assignee);
        _log("after assign to junior");
        assertGt(pool.juniorPrincipal(), 0);
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "J > 0 while S < HWM");
    }

    /// Exit: the assignee redeems the assigned value while senior stays impaired.
    function test_AssignJuniorWhileImpaired_JuniorExitsBeforeSeniorRestored() public {
        _setup();
        uint256 shares = juniorVault.balanceOf(assignee);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // deposit cooldown on bootstrap mint
        vm.prank(assignee);
        uint256 id = juniorVault.requestRedeem(shares, assignee, assignee);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _mark(1e8);
        vm.prank(address(router));
        IHousePool.LpEpochSettlementResult memory r = pool.settleLpEpoch(1e8, vm.getBlockTimestamp());
        _log("after junior redemption settled at 1e8");
        console2.log("junior funded assets", r.juniorFundedAssets);
        console2.log("claimable", juniorVault.claimableRedeemAssets(id, assignee));
        assertGt(r.juniorFundedAssets, 0, "junior extracted while senior impaired");
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior still impaired");
    }

    /// One favorable tick and back moves value J -> S compared with a control.
    function test_AssignJuniorWhileImpaired_MarkRoundTripMovesJuniorToSenior() public {
        _setup();
        uint256 snap = vm.snapshotState();

        vm.warp(vm.getBlockTimestamp() + 600);
        _mark(1e8);
        _rec();
        uint256 cS = pool.seniorPrincipal();
        uint256 cJ = pool.juniorPrincipal();
        uint256 cU = pool.unassignedAssets();

        vm.revertToState(snap);
        vm.warp(vm.getBlockTimestamp() + 300);
        _mark(1.05e8);
        _rec();
        vm.warp(vm.getBlockTimestamp() + 298);
        _mark(1e8);
        _rec();

        uint256 sGain = pool.seniorPrincipal() - cS;
        uint256 jLoss = cJ - pool.juniorPrincipal();
        console2.log("senior gained vs control", sGain);
        console2.log("junior lost vs control  ", jLoss);
        assertEq(pool.unassignedAssets(), cU);
        assertApproxEqAbs(sGain, 3000e6, 10e6, "favorable tick value transferred to senior");
        assertApproxEqAbs(jLoss, sGain, 2, "1:1 from junior");
    }

    /// Repeated mark noise drives the full senior gap out of the junior assignment.
    function test_AssignJuniorWhileImpaired_RepeatedTicksTransferWholeGap() public {
        _setup();
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        uint256 j0 = pool.juniorPrincipal();
        for (uint256 i; i < 6; ++i) {
            vm.warp(vm.getBlockTimestamp() + 60);
            _mark(1.05e8);
            _rec();
            vm.warp(vm.getBlockTimestamp() + 60);
            _mark(1e8);
            _rec();
        }
        console2.log("initial gap        ", gap);
        console2.log("junior lost        ", j0 - pool.juniorPrincipal());
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior fully restored by noise");
        assertApproxEqAbs(j0 - pool.juniorPrincipal(), gap, 50e6, "junior paid the whole gap (plus coupon)");
    }
}
```

Result (key lines):

```
after realized loss       senior 86000000085  seniorHWM 100000000760  junior 0            unassign 60000000000
after assign to junior    senior 86000000085  seniorHWM 100000000760  junior 60000000000  unassign 0

[PASS] test_AssignJuniorWhileImpaired_JuniorExitsBeforeSeniorRestored()
  junior funded assets 59987642399      senior 86002356685 < HWM 100000000760
[PASS] test_AssignJuniorWhileImpaired_MarkRoundTripMovesJuniorToSenior()
  senior gained vs control 3000002057   junior lost vs control 3000002057
[PASS] test_AssignJuniorWhileImpaired_RepeatedTicksTransferWholeGap()
  initial gap 14000000675               junior lost 14000047097   (S == HWM)

Suite result: ok. 3 passed; 0 failed; 0 skipped
```

