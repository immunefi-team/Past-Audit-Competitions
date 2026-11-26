# \#98559 [SC-High] Permissionless protection retries prevent owners from amending failed close limits, freezing position funds for over 24 hours

**Submitted on Nov 8th 2026 at 17:46:26 UTC by @Mukymuk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98559
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

When a stop-loss fires on Plether, the close can fail if the price has gapped past the stop and the close now costs more than the owner's own debit cap. The protocol's recovery for that case is a state, `Latched`, in which the owner can raise the cap and retry.

Anyone can take that state away. `retryPositionProtectionClose` is permissionless and moves the protection from `Latched` back to `Triggered`, where the owner can no longer amend the cap, cancel the protection or close the position. A third party who repeats this every time the 60-second close order expires keeps a plain-EOA owner locked out.

With the release parameters, a local test keeps the lock for 24 h 39 s. At the end, with a fresh oracle, the position is still open and only the 500 USDG of free margin outside it can be withdrawn. From the same state without the third party, the owner closes the position and withdraws 3,491.20 USDG in total, about 2,991.20 USDG more than the lock allows.

While the lock lasts, the position stays open and its loss keeps growing. If the price keeps moving against it, the owner ends with 2,000.80 USDG less than without the third party (price rising to 1.40), or 2,985.70 USDG less if the position reaches its liquidation price. The third party's gas for the cycles in those tests is under $1.

The lock depends on ordering: the third party's retry has to land before the owner's amendment each time. A local test chooses that order; it cannot show who wins inclusion on a live network. Owners whose account can batch calls in one transaction (EIP-7702) escape at any point; the affected owners are plain EOAs.

## Vulnerability Details

All references are under `packages/perps/` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.

**The owner's remedy.** After a failed protected close, the owner is meant to amend the close limits and retry. The interface gives the owner this remedy after a relatch (`src/interfaces/IPositionProtectionActions.sol` L32–37). `amendLatchedPositionProtectionCloseLimits` allows it only for the owner, only in `Latched`, and only with no pending order (`src/PositionProtectionBook.sol` L315–327). Failed-attempt relatching and owner amendment are documented recovery behaviour (`WORKING_WITH_AI_AGENTS.md` L282–286, `SECURITY.md` L730–731).

**Who else can use the state.** `retryPositionProtectionClose` accepts any caller while the protection is `Latched` (`src/PositionProtectionBook.sol` L382–406). It returns the protection to `Triggered` with a new close order that carries the old limits. `Latched` has a single writer, `handleFailedProtectionAttempt` (L666), reached both from expiry (`src/OrderRouter.sol` L226) and from a failed execution (`src/OrderRouter.sol` L328). It records nothing about where it came from, so the permissionless retry accepts both.

**The owner's other exits are closed while protection is active.** An ordinary close reverts with `ProtectionActive` (`src/router/OrderLiquidationBatchLogic.sol` L313–314). `cancelPositionProtection` and the atomic `cancelPositionProtectionAndCommitClose` revert with `ProtectionNotArmed` outside `Armed` (the latter at `src/PositionProtectionBook.sol` L346–348; both asserted in the PoC). Amendment needs `Latched`.

**The cycle.**

1. The stop fires. Protection becomes `Triggered` and a close order is queued with the default 60-second execution window.
2. The price has gapped past the stop, so the close debits more than the owner's `maxGrossAccountDebitAssets`. An honest keeper executes the order and it fails with `ConstraintViolation/GrossAccountDebit`. Protection becomes `Latched`.
3. If a third party's `retryPositionProtectionClose` lands before the owner's amendment, protection is back in `Triggered` with a new order and the old cap. The owner's amendment reverts (`ProtectionNotLatched`), and cancel and close revert as before.
4. The new order cannot fill, because the cap has not changed. When its window ends, the third party expires it and retries in the same transaction (`executeOrder` and `expireOrder` are permissionless, `src/OrderRouter.sol` L240 and L191), so `Latched` is never visible between blocks.
5. Step 4 repeats. Honest keeper executions in between also fail on the cap and relatch, and the third party retries again.

The repository already tests repeated retries and owner amendment separately. What this report adds is their composition: a permissionless retry consumes the only state in which the owner's documented recovery works, and nothing reserves that opportunity for the owner. Respecting the owner's financial cap is correct; making its amendment unreachable is the failure.

**Why this is not the intended design.** The repository documents every piece separately, and this report relies on that documentation:

- The program treats keepers as untrusted: "anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope." The third party here only calls permissionless keeper functions.
- The acknowledged Cecuro finding MED-9 ("Keepers can retry protection closes without the trader's financial limits") was answered by keeping the owner's ceilings on every retry, with this caveat: "Tight limits may prevent execution until conditions improve or the owner amends them." That accepted limitation depends on the owner being able to amend. The cycle removes exactly that: the owner's ceilings stay, and the amendment never becomes reachable. The program accepts reports on acknowledged findings that show the fix is incomplete.
- The operational safeguard is "operators must alert on prolonged Latched state without a live child" (`SECURITY.md` L1356–1359). The cycle always keeps a live child, so `Latched` never persists and that alert never fires.
- Keeper guidance says a conservative worker should "not continuously queue the same failing attempt" (`DEPLOYMENT.md` L419–421). An untrusted keeper does not have to follow it.
- The repository's own test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` (`PositionProtection.t.sol` L954–1004) asserts that once a third party retries, the owner's amendment reverts with `ProtectionNotLatched` (L995); in that test the owner had already amended, so the retry closes. `test_RepeatedProtectionAttemptExpiryRetry_ConservesSingleBounty` (L1558) runs repeated third-party expire-and-retry cycles and checks only that bounties and settlement are conserved. Neither considers a third party repeating the retry before the owner can amend.

**Relationship to known issues.** The AI-scan item "Protection trigger bounty is paid before a zero-bounty close is executed" (`audits/scan-plether-fi-plether-core-d4541b49.md` L542–584) describes the same exposure (the position "can remain open through its stop-loss/take-profit threshold and incur further loss or liquidation"), but from a different cause: keepers having no incentive to execute a zero-bounty close. In that case nothing blocks recovery: the order expires, protection becomes `Latched`, and the owner amends and exits (X5b in the PoC). Here honest keepers do execute and the close fails on the cap (X5a); what blocks the owner is the permissionless retry consuming `Latched`. Cecuro MED-15 accepts "temporary execution unavailability when no positive valid quote exists"; here a valid quote exists and is executed. Cecuro MED-30 is about stops not firing while the market is closed; here the stop fires. Cecuro MED-7 needs an older signed price; nothing like that is used here.

## Impact Details

Selected impact: **High, Temporary freezing of funds for at least 24 hours.**

Measured with the release parameters (LONG of 10,000 at 1.00, 5,000 USDG margin, stop at 1.10, cap 1.5 times the stop-close debit, price gapping to 1.20):

| After 24 h 39 s of the cycle, fresh oracle at 1.20 | Value |
|---|---|
| Position | still open (size 10,000) |
| Free margin outside the position | 500 USDG, withdrawable |
| Position-backed balance | 4,995.50 USDG; withdrawing even 1 unit reverts `MarginClearinghouse__InsufficientFreeEquity` |
| Control, same state, no third party: owner expires, amends, retries, keeper closes at 1.20, owner withdraws | 3,491.20 USDG in total (including the 500 USDG free margin) |

The value kept from the owner for over 24 hours is about 2,991.20 USDG: the 3,491.20 USDG the owner withdraws in the control, less the 500 USDG that stays withdrawable during the lock. During the lock the position also stays exposed to price; the stop-loss that should have closed it cannot complete. Carry over the 24 hours is negligible (0.003436 USDG at the checkpoint).

**Damage while locked.** The stop-loss exists to cap the loss. While the owner is locked out, the position keeps losing if the price keeps moving. Owner USDG actually received at the end (`balanceOf` after withdrawing everything allowed), same position and cap:

| Branch | Owner receives | vs control |
|---|---:|---:|
| Stop works: close at 1.10 | 3,991.60 USDG | +1,000.40 |
| Control: the gap close fails at 1.20; no third party; the owner amends and closes at 1.20 | 2,991.20 USDG | 0 |
| D1: the cycle runs while the price rises to 1.30 and 1.40; then the owner closes at 1.40 | 990.40 USDG | −2,000.80 |
| D2: the cycle runs until the first liquidatable price (1.498); the position is liquidated while `Triggered` | 5.50 USDG | −2,985.70 |

