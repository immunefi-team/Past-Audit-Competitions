# \#99889 [SC-Low] The MED-28 fix makes stop-loss/take-profit arming revert after any order execution, so any account can block every trader's protection at no cost

**Submitted on Nov 15th 2026 at 16:33:27 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99889
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

**Severity: Medium.** Impact selected from the program's list: "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/PositionProtectionBook.sol` (`_freshProtectionObservation`, lines 862-886), reaching `packages/perps/src/PletherOracle.sol` (`_snapshotFromBasket`, lines 892-896).

**Relation to prior findings:** a new bug introduced by the fix for Cecuro **MED-28** (the program's known-issues rule makes a report on a fixed root valid when it shows the fix "is incomplete or introduces a new bug"). It is not MED-28, MED-17, LOW-73 or the SECURITY.md keeper limitation; the section "Relation to prior findings" explains each.

## Summary

To fix Cecuro MED-28, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` now validate the trader's trigger prices against "the current accepted oracle observation". They do this by calling `oracle.getLatestPrice(PriceMode.MarkRefresh)` (`PositionProtectionBook.sol:880`). That view enforces publish order: it reverts `PletherOracle__PriceOutOfOrder` whenever the accepted live basket is older than the Engine mark (`PletherOracle.sol:892-896`).

Ordinary order execution puts the system in exactly that state. A keeper's `executeOrder` pushes the order's historical Pyth tick into the Engine mark (`OrderRouterExecutionSidecar.sol:979`, `:1053-1061`) but never writes the accepted basket. From then until someone pays for a fresh `updateMarkPrice`, **every trader's protection create, replace and open-with-protection reverts**, even though the oracle is live, the Engine mark is fresh, and the MED-28 comment and the Book's own predicate `_acceptedMarkInstalled()` (`PositionProtectionBook.sol:888-895`) both accept this state. The other arming path in the same contract, the one that arms protection when a protected open fills (`PositionProtectionBook.sol:598-602`), accepts the state and arms from `ENGINE.lastMarkTime()`.

Anyone can cause the state on purpose and at no asset cost. An account with no position commits an order whose slippage bound cannot be met and executes it itself. The order fails on slippage, but its tick has already advanced the Engine mark (the oracle step runs before settlement), and the execution bounty comes back to the same account as executor. The PoC repeats this after each of five refreshes by the victim and every arm attempt is refused, at zero settlement-asset cost to the account causing it.

The damage is to traders who rely on the protection. In the PoC, a trader whose one attempt to arm a 1.10 stop-loss is refused is liquidated at 1.19 instead of being stopped out at 1.10, and ends **905.400002 USDC** worse off than the same trader in the same market whose arm was accepted. The LP pool ends 905.000002 USDC better off.

The same call also produces a second symptom: for up to 15 seconds after each refresh, arming reverts `PletherOracle__StalePrice` while the Engine mark is still fresh (section "Second symptom").

The limits are stated plainly in the Impact section: the refusal is a visible revert, one refresh in the same second clears it, and closing a position is never blocked.

## Relation to prior findings

**Cecuro MED-28** (Acknowledged, "Addressed"), "Trader can arm a new protection from a stale engine mark after oracle state already crossed it". The maintainers' comment on the fix: "New protection, replacement and [open-with-protection] staging validate trigger geometry against the current accepted oracle observation, rather than only the cached Engine mark. … The integrated synchronization guard also rejects arming **when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark**." MED-28 was the case where the *oracle is ahead* of a stale Engine mark. This report is the opposite state, where the *Engine mark is ahead* of the accepted basket. The fix's own policy allows that state, and the fix itself causes the revert:

1. The `getLatestPrice(MarkRefresh)` call at `PositionProtectionBook.sol:880` is the MED-28 remediation (MED-28 p. 66: "read `pletherOracle.getLatestPrice(PriceMode.MarkRefresh)` … during protection creation and replacement"). It arrived with the protection hardening commit `eadc5c58` (#124), after the audited code.
2. `_acceptedMarkInstalled()` (`PositionProtectionBook.sol:888-895`), the synchronization predicate the MED-28 comment refers to (commit `81919218`, #120), returns `true` for `acceptedTime < markTime`. On create/replace that branch can never be reached, because the line before it has already reverted. The open-with-protection fill path (`:598-602`) does reach it on every fill.
3. The repository documents only the reverse failure. `WORKING_WITH_AI_AGENTS.md:270-272` says protection fails "If the accepted live basket is newer than the Engine mark or disagrees at the same timestamp". No maintainer test expects `PriceOutOfOrder` from the protection Book; the `OracleMarkAhead` tests in `PositionProtection.t.sol` cover only the oracle-ahead case.
4. `SECURITY.md:563-564` states the property this breaks: "Existing-position protection creation remains available in degraded mode because it adds no exposure. Router pause, frozen-oracle policy, freshness, position/queue checks, and post-lock collateral admission still apply." Publish order relative to the Engine mark is not in that list.

Other nearby items, and why they are different:
- **MED-17** (trigger publish time versus the `armedAt` wall clock) says creation and replacement "use the accepted current oracle observation". It does not describe or accept a refusal.
- **LOW-73** (create reverted while the old `degradedMode` latch was set) has the same user-facing symptom but a different root (the latch, since removed).
- **MED-18** (the cached mark rewritten at the same timestamp) has no protection-arming consequence.
- **MED-30** (known issue: no automatic stop-loss trigger while the oracle is frozen) needs a frozen market. This report needs a live market and concerns create/replace, not the trigger. In a frozen market `_freshProtectionObservation` reverts `ConditionalTriggerFrozen` before reaching the oracle call.
- **SECURITY.md:1341** ("execution remains dependent on fresh keeper infrastructure and oracle publication outside frozen windows") is about *execution*. Arming is not execution, and the refusal happens while the accepted basket is inside its staleness limit and the Engine mark is fresher still.
- **ORACLE_SYNCHRONIZATION.md:149-151** acknowledges the state ("historical execution may advance the Engine without advancing the accepted live basket") but never says that create/replace are refused in it.
- AI-scan info item "Later mark refresh can make live historical-price orders permanently out of order" runs the other way (a refresh blocks a historical *order*). Here a historical *execution* blocks *protection arming*.

## Vulnerability details

### The refused state

`_freshProtectionObservation()` (`PositionProtectionBook.sol:862-886`):

```solidity
        uint64 markTime = ENGINE.lastMarkTime();
        cachedMarkPrice = ENGINE.lastMarkPrice();
        if (
            cachedMarkPrice == 0 || markTime == 0
                || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
        ) {
            revert OrderRouter__ProtectionMarkTooStale();
        }
        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
        currentMarkPrice = current.markPrice;
        currentPublishTime = current.publishTime;
        if (!_acceptedMarkInstalled()) {
            revert PositionProtectionBook__OracleMarkAhead();
        }
```

`getLatestPrice(MarkRefresh)` builds the snapshot from the stored accepted basket (`_acceptedLiveBasket`, `PletherOracle.sol:706-727`) and calls `_snapshotFromBasket(basket, policy, true)` (`PletherOracle.sol:578`). With `enforcePublishOrder = true` (`PletherOracle.sol:892-896`):

```solidity
        if (enforcePublishOrder) {
            uint64 markTime = engine.lastMarkTime();
            if (snapshot.publishTime < markTime) {
                revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
            }
```

The predicate checked one line later says this state is fine (`PositionProtectionBook.sol:888-895`):

```solidity
    function _acceptedMarkInstalled() private view returns (bool) {
        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
        if (!accepted) {
            return true;
        }
        uint64 markTime = ENGINE.lastMarkTime();
        return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
    }
```

### How ordinary execution reaches it

`executeOrder` → `_prepareSingleOracle` validates the order's historical tick (the first Pyth update after the order's commit time) and then calls `_updateEngineMark` (`OrderRouterExecutionSidecar.sol:979`), which writes it into the Engine whenever it is newer (`:1053-1061`):

```solidity
    function _updateEngineMark(
        IOrderRouterExecutionHost host,
        OracleResult memory result
    ) private {
        ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
        if (result.publishTime > engine_.lastMarkTime()) {
            engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
        }
    }
```

The accepted basket is written only by a live `updateMarkPrice` intake, so after any executed (or failed) order whose tick is newer than the last refresh, `acceptedBasket.publishTime < engine.lastMarkTime()`, and protection arming reverts for every account. Because `executeOrder` has no executor allow-list (the executor is `msg.sender`, `OrderRouterExecutionSidecar.sol:129-139`), any address can create this state. The slippage check runs after the oracle step (`OrderRouterExecutionSidecar.sol:467-473`), so even an order that fails on its own slippage bound moves the Engine mark.

### Second symptom (same call): `StalePrice` while the mark is fresh

The same `getLatestPrice(MarkRefresh)` call also checks every basket *component* against the 60 s MarkRefresh staleness limit (`PletherOracle.sol:720-722`), while the Book's own freshness check tests only the Engine mark time (`PositionProtectionBook.sol:874-879`), which is the *newest* component. Live intake allows components to differ by up to `LIVE_TICK_LOOKBACK` = 15 s (`PletherOracle.sol:29`, `:762-764`). So after each refresh there is a window of up to 15 s (oldest component + 60 s < now ≤ newest component + 60 s) in which create/replace/open-with-protection revert `PletherOracle__StalePrice` although the mark check passes. The PoC shows a 12 s window with ordinary skew. It is the same line and the same fix as above, so it is reported here as a second symptom, not as a separate finding.

## Impact

**Who:** any address. No role, no position and no market exposure are needed. Ordinary keeper execution of other traders' orders also reaches the state without anyone intending it.

**What breaks:** `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert for every trader while the oracle is live and the Engine mark is fresh. The PoC shows a trader cannot arm a stop on an open position, cannot tighten an existing stop, and cannot open with protection.

**Cost to the account causing it:** zero settlement assets per cycle. A self-executed order that fails on slippage pays its execution bounty back to the same account (asserted in the PoC; gas only).

**Damage to users (measured):** in the PoC market (10,000-notional LONG, 2,000 USDC margin), a trader who tries once to arm a 1.10 stop-loss, is refused, and does not retry is liquidated at 1.19 and ends **905.400002 USDC** worse off than the same trader whose arm was accepted. The LP pool ends 905.000002 USDC better off in the same comparison. Two more cases in the second test file, each blocked by an account with no position: a stop at 1.05 that could not be armed before a fast move to 1.20 (the trader is 1,386.36 USDC worse off than with the stop), and a refused *tightening* of an existing 1.10 stop to 1.05 (the trader is stopped out at the old level, 400.16 USDC worse off).

This matches the program's Medium impact "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": the account causing it gains nothing, and the damage lands on traders who rely on the protection the protocol says stays available (`SECURITY.md:563-564`).

**Limits, stated plainly:**
- The refusal is a visible revert, not a silent failure. A trader who notices it can retry.
- One `updateMarkPrice` with a current Pyth update clears it. A refresh and an arm in the same second cannot be split by another account: the only tick that account can use is the first one after its own commit, and if that is not newer than the refresh, the Engine mark does not move (PoC `test_A_Control_SameSecondRefreshThenArmCannotBeInterleaved`). Repeated blocking therefore works against a victim whose refresh and arm land in different seconds, for example two ordinary transactions from an EOA.
- Exits are never blocked. A plain close commits and executes in the blocked state, and already-armed protection still triggers.
- The loss figures are counterfactuals: they assume the trader does not retry, refresh or close by hand while the price moves against them (19% in the 905.40 USDC case).


## Recommendation

Make create/replace/open-with-protection accept exactly the state the MED-28 policy and `_acceptedMarkInstalled()` accept. When the accepted basket is older than, or equal to (same price), the Engine mark, the freshness-checked Engine mark *is* the newest accepted observation. Use it, as the open-with-protection fill path already does (`PositionProtectionBook.sol:598-602` arms from `ENGINE.lastMarkTime()` once `_acceptedMarkInstalled()` passes), and keep the MED-28 refusal when the accepted basket is ahead or disagrees:

```solidity
    function _freshProtectionObservation()
        private
        view
        returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime)
    {
        IPletherOracle oracle = _activeOracle();
        IPletherOracle.PolicySnapshot memory policy = oracle.getOrderExecutionPolicy(true);
        if (policy.oracleFrozen) {
            revert OrderRouter__ConditionalTriggerFrozen();
        }
        uint64 markTime = ENGINE.lastMarkTime();
        cachedMarkPrice = ENGINE.lastMarkPrice();
        if (
            cachedMarkPrice == 0 || markTime == 0
                || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
        ) {
            revert OrderRouter__ProtectionMarkTooStale();
        }
        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
        if (!accepted) {
            revert IPletherOracle.PletherOracle__NoAcceptedBasket(); // unchanged outcome before the first intake
        }
        // MED-28: refuse when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark.
        if (acceptedTime > markTime || (acceptedTime == markTime && acceptedPrice != cachedMarkPrice)) {
            revert PositionProtectionBook__OracleMarkAhead();
        }
        // Accepted basket older than (historical execution advanced the Engine) or equal to the Engine mark:
        // the freshness-checked Engine mark is the current accepted observation.
        currentMarkPrice = cachedMarkPrice;
        currentPublishTime = markTime;
    }
```

This removes the `getLatestPrice(MarkRefresh)` call, so it fixes both symptoms: the publish-order revert, and the per-component `StalePrice` revert while the mark itself is fresh. The trigger path is unchanged and still ingests and validates a fresh Pyth update when the protection fires (`triggerPositionProtection` dispatches `updateMarkPrice`, `PositionProtectionBook.sol:375-378`, which reaches `oracle.updatePrice(…, MarkRefresh)` in `router/OrderLiquidationBatchLogic.sol:607-609`).

If the maintainers want to keep the component-level re-validation at arming, the narrower alternative is to call `getLatestPrice(MarkRefresh)` only when `acceptedTime == markTime`, and to use the Engine mark when `acceptedTime < markTime`. That fixes the main symptom but not the second one.

Either way, a regression test like `test_Property_RouterPath_TraderCanArmProtectionAfterKeeperExecution` above (arming after another trader's keeper execution) would catch this case; the current suite only tests the oracle-ahead direction.


## Link to Proof of Concept

https://gist.github.com/Richter2552/76e3fa5cae573b6902104c8669d89aaf

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The three test files below extend the maintainers' `PositionProtectionTest` fixture and use only public entry points from unprivileged accounts after the fixture's `setUp`: production `OrderRouter.commitOrder(OrderRequest)`, `router.executeOrder` from an address with no role, the permissionless `router.updateMarkPrice`, `HousePool.settleLpEpoch`, `TrancheVault` request/claim, `createPositionProtection`, `replacePositionProtection`, `triggerPositionProtection` and `executeLiquidationBatch`. Pyth is the maintainers' `MockPyth` (Pyth is outside the scope). No protocol contract is pranked and no storage is written. The fixture's junior deposit helper is switched off (`_initialJuniorDeposit() = 0`) because it impersonates the router; the junior LP deposits through the public vault flow instead. Because of that override, the tests inherited from `PositionProtectionTest` are not meant to run in these contracts; run only the tests named below.

Each **property** test states the behaviour the code should have and **fails** on the pinned code with the named error. Each **characterize/control** test passes and pins the exact revert, the cost, the remedy and the measured value.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card1351S2RouterPathProtectionBlocked.t.sol Card1351S2PersistenceCostValue.t.sol \
   Card1351S2ValueAndStaleWindow.t.sol packages/perps/test/perps/spec/trader/
cd packages/perps

forge test --match-contract '^Card1351S2RouterPathProtectionBlockedTest$' --match-test '^test_Property_RouterPath_' -vv
forge test --match-contract '^Card1351S2RouterPathProtectionBlockedTest$' --match-test '^test_(Control|Characterize)_RouterPath_' -vv

forge test --match-contract '^Card1351S2PersistenceCostValueTest$' --match-test '^test_((A|B|C)_|ReadDown_)' -vv

forge test --match-contract '^Card1351S2ValueAndStaleWindowTest$' --match-test '^test_(Value_|Recovery_|StaleWindow_Characterize_|StaleWindow_RemedyWithMaxSkew)' -vv
forge test --match-contract '^Card1351S2ValueAndStaleWindowTest$' --match-test '^test_StaleWindow_Property_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run on a fresh tree from `git archive a82d4fef` with the three files installed as above (forge 1.8.1, solc 0.8.35, the repository's `packages/perps/foundry.toml`). The result: 12 characterize/control tests pass, and the 3 property tests fail with the named errors. Excerpt (forge `-vv` logs):

```console
[FAIL: PletherOracle__PriceOutOfOrder(1709535618 [1.709e9], 1709535629 [1.709e9])] test_Property_RouterPath_TraderCanArmProtectionAfterKeeperExecution() (gas: 9375298)
[FAIL: PletherOracle__PriceOutOfOrder(1709535610 [1.709e9], 1709535612 [1.709e9])] test_Property_RouterPath_TraderCanArmProtectionAfterOwnOpen() (gas: 5662967)
[PASS] test_Characterize_RouterPath_RevertAndRemedy() (gas: 10108091)
Logs:
  accepted basket publishTime 1709535618
  engine lastMarkTime         1709535629
[PASS] test_Control_RouterPath_ArmSucceedsWhenAcceptedBasketCurrent() (gas: 6060569)

[PASS] test_A_FailedSelfExecutedOrderBlocksArmingAtZeroAssetCost() (gas: 8013255)
Logs:
  MALLORY settlement delta per cycle (USDC 1e6): 0
  gas: commit 1370878
  gas: execute 654070
[PASS] test_A_GriefLoop_ReblocksEveryRefreshThenArmAcrossSeconds() (gas: 19524625)
Logs:
  rounds, all arm attempts refused: 5
  victim refresh gas total: 1160944
  griefer gas total: 10124936
  griefer settlement delta total: 0
[PASS] test_A_Control_SameSecondRefreshThenArmCannotBeInterleaved() (gas: 8523669)
[PASS] test_B_FastMarket_BlockedStopLossDifferential() (gas: 13927739)
Logs:
  ALICE settlement, control (stop at 1.05, filled 1.06): 19391360000
  ALICE settlement, griefed (no stop, closed 1.20):      18005000000
  difference (USDC 1e6): 1386360000
[PASS] test_C_TighteningExistingStopIsBlocked_Differential() (gas: 13930521)
Logs:
  ALICE settlement, control (tightened to 1.05): 19391360000
  ALICE settlement, griefed (old 1.10 stop):     18991199999
  difference (USDC 1e6): 400160001
[PASS] test_ReadDown_VictimCanExitOrSelfClear() (gas: 11011837)

[PASS] test_Value_RefusedStopLossCostsTheTrader() (gas: 40578379)
Logs:
  world control (stop armed)
    ALICE armed / stop fired / liquidated true true false
    liquidation price (1e8) 119000000
    ALICE balance (USDC 1e6) 18991199996
    CAROL balance (USDC 1e6) 18085799994
    pool totalAssets (USDC 1e6) 1004905000010
  world refused (arm reverted, no retry)
    ALICE armed / stop fired / liquidated false false true
    liquidation price (1e8) 119000000
    ALICE balance (USDC 1e6) 18085799994
    CAROL balance (USDC 1e6) 18085799994
    pool totalAssets (USDC 1e6) 1005810000012
  ALICE loss vs control (USDC 1e6) 905400002
  pool gain vs control (USDC 1e6) 905000002
[PASS] test_Recovery_RefreshThenArm() (gas: 15639642)
[PASS] test_StaleWindow_Characterize_WindowAndRemedy() (gas: 11058835)
Logs:
  stale-component window seconds 12
[PASS] test_StaleWindow_RemedyWithMaxSkewStillArms() (gas: 13021389)
[FAIL: PletherOracle__StalePrice(1, 0x0000000000000000000000000000000000000000000000000000000000000001, 1709535629 [1.709e9], 60, 1709535690 [1.709e9])] test_StaleWindow_Property_ArmWhileMarkFresh() (gas: 9504169)
```

Expected output on a fixed contract: the three property tests pass. The characterize tests that pin the revert (`test_Characterize_RouterPath_RevertAndRemedy`, `test_A_*`, `test_StaleWindow_Characterize_*`) then fail at their `expectRevert`, because the revert they pin no longer happens.

### What the tests show

| step | test | result on the pinned code |
|---|---|---|
| ALICE opens through `commitOrder` + keeper `executeOrder`; BOB's ordinary order is then keeper-executed with the next Pyth tick; ALICE tries to arm a take-profit | `test_Property_RouterPath_TraderCanArmProtectionAfterKeeperExecution` | **FAILS**: `PletherOracle__PriceOutOfOrder(accepted, mark)` |
| ALICE tries to arm right after her own open is keeper-executed | `test_Property_RouterPath_TraderCanArmProtectionAfterOwnOpen` | **FAILS**: `PletherOracle__PriceOutOfOrder` |
| exact revert, no bounty held, one public refresh clears it | `test_Characterize_RouterPath_RevertAndRemedy` | passes |
| with the accepted basket current, the same create succeeds | `test_Control_RouterPath_ArmSucceedsWhenAcceptedBasketCurrent` | passes |
| an account with no position self-executes an order that fails on slippage; the Engine mark moves; arming is refused; the account's settlement balance is unchanged | `test_A_FailedSelfExecutedOrderBlocksArmingAtZeroAssetCost` | passes |
| the same after each of five refreshes by the victim; all five arm attempts refused; zero settlement-asset cost | `test_A_GriefLoop_ReblocksEveryRefreshThenArmAcrossSeconds` | passes |
| limit: a refresh and an arm in the same second cannot be split | `test_A_Control_SameSecondRefreshThenArmCannotBeInterleaved` | passes |
| a refused stop in a fast market, and a refused tightening of an existing stop, compared with the accepted case | `test_B_FastMarket_BlockedStopLossDifferential`, `test_C_TighteningExistingStopIsBlocked_Differential` | passes (control balance > refused balance) |
| limit: in the blocked state a manual close works, and a self-refresh clears it | `test_ReadDown_VictimCanExitOrSelfClear` | passes |
| control world vs refused world, same market: ALICE ends 905.400002 USDC worse, the pool 905.000002 USDC better, CAROL identical | `test_Value_RefusedStopLossCostsTheTrader` | passes |
| one public refresh lets ALICE arm, and her stop then fires at 1.10 | `test_Recovery_RefreshThenArm` | passes |
| second symptom: mark 49 s old (fresh), oldest component 61 s old; ALICE arms | `test_StaleWindow_Property_ArmWhileMarkFresh` | **FAILS**: `PletherOracle__StalePrice` |
| second symptom: arm works at +60 s, refused at +61 s, a refresh clears it | `test_StaleWindow_Characterize_WindowAndRemedy` | passes |
| the remedy refresh for the main symptom, at the widest skew intake allows, still lets ALICE arm at once | `test_StaleWindow_RemedyWithMaxSkewStillArms` | passes |

### Test file 1 of 3: `packages/perps/test/perps/spec/trader/Card1351S2RouterPathProtectionBlocked.t.sol` (minimal reproduction)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title Protection arming is refused after an ordinary keeper execution (public entry points, unprivileged accounts)
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to PositionProtection.t.sol in
///         packages/perps/test/perps/spec/trader/ and run with the repository's own foundry.toml.
/// @dev `BasePerpTest._open` pranks the router into `engine.processOrderTyped`, and the fixture's junior LP funding
///      pranks the router to install a mark. This file uses neither:
///      - Deployment and owner configuration come from the maintainers' fixture (BasePerpTest.setUp). Both seeds and
///        `pool.activateTrading()` run there; the test asserts `isTradingActive()` and `canIncreaseRisk()`.
///      - `_initialJuniorDeposit()` is 0, so setUp makes no router-impersonated call. The junior LP deposits through
///        `TrancheVault.requestDeposit`, a public `router.updateMarkPrice`, the permissionless
///        `HousePool.settleLpEpoch(0, 0)` (no open positions, HousePool.sol:809-819), and `claimDeposit`.
///      - Every trade is a production V3 `OrderRouter.commitOrder(OrderRequest)` from the trader's own account. The
///        harness's scalar `commitOrder` overload is not used.
///      - Every execution is `router.executeOrder` from KEEPER, an address with no role. The executor is
///        `msg.sender` with no allow-list (OrderRouterExecutionSidecar.sol:129-139).
///      - Every mark refresh is the permissionless `router.updateMarkPrice` (OrderRouter.sol:424-434).
///      - Pyth is the maintainers' MockPyth. A keeper's update is the first tick after commitTime: the
///        `parsePriceFeedUpdatesUnique` window is [commitTime+1, ...] (PletherOracle.sol:608-614), with
///        `previousPublishTime = commitTime`. No prank, no stdstore write, and no `vm.store` touches protocol state.
contract Card1351S2RouterPathProtectionBlockedTest is PositionProtectionTest {

    address internal constant LP = address(0x1B1B);
    address internal constant KEEPER = address(0x5EE5); // no role anywhere; any EOA
    address internal constant REFRESHER = address(0x4EF4); // any EOA
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;

    uint256 private _clientNonce;

    /// No junior deposit in the fixture setUp: its helper pranks the router into `engine.updateMarkPrice`.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Public-entry-point helpers
    // ---------------------------------------------------------------------------------------------------------------

    /// A live Pyth tick at the current block time, chained to the last engine mark time.
    function _tickNow(
        uint256 price
    ) private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    /// Anyone installs a fresh accepted basket via the permissionless `router.updateMarkPrice`.
    function _publicRefresh(
        address caller,
        uint256 price
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow(price);
        vm.prank(caller);
        router.updateMarkPrice(data);
    }

    /// An LP funds junior through the vault's public request/claim flow. The epoch is settled permissionlessly.
    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount); // the LP's own USDC
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        uint256 activationTime = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _publicRefresh(lp, MARK_PRICE);
        vm.prank(lp);
        pool.settleLpEpoch(0, 0); // permissionless while no position is open (HousePool.sol:809-819)

        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "ghost: LP deposit finalized");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimable, lp, lp);
    }

    /// A trader commits through the production V3 `OrderRouter.commitOrder(OrderRequest)`.
    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card1351s2-router", trader, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1, // no slippage limit
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    /// KEEPER (no role) executes with the first Pyth tick after commitTime, as `parsePriceFeedUpdatesUnique` requires.
    function _keeperExecute(
        uint64 orderId
    ) private returns (uint64 executionPublishTime) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        vm.warp(uint256(committedAt) + 2);
        vm.roll(vm.getBlockNumber() + 1);
        executionPublishTime = committedAt + 1;
        bytes[] memory data = _mockUniquePythUpdateDataAt(MARK_PRICE, executionPublishTime, committedAt);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    /// Activated pool + public junior LP + ALICE LONG opened through commit -> keeper execute.
    function _reachAliceOpenThroughRouter() private {
        assertTrue(pool.isTradingActive(), "ghost: trading activated");
        assertTrue(pool.canIncreaseRisk(), "ghost: commit-time risk gate open");
        _publicJuniorDeposit(LP, JUNIOR_LP_ASSETS);

        vm.warp(vm.getBlockTimestamp() + 10);
        _publicRefresh(REFRESHER, MARK_PRICE);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint64 aliceOrder = _commitV3(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false);
        _keeperExecute(aliceOrder);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "ghost: ALICE holds the LONG via the router");
    }

    /// After ALICE's open, someone refreshes (the accepted basket is current). Then BOB's ordinary order is
    /// keeper-executed at the first tick after his commit. Returns the accepted and engine mark times.
    function _reachMarkAheadViaOtherTradersExecution() private returns (uint64 acceptedTime, uint64 markTime) {
        _reachAliceOpenThroughRouter();
        vm.warp(vm.getBlockTimestamp() + 5);
        _publicRefresh(REFRESHER, MARK_PRICE);
        (, , uint64 acceptedBefore) = pletherOracle.getAcceptedBasketMark();
        assertEq(acceptedBefore, engine.lastMarkTime(), "ghost: accepted basket current before BOB's execution");

        vm.warp(vm.getBlockTimestamp() + 10);
        vm.roll(vm.getBlockNumber() + 1);
        uint64 bobOrder = _commitV3(BOB, CfdTypes.Side.SHORT, POSITION_SIZE, POSITION_MARGIN_ASSETS, false);
        uint64 bobPublish = _keeperExecute(bobOrder);

        (, , acceptedTime) = pletherOracle.getAcceptedBasketMark();
        markTime = engine.lastMarkTime();
        assertEq(acceptedTime, acceptedBefore, "ghost: execution left the accepted basket untouched");
        assertEq(markTime, bobPublish, "ghost: engine mark advanced to BOB's execution tick");
        assertGt(markTime, acceptedTime, "ghost: engine mark AHEAD of accepted basket");
        assertFalse(engine.isOracleFrozen(), "ghost: live oracle policy");
        console2.log("accepted basket publishTime", acceptedTime);
        console2.log("engine lastMarkTime        ", markTime);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Tests
    // ---------------------------------------------------------------------------------------------------------------

    /// PROPERTY: after another trader's keeper execution, ALICE can arm a TP while the oracle is live.
    /// FAILS on the pinned code (PletherOracle__PriceOutOfOrder).
    function test_Property_RouterPath_TraderCanArmProtectionAfterKeeperExecution() public {
        _reachMarkAheadViaOtherTradersExecution();
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        assertGt(protectionId, 0, "protection armed");
    }

    /// PROPERTY, ALICE's own open: right after her open is keeper-executed, she can arm a TP. FAILS on the pinned code.
    /// Her order is committed after a refresh, so its first tick (commitTime+1) is newer than the accepted basket.
    function test_Property_RouterPath_TraderCanArmProtectionAfterOwnOpen() public {
        _reachAliceOpenThroughRouter();
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        console2.log("accepted / mark after own open", acceptedTime, engine.lastMarkTime());
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        assertGt(protectionId, 0, "protection armed");
    }

    /// CONTROL: with the accepted basket current, the same create succeeds. The block comes from the execution.
    function test_Control_RouterPath_ArmSucceedsWhenAcceptedBasketCurrent() public {
        _reachAliceOpenThroughRouter();
        vm.warp(vm.getBlockTimestamp() + 5);
        _publicRefresh(REFRESHER, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        assertGt(protectionId, 0, "control: protection arms with a current accepted basket");
    }

    /// CHARACTERIZE: the exact revert, that no bounty stays held, and that one public refresh clears it.
    function test_Characterize_RouterPath_RevertAndRemedy() public {
        (uint64 acceptedTime, uint64 markTime) = _reachMarkAheadViaOtherTradersExecution();

        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        assertEq(router.getAccountReservations(ALICE).executionBountyAssets, 0, "failed create holds no bounty");

        // Remedy: ALICE (or anyone) refreshes with a current tick through the public router entry point.
        vm.warp(vm.getBlockTimestamp() + 1);
        _publicRefresh(ALICE, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        assertGt(protectionId, 0, "one public refresh clears the block");
    }

}
```

### Test file 3 of 3: `packages/perps/test/perps/spec/trader/Card1351S2ValueAndStaleWindow.t.sol` (measured loss, recovery, second symptom)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title What a refused stop-loss costs a trader, and the second symptom (StalePrice while the mark is fresh)
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to PositionProtection.t.sol in
///         packages/perps/test/perps/spec/trader/ and run with the repository's own foundry.toml.
/// @dev One market, two worlds that differ only in the ORDER of two ordinary events before ALICE arms her stop:
///        - control: BOB's ordinary order is keeper-executed, THEN someone refreshes. ALICE arms a 1.10 stop.
///        - refused: someone refreshes, THEN BOB's ordinary order is keeper-executed. ALICE's arm reverts
///          PletherOracle__PriceOutOfOrder.
///      Same price path afterwards in both worlds: 1.00 -> 1.10 (ALICE's stop level) -> first liquidatable price.
///      ALICE tries to arm ONCE and does not retry; that is the counterfactual being measured.
///      LP funds junior publicly; ALICE and CAROL open the same LONG through `commitOrder` + permissionless keeper
///      `executeOrder`; DAVE's ordinary SHORT is executed in both worlds. No router-impersonating helper is used after
///      the fixture setUp, and no protocol contract is pranked.
contract Card1351S2ValueAndStaleWindowTest is PositionProtectionTest {
    address internal constant LP = address(0x1B1B);
    address internal constant KEEPER = address(0x5EE5); // any EOA, no role
    address internal constant LIQ_KEEPER = address(0x11CC); // any EOA, no role
    address internal constant REFRESHER = address(0x4EF4); // any EOA
    address internal constant DAVE = address(0xDA7E);
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant STOP_PRICE = 110_000_000; // LONG stop-loss sits above the mark (LONG loses as price rises)

    uint256 private _clientNonce;

    struct Measure {
        bool aliceArmed;
        bytes4 aliceArmRevert;
        bool aliceStopFired;
        bool aliceLiquidated;
        bool carolLiquidated;
        uint256 liqPrice;
        uint256 aliceBal;
        uint256 carolBal;
        uint256 poolAssets;
    }

    Measure internal m;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // -------------------------------------------------------------------------------------------------------------
    // Public-entry-point helpers
    // -------------------------------------------------------------------------------------------------------------

    function _tickAt(
        uint256 price,
        uint256 publishTime
    ) private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _skewedTick(
        uint256 price,
        uint256 oldest,
        uint256 newest
    ) private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            uint256 t = i == 0 ? oldest : newest;
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, t, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _nextBlock(
        uint256 dt
    ) private {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _publicRefresh(
        address caller,
        uint256 price
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickAt(price, vm.getBlockTimestamp());
        vm.prank(caller);
        router.updateMarkPrice(data);
    }

    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activationTime = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _publicRefresh(lp, MARK_PRICE);
        vm.prank(lp);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("caper1351s2", trader, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    /// KEEPER executes with the first tick after commitTime (historical execution), at `price`.
    function _keeperExecute(
        uint64 orderId,
        uint256 price
    ) private {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        if (vm.getBlockTimestamp() < uint256(committedAt) + 2) {
            vm.warp(uint256(committedAt) + 2);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockUniquePythUpdateDataAt(price, committedAt + 1, committedAt);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    function _commitAndExecute(
        address trader,
        CfdTypes.Side side,
        uint256 price
    ) private {
        _nextBlock(1);
        uint64 id = _commitV3(trader, side, POSITION_SIZE, POSITION_MARGIN_ASSETS, false);
        _keeperExecute(id, price);
    }

    // -------------------------------------------------------------------------------------------------------------
    // The one scenario, parameterised by the order of BOB's execution and the refresh
    // -------------------------------------------------------------------------------------------------------------

    function _market() private {
        assertTrue(pool.isTradingActive(), "ghost: trading active");
        _fundTrader(DAVE, 20_000e6);
        _publicJuniorDeposit(LP, JUNIOR_LP_ASSETS);
        _nextBlock(10);
        _publicRefresh(REFRESHER, MARK_PRICE);
        _commitAndExecute(ALICE, CfdTypes.Side.LONG, MARK_PRICE);
        _commitAndExecute(CAROL, CfdTypes.Side.LONG, MARK_PRICE);
        _nextBlock(5);
        _publicRefresh(REFRESHER, MARK_PRICE);
    }

    function _armPhase(
        bool refusedWorld
    ) private {
        if (refusedWorld) {
            // refresh (in _market), THEN BOB's ordinary execution: the engine mark is now ahead of the accepted basket
            _commitAndExecute(BOB, CfdTypes.Side.SHORT, MARK_PRICE);
            (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
            assertGt(engine.lastMarkTime(), acceptedTime, "ghost: engine mark ahead of accepted basket");
        } else {
            // BOB's ordinary execution, THEN a refresh: the accepted basket is current
            _commitAndExecute(BOB, CfdTypes.Side.SHORT, MARK_PRICE);
            _nextBlock(1);
            _publicRefresh(REFRESHER, MARK_PRICE);
        }
        _nextBlock(1);
        vm.prank(ALICE);
        try protectionActions.createPositionProtection(_params(0, STOP_PRICE)) returns (uint64) {
            m.aliceArmed = true;
        } catch (bytes memory reason) {
            m.aliceArmRevert = bytes4(reason);
        }
    }

    function _stopPhase() private {
        _nextBlock(10);
        _publicRefresh(REFRESHER, STOP_PRICE);
        if (m.aliceArmed) {
            uint64 pid = protectionViews.activePositionProtectionId(ALICE);
            _nextBlock(1);
            bytes[] memory data = _tickAt(STOP_PRICE, vm.getBlockTimestamp());
            vm.prank(TRIGGER_KEEPER);
            uint64 linked = protectionActions.triggerPositionProtection(pid, data);
            _keeperExecute(linked, STOP_PRICE);
            (uint256 size,,,,,,) = engine.positions(ALICE);
            m.aliceStopFired = size == 0;
        }
    }

    function _firstLiquidatablePrice() private view returns (uint256 p) {
        uint256 depth = pool.totalAssets();
        for (p = STOP_PRICE; p <= 200_000_000; p += 1_000_000) {
            if (engineLens.isLiquidatableAt(CAROL, p, depth)) return p;
        }
        revert("no liquidation price in range");
    }

    function _openAccounts() private view returns (address[] memory accts) {
        (uint256 aSize,,,,,,) = engine.positions(ALICE);
        accts = new address[](aSize == 0 ? 1 : 2);
        accts[0] = CAROL;
        if (aSize != 0) accts[1] = ALICE;
    }

    /// The price reaches the first liquidatable level. DAVE's ordinary order executes, a refresh follows, and a
    /// liquidation keeper batches every underwater account with a current tick.
    function _liqPhase() private {
        m.liqPrice = _firstLiquidatablePrice();
        _nextBlock(10);
        uint64 daveOrder = _commitV3(DAVE, CfdTypes.Side.SHORT, POSITION_SIZE, POSITION_MARGIN_ASSETS, false);
        _keeperExecute(daveOrder, STOP_PRICE);
        _nextBlock(1);
        _publicRefresh(REFRESHER, STOP_PRICE);
        _nextBlock(1);
        uint256 nowTs = vm.getBlockTimestamp();
        assertGt(nowTs, engine.lastMarkTime(), "ghost: current tick strictly newer than the engine mark");
        bytes[] memory current = _tickAt(m.liqPrice, nowTs);
        vm.prank(LIQ_KEEPER);
        router.executeLiquidationBatch(_openAccounts(), current);
        (uint256 cSize,,,,,,) = engine.positions(CAROL);
        (uint256 aSize,,,,,,) = engine.positions(ALICE);
        m.carolLiquidated = cSize == 0;
        m.aliceLiquidated = !m.aliceStopFired && aSize == 0;
    }

    function _run(
        string memory world,
        bool refusedWorld
    ) private {
        _market();
        _armPhase(refusedWorld);
        _stopPhase();
        _liqPhase();
        m.aliceBal = clearinghouse.balanceAssets(ALICE);
        m.carolBal = clearinghouse.balanceAssets(CAROL);
        m.poolAssets = pool.totalAssets();
        console2.log("world", world);
        console2.log("  ALICE armed / stop fired / liquidated", m.aliceArmed, m.aliceStopFired, m.aliceLiquidated);
        console2.log("  liquidation price (1e8)", m.liqPrice);
        console2.log("  ALICE balance (USDC 1e6)", m.aliceBal);
        console2.log("  CAROL balance (USDC 1e6)", m.carolBal);
        console2.log("  pool totalAssets (USDC 1e6)", m.poolAssets);
        console2.log("  trigger keeper balance (USDC 1e6)", clearinghouse.balanceAssets(TRIGGER_KEEPER));
        console2.log("  execution keeper balance (USDC 1e6)", clearinghouse.balanceAssets(KEEPER));
        console2.log("  liquidation keeper balance (USDC 1e6)", clearinghouse.balanceAssets(LIQ_KEEPER));
    }

    // -------------------------------------------------------------------------------------------------------------
    // Value: the stop ALICE could not arm, and what it would have saved her
    // -------------------------------------------------------------------------------------------------------------

    /// Two worlds from the same state. In the control world ALICE's 1.10 stop arms and closes her at 1.10. In the
    /// refused world her one arm attempt reverts PriceOutOfOrder, she does not retry, and she is liquidated at the
    /// first liquidatable price with CAROL. CAROL (no protection in either world) is the control for the market path.
    function test_Value_RefusedStopLossCostsTheTrader() public {
        uint256 snap = vm.snapshotState();

        _run("control (stop armed)", false);
        assertTrue(m.aliceArmed, "control: ALICE arms");
        assertTrue(m.aliceStopFired, "control: ALICE's stop closes her at the stop level");
        assertTrue(m.carolLiquidated, "control: CAROL is liquidated");
        uint256 aliceControl = m.aliceBal;
        uint256 carolControl = m.carolBal;
        uint256 poolControl = m.poolAssets;

        vm.revertToState(snap);

        _run("refused (arm reverted, no retry)", true);
        assertFalse(m.aliceArmed, "refused: ALICE's arm reverted");
        assertEq(m.aliceArmRevert, IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "PriceOutOfOrder");
        assertTrue(m.aliceLiquidated, "refused: ALICE is liquidated instead of stopped out");
        assertEq(m.carolBal, carolControl, "same market path: CAROL ends identical in both worlds");

        uint256 aliceLoss = aliceControl - m.aliceBal;
        uint256 poolGain = m.poolAssets - poolControl;
        console2.log("ALICE loss vs control (USDC 1e6)", aliceLoss);
        console2.log("pool gain vs control (USDC 1e6)", poolGain);
        assertEq(aliceLoss, 905_400_002, "ALICE ends 905.400002 USDC worse");
        assertEq(poolGain, 905_000_002, "the LP pool ends 905.000002 USDC better");
    }

    /// RECOVERY: in the refused state, one public refresh with a current tick lets ALICE arm, and her stop then
    /// protects her through the same price path as control.
    function test_Recovery_RefreshThenArm() public {
        _market();
        _armPhase(true);
        assertFalse(m.aliceArmed, "ghost: arm refused");
        _nextBlock(1);
        _publicRefresh(ALICE, MARK_PRICE); // the trader herself, or anyone
        _nextBlock(1);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        assertGt(pid, 0, "recovery: a public refresh lets ALICE arm");
        m.aliceArmed = true;
        _stopPhase();
        assertTrue(m.aliceStopFired, "recovery: stop fires at the stop level");
    }

    // -------------------------------------------------------------------------------------------------------------
    // Second symptom: StalePrice while the cached mark is fresh (same call, PositionProtectionBook.sol:880)
    // -------------------------------------------------------------------------------------------------------------

    function _aliceOpenWithSkewedRefresh() private returns (uint256 oldest, uint256 newest) {
        _market();
        _nextBlock(20);
        newest = vm.getBlockTimestamp();
        oldest = newest - 12; // inside LIVE_TICK_LOOKBACK (15 s) and the MarkRefresh divergence limit
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _skewedTick(MARK_PRICE, oldest, newest);
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
        assertEq(engine.lastMarkTime(), newest, "ghost: engine mark time = newest component");
    }

    /// PROPERTY: while the cached mark is fresh, a trader can arm. FAILS on the pinned code (PletherOracle__StalePrice).
    function test_StaleWindow_Property_ArmWhileMarkFresh() public {
        (uint256 oldest,) = _aliceOpenWithSkewedRefresh();
        vm.warp(oldest + 61); // oldest component stale (>60 s), mark (newest) 49 s old
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        assertGt(pid, 0, "armed while the mark is fresh");
    }

    /// CHARACTERIZE: exact revert, window bounds, and the same remedy as the main symptom.
    function test_StaleWindow_Characterize_WindowAndRemedy() public {
        (uint256 oldest, uint256 newest) = _aliceOpenWithSkewedRefresh();
        // just before the window: arm works
        vm.warp(oldest + 60);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 snap = vm.snapshotState();
        vm.prank(ALICE);
        protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        vm.revertToState(snap);
        // inside the window: the mark is 49 s old (fresh under the 60 s limit) but arming reverts StalePrice
        vm.warp(oldest + 61);
        vm.roll(vm.getBlockNumber() + 1);
        assertLe(vm.getBlockTimestamp() - engine.lastMarkTime(), 60, "ghost: cached mark is fresh");
        vm.prank(ALICE);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        console2.log("stale-component window seconds", newest + 60 - (oldest + 60));
        // remedy: anyone refreshes
        _publicRefresh(REFRESHER, MARK_PRICE);
        _nextBlock(1);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        assertGt(pid, 0, "remedy: a refresh clears it");
    }

    /// The remedy for the main symptom (a refresh) installs the most-skewed basket intake allows (oldest component at
    /// now-15). Arming still succeeds at once; the stale-component window opens only 45 s later.
    function test_StaleWindow_RemedyWithMaxSkewStillArms() public {
        _market();
        _commitAndExecute(BOB, CfdTypes.Side.SHORT, MARK_PRICE); // main-symptom state
        _nextBlock(20);
        vm.prank(ALICE);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        uint256 nowTs = vm.getBlockTimestamp();
        bytes[] memory data = _skewedTick(MARK_PRICE, nowTs - 15, nowTs);
        vm.prank(ALICE);
        router.updateMarkPrice(data);
        _nextBlock(1);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_params(0, STOP_PRICE));
        assertGt(pid, 0, "remedy with max skew still arms");
    }
}
```

_Test file 2 of 3 (`Card1351S2PersistenceCostValue.t.sol`: persistence and cost) is in the linked secret Gist, because this field is limited to 65,536 characters. Test files 1 and 3 above run on their own._

