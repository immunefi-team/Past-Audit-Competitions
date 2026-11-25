# \#98406 [SC-Low] LOW-59 fix is incomplete: CfdEngine can still renounce ownership, so the next degraded-mode latch can never be cleared and LP epoch settlement stops for good

**Submitted on Nov 7th 2026 at 17:57:04 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98406
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
The program treats "the owner renounces and loses recovery authority" as a real defect. It acknowledged Cecuro LOW-59 ("Owner can leave emergency recovery ownerless") and fixed it by making `renounceOwnership()` revert, with this stated reason:

> "Ownership cannot be renounced because governance must retain emergency recovery authority." (`HousePool.sol:264`)

That fix pattern was applied to HousePool, OrderRouterAdmin and EmergencyPauseCoordinator. It was not applied to `CfdEngine`, which keeps the original OpenZeppelin `renounceOwnership()` through `CfdTreasury is Ownable2Step` (`CfdTreasury.sol:10`). Yet the engine holds the protocol's only exit from degraded mode:

> "Engine degraded mode latches when a close or liquidation leaves raw `E < L` [...]. It blocks new risk, LP entry, and all LP epoch settlement. The Engine owner can call `clearDegradedMode()` once raw `E >= L`." (`EMERGENCY_RESPONSE_GUIDE.md:45-48`)

The trigger is not a hypothetical admin mistake. Renouncing the engine is an operation the team supports and tests: SECURITY.md:128 says "renouncing ownership does not prevent activation", and `test_TreasuryProposal_ActivationDoesNotRequireAnOwner` renounces the engine on purpose. After that supported governance step, an ordinary market move that leaves `E < L` latches degraded mode, and the latch can never be cleared:
- `clearDegradedMode()` and `recapitalizePool()` are `onlyOwner` (`CfdEngine.sol:658`, `:632`). `clearDegradedMode()` is the only code that sets `degradedMode` back to `false`: the only writers of the flag are `CfdEngine.sol:665` (false) and `:1366` (true).
- CfdEngine is not upgradeable. It has no proxy, its dependencies are immutable or set once (`HousePool.ENGINE` is immutable, `MarginClearinghouse.setEngine` is one-time), and DEPLOYMENT.md:21 says "Immutable and one-time bindings prevent [...] an in-place upgrade of a populated deployment. There is no live-state migration tool." So there is no way to restore the clear function.
- Every `settleLpEpoch` then reverts with `HousePool__DegradedMode`, and this continues after solvency has fully returned. No LP redemption can be funded and no LP deposit can be activated again; only claims funded before the latch stay claimable. HousePool keeps its owner thanks to the LOW-59 fix, but no HousePool function can clear the engine latch.

The program's known-issue rule covers exactly this case: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fef [...] is incomplete or introduces a new bug" (program page, Known Issues and Previous Audits). This report shows that the LOW-59 fix is incomplete.