The control already contains the gap loss; the difference from the control is what the lock adds. It scales with how far the price moves while the owner is locked out: on this 10,000 position, each 0.01 of further rise costs the owner about 100 USDG (D1: 2,000.80 for 0.20). In D1 and D2 the moves happen over 15 cycles (about 15 minutes). Liquidation works normally while the protection is `Triggered` (`PositionLiquidated` is emitted and the protection ends `Liquidated`), and in D2 it leaves no bad debt. The lock does not create a solvency problem; it removes the owner's exit while the loss grows.

Who is affected: owners using a plain EOA. An owner with EIP-7702 batching escapes in one transaction at any point, even after the cycle has started (tests E4a–E4c in the PoC). A plain EOA cannot: its own execution relatches the protection, and a third-party retry that lands before its amendment closes the window again (E4d).

Preconditions:

- The owner set a finite close cap and a price gap pushed the close debit above it. In the test the cap is 1.5 times the debit at the stop (1,505.85 USDG); at the stop the close succeeds (1,003.90 USDG, G1), and after the gap it fails (2,004.30 USDG, G2). No claim is made about gap frequency.
- The position has enough margin to stay open during the lock (it does, in the test).
- A third party keeps retrying, and each retry lands before the owner's amendment.

**Cost to the third party.** It earns nothing and pays gas for one expire-and-retry transaction per window. Measured execution gas is 1,113,356 per cycle, identical over 10 cycles. An estimated execution gas after a cold-access adjustment, adding EIP-2929 cold surcharges for the 13 accounts and 112 storage slots one cycle touches, is about 1,367,356; this is an estimate, not a measurement or a strict upper bound. Adding the 21,000 base gas, at 0.020 gwei ([Arbiscan](https://arbiscan.io/gasTracker), 7 Nov 2026) and ETH at $2,243.46 ([DappAtlas, citing CoinGecko](https://dappatlas.com/tools/gas-tracker/arbitrum/), 7 Nov 2026), one cycle costs about $0.05–0.06:

| Period | Cycles | Third-party cost |
|---|---:|---:|
| D1 / D2 lock in the tests | 15 | $0.76–0.93 |
| One hour | 59 | $3.00–3.68 |
| One day | 1,417 | $72–88 |

L1 data posting and extra inclusion fees are not included.

The program's griefing rule: "spending USD 1 to cause USD 10 or less in damage is Griefing, while spending USD 1 to cause USD 100 or more in damage is not." Cost and damage depend on how long the lock lasts and how far the price moves meanwhile. In D1 and D2, 15 cycles (under $1 of gas) keep the owner locked out while the position loses 2,000.80–2,985.70 USDG more than without the third party, a ratio well above 100. In the 24-hour run the price stays at 1.20: the third party spends about $72–88 and the effect is the freeze itself, with no further loss. The damage therefore requires the price to keep moving against the position during the lock; the third party does not cause that move, it blocks the owner's documented exit while it happens.

The program states: "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." If triage applies the program's griefing rule, or accepts a duration shorter than 24 hours, the same evidence meets Medium (Temporary freezing of funds for at least 1 hour).

## Recommended fix

Let the owner stage new limits while the protection is `Triggered`, so the next attempt uses them, while the live attempt keeps its committed bounds. A prototype:

```solidity
// in amendLatchedPositionProtectionCloseLimits, after loading the owned protection:
if (protection.status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
    // Stage limits for the next attempt; the live attempt keeps its committed bounds.
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
    return;
}
```

Two regression tests start after the third party's retry, with a plain EOA owner (P1, P2 in the PoC). Both fail on the original code with `ProtectionNotLatched` and pass with the change: the next attempt carries the owner's new cap and the position closes. In the repository's `PositionProtection.t.sol` suite, the change passes 103 of 104 tests. The one failure, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, asserts the exact rule the change relaxes; the part of it that checks the live order keeps its committed limits still holds (P2). The function's NatSpec, which currently promises amendment only in `Latched`, should be updated with the change. An owner-only window after a relatch would be an alternative.

## Proof of Concept

## Step-by-step

All results come from local Foundry tests at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. No public network was used.

Configuration: the release parameters from `DEPLOYMENT.md` and the deploy script (margins 10/20 bps, execution fee 4 bps, Senior cap 80%, 1 USDG seeds, protection trigger bounty, router and oracle values, 60-second execution window). `HT1V6Base._assertLaunchConfig` asserts each one.

1. Alice opens a LONG of 10,000 at 1.00 with 5,000 USDG margin and a stop-loss at 1.10 (in this protocol a LONG loses when the price rises). Her protection's `maxGrossAccountDebitAssets` is 1.5 times the close debit at the stop: 1,505.85 USDG. Control G1: executed at the stop, the close debits 1,003.90 USDG and succeeds.

2. The stop fires and the price gaps to 1.20. The close would now debit 2,004.30 USDG. An honest keeper executes it inside the 60-second window and it fails with `ConstraintViolation/GrossAccountDebit`. Protection becomes `Latched` and the position stays open (G2).

3. Before Alice amends, a third party (`RetryCycler`, a plain helper contract) calls `retryPositionProtectionClose`. Protection is `Triggered` again, with a new close order and the same cap.

4. Alice tries every exit. Each reverts with an exact selector:

```
amendLatchedPositionProtectionCloseLimits  -> OrderRouter__ProtectionNotLatched()   0x5b54e624
cancelPositionProtection                   -> OrderRouter__ProtectionNotArmed()     0x2dee7d8f
commitOrder (ordinary close)               -> OrderRouter__ProtectionActive()       0x3a0535a3
```

5. The new order cannot fill, because the cap is unchanged. When its window ends, the third party expires it and retries in one transaction. `Latched` never appears between blocks, and Alice's three calls revert again.

6. Step 5 repeats 1,417 times. Halfway through, a second honest keeper execution fails the same way and relatches, and the third party retries again before Alice. The lock lasts 86,439 seconds (24 h 39 s) with the window still at 60 seconds (asserted).

7. At the end, with a fresh oracle at 1.20:

| | Value |
|---|---|
| Position size | 10,000 (still open) |
| Free margin (500 USDG extra deposit) | withdrawable: 500 USDG |
| Position-backed balance | 4,995.50 USDG; `withdraw(1)` reverts `MarginClearinghouse__InsufficientFreeEquity()` 0x024ec6ee |
| Control from the same state, no third party: expire, amend, retry, keeper closes at 1.20, withdraw | 3,491.20 USDG withdrawn in total (the snapshot is taken before the free-margin withdrawal, so this includes the 500 USDG); size 0 |

## Who can escape (EIP-7702)

`executeOrder` and `expireOrder` are permissionless, so an owner who can batch calls in one transaction escapes even after the cycle has started:

| Test | Owner action in one transaction | Result |
|---|---|---|
| E4a | executeOrder(live order) fails on the cap → Latched; amend; retry | new order executes at 1.20; closed |
| E4b | after the deadline: expireOrder; amend; retry | closed |
| E4c | the third party wins the expiry block; the owner repeats E4a on the new live order | closed |
| d | a batch that names an order the third party already replaced | reverts `OrderRouter__OrderNotPending()` (stale target) |
| E4d | plain EOA: its own executeOrder relatches; a third-party retry lands before its amend | amend reverts `ProtectionNotLatched`; position open |

## Controls

| Test | Result |
|---|---|
| X1, ordinary close during the lock (both commit overloads) | `ProtectionActive`; no order created |
| X2, atomic cancel-and-close while `Triggered` | `ProtectionNotArmed` |
| X2C, control while `Armed` | atomic cancel-and-close executes; size 0 |
| X3, free margin during the lock | 250 of the 500 USDG free margin withdrawn; withdrawing the rest of the balance reverts `InsufficientFreeEquity` |
| X4, owner expires the order herself | `Latched`, but cancel-and-close, ordinary close and cancel still fail |
| X5a, retry without amending | keeper at 1.20 fails on the cap (2,004.30 > 1,505.85 USDG); relatches |
| X5b, expire, amend, retry as three owner transactions, no third party | cap raised to 25,000 USDG; keeper executes at 1.20; size 0 |
| a, three cycles | amend, cancel and close fail each time with the selectors above |
| b, 5,000 USDG deposit during the lock | owner paths still fail with the same selectors |
| c, 7702 batch before any cycle | expire, amend, retry; closed at 1.20 |

## Damage while the lock lasts (HT1V7Damage)

Same position, cap and gap. "Received" is the owner's wallet USDG after withdrawing everything `getWithdrawableAssets` allows, measured from the moment the stop fires.

| Branch | What happens | Owner receives | vs control |
|---|---|---:|---:|
| STOP | the order executes at 1.10 | 3,991.60 | +1,000.40 |
| CONTROL | fails at 1.20 on the cap → `Latched`; no third party; owner amends and retries; keeper closes at 1.20 | 2,991.20 | 0 |
| D1 | gap failure → third-party retry → cycle: 5 cycles at 1.20, fresh mark 1.30 and 5 cycles, fresh mark 1.40 and 5 cycles; then the cycle stops and the owner expires, amends, retries; keeper closes at 1.40 | 990.40 | −2,000.80 |
| D2 | same cycle up to 1.40; then the first liquidatable price on a 0.001 grid (1.498); `router.executeLiquidation` with fresh price data while the protection is `Triggered` and the third party's order is live | 5.50 | −2,985.70 |

D2 details: `previewLiquidation` reports liquidatable with equity 5.499965 USDG and a 5 USDG keeper bounty; `executeLiquidation` does not revert; `PositionLiquidated` is emitted for the owner at 1.498; position size 0; protection status `Liquidated`; the third party's order ends `Failed`; the owner withdraws the remaining 5.499965 USDG.

## Third-party cost (HT1V7Gas)

The measured call is `RetryCycler.atomicExpireAndRetry` (`router.expireOrder` + `retryPositionProtectionClose`) under the release configuration. Execution gas only; the 21,000 base gas and calldata are not included.

| | Value |
|---|---:|
| G1: 10 consecutive cycles, `vm.lastCallGas().gasTotalUsed` | 1,113,356 every cycle |
| G1: same cycles, `gasleft()` delta (includes the test's call overhead) | 1,116,951 average |
| G2: accounts / storage slots touched by one cycle | 13 / 112 |
| G2: estimated execution gas after a cold-access adjustment, warm gas + EIP-2929 surcharges ((13 − 1) × 2,500 + 112 × 2,000) | about 1,367,356 (estimate, not a strict upper bound) |

`vm.cool()` and `--isolate` did not change the measured gas on this forge version, so the cold figure is derived from the access counts rather than measured.

## Fix prototype

With the change from the Description applied, P1 and P2 (both start after the third party's retry, plain EOA owner) pass: the owner's new cap is used by the next attempt and the position closes. On the original code both fail with `ProtectionNotLatched`. Tests that assert the original rule (a, b, E4d, H24) fail on the patched copy, as expected.

## Running the tests

Save the five files below, plus the two posted as a comment on this report (`HT1V7Gas.t.sol`, `HT1V6OwnerAmendWhileTriggered.t.sol`), in `packages/perps/test/perps/spec/trader/` (they import `./PositionProtection.t.sol` from that folder, so they must not go in a subfolder). From the repository root:

```
forge test --offline --root packages/perps --match-path 'test/perps/spec/trader/HT1V6*.t.sol' --match-test test_HT1V6 --gas-limit 18446744073709551615 -vv
forge test --offline --root packages/perps --match-path 'test/perps/spec/trader/HT1V7*.t.sol' --match-test test_HT1V7 -vv
```

With all seven files, on the original code: the HT1V6 files give 20 passing tests, and P1, P2 fail (they test the fix); the HT1V7 files give 5 passing tests. The gas test and the fix tests (P1, P2) did not fit in this field's 65,536-character limit and are posted as a comment right after submission.

### RetryCycler.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Companion helper only; no additional scenario tests. Run the importing test files with forge test --offline --root packages/perps --match-path 'test/perps/spec/trader/<File>.t.sol' -vv.
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";

/// @notice Test helper. Property under test: once a protection close child fails or expires, the
/// documented `Latched` remedy (owner amends close limits) stays available to the owner. This helper
/// sequences permissionless calls (execute/expire, then retryPositionProtectionClose) to check whether
/// `Latched` remains observable between transactions.
contract RetryCycler {

    OrderRouter public immutable router;
    IPositionProtectionActions public immutable protectionActions;

    constructor(
        OrderRouter router_,
        IPositionProtectionActions protectionActions_
    ) {
        router = router_;
        protectionActions = protectionActions_;
    }

    function atomicFailAndRetry(
        uint64 childOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory execResult, uint64 newChildOrderId) {
        execResult = router.executeOrder{value: msg.value}(childOrderId, pythUpdateData);
        newChildOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    function atomicExpireAndRetry(
        uint64 childOrderId,
        uint64 protectionId
    ) external returns (OrderV3Types.ExecutionResult memory execResult, uint64 newChildOrderId) {
        execResult = router.expireOrder(childOrderId);
        newChildOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    /// @notice Single permissionless retry, sent as its own transaction (not atomic with any execution).
    function retryClose(
        uint64 protectionId
    ) external returns (uint64) {
        return protectionActions.retryPositionProtectionClose(protectionId);
    }

    receive() external payable {}

}
```

### HT1V6Base.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Shared launch-configuration base for the HT1 v6 tests (helper only; no tests of its own).
// Place in packages/perps/test/perps/spec/trader/ next to PositionProtection.t.sol and RetryCycler.t.sol.
//
// Launch parameters (packages/perps/DEPLOYMENT.md "Initial release parameters" and
// script/DeployPerpsArbitrumSepolia.s.sol): risk params set through the Engine constructor (_riskParams),
// Senior limits $40M / 80% and the Router/Oracle release values set through the governed 72h paths, 1 USDG seeds.
// Sign convention (PositionProtection.t.sol:65-66): a LONG loses when the price RISES (stop-loss above entry).
import {RetryCycler} from "./RetryCycler.t.sol";
import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {Vm} from "forge-std/Vm.sol";

/// @dev Minimal EIP-7702 delegation: the EOA calls itself and executes N calls atomically (reverts as a whole).
contract BatchExec7702V6 {
    struct Call {
        address target;
        uint256 value;
        bytes data;
    }

    function exec(Call[] calldata calls) external payable {
        require(msg.sender == address(this), "only self");
        for (uint256 i; i < calls.length; ++i) {
            (bool ok, bytes memory ret) = calls[i].target.call{value: calls[i].value}(calls[i].data);
            if (!ok) {
                assembly {
                    revert(add(ret, 32), mload(ret))
                }
            }
        }
    }

    receive() external payable {}
}

abstract contract HT1V6Base is PositionProtectionTest {
    uint256 internal constant OWNER_PK = 0xA11CE7702;
    uint256 internal constant SIZE = 10_000e18; // 10,000 synthetic units; multiple of SIZE_QUANTUM (1e20)
    uint256 internal constant MARGIN = 5_000e6; // survives the gap price below with launch margins 10/20 bps
    uint256 internal constant ENTRY = 1e8;
    uint256 internal constant STOP = 1.1e8; // LONG stop-loss (LONG loses as price rises)
    uint256 internal constant GAP = 1.2e8; // price jumps past the stop before the child executes
    uint256 internal constant WIDE = 25_000e6;

    struct Exec {
        uint8 status;
        uint8 reason;
        uint8 constraint;
        uint256 actual;
        uint256 limit;
        bool receipt;
    }

    // ---- Launch configuration ----------------------------------------------------------------------------
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

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

    /// PositionProtectionTest.setUp is not virtual: every HT1 v6 test applies the launch configuration first.
    modifier launched() {
        _launch();
        _;
    }

    function _launch() internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.maxPendingOrders = 5;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(rc);
        // The governed paths warp 2 x 72h (into a weekend, oracle frozen): move to the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(ENTRY);
        _assertLaunchConfig();
    }

    function _assertLaunchConfig() internal view {
        CfdTypes.RiskParams memory rp = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(rp.vpiFactor, 0.01e18, "vpiFactor");
        assertEq(rp.maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(rp.maintMarginBps, 10, "maintMarginBps");
        assertEq(rp.initMarginBps, 20, "initMarginBps");
        assertEq(rp.fadMarginBps, 300, "fadMarginBps");
        assertEq(rp.baseCarryBps, 500, "baseCarryBps");
        assertEq(rp.minBountyAssets, 1e6, "minBountyAssets");
        assertEq(rp.bountyBps, 10, "bountyBps");
        assertEq(rp.keeperShareBps, 5000, "keeperShareBps");
        assertEq(rp.protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "fadRunwaySeconds");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(router.maxExecutionWindowSeconds(), 60, "maxExecutionWindowSeconds");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), 5, "maxPendingOrders");
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "positionProtectionTriggerBountyAssets");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "basketMaxConfidenceRatioBps");
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "juniorMaintenanceFeeAprBps");
        assertEq(SIZE % CfdTypes.SIZE_QUANTUM, 0, "size quantum");
    }

    // ---- Helpers -------------------------------------------------------------------------------------------
    function _owner() internal pure returns (address) {
        return vm.addr(OWNER_PK);
    }

    function _status(uint64 id) internal view returns (uint8) {
        return uint8(protectionViews.getPositionProtection(id).status);
    }

    function _size(address a) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _wide() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits(WIDE, WIDE, WIDE, WIDE);
    }

    /// Opens the LONG and arms a stop-loss with the given gross-debit cap (other limits unlimited).
    function _setupArmed(address owner, uint256 grossCap) internal returns (uint64 protectionId) {
        _fundTrader(owner, 20_000e6);
        _open(owner, CfdTypes.Side.LONG, SIZE, MARGIN, ENTRY);
        _withdrawAllFreeSettlement(owner);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, STOP);
        params.closeLimits.maxGrossAccountDebitAssets = grossCap;
        _fundTrader(owner, _totalProtectionBountyAssets());
        vm.prank(owner);
        protectionId = protectionActions.createPositionProtection(params);
    }

    /// Gross account debit of the stop close at STOP, measured with a probe protection whose cap is 1 base unit;
    /// state is restored afterwards.
    function _probeStopDebit(address owner) internal returns (uint256 debit) {
        uint256 snap = vm.snapshotState();
        uint64 pid = _setupArmed(owner, 1);
        uint64 child = _triggerAt(pid, STOP);
        Exec memory e = _execAt(child, STOP, EXECUTION_KEEPER);
        assertEq(e.constraint, uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "probe: GrossAccountDebit");
        debit = e.actual;
        vm.revertToState(snap);
    }

    /// Cap that covers the close at the stop price with costs plus 50% headroom.
    function _capWithHeadroom(address owner) internal returns (uint256 stopDebit, uint256 cap) {
        stopDebit = _probeStopDebit(owner);
        cap = stopDebit * 3 / 2;
    }

    function _execAt(uint64 orderId, uint256 price, address executor) internal returns (Exec memory e) {
        bytes[] memory pyth = _mockPythUpdateData(price); // advances one block and one second
        vm.recordLogs();
        vm.prank(executor);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, pyth);
        e.status = uint8(res.status);
        e.reason = uint8(res.terminalReason);
        _readReceipt(vm.getRecordedLogs(), orderId, e);
    }

    function _readReceipt(Vm.Log[] memory logs, uint64 orderId, Exec memory e) internal pure {
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].topics.length >= 2 && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == uint256(orderId)
            ) {
                (,,, OrderV3Types.OrderReceipt memory r) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                e.constraint = uint8(r.failure.constraint);
                e.actual = r.failure.actual;
                e.limit = r.failure.limit;
                e.receipt = true;
            }
        }
    }

    function _logExec(string memory tag, Exec memory e) internal {
        emit log_string(tag);
        emit log_named_uint("  LifecycleStatus (2=Executed,3=Failed)", e.status);
        emit log_named_uint("  TerminalReason (1=Executed,8=ConstraintViolation)", e.reason);
        emit log_named_uint("  ConstraintKind (3=GrossAccountDebit)", e.constraint);
        emit log_named_uint("  failure.actual", e.actual);
        emit log_named_uint("  failure.limit", e.limit);
    }

    function _deadline(uint64 child) internal view returns (uint64) {
        return router.lifecycleBook().orderTiming(child).executionDeadline;
    }

    function _warpPastDeadline(uint64 child) internal {
        vm.warp(uint256(_deadline(child)) + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _newCycler() internal returns (RetryCycler bot) {
        bot = new RetryCycler(router, protectionActions);
        vm.deal(address(bot), 10 ether);
    }

    /// One third-party cycle: expire the child after its deadline and retry in the same transaction.
    function _cycle(RetryCycler bot, uint64 child, uint64 pid) internal returns (uint64 next) {
        _warpPastDeadline(child);
        OrderV3Types.ExecutionResult memory r;
        (r, next) = bot.atomicExpireAndRetry(child, pid);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "cycle: previous child Failed");
        assertEq(_status(pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "cycle: Triggered");
    }

    /// Standard locked setup: cap = 1.5 x stop debit, trigger at STOP, one third-party cycle (child live).
    function _setupLocked(address owner)
        internal
        returns (uint64 pid, uint64 child, RetryCycler bot, uint256 stopDebit, uint256 cap)
    {
        (stopDebit, cap) = _capWithHeadroom(owner);
        pid = _setupArmed(owner, cap);
        bot = _newCycler();
        child = _triggerAt(pid, STOP);
        child = _cycle(bot, child, pid);
    }
}
```

### HT1V6H24.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// HT1 v6 item 3: funds unavailable for 24 hours, launch configuration (see HT1V6Base), honest gap scenario.
// Cap = 1.5 x stop-close debit; trigger at the stop (1.10); the price then gaps to 1.20 and stays there.
// Two honest keeper executions (right after the trigger and halfway) fail on the cap and relatch; a third-party
// retry follows each; the rest are expire-and-retry cycles with the unchanged 60-second window.
// At the end, with a fresh oracle: position still open; position margin not withdrawable (exact selector);
// free margin withdrawable. Control from the same state: expire -> amend -> retry -> close -> withdraw.
// Run: forge test --offline --root packages/perps --match-path 'test/perps/spec/trader/HT1V6H24.t.sol' --match-test 'test_HT1V6' --gas-limit 18446744073709551615 -vv
import {RetryCycler} from "./RetryCycler.t.sol";
import {HT1V6Base} from "./HT1V6Base.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

contract HT1V6H24Test is HT1V6Base {
    uint256 internal constant CYCLES_24H = 1417; // ceil(86400 / 61)
    uint256 internal constant FREE_EXTRA = 500e6;

    function _ownerPathsFail(address owner, uint64 pid) internal {
        vm.startPrank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _wide());
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(pid);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.stopPrank();
    }

    function _keeperFailThenRetry(address owner, uint64 pid, uint64 child, uint256 cap, RetryCycler bot)
        internal
        returns (uint64 next)
    {
        assertGe(uint256(_deadline(child)), block.timestamp + 1, "keeper: child live");
        Exec memory e = _execAt(child, GAP, EXECUTION_KEEPER);
        _logExec("H24 honest keeper execution at GAP", e);
        assertEq(e.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "keeper: ConstraintViolation");
        assertEq(e.constraint, uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "keeper: GrossAccountDebit");
        assertEq(e.limit, cap, "keeper: limit is cap");
        assertGt(e.actual, cap, "keeper: debit exceeds cap");
        assertEq(_status(pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "keeper: Latched");
        next = bot.retryClose(pid);
        assertEq(_status(pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "retry: Triggered");
        _ownerPathsFail(owner, pid);
    }

    function _logAccount(string memory tag, address owner) internal {
        ICfdEngineTypes.AccountCollateralView memory v = engineAccountLens.getAccountCollateralView(owner);
        (uint256 size, uint256 margin,,,,,) = engine.positions(owner);
        emit log_string(tag);
        emit log_named_uint("  position size", size);
        emit log_named_uint("  position margin", margin);
        emit log_named_uint("  settlementBalance", v.settlementBalanceAssets);
        emit log_named_uint("  activePositionMargin", v.activePositionMarginAssets);
        emit log_named_uint("  otherLockedMargin", v.otherLockedMarginAssets);
        emit log_named_uint("  freeSettlement", v.freeSettlementAssets);
        emit log_named_uint("  accountEquity", v.accountEquityAssets);
        emit log_named_uint("  getWithdrawableAssets", engineAccountLens.getWithdrawableAssets(owner));
    }

    function test_HT1V6_H24_GapLock_FundsUnavailable() public launched {
        address owner = _owner();
        (uint256 stopDebit, uint256 cap) = _capWithHeadroom(owner);
        uint64 pid = _setupArmed(owner, cap);
        RetryCycler bot = _newCycler();
        uint64 child = _triggerAt(pid, STOP);
        _fundTrader(owner, FREE_EXTRA); // free margin outside the position
        uint256 startTs = router.lifecycleBook().orderTiming(child).commitTimestamp;
        emit log_named_uint("H24 stop-close debit", stopDebit);
        emit log_named_uint("H24 cap", cap);
        _logAccount("H24 START (after trigger)", owner);

        vm.pauseGasMetering();
        child = _keeperFailThenRetry(owner, pid, child, cap, bot);
        for (uint256 i; i < CYCLES_24H; ++i) {
            child = _cycle(bot, child, pid);
            _ownerPathsFail(owner, pid);
            if (i == CYCLES_24H / 2) child = _keeperFailThenRetry(owner, pid, child, cap, bot);
        }
        vm.resumeGasMetering();

        uint256 dur = router.lifecycleBook().orderTiming(child).commitTimestamp - startTs;
        emit log_named_uint("H24 lock duration (s)", dur);
        assertGe(dur, 86_400, "lock >= 24h");
        assertEq(router.maxExecutionWindowSeconds(), 60, "window 60 s");

        // Fresh oracle at the end (permissionless mark update at the gap price).
        _refreshMark(GAP);
        assertEq(uint256(engine.lastMarkTime()), block.timestamp, "fresh mark");
        assertGt(_size(owner), 0, "position still open after 24h");
        _logAccount("H24 END (fresh mark)", owner);

        uint256 snap = vm.snapshotState();
        // Locked branch. Step 1: withdraw exactly the free margin; this checkpoints the 24 h carry first.
        uint256 bal0 = _settlementBalance(owner);
        uint256 withdrawable = engineAccountLens.getWithdrawableAssets(owner);
        uint256 t0 = settlementToken.balanceOf(owner);
        vm.prank(owner);
        clearinghouse.withdraw(owner, withdrawable);
        uint256 got = settlementToken.balanceOf(owner) - t0;
        uint256 bal1 = _settlementBalance(owner);
        emit log_named_uint("H24 withdrawn free margin", got);
        emit log_named_uint("H24 carry realised at checkpoint (bal0 - withdrawn - bal1)", bal0 - got - bal1);
        _logAccount("H24 END after free withdrawal (carry checkpointed)", owner);
        // Step 2: the margin backing the position is refused, with the exact selector.
        emit log_named_uint("H24 locked balance (attempted withdrawal)", bal1);
        vm.prank(owner);
        vm.expectRevert(IMarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdraw(owner, bal1);
        vm.prank(owner);
        vm.expectRevert(IMarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdraw(owner, 1);
        assertEq(got, withdrawable, "free margin fully withdrawn");
        assertGt(_size(owner), 0, "still open");
        vm.revertToState(snap);

        // Control, same state: owner (no third party) expires, amends, retries; keeper closes; owner withdraws.
        _warpPastDeadline(child);
        vm.prank(owner);
        router.expireOrder(child);
        vm.prank(owner);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _wide());
        vm.prank(owner);
        uint64 nc = protectionActions.retryPositionProtectionClose(pid);
        Exec memory e = _execAt(nc, GAP, EXECUTION_KEEPER);
        _logExec("H24 CONTROL close at GAP", e);
        assertEq(e.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: closes");
        assertEq(_size(owner), 0, "control: closed");
        _logAccount("H24 CONTROL after close", owner);
        uint256 w = _freeSettlementAssets(owner);
        uint256 t1 = settlementToken.balanceOf(owner);
        vm.prank(owner);
        clearinghouse.withdraw(owner, w);
        emit log_named_uint("H24 CONTROL withdrawn after close", settlementToken.balanceOf(owner) - t1);
        assertEq(settlementToken.balanceOf(owner) - t1, w, "control: full free balance withdrawn");
    }
}
```

### HT1V7Damage.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// HT1 v7 item 1: economic damage while the price keeps moving against the owner during the lock.
// Launch configuration and helpers from HT1V6Base (same asserts). LONG 10,000 at 1.00, margin 5,000 USDG,
// stop-loss 1.10 (a LONG loses as the price rises), cap = 1.5 x the stop-close debit, gap to 1.20.
// Common state A: armed, triggered at 1.10, child live. "Received" = owner wallet USDG after withdrawing
// everything getWithdrawableAssets allows, minus the owner wallet balance at state A.
//   STOP    : the child executes at 1.10 (the stop works).
//   CONTROL : the child fails at 1.20 (cap) -> Latched; no third party; owner amends + retries at once;
//             keeper closes at 1.20 (same control as v6).
//   S       : the child fails at 1.20 -> Latched -> third-party retry -> expire-and-retry cycle runs.
//   D1      : from S, fresh marks 1.30 then 1.40 (5 cycles per step); the cycle stops; owner expires, amends,
//             retries; keeper closes at 1.40.
//   D2      : from S, 1.30, 1.40, then the first price (0.001 grid) at which the engine preview says liquidatable;
//             liquidation through router.executeLiquidation with fresh Pyth data while the protection is Triggered.
// Run: forge test --offline --root packages/perps --match-path 'test/perps/spec/trader/HT1V7Damage.t.sol' --match-test test_HT1V7 -vv
import {RetryCycler} from "./RetryCycler.t.sol";
import {HT1V6Base} from "./HT1V6Base.t.sol";
import {Vm} from "forge-std/Vm.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

contract HT1V7DamageTest is HT1V6Base {
    uint256 internal constant STEPS_CYCLES = 5;
    bytes32 internal constant POSITION_LIQUIDATED_TOPIC =
        keccak256("PositionLiquidated(address,uint8,uint256,uint256,uint256)");
    bytes32 internal constant LIQ_BATCH_ITEM_TOPIC =
        keccak256("LiquidationBatchItem(uint256,address,uint8,uint256,bytes4)");

    struct StateA {
        address owner;
        uint64 pid;
        uint64 child;
        uint256 cap;
        uint256 wallet0;
    }

    function _stateA() internal returns (StateA memory a) {
        a.owner = _owner();
        (, a.cap) = _capWithHeadroom(a.owner);
        a.pid = _setupArmed(a.owner, a.cap);
        a.child = _triggerAt(a.pid, STOP);
        a.wallet0 = settlementToken.balanceOf(a.owner);
    }

    function _logAcct(string memory tag, address owner) internal {
        ICfdEngineTypes.AccountCollateralView memory v = engineAccountLens.getAccountCollateralView(owner);
        emit log_string(tag);
        emit log_named_uint("  size", _size(owner));
        emit log_named_uint("  settlementBalance", v.settlementBalanceAssets);
        emit log_named_uint("  activePositionMargin", v.activePositionMarginAssets);
        emit log_named_uint("  otherLockedMargin", v.otherLockedMarginAssets);
        emit log_named_uint("  traderClaimBalance", v.traderClaimBalanceAssets);
        emit log_named_uint("  getWithdrawableAssets", engineAccountLens.getWithdrawableAssets(owner));
    }

    /// Withdraws everything withdrawable and returns wallet USDG received since state A.
    function _withdrawAllReceived(StateA memory a, string memory tag) internal returns (uint256 received) {
        uint256 w = engineAccountLens.getWithdrawableAssets(a.owner);
        if (w != 0) {
            vm.prank(a.owner);
            clearinghouse.withdraw(a.owner, w);
        }
        received = settlementToken.balanceOf(a.owner) - a.wallet0;
        _logAcct(string.concat(tag, " after withdrawing all withdrawable"), a.owner);
        emit log_named_uint(string.concat(tag, " owner USDG received"), received);
        assertEq(engineAccountLens.getWithdrawableAssets(a.owner), 0, "nothing left withdrawable");
    }

    function _branchStop(StateA memory a) internal returns (uint256) {
        Exec memory e = _execAt(a.child, STOP, EXECUTION_KEEPER);
        _logExec("STOP close at 1.10", e);
        assertEq(e.status, uint8(OrderV3Types.LifecycleStatus.Executed), "stop: executes");
        assertEq(_size(a.owner), 0, "stop: closed");
        return _withdrawAllReceived(a, "STOP");
    }

    function _gapFail(StateA memory a) internal {
        Exec memory e = _execAt(a.child, GAP, EXECUTION_KEEPER);
        _logExec("gap execution at 1.20", e);
        assertEq(e.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "gap: ConstraintViolation");
        assertEq(e.constraint, uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "gap: GrossAccountDebit");
        assertGt(e.actual, a.cap, "gap: debit > cap");
        assertEq(_status(a.pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "gap: Latched");
    }

    function _ownerCloseAt(StateA memory a, uint256 price, string memory tag) internal returns (uint256) {
        vm.prank(a.owner);
        protectionActions.amendLatchedPositionProtectionCloseLimits(a.pid, _wide());
        vm.prank(a.owner);
        uint64 nc = protectionActions.retryPositionProtectionClose(a.pid);
        Exec memory e = _execAt(nc, price, EXECUTION_KEEPER);
        _logExec(string.concat(tag, " keeper close"), e);
        assertEq(e.status, uint8(OrderV3Types.LifecycleStatus.Executed), "owner close executes");
        assertEq(_size(a.owner), 0, "closed");
        return _withdrawAllReceived(a, tag);
    }

    function _branchControl(StateA memory a) internal returns (uint256) {
        _gapFail(a);
        return _ownerCloseAt(a, GAP, "CONTROL");
    }

    /// From state A to S: gap failure, third-party retry, cycle running. Returns the live child.
    function _toS(StateA memory a, RetryCycler bot) internal returns (uint64 child) {
        _gapFail(a);
        child = bot.retryClose(a.pid);
        assertEq(_status(a.pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "S: Triggered");
    }

    function _rise(StateA memory a, RetryCycler bot, uint64 child, uint256[2] memory prices)
        internal
        returns (uint64)
    {
        for (uint256 i; i < STEPS_CYCLES; ++i) child = _cycle(bot, child, a.pid);
        for (uint256 s; s < prices.length; ++s) {
            _refreshMark(prices[s]);
            assertEq(uint256(engine.lastMarkTime()), block.timestamp, "fresh mark at step");
            emit log_named_uint("fresh mark step price", prices[s]);
            for (uint256 i; i < STEPS_CYCLES; ++i) child = _cycle(bot, child, a.pid);
            assertGt(_size(a.owner), 0, "open during lock");
        }
        return child;
    }

    function _branchD1(StateA memory a) internal returns (uint256) {
        RetryCycler bot = _newCycler();
        uint64 child = _toS(a, bot);
        child = _rise(a, bot, child, [uint256(1.3e8), uint256(1.4e8)]);
        _logAcct("D1 at 1.40 before owner exit", a.owner);
        // The cycle stops: the owner expires the last child after its deadline, amends and retries.
        _warpPastDeadline(child);
        vm.prank(a.owner);
        router.expireOrder(child);
        return _ownerCloseAt(a, 1.4e8, "D1");
    }

    function _branchD2(StateA memory a) internal returns (uint256 received, uint256 liqPrice) {
        RetryCycler bot = _newCycler();
        uint64 child = _toS(a, bot);
        child = _rise(a, bot, child, [uint256(1.3e8), uint256(1.4e8)]);
        for (liqPrice = 1.4e8; liqPrice <= 2e8; liqPrice += 1e5) {
            if (engineLens.previewLiquidation(a.owner, liqPrice).liquidatable) break;
        }
        ICfdEngineTypes.LiquidationPreview memory pv = engineLens.previewLiquidation(a.owner, liqPrice);
        emit log_named_uint("D2 first liquidatable price (0.001 grid)", liqPrice);
        emit log_named_int("D2 preview equity", pv.equityAssets);
        emit log_named_uint("D2 preview keeper bounty", pv.keeperBountyAssets);
        emit log_named_uint("D2 preview immediate payout", pv.immediatePayoutAssets);
        assertTrue(pv.liquidatable, "D2: liquidatable price found");
        assertEq(_status(a.pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "D2: Triggered");
        assertGe(uint256(_deadline(child)), block.timestamp, "D2: cycler child live");
        _logAcct("D2 before liquidation", a.owner);

        _mockPythUpdateData(liqPrice);
        bytes[] memory data = _livePriceData(liqPrice);
        vm.recordLogs();
        vm.prank(EXECUTION_KEEPER);
        (bool ok, bytes memory ret) = address(router).call(abi.encodeCall(router.executeLiquidation, (a.owner, data)));
        if (!ok) emit log_named_bytes("D2 executeLiquidation REVERT data", ret);
        assertTrue(ok, "D2: executeLiquidation did not revert");
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bool liquidated;
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics[0] == POSITION_LIQUIDATED_TOPIC && address(uint160(uint256(logs[i].topics[1]))) == a.owner) {
                liquidated = true;
                (, uint256 sz, uint256 px, uint256 kb) = abi.decode(logs[i].data, (uint8, uint256, uint256, uint256));
                emit log_named_uint("D2 PositionLiquidated size", sz);
                emit log_named_uint("D2 PositionLiquidated price", px);
                emit log_named_uint("D2 PositionLiquidated keeperBounty", kb);
            }
            if (logs[i].topics[0] == LIQ_BATCH_ITEM_TOPIC) {
                (uint8 result, uint256 kb2, bytes4 sel) = abi.decode(logs[i].data, (uint8, uint256, bytes4));
                emit log_named_uint("D2 LiquidationBatchItem result", result);
                emit log_named_uint("D2 LiquidationBatchItem keeperBounty", kb2);
                emit log_named_bytes32("D2 LiquidationBatchItem errorSelector", bytes32(sel));
            }
        }
        assertTrue(liquidated, "D2: PositionLiquidated emitted while protection Triggered");
        assertEq(_size(a.owner), 0, "D2: position removed");
        emit log_named_uint("D2 protection status after", _status(a.pid));
        assertEq(_status(a.pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated), "D2: Liquidated");
        assertEq(
            uint8(router.lifecycleBook().lifecycleStatus(child)),
            uint8(OrderV3Types.LifecycleStatus.Failed),
            "D2: cycler child terminally failed by liquidation cleanup"
        );
        emit log_named_uint("D2 lifecycle status of cycler child", uint8(router.lifecycleBook().lifecycleStatus(child)));
        received = _withdrawAllReceived(a, "D2");
    }

    function _measureCommon() internal returns (StateA memory a, uint256 stopRecv, uint256 controlRecv) {
        a = _stateA();
        uint256 snap = vm.snapshotState();
        stopRecv = _branchStop(a);
        vm.revertToState(snap);
        controlRecv = _branchControl(a);
        vm.revertToState(snap);
    }

    function _deltas(string memory tag, uint256 branch, uint256 controlRecv, uint256 stopRecv) internal {
        emit log_named_int(string.concat(tag, " minus CONTROL"), int256(branch) - int256(controlRecv));
        emit log_named_int(string.concat(tag, " minus STOP"), int256(branch) - int256(stopRecv));
    }

    function test_HT1V7_D0_StopAndControl() public launched {
        (, uint256 stopRecv, uint256 controlRecv) = _measureCommon();
        _deltas("CONTROL", controlRecv, controlRecv, stopRecv);
        assertLt(controlRecv, stopRecv, "control receives less than a working stop");
    }

    function test_HT1V7_D1_ModerateRise() public launched {
        (StateA memory a, uint256 stopRecv, uint256 controlRecv) = _measureCommon();
        uint256 d1 = _branchD1(a);
        _deltas("D1", d1, controlRecv, stopRecv);
        assertLt(d1, controlRecv, "D1 receives less than control");
    }

    function test_HT1V7_D2_LiquidationWhileTriggered() public launched {
        (StateA memory a, uint256 stopRecv, uint256 controlRecv) = _measureCommon();
        (uint256 d2,) = _branchD2(a);
        _deltas("D2", d2, controlRecv, stopRecv);
        assertLt(d2, controlRecv, "D2 receives less than control");
    }
}
```

### HT1V6Controls.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// HT1 v6, launch configuration (see HT1V6Base). Items 2, 4 and 6 of the rework.
// Property: after the stop-loss fires, the owner can always reach a terminal exit and withdraw free funds.
// Scenario: LONG 10,000 units at 1.00, margin 5,000 USDG, stop-loss 1.10 (a LONG loses as the price rises),
// gross-debit cap = 1.5 x the measured debit of the close at the stop price (other limits unlimited).
// Gap case: the child is executed at 1.20 instead of 1.10.
// Run: forge test --offline --root packages/perps --match-path 'test/perps/spec/trader/HT1V6Controls.t.sol' --match-test 'test_HT1V6' -vv
import {RetryCycler} from "./RetryCycler.t.sol";
import {HT1V6Base, BatchExec7702V6} from "./HT1V6Base.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

contract HT1V6ControlsTest is HT1V6Base {
    uint8 internal constant TRIGGERED = uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered);
    uint8 internal constant LATCHED = uint8(PositionProtectionTypes.PositionProtectionStatus.Latched);
    uint8 internal constant P_EXECUTED = uint8(PositionProtectionTypes.PositionProtectionStatus.Executed);
    uint8 internal constant L_EXECUTED = uint8(OrderV3Types.LifecycleStatus.Executed);
    uint8 internal constant L_FAILED = uint8(OrderV3Types.LifecycleStatus.Failed);

    function _assertOwnerPathsFail(address owner, uint64 pid) internal {
        vm.startPrank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _wide());
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(pid);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.stopPrank();
    }

    function _attach7702(address owner) internal {
        BatchExec7702V6 impl = new BatchExec7702V6();
        vm.signAndAttachDelegation(address(impl), OWNER_PK);
        assertGt(owner.code.length, 0, "7702 delegation attached");
    }

    function _amendRetryCalls(uint64 pid) internal view returns (BatchExec7702V6.Call[] memory c) {
        c = new BatchExec7702V6.Call[](2);
        c[0] = BatchExec7702V6.Call(
            address(protectionActions), 0,
            abi.encodeCall(protectionActions.amendLatchedPositionProtectionCloseLimits, (pid, _wide()))
        );
        c[1] = BatchExec7702V6.Call(
            address(protectionActions), 0, abi.encodeCall(protectionActions.retryPositionProtectionClose, (pid))
        );
    }

    function _withFirst(BatchExec7702V6.Call memory first, uint64 pid)
        internal
        view
        returns (BatchExec7702V6.Call[] memory c)
    {
        BatchExec7702V6.Call[] memory tail = _amendRetryCalls(pid);
        c = new BatchExec7702V6.Call[](3);
        c[0] = first;
        c[1] = tail[0];
        c[2] = tail[1];
    }

    function _assertClosedByKeeper(address owner, uint64 pid, string memory tag) internal {
        uint64 newChild = protectionViews.getPositionProtection(pid).linkedOrderId;
        assertEq(protectionViews.closeFinancialLimits(pid).maxGrossAccountDebitAssets, WIDE, "amended cap stored");
        Exec memory e = _execAt(newChild, GAP, EXECUTION_KEEPER);
        _logExec(tag, e);
        assertEq(e.status, L_EXECUTED, "new child executes at the gap price");
        assertEq(_size(owner), 0, "position closed");
        assertEq(_status(pid), P_EXECUTED, "protection Executed");
    }

    // ---- Item 1 ------------------------------------------------------------------------------------------
    function test_HT1V6_00_LaunchConfig() public launched {
        _assertLaunchConfig();
    }

    // ---- Item 2: honest gap scenario ---------------------------------------------------------------------
    function test_HT1V6_G1_Control_ExecAtStop_ClosesWithinCap() public launched {
        address owner = _owner();
        (uint256 stopDebit, uint256 cap) = _capWithHeadroom(owner);
        uint64 pid = _setupArmed(owner, cap);
        uint64 child = _triggerAt(pid, STOP);
        uint256 bal0 = _settlementBalance(owner);
        Exec memory e = _execAt(child, STOP, EXECUTION_KEEPER);
        uint256 bal1 = _settlementBalance(owner);
        emit log_named_uint("G1 stop-close gross debit (probe, same state)", stopDebit);
        emit log_named_uint("G1 owner cap (1.5 x stop debit)", cap);
        emit log_named_uint("G1 settlement balance before", bal0);
        emit log_named_uint("G1 settlement balance after", bal1);
        emit log_named_uint("G1 settlement balance decrease", bal0 - bal1);
        _logExec("G1 execution at STOP", e);
        assertEq(e.status, L_EXECUTED, "G1: executes");
        assertEq(e.reason, uint8(OrderV3Types.TerminalReason.Executed), "G1: Executed");
        assertLe(stopDebit, cap, "G1: debit within cap");
        assertEq(_size(owner), 0, "G1: closed");
        assertEq(_status(pid), P_EXECUTED, "G1: protection Executed");
    }

    function test_HT1V6_G2_GapBeyondStop_ExceedsCap_Latched() public launched {
        address owner = _owner();
        (uint256 stopDebit, uint256 cap) = _capWithHeadroom(owner);
        uint64 pid = _setupArmed(owner, cap);
        uint64 child = _triggerAt(pid, STOP);
        Exec memory e = _execAt(child, GAP, EXECUTION_KEEPER);
        emit log_named_uint("G2 stop-close gross debit (probe)", stopDebit);
        emit log_named_uint("G2 owner cap (1.5 x stop debit)", cap);
        _logExec("G2 execution at GAP", e);
        assertEq(e.status, L_FAILED, "G2: child Failed");
        assertEq(e.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "G2: ConstraintViolation");
        assertEq(e.constraint, uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "G2: GrossAccountDebit");
        assertEq(e.limit, cap, "G2: limit is the owner cap");
        assertGt(e.actual, cap, "G2: debit exceeds cap");
        assertEq(_status(pid), LATCHED, "G2: Latched");
        assertGt(_size(owner), 0, "G2: position open");
    }

    // ---- Item 6: X1-X5b ----------------------------------------------------------------------------------
    function test_HT1V6_X1_OrdinaryCloseDuringLock() public launched {
        address owner = _owner();
        (uint64 pid,,,,) = _setupLocked(owner);
        uint64 nextId = router.nextCommitId();
        OrderV3Types.OrderRequest memory req = _manualCloseRequest(CfdTypes.Side.LONG);
        req.sizeDelta = SIZE;
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(req);
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        assertEq(router.nextCommitId(), nextId, "X1: no order created");
        assertEq(_status(pid), TRIGGERED, "X1: Triggered");
        assertGt(_size(owner), 0, "X1: open");
    }

    function test_HT1V6_X2_AtomicCancelAndCloseDuringLock() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,,) = _setupLocked(owner);
        OrderV3Types.OrderRequest memory req = _manualCloseRequest(CfdTypes.Side.LONG);
        req.sizeDelta = SIZE;
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(pid, req);
        assertEq(_status(pid), TRIGGERED, "X2: Triggered");
        assertEq(protectionViews.getPositionProtection(pid).linkedOrderId, child, "X2: same child");
        assertGt(_size(owner), 0, "X2: open");
    }

    function test_HT1V6_X2C_Control_ArmedAtomicExitCloses() public launched {
        address owner = _owner();
        (, uint256 cap) = _capWithHeadroom(owner);
        uint64 pid = _setupArmed(owner, cap);
        OrderV3Types.OrderRequest memory req = _manualCloseRequest(CfdTypes.Side.LONG);
        req.sizeDelta = SIZE;
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(req);
        vm.prank(owner);
        uint64 orderId = protectionActions.cancelPositionProtectionAndCommitClose(pid, req);
        Exec memory e = _execAt(orderId, ENTRY, EXECUTION_KEEPER);
        assertEq(e.status, L_EXECUTED, "X2C: manual close executes");
        assertEq(_size(owner), 0, "X2C: closed");
    }

    function test_HT1V6_X3_FreeMarginWithdrawableDuringLock() public launched {
        address owner = _owner();
        (uint64 pid,,,,) = _setupLocked(owner);
        _fundTrader(owner, 500e6);
        _refreshMark(GAP);
        uint256 free = _freeSettlementAssets(owner);
        uint256 withdrawable = engineAccountLens.getWithdrawableAssets(owner);
        emit log_named_uint("X3 free settlement", free);
        emit log_named_uint("X3 getWithdrawableAssets", withdrawable);
        uint256 t0 = settlementToken.balanceOf(owner);
        vm.prank(owner);
        clearinghouse.withdraw(owner, 250e6);
        assertEq(settlementToken.balanceOf(owner) - t0, 250e6, "X3a: 250 USDG withdrawn");
        uint256 all = _settlementBalance(owner);
        emit log_named_uint("X3b remaining balance (attempted)", all);
        vm.prank(owner);
        vm.expectRevert(IMarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdraw(owner, all);
        assertEq(_status(pid), TRIGGERED, "X3: Triggered");
        assertGt(_size(owner), 0, "X3: open");
    }

    function test_HT1V6_X4_OwnerExpiresThenOwnerPathsStillFail() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,,) = _setupLocked(owner);
        _warpPastDeadline(child);
        vm.prank(owner);
        router.expireOrder(child);
        assertEq(_status(pid), LATCHED, "X4: Latched after owner expiry");
        OrderV3Types.OrderRequest memory req = _manualCloseRequest(CfdTypes.Side.LONG);
        req.sizeDelta = SIZE;
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(pid, req);
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(pid);
        assertGt(_size(owner), 0, "X4: open");
    }

    function test_HT1V6_X5a_OwnerRetryWithoutAmend_FailsAtGap() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,, uint256 cap) = _setupLocked(owner);
        _warpPastDeadline(child);
        vm.prank(owner);
        router.expireOrder(child);
        vm.prank(owner);
        uint64 retryChild = protectionActions.retryPositionProtectionClose(pid);
        Exec memory e = _execAt(retryChild, GAP, EXECUTION_KEEPER);
        _logExec("X5a keeper execution at GAP", e);
        assertEq(e.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "X5a: ConstraintViolation");
        assertEq(e.constraint, uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "X5a: GrossAccountDebit");
        assertEq(e.limit, cap, "X5a: limit is cap");
        assertGt(e.actual, cap, "X5a: debit exceeds cap");
        assertEq(_status(pid), LATCHED, "X5a: relatched");
        assertGt(_size(owner), 0, "X5a: open");
    }

    function test_HT1V6_X5b_OwnerAmendThenRetry_Closes() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,, uint256 cap) = _setupLocked(owner);
        _warpPastDeadline(child);
        vm.prank(owner);
        router.expireOrder(child); // owner tx 1
        vm.prank(owner);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _wide()); // owner tx 2
        emit log_named_uint("X5b cap before", cap);
        emit log_named_uint("X5b cap after", protectionViews.closeFinancialLimits(pid).maxGrossAccountDebitAssets);
        vm.prank(owner);
        protectionActions.retryPositionProtectionClose(pid); // owner tx 3
        _assertClosedByKeeper(owner, pid, "X5b keeper execution at GAP");
    }

    // ---- Item 6: (a)-(d) -----------------------------------------------------------------------------------
    function test_HT1V6_a_BaselineEOA_ThreeCycles_OwnerPathsFail() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child, RetryCycler bot,,) = _setupLocked(owner);
        for (uint256 i; i < 3; ++i) {
            _assertOwnerPathsFail(owner, pid);
            child = _cycle(bot, child, pid);
        }
        _assertOwnerPathsFail(owner, pid);
        assertGt(_size(owner), 0, "a: open");
    }

    function test_HT1V6_b_DepositDuringLock_DoesNotUnlock() public launched {
        address owner = _owner();
        (uint64 pid,,,,) = _setupLocked(owner);
        uint256 b0 = _settlementBalance(owner);
        _fundTrader(owner, 5000e6);
        emit log_named_uint("b deposit credit", _settlementBalance(owner) - b0);
        _assertOwnerPathsFail(owner, pid);
        assertEq(_status(pid), TRIGGERED, "b: Triggered");
        assertGt(_size(owner), 0, "b: open");
    }

    function test_HT1V6_c_7702BeforeCycle_ExpireAmendRetry_Closes() public launched {
        address owner = _owner();
        (, uint256 cap) = _capWithHeadroom(owner);
        uint64 pid = _setupArmed(owner, cap);
        uint64 child = _triggerAt(pid, STOP);
        _attach7702(owner);
        _warpPastDeadline(child);
        BatchExec7702V6.Call[] memory c =
            _withFirst(BatchExec7702V6.Call(address(router), 0, abi.encodeCall(router.expireOrder, (child))), pid);
        vm.prank(owner, owner);
        BatchExec7702V6(payable(owner)).exec(c);
        assertEq(_status(pid), TRIGGERED, "c: Triggered with amended cap");
        _assertClosedByKeeper(owner, pid, "c keeper execution at GAP");
    }

    /// (d) honest control: the cycler's expire-and-retry lands first in the block; a batch that still names the
    /// expired child reverts. This is a stale-target artifact, not a final "too late" (see E4c).
    function test_HT1V6_d_StaleChildBatch_Reverts() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child, RetryCycler bot,,) = _setupLocked(owner);
        _attach7702(owner);
        uint64 stale = child;
        _warpPastDeadline(child);
        (, uint64 next) = bot.atomicExpireAndRetry(child, pid);
        BatchExec7702V6.Call[] memory c =
            _withFirst(BatchExec7702V6.Call(address(router), 0, abi.encodeCall(router.expireOrder, (stale))), pid);
        vm.prank(owner, owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotPending.selector);
        BatchExec7702V6(payable(owner)).exec(c);
        assertEq(protectionViews.getPositionProtection(pid).linkedOrderId, next, "d: cycler child linked");
        assertGt(_size(owner), 0, "d: open");
    }

    // ---- Item 4: 7702 escapes after the cycle has started --------------------------------------------------
    /// E4a: executeOrder is permissionless. Inside the live window, the owner batches
    /// [executeOrder(child) -> fails on the cap -> Latched, amend, retry] in one transaction.
    function test_HT1V6_E4a_7702_ExecFailAmendRetry_AfterCycleStarted() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,,) = _setupLocked(owner);
        _attach7702(owner);
        bytes[] memory pyth = _mockPythUpdateData(GAP);
        assertGt(uint256(_deadline(child)), block.timestamp, "E4a: child live (inside window)");
        BatchExec7702V6.Call[] memory c = _withFirst(
            BatchExec7702V6.Call(address(router), 0, abi.encodeCall(router.executeOrder, (child, pyth))), pid
        );
        vm.prank(owner, owner);
        BatchExec7702V6(payable(owner)).exec(c);
        assertTrue(protectionViews.getPositionProtection(pid).linkedOrderId != child, "E4a: new child linked");
        assertEq(_status(pid), TRIGGERED, "E4a: Triggered with amended cap");
        _assertClosedByKeeper(owner, pid, "E4a keeper execution at GAP");
    }

    /// E4a2: same batch, then the owner executes the new child itself in the next block (no keeper).
    function test_HT1V6_E4a2_7702_ExecFailAmendRetry_OwnerExecutesNewChild() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,,) = _setupLocked(owner);
        _attach7702(owner);
        bytes[] memory pyth = _mockPythUpdateData(GAP);
        BatchExec7702V6.Call[] memory c = _withFirst(
            BatchExec7702V6.Call(address(router), 0, abi.encodeCall(router.executeOrder, (child, pyth))), pid
        );
        vm.prank(owner, owner);
        BatchExec7702V6(payable(owner)).exec(c);
        uint64 newChild = protectionViews.getPositionProtection(pid).linkedOrderId;
        Exec memory e = _execAt(newChild, GAP, owner);
        _logExec("E4a2 owner execution of new child at GAP", e);
        assertEq(e.status, L_EXECUTED, "E4a2: owner executes its new child");
        assertEq(_size(owner), 0, "E4a2: closed");
    }

    /// E4b: after expiry of the live (cycler-created) child, owner batch [expire, amend, retry] lands first.
    function test_HT1V6_E4b_7702_ExpireAmendRetry_AfterCycleStarted() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child,,,) = _setupLocked(owner);
        _attach7702(owner);
        _warpPastDeadline(child);
        BatchExec7702V6.Call[] memory c =
            _withFirst(BatchExec7702V6.Call(address(router), 0, abi.encodeCall(router.expireOrder, (child))), pid);
        vm.prank(owner, owner);
        BatchExec7702V6(payable(owner)).exec(c);
        _assertClosedByKeeper(owner, pid, "E4b keeper execution at GAP");
    }

    /// E4c: the cycler wins the expiry block (as in (d)); the owner then re-targets the new live child
    /// with the exec-fail route inside its window.
    function test_HT1V6_E4c_CyclerFirst_ThenOwnerExecFailRoute_Closes() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child, RetryCycler bot,,) = _setupLocked(owner);
        _attach7702(owner);
        _warpPastDeadline(child);
        (, uint64 next) = bot.atomicExpireAndRetry(child, pid);
        bytes[] memory pyth = _mockPythUpdateData(GAP);
        BatchExec7702V6.Call[] memory c = _withFirst(
            BatchExec7702V6.Call(address(router), 0, abi.encodeCall(router.executeOrder, (next, pyth))), pid
        );
        vm.prank(owner, owner);
        BatchExec7702V6(payable(owner)).exec(c);
        _assertClosedByKeeper(owner, pid, "E4c keeper execution at GAP");
    }

    /// E4d: plain EOA (no delegation) cannot use the exec-fail route atomically: after its own executeOrder
    /// relatches, a third-party retry in between leaves amendment unavailable.
    function test_HT1V6_E4d_PlainEOA_ExecFail_ThenThirdPartyRetry_AmendFails() public launched {
        address owner = _owner();
        (uint64 pid, uint64 child, RetryCycler bot,,) = _setupLocked(owner);
        Exec memory e = _execAt(child, GAP, owner);
        assertEq(e.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "E4d: owner exec fails on cap");
        assertEq(_status(pid), LATCHED, "E4d: Latched");
        bot.retryClose(pid);
        _assertOwnerPathsFail(owner, pid);
        assertGt(_size(owner), 0, "E4d: open");
    }
}
```
