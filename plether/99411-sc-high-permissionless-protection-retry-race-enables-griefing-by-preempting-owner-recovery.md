# \#99411 [SC-High] Permissionless protection retry race enables griefing by preempting owner recovery

**Submitted on Nov 13th 2026 at 14:44:29 UTC by @maptool for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99411
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - An attacker can repeatedly preempt owner amendment, delaying closure and exposing the trader to additional market losses and carry, potentially resulting in otherwise avoidable liquidation.

## Description

## Brief/Intro
An unprivileged caller can repeatedly interfere with a trader’s recovery from a failed position-protection close. The owner can amend the close’s financial limits only while the protection is Latched, but any caller can immediately retry it and restore Triggered, where amendment is rejected. An attacker can combine failure or expiry processing with retry in one transaction, or insert a retry between the owner’s separate cleanup and amendment transactions. Successful interference delays position closure and prolongs exposure to market movements and carry, potentially causing additional losses or otherwise avoidable liquidation.

## Vulnerability Details
The issue exists primarily in `PositionProtectionBook.sol`.

The recovery mechanism combines these rules:

1. `amendLatchedPositionProtectionCloseLimits` requires the caller to own the protection, its status to be `Latched`, and the account to have no pending orders.
2. A failed or expired protection child returns the protection to `Latched` if the protected position’s side and size remain unchanged.
3. `retryPositionProtectionClose` is permissionless. It queues a replacement close and changes the protection back to `Triggered`.
4. The replacement retains the same financial limits and receives a fresh execution window.
5. While `Triggered`, the owner cannot amend, cancel, replace, or use the armed-only cancel-and-close operation. The active protection also prevents an ordinary fresh close order.

Consequently, an attacker contract can execute this sequence atomically:

`executeOrder(child, admissiblePriceData)`
    -> financial constraint fails
    -> child becomes terminal
    -> protection becomes `Latched`

`retryPositionProtectionClose(protectionId)`
    -> replacement child is queued
    -> protection becomes `Triggered

After the child’s deadline, the attacker can alternatively compose:

`expireOrder(child)`
`retryPositionProtectionClose(protectionId)`

The owner cannot interleave an amendment inside either transaction. Individual reentrancy guards do not prevent these sequential calls.

The attacker can also interfere with an active owner:

Transaction 1 — owner:
    execute the failing child
    -> protection becomes `Latched`

Transaction 2 — attacker:
    `retryPositionProtectionClose`
    -> protection becomes `Triggered`

Transaction 3 — owner:
    `amendLatchedPositionProtectionCloseLimits`
    -> reverts `ProtectionNotLatched`

### Reachable financial-limit failure:

The supplemental PoC opens a 10,000-token LONG USD position near basket mark 1.00, with 2,000 USDG position margin and sufficient free collateral. It configures:

- Stop-loss threshold: 1.001.
- Maximum execution notional: 10,020 USDG.
- Subsequent trigger/execution observation: 1.003.

The ceiling covers both the approximately 10,000 USDG entry notional and the 10,010 USDG notional at the intended stop. An observation at 1.003 produces approximately 10,030 USDG execution notional, exceeding the retained ceiling.

Rejecting that execution correctly enforces the trader’s limit. The vulnerability concerns an attacker’s ability to interfere with the owner’s subsequent attempt to amend it.

### Preconditions and recovery boundaries:

The attack requires an unchanged protected position and a close that remains unexecutable under its retained limits. Sustained interference depends on transaction ordering; it does not assume a malicious sequencer or guaranteed censorship.

An owner or honest keeper can execute a failing replacement before expiry, creating another amendment opportunity. A position-owning contract can batch failure processing, amendment, and retry once the child is eligible. If market conditions return within the retained limits, an honest executor can close without amendment. Once amendment succeeds, future retries inherit the new limits.

Accordingly, this report claims repeatable interference with recovery.

### Relationship to previously disclosed behavior

MED-9 concerned generated protection closes lacking trader-selected financial ceilings. Those ceilings are correctly retained and enforced in the frozen implementation. This report concerns a separate liveness issue in the associated owner-amendment recovery path.

Previously disclosed post-trigger noncancellation, unpaid close execution, and repeated-retry queue pressure are not claimed as new findings. The distinguishing behavior is active preemption of amendment despite an owner or executor willing to process the failed close.

## Impact Details

Selected impact: Griefing — damage to users without requiring attacker profit.

The demonstrated impact is denial of particular owner amendment attempts and repeatable delay of recovery from a failed protection close. During successful interference:

- The trader continues holding exposure they are attempting to terminate.
- Carry can continue accruing.
- Adverse market movement can produce additional losses.
- If equity falls sufficiently before recovery, the position may become liquidatable.

These financial consequences are conditional on market movement, account health, and attack duration. The PoC demonstrates interference with recovery; it does not demonstrate liquidation or establish a fixed monetary loss.

The attacker needs no USDG deposit or retry bounty. They pay native transaction costs; failure-driven execution also requires qualifying oracle data and applicable fees. The expiry-and-retry route requires no oracle update.

The repeatability test performs 60 expiry/retry cycles over 3,660 seconds and observes rejected owner amendments after each renewal.

Router pause does not prevent the retry path, and risk-off cleanup for opening orders does not terminate an already-triggered protection close.

## References
- [Owner amendment restrictions — PositionProtectionBook.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)

- [Permissionless retry — PositionProtectionBook.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)
- [Failure relatching — PositionProtectionBook.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675)
- [Retained financial limits and fresh execution window](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L679)
- [Active protection blocks ordinary orders](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)
- [Public expiry — OrderRecoverySidecar.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L63)
- [MED-9 remediation and accepted tradeoffs — PR #124](https://github.com/Plether-Fi/plether-core/pull/124)

- [Previously disclosed retry behavior — PR #78](https://github.com/Plether-Fi/plether-core/pull/78)


## Link to Proof of Concept

https://gist.github.com/mrKaplan-cg/b141ae7d4215d7d93db87aa7562a3a9a

## Proof of Concept

## Proof of Concept
Place the supplied files (provided in the gist) at:

packages/perps/test/audit/Lead8Validation.t.sol
packages/perps/test/audit/Lead8ProtectionStarvation.t.sol

Run:

cd packages/perps

FOUNDRY_VIA_IR=false forge test \
  --match-path test/audit/Lead8Validation.t.sol \
  --match-test 'test_Lead8Review_*' -vv

Reproduction sequence

1. Open the position through the public V3 order interface.
2. Create stop-loss protection with threshold 1.001 and maximum execution notional 10,020 USDG.
3. Trigger at mark 1.003.
4. Through an unprivileged helper contract, execute the child at an admissible price exceeding the ceiling and immediately retry within the same transaction.
5. Verify that the replacement child leaves the protection Triggered.
6. Attempt owner amendment and observe ProtectionNotLatched.
7. Advance past the replacement’s deadline and atomically expire and retry it.
8. Repeat for 60 cycles, attempting owner amendment after each renewal.
9. Verify that the elapsed time is 3,660 seconds, the position remains open, and the attacker requires no USDG balance.

A separate test has the owner terminally fail the child first, then inserts an attacker retry before amendment. The owner’s amendment again reverts.