## Vulnerability Details
**Root cause** (pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`)
```solidity
// CfdTreasury.sol:10 - inherited by CfdEngine; no renounceOwnership override
abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {

// CfdEngine.sol:658 - the only code path that sets degradedMode back to false
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
    degradedMode = false;
    emit DegradedModeCleared();
}
```
The fix pattern, as applied to the three other contracts:
```solidity
// HousePool.sol:264-267 (same in OrderRouterAdmin.sol:125, EmergencyPauseCoordinator.sol:102)
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

**Where the fix pattern was applied and where it was not**

| | LOW-59 fix (1151d01f) | This report |
|---|---|---|
| Contracts | HousePool, OrderRouterAdmin, EmergencyPauseCoordinator | CfdEngine (via `CfdTreasury.sol`) |
| Recovery functions lost after renounce | `unpause`, `unpauseLpEpochSettlement`, `setPauser`, config finalization, `setGuardian` | `clearDegradedMode`, `recapitalizePool` |
| Event that makes the loss permanent | a later pause / settlement hold | a later degraded latch, which trading sets on its own (`CfdEngine.sol:1355-1367`; ACCOUNTING_SPEC "Degraded Mode"; WHITEPAPER 2.3) |
| Status at a82d4fef | renounce reverts (PoC test 1) | renounce succeeds (PoC test 1) |

The omission is not intended. Cecuro's remediation for LOW-59 was "Override renounceOwnership() in every governance-bearing perps contract", and the engine is governance-bearing: owner-only setters, treasury proposals, recapitalization and the degraded-mode clear. The team marked LOW-59 "Addressed", but the override exists in three contracts only.

**What we do not dispute**
An engine without an owner is a state the team supports for the treasury schedule (SECURITY.md:128 and the team test above), and that part works as documented. This report is about what the documents and tests do not cover: an ownerless engine that later latches degraded mode. No document says that renouncing the engine also gives up the only degraded-mode recovery, and no test covers that state.

**The obvious counterarguments, answered up front**
- *"Renouncing is a privileged action."* It is the same privileged action as LOW-59, which the program treated as a defect and fixed, and the precondition is an operation the team itself supports. The program rules keep this case in scope: "an in-scope impact arises from a logic error that a privileged address can trigger, which remains valid" (Attacks Involving Access to Privileged Addresses). The logic error is the missing override. The impact needs no privileged action after the renounce, because trading sets the latch.
- *"LOW-59 is a known issue."* The known-issue rule quoted above makes an incomplete fix valid. LOW-59's fix is verified for the three contracts it changed (PoC test 1). The remaining instance is in a different file (`CfdTreasury.sol`), loses a different recovery path (`clearDegradedMode` / `recapitalizePool`), and is fixed by a change those three overrides do not make.

## Impact Details
**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value"

After the engine owner has renounced and a later close leaves `E < L`:
- Every `settleLpEpoch` reverts `HousePool__DegradedMode`, permanently (PoC: +7, +28 and +364 days). No Junior or Senior redemption is ever funded again, and no LP deposit is ever activated. The LP capital stays in HousePool but can never be redeemed: 1,007,270.79 USDG in the PoC.
- Matured Junior deposit escrow cannot be cancelled (`TrancheVault__DepositEpochAlreadyActive`) and never activates: 50,000 USDG in the PoC.
- Traders with a live position cannot withdraw free collateral until they close (`CfdEngine__DegradedMode`). New opens are blocked for good.
- This holds after solvency has returned. In the PoC both legs that caused the latch are closed and the `clearDegradedMode` solvency check passes. Only the `onlyOwner` gate blocks it, and no address can pass that gate any more.
- The HousePool owner, who still exists, can only hold settlement. That lets Senior escrow out, but it does not clear the engine latch, so the Junior escrow and all LP redemptions stay blocked.

**Measured (PoC tests 2-4).** All three branches run the same market history: pool 1,002,000 USDG, release risk parameters, basket -1 %, then both legs closed. R2 is a Junior redemption the incumbent LP queues before the latch, for 10 % of its shares. The bug branches (tests 1-3) use no cheatcode on protocol state; the only privileged call is the owner's `renounceOwnership()`. `vm.etch` appears only in test 4, to install the suggested fix. Amounts in USDG.

| After both legs are closed (E = 1,007,270.79, L = 10,000) | Owner present (control) | Engine renounced (variant) | Fix installed |
|---|---|---|---|
| `clearDegradedMode()` | succeeds | reverts for every caller | succeeds |
| `settleLpEpoch` | succeeds | `HousePool__DegradedMode` at +7 / +28 / +364 days | succeeds |
| Junior redemption R2 funded | 100,516.56 | 0 | 100,516.56 |
| Junior deposit (50,000) | activated | stuck, not cancellable | activated |
| Senior deposit (100,000) | activated | refundable only through the HousePool owner's settlement hold | activated |

**Suggested fix:** apply the same override to `CfdTreasury` (covers `CfdEngine`), and for consistency to `CfdEngineAdmin` and `MarginClearinghouse`, which also still inherit `renounceOwnership`:
```solidity
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```
If renouncing the engine is meant to stay possible (SECURITY.md:128), the alternative fix is to make `clearDegradedMode()` callable without the owner once its on-chain solvency check passes, because that check already decides when clearing is safe.

**Negative control:** PoC test 4 etches exactly this override onto the live engine address. `renounceOwnership()` then reverts, the owner stays, and the same market history ends with `clearDegradedMode` and `settleLpEpoch` succeeding.

## References
- `packages/perps/src/CfdTreasury.sol#L10`, `packages/perps/src/CfdEngine.sol#L632-L667` (recovery), `#L1355-L1367` (latch)
- `packages/perps/src/HousePool.sol#L264-L267`, `OrderRouterAdmin.sol#L125`, `EmergencyPauseCoordinator.sol#L102` (LOW-59 fix)
- `packages/perps/EMERGENCY_RESPONSE_GUIDE.md` (Control inventory, degraded mode), `ACCOUNTING_SPEC.md` "Degraded Mode" (l.1201-1212), `SECURITY.md:128`, `DEPLOYMENT.md:21`
- Cecuro AI audit 2026-10-26, LOW-59 (comment, locations, remediation); fix commit 1151d01f
- `packages/perps/test/perps/spec/admin/TreasuryProposal.t.sol:340`

## Proof of Concept

## Proof of Concept
One runnable Foundry test file in the repository's own perps test suite, at the frozen commit, with no change to any in-scope contract.
- **Fixture:** the repo's `BasePerpTest`. The only mocks are MockPyth and the USDG mock, both out of scope.
- **Parameters:** release values from DEPLOYMENT.md (vpiFactor 0.01, maxSkewRatio 0.4, MM 10 / IM 20 / FAD 300 bps, carry 500 bps, bounty 10 bps, keeper share 50 %).
- **Entry points:** public only, plus one owner call in the variant: `engine.renounceOwnership()`.
  - LPs call `TrancheVault.requestDeposit / requestRedeem / cancelPendingDeposit`.
  - Traders call `router.commitOrder` and `clearinghouse.withdraw`.
  - An untrusted keeper calls `router.executeOrder`, `router.updateMarkPrice` and `router.settleLpEpoch`.
- The test contract is the fixture's governance (owner of every component) and its initial Junior LP, as in the repo's own tests.
- No `vm.store`. Test 4 uses `vm.etch` only to install the suggested fix for the negative control.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test code below as packages/perps/test/PocS4_EngineRenounceRecovery.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/PocS4_EngineRenounceRecovery.t.sol -vv   # Foundry 1.5.1
```

No env vars or RPC needed. Expected result: `4 passed`. `FOUNDRY_SPARSE_MODE=true` compiles only this test and its dependencies. Without it, forge first builds the whole via-IR suite, which takes far longer.

**Shared history (tests 2-4).** Amounts in USDG.
1. LP queue for the next epoch: the incumbent Junior LP requests redemption R2 (10 % of its shares), a new Junior LP deposits 50,000, a new Senior LP deposits 100,000.
2. TRADER_B opens a small LONG (100 lots). TRADER_L and TRADER_S open a balanced pair at admission capacity (9,985 lots per side); pool assets 1,002,000.46.
3. The basket moves -1 % and TRADER_L closes the winning LONG leg. The close leaves `E < L`, and the engine latches degraded mode (`CfdEngine.sol:1366`). This is the documented latch (ACCOUNTING_SPEC "Degraded Mode", WHITEPAPER 2.3).
4. TRADER_S closes the SHORT leg. E = 1,007,270.79 and L = 10,000, so solvency has fully returned.

**Test 1, `test_S4_1_FixedContractsRejectRenounce_EngineAccepts`.** `HousePool.renounceOwnership()` and `OrderRouterAdmin.renounceOwnership()` revert (the LOW-59 fix). `CfdEngine.renounceOwnership()` succeeds and the owner becomes `address(0)`.

**Test 2, `test_S4_2_Control_OwnerPresent_RecoveryWorks` (control).** While latched, `settleLpEpoch` reverts `HousePool__DegradedMode`. The owner calls `clearDegradedMode()`, then the keeper settles: R2 is funded with 100,516.56, and the Junior (50,000) and Senior (100,000) deposits activate.

**Test 3, `test_S4_3_Variant_EngineRenounced_LatchIsPermanent` (variant).** Governance renounces the engine before step 1; then the same history runs.
- The `clearDegradedMode` solvency check passes. This is shown in a snapshot by calling as `address(0)`, the value `owner()` now returns, which no real account can use.
- `clearDegradedMode()` from the former owner, a keeper, an LP and an outsider: `OwnableUnauthorizedAccount`. `recapitalizePool()` from the former owner: `OwnableUnauthorizedAccount`.
- `settleLpEpoch` reverts `HousePool__DegradedMode` at +7, +28 and +364 days. R2 funded: 0. The matured Junior deposit escrow (50,000) cannot be cancelled.
- The HousePool owner's settlement hold lets the Senior escrow (100,000) out. The Junior escrow stays stuck, and settlement still reverts after the hold is released. Pool assets that can never be redeemed: 1,007,270.79.
- TRADER_B, who has a live position, cannot withdraw free collateral (`CfdEngine__DegradedMode`).

**Test 4, `test_S4_4_FixInstalled_RenounceRejected_RecoveryWorks` (negative control).** The suggested override (`CfdEngineRenounceGuarded`, the same code as `HousePool.sol:264-267`) is etched onto the live engine address. The engine's planning reader is bound to its creator, so the test gives the patched reader the original reader's code. `renounceOwnership()` reverts and the owner stays. The same history ends with `clearDegradedMode` and `settleLpEpoch` succeeding, and R2 is funded with 100,516.56.

Output of the run:
```
Ran 4 tests for test/PocS4_EngineRenounceRecovery.t.sol:PocS4_EngineRenounceRecovery
[PASS] test_S4_1_FixedContractsRejectRenounce_EngineAccepts() (gas: 38947)
Logs:
  == S4.1 LOW-59 fix scope
  HousePool / OrderRouterAdmin renounceOwnership: revert (fix 1151d01f)
  CfdEngine renounceOwnership: succeeds, owner = 0x0000000000000000000000000000000000000000
[PASS] test_S4_2_Control_OwnerPresent_RecoveryWorks() (gas: 15937117)
Logs:
  == S4.2 CONTROL: engine owner present
  Step 1 - LP queue: Junior redemption R2 shares      100000913208338
           Junior deposit escrow (USDG atoms)          50000000000
           Senior deposit escrow (USDG atoms)          100000000000
  Step 2 - book open: lots per side, pool assets       9985 1002000460682
  Step 3 - basket -1 %, LONG leg closed: degradedMode   true
  Step 4 - SHORT leg closed: E (pool - trader claims)    1007270789391
           L (max directional liability)                10000000000
  Step 5 - keeper settleLpEpoch while latched: HousePool__DegradedMode
  Step 6 - owner clearDegradedMode: ok, degradedMode    false
  Step 7 - keeper settleLpEpoch: ok
           R2 funded (USDG atoms)                       100516559643
           Junior deposit activated (USDG atoms)        50000000000
           Senior deposit activated (USDG atoms)        100000000000
[PASS] test_S4_3_Variant_EngineRenounced_LatchIsPermanent() (gas: 17074472)
Logs:
  == S4.3 VARIANT: engine owner renounced before the market event
  Step 0 - governance calls engine.renounceOwnership(); owner = 0x0000000000000000000000000000000000000000
  Step 1 - LP queue: Junior redemption R2 shares      100000913208338
           Junior deposit escrow (USDG atoms)          50000000000
           Senior deposit escrow (USDG atoms)          100000000000
  Step 2 - book open: lots per side, pool assets       9985 1002000460682
  Step 3 - basket -1 %, LONG leg closed: degradedMode   true
  Step 4 - SHORT leg closed: E (pool - trader claims)    1007270789391
           L (max directional liability)                10000000000
  Step 5 - clearDegradedMode solvency check passes (snapshot as owner() == address(0))
  Step 6 - clearDegradedMode by former owner / keeper / LP / outsider: OwnableUnauthorizedAccount
           recapitalizePool by the former owner: OwnableUnauthorizedAccount
  Step 7 - +days after maturity: settleLpEpoch reverts HousePool__DegradedMode 7
  Step 7 - +days after maturity: settleLpEpoch reverts HousePool__DegradedMode 28
  Step 7 - +days after maturity: settleLpEpoch reverts HousePool__DegradedMode 364
           R2 funded (USDG atoms)                       0
           matured Junior deposit escrow: cancelPendingDeposit reverts (USDG atoms held) 50000000000
  Step 8 - HousePool owner settlement hold: Senior escrow refunded 100000000000
           Junior escrow still stuck; settlement still HousePool__DegradedMode after unpause
           LP capital that can never be redeemed (pool.totalAssets, USDG atoms) 1007270789391
           TRADER_B (live position) withdraw: CfdEngine__DegradedMode (must close first)
[PASS] test_S4_4_FixInstalled_RenounceRejected_RecoveryWorks() (gas: 21390967)
Logs:
  == S4.4 NEGATIVE CONTROL: suggested fix installed (renounce reverts on CfdEngine)
  Step 0 - engine.renounceOwnership(): reverts; owner kept
  Step 1 - LP queue: Junior redemption R2 shares      100000913208338
           Junior deposit escrow (USDG atoms)          50000000000
           Senior deposit escrow (USDG atoms)          100000000000
  Step 2 - book open: lots per side, pool assets       9985 1002000460682
  Step 3 - basket -1 %, LONG leg closed: degradedMode   true
  Step 4 - SHORT leg closed: E (pool - trader claims)    1007270789391
           L (max directional liability)                10000000000
  Step 5 - owner clearDegradedMode + keeper settleLpEpoch: ok; R2 funded 100516559643
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 42.56ms (69.79ms CPU time)
Ran 1 test suite in 55.04ms (42.56ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

### Full test code (`packages/perps/test/PocS4_EngineRenounceRecovery.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// S4: the LOW-59 fix ("renounceOwnership reverts") covers HousePool, OrderRouterAdmin and EmergencyPauseCoordinator,
// but not CfdEngine (CfdTreasury is Ownable2Step). CfdEngine.clearDegradedMode() is onlyOwner and is the only way out
// of degraded mode. After the engine owner renounces, a later market-driven latch is permanent: LP epoch settlement
// reverts HousePool__DegradedMode forever, even after solvency has fully returned.
//
// Release risk parameters (DEPLOYMENT.md "Initial release parameters"): vpiFactor 0.01e18, maxSkewRatio 0.4e18,
// MM 10 bps, IM 20 bps, FAD 300 bps, baseCarry 500 bps, bounty 10 bps / keeper share 50 %. After setUp only public,
// untrusted entrypoints are used (router.commitOrder, router.executeOrder, router.settleLpEpoch,
// router.updateMarkPrice, vault requests), plus exactly one owner call in the variant: engine.renounceOwnership().
//
// Tests:
//   test_S4_1_FixedContractsRejectRenounce_EngineAccepts   - LOW-59 fix works where it was applied; engine is open
//   test_S4_2_Control_OwnerPresent_RecoveryWorks            - latch, solvency returns, owner clears, settlement resumes
//   test_S4_3_Variant_EngineRenounced_LatchIsPermanent      - same history after renounce: no caller can clear, LP
//                                                              settlement blocked at +7, +28 and +364 days
//   test_S4_4_FixInstalled_RenounceRejected_RecoveryWorks   - patched engine code (renounce reverts) etched in place
//
// Run (from the repo root of audit-comp-plether @ a82d4fef):
//   save as packages/perps/test/PocS4_EngineRenounceRecovery.t.sol
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/PocS4_EngineRenounceRecovery.t.sol -vv

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {console} from "forge-std/console.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @dev The suggested fix, test-only: the same override HousePool uses (HousePool.sol:265), applied to the engine.
contract CfdEngineRenounceGuarded is CfdEngine {
    constructor(
        address settlementToken_,
        address clearinghouse_,
        uint256 capPrice_,
        CfdTypes.RiskParams memory riskParams_,
        uint256 frozenCloseSpreadBps_
    ) CfdEngine(settlementToken_, clearinghouse_, capPrice_, riskParams_, frozenCloseSpreadBps_) {}

    function renounceOwnership() public view override onlyOwner {
        revert OwnableInvalidOwner(address(0));
    }
}

contract PocS4_EngineRenounceRecovery is BasePerpTest {

    address internal constant TRADER_L = address(0xA11A01); // LONG leg
    address internal constant TRADER_S = address(0xA11A02); // SHORT leg
    address internal constant TRADER_B = address(0xB0B001); // ordinary trader with a small live LONG
    address internal constant KEEPER = address(0xBEEF01);
    address internal constant LP_JR_ENTRY = address(0xD0D001);
    address internal constant LP_SR_ENTRY = address(0xD0D002);
    address internal constant OUTSIDER = address(0xC0FFEE);

    uint256 internal constant LOT = 1e20;
    uint256 internal constant LOT_ASSETS_AT_PAR = 100e6;
    uint256 internal constant ENTRY = 1e8;
    uint256 internal constant B_LOTS = 100;
    uint256 internal constant JR_DEPOSIT = 50_000e6;
    uint256 internal constant SR_DEPOSIT = 100_000e6;

    uint256 internal r2; // Junior redemption of the incumbent LP, matures after the latch
    uint256 internal r2Shares;
    uint256 internal jrDepositId;
    uint256 internal srDepositId;
    uint256 internal pairLots;
    uint256 internal clientNonce;
    uint256 internal movedPrice;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    // ------------------------------------------------------------------ helpers

    function _effectiveAssets() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _request(
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        clientNonce += 1;
        request.clientOrderId = bytes32(clientNonce);
        request.side = side;
        request.sizeDelta = lots * LOT;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 1 hours),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        });
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, lots, margin, isClose);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _keeperSettle(
        uint256 price
    ) internal returns (bytes4 selector) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        try router.settleLpEpoch(data) {
            return bytes4(0);
        } catch (bytes memory err) {
            return bytes4(err);
        }
    }

    function _keeperRefreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _tryClear(
        address caller
    ) internal returns (bytes4 selector) {
        vm.prank(caller);
        try engine.clearDegradedMode() {
            return bytes4(0);
        } catch (bytes memory err) {
            return bytes4(err);
        }
    }

    // ------------------------------------------------------------------ shared history

    /// @dev LP queue (redemption + two deposits maturing at the next epoch), a small ordinary LONG, and a balanced pair
    ///      opened at admission capacity, all through public entrypoints. Same construction as the repo's documented
    ///      degraded-mode example (WHITEPAPER 2.3): one side realizes a gain, the other side stays open.
    function _queueAndOpenBook() internal {
        uint256 cooldownEnd = juniorVault.lastDepositTime(address(this)) + juniorVault.DEPOSIT_COOLDOWN() + 1;
        if (cooldownEnd > vm.getBlockTimestamp()) {
            vm.warp(cooldownEnd);
        }
        r2Shares = juniorVault.maxRequestRedeem(address(this)) / 10;
        r2 = juniorVault.requestRedeem(r2Shares, address(this), address(this));
        settlementToken.mint(LP_JR_ENTRY, JR_DEPOSIT);
        vm.startPrank(LP_JR_ENTRY);
        settlementToken.approve(address(juniorVault), JR_DEPOSIT);
        jrDepositId = juniorVault.requestDeposit(JR_DEPOSIT, LP_JR_ENTRY, LP_JR_ENTRY);
        vm.stopPrank();
        settlementToken.mint(LP_SR_ENTRY, SR_DEPOSIT);
        vm.startPrank(LP_SR_ENTRY);
        settlementToken.approve(address(seniorVault), SR_DEPOSIT);
        srDepositId = seniorVault.requestDeposit(SR_DEPOSIT, LP_SR_ENTRY, LP_SR_ENTRY);
        vm.stopPrank();
        console.log("Step 1 - LP queue: Junior redemption R2 shares     ", r2Shares);
        console.log("         Junior deposit escrow (USDG atoms)         ", JR_DEPOSIT);
        console.log("         Senior deposit escrow (USDG atoms)         ", SR_DEPOSIT);

        _fundTrader(TRADER_B, 7000e6);
        _trade(TRADER_B, CfdTypes.Side.LONG, B_LOTS, 2000e6, false, ENTRY);

        uint256 e0 = _effectiveAssets();
        pairLots = (e0 * 10_000) / (10_025 * LOT_ASSETS_AT_PAR) - 10;
        uint256 maxSkewLots = (e0 * 4) / (10 * LOT_ASSETS_AT_PAR) - 1;
        uint256 longLots = pairLots - B_LOTS;
        _fundTrader(TRADER_L, 10_000e6);
        _fundTrader(TRADER_S, 30_000e6);
        uint256 s1 = maxSkewLots - B_LOTS;
        uint256 l1 = s1 + maxSkewLots - B_LOTS;
        _trade(TRADER_S, CfdTypes.Side.SHORT, s1, 12_000e6, false, ENTRY);
        _trade(TRADER_L, CfdTypes.Side.LONG, l1, 3000e6, false, ENTRY);
        _trade(TRADER_S, CfdTypes.Side.SHORT, pairLots - s1, 12_000e6, false, ENTRY);
        _trade(TRADER_L, CfdTypes.Side.LONG, longLots - l1, 1000e6, false, ENTRY);
        console.log("Step 2 - book open: lots per side, pool assets      ", pairLots, pool.totalAssets());
        assertFalse(engine.degradedMode());
    }

    /// @dev Basket -1 %: the winning LONG leg closes and the engine latches degraded mode (CfdEngine.sol:1366).
    ///      Then the SHORT leg closes as well, so the liability that caused the latch is gone.
    function _latchThenSolvencyReturns() internal {
        movedPrice = (ENTRY * 99) / 100;
        _trade(TRADER_L, CfdTypes.Side.LONG, pairLots - B_LOTS, 0, true, movedPrice);
        assertTrue(engine.degradedMode(), "market move latches degraded mode");
        console.log("Step 3 - basket -1 %, LONG leg closed: degradedMode  ", engine.degradedMode());
        _trade(TRADER_S, CfdTypes.Side.SHORT, pairLots, 0, true, movedPrice);
        console.log("Step 4 - SHORT leg closed: E (pool - trader claims)   ", _effectiveAssets());
        console.log("         L (max directional liability)               ", _maxLiability());
        assertGe(_effectiveAssets(), _maxLiability(), "solvency has returned");
        vm.warp(pool.lpEpochStart(r2) + 1 hours);
        _keeperRefreshMark(movedPrice);
    }

    // ------------------------------------------------------------------ tests

    function test_S4_1_FixedContractsRejectRenounce_EngineAccepts() public {
        console.log("== S4.1 LOW-59 fix scope");
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        pool.renounceOwnership();
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        routerAdmin.renounceOwnership();
        console.log("HousePool / OrderRouterAdmin renounceOwnership: revert (fix 1151d01f)");
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        console.log("CfdEngine renounceOwnership: succeeds, owner =", engine.owner());
    }

    function test_S4_2_Control_OwnerPresent_RecoveryWorks() public {
        console.log("== S4.2 CONTROL: engine owner present");
        _queueAndOpenBook();
        _latchThenSolvencyReturns();
        assertEq(_keeperSettle(movedPrice), IHousePool.HousePool__DegradedMode.selector, "blocked while latched");
        console.log("Step 5 - keeper settleLpEpoch while latched: HousePool__DegradedMode");
        assertEq(_tryClear(address(this)), bytes4(0), "owner clears once solvent");
        console.log("Step 6 - owner clearDegradedMode: ok, degradedMode   ", engine.degradedMode());
        bytes4 sel = _keeperSettle(movedPrice);
        assertEq(sel, bytes4(0), "settlement resumes");
        uint256 funded = juniorVault.claimableRedeemAssets(r2, address(this));
        uint256 jrActive = juniorVault.claimableDepositRequest(jrDepositId, LP_JR_ENTRY);
        uint256 srActive = seniorVault.claimableDepositRequest(srDepositId, LP_SR_ENTRY);
        console.log("Step 7 - keeper settleLpEpoch: ok");
        console.log("         R2 funded (USDG atoms)                      ", funded);
        console.log("         Junior deposit activated (USDG atoms)       ", jrActive);
        console.log("         Senior deposit activated (USDG atoms)       ", srActive);
        assertGt(funded, 0, "R2 funded");
        assertEq(jrActive, JR_DEPOSIT, "Junior deposit activated");
        assertEq(srActive, SR_DEPOSIT, "Senior deposit activated");
    }

    function test_S4_3_Variant_EngineRenounced_LatchIsPermanent() public {
        console.log("== S4.3 VARIANT: engine owner renounced before the market event");
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        console.log("Step 0 - governance calls engine.renounceOwnership(); owner =", engine.owner());
        _queueAndOpenBook();
        _latchThenSolvencyReturns();

        // The solvency condition of clearDegradedMode holds: only the owner gate blocks it. (Shown in a snapshot by
        // calling as address(0), which is what owner() now returns; no real account can do this.)
        uint256 snap = vm.snapshotState();
        assertEq(_tryClear(address(0)), bytes4(0), "solvency check passes");
        assertTrue(vm.revertToState(snap));
        console.log("Step 5 - clearDegradedMode solvency check passes (snapshot as owner() == address(0))");

        address[4] memory callers = [address(this), KEEPER, LP_JR_ENTRY, OUTSIDER];
        for (uint256 i = 0; i < callers.length; i++) {
            assertEq(
                _tryClear(callers[i]), Ownable.OwnableUnauthorizedAccount.selector, "nobody can clear degraded mode"
            );
        }
        console.log("Step 6 - clearDegradedMode by former owner / keeper / LP / outsider: OwnableUnauthorizedAccount");
        settlementToken.mint(address(this), 1_000_000e6);
        settlementToken.approve(address(engine), 1_000_000e6);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.recapitalizePool(1_000_000e6);
        console.log("         recapitalizePool by the former owner: OwnableUnauthorizedAccount");

        uint256[3] memory days_ = [uint256(7), 28, 364]; // whole weeks: same weekday and time as the first check
        for (uint256 i = 0; i < days_.length; i++) {
            vm.warp(pool.lpEpochStart(r2) + days_[i] * 1 days);
            _keeperRefreshMark(movedPrice);
            assertEq(
                _keeperSettle(movedPrice), IHousePool.HousePool__DegradedMode.selector, "settlement blocked forever"
            );
            assertTrue(engine.degradedMode());
            console.log("Step 7 - +days after maturity: settleLpEpoch reverts HousePool__DegradedMode", days_[i]);
        }

        uint256 funded = juniorVault.claimableRedeemAssets(r2, address(this));
        console.log("         R2 funded (USDG atoms)                      ", funded);
        assertEq(funded, 0, "R2 never funded");
        vm.prank(LP_JR_ENTRY);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(jrDepositId, LP_JR_ENTRY, LP_JR_ENTRY);
        console.log("         matured Junior deposit escrow: cancelPendingDeposit reverts (USDG atoms held)", JR_DEPOSIT);

        // HousePool keeps its owner (LOW-59 fix), but none of its tools clears the engine latch.
        pool.pauseLpEpochSettlement();
        vm.prank(LP_SR_ENTRY);
        uint256 refunded = seniorVault.cancelPendingDeposit(srDepositId, LP_SR_ENTRY, LP_SR_ENTRY);
        vm.prank(LP_JR_ENTRY);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(jrDepositId, LP_JR_ENTRY, LP_JR_ENTRY);
        pool.unpauseLpEpochSettlement();
        assertEq(_keeperSettle(movedPrice), IHousePool.HousePool__DegradedMode.selector, "pool owner cannot help");
        console.log("Step 8 - HousePool owner settlement hold: Senior escrow refunded", refunded);
        console.log("         Junior escrow still stuck; settlement still HousePool__DegradedMode after unpause");
        console.log("         LP capital that can never be redeemed (pool.totalAssets, USDG atoms)", pool.totalAssets());

        vm.prank(TRADER_B);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(TRADER_B, 1000e6);
        console.log("         TRADER_B (live position) withdraw: CfdEngine__DegradedMode (must close first)");
    }

    function test_S4_4_FixInstalled_RenounceRejected_RecoveryWorks() public {
        console.log("== S4.4 NEGATIVE CONTROL: suggested fix installed (renounce reverts on CfdEngine)");
        CfdEngineRenounceGuarded patched = new CfdEngineRenounceGuarded(
            address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(), _frozenCloseSpreadBps()
        );
        // The engine's planning reader is bound to its creator; give the patched reader the original reader's code
        // (ENGINE = the live engine address), then put the patched engine code at the live engine address.
        vm.etch(address(patched.planningReader()), address(engine.planningReader()).code);
        vm.etch(address(engine), address(patched).code);

        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        engine.renounceOwnership();
        assertEq(engine.owner(), address(this));
        console.log("Step 0 - engine.renounceOwnership(): reverts; owner kept");
        _queueAndOpenBook();
        _latchThenSolvencyReturns();
        assertEq(_tryClear(address(this)), bytes4(0), "owner clears");
        assertEq(_keeperSettle(movedPrice), bytes4(0), "settlement resumes");
        assertGt(juniorVault.claimableRedeemAssets(r2, address(this)), 0, "R2 funded");
        assertEq(juniorVault.claimableDepositRequest(jrDepositId, LP_JR_ENTRY), JR_DEPOSIT);
        console.log("Step 5 - owner clearDegradedMode + keeper settleLpEpoch: ok; R2 funded",
            juniorVault.claimableRedeemAssets(r2, address(this)));
    }
}
```
