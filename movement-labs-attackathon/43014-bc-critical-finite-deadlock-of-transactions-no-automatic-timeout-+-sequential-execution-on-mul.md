# #43014 \[BC-Critical] finite Deadlock of Transactions (No Automatic Timeout + Sequential Execution) on multisig implementation

**Submitted on Mar 31st 2025 at 16:23:22 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43014
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Direct loss of funds
  * Permanent freezing of funds (fix requires hardfork)
  * Multisig owner-induced deadlock (escalation of privilege)

## Description

**Summary**\
A single proposal in the multisig contract can become permanently stuck in “pending” status if it never meets the threshold for either approval or rejection. Because each proposal must be resolved in strict sequence order, all future proposals are indefinitely blocked, effectively locking any funds governed by the multisig. Contrary to the assumption that this requires “privileged access,” in this design the multisig owner role is held by normal users, not administrators—any unresponsive or uncooperative user can freeze the contract.

***

## Vulnerability Details

### Location

* `aptos-framework/sources/multisig_account.move`

### Description

1. **Sequential Execution Constraint**\
   The contract enforces strict sequential processing of proposals. Specifically:

   ```rust
   public fun can_be_executed(multisig_account: address, sequence_number: u64): bool acquires MultisigAccount {
       ...
       sequence_number == last_resolved_sequence_number(multisig_account) + 1
       && num_approvals >= num_signatures_required
       // ...
   }

   public fun can_be_rejected(multisig_account: address, sequence_number: u64): bool acquires MultisigAccount {
       ...
       sequence_number == last_resolved_sequence_number(multisig_account) + 1
       && num_rejections >= num_signatures_required
       // ...
   }
   ```

   No proposal with `sequence_number = X+1` can be acted upon until proposal `X` is fully approved or rejected.
2. **No Timeout or Override**\
   Because there is no time-based fallback mechanism (e.g., “auto-reject after 7 days”) or override function (e.g., a forced skip), a proposal `X` that never achieves enough approvals or rejections gets stuck in a permanent “limbo.”
   * As a result, `last_resolved_sequence_number` never advances, preventing all future proposals from proceeding.
3. **Owners Are Regular Users, Not Admins**\
   Unlike systems where a privileged admin could step in, any participant in the multisig—simply by refusing to vote—can block all future transactions. This is a fundamental design flaw, granting ordinary users disproportionate power to freeze funds indefinitely.
4. **Why This Is a Problem**
   * **Permanent Freeze of Funds**: If enough owners refuse to vote or go offline, the transaction never reaches a final decision.
   * **No Built-In Remedy**: If the code is non-upgradable, a full chain-level intervention may be the only fix.
   * **Defeats Multisig Purpose**: By design, a multisig is meant to handle multi-party consensus; deadlocks undermine that purpose entirely - as sometimes it is expected that not all members will be available.

### Impact

* **Permanent Freeze of Funds**: All subsequent proposals and, therefore, all future actions on the multisig are blocked.
* **Wider Ecosystem Risk**: Since this multisig is a standard, permissionless tool, anyone deploying it is at risk of having their funds locked by some of the owners of multisig instances.
* **No Privilege Required**: A single regular owner’s silence is enough to deadlock the system - depending on the voting threshold.

***

## Recommended Fix

1. **Implement a Timeout Mechanism**
   * After a set period, automatically reject any proposal that has not been fully approved.
   * Example approach:

     ```rust
     if current_time - proposal_start_time > TIMEOUT {
         force_reject_proposal(seq);
     }
     ```
2. **Add an Override Feature**
   * Provide a method (e.g., supermajority vote) to forcibly resolve or skip a stalled proposal. This ensures a single unresponsive or hostile user cannot lock all funds indefinitely.

## Proof of Concept

1. Deploy a **2-of-3** multisig using the above contract.
2. Create proposal #1 (sequence\_number = 1).
   * Owner A votes “yes,” Owner B votes “no,” Owner C remains silent.
3. Approvals = 1, Rejections = 1—neither reaches the required 2.
4. Proposal #1 remains in “pending” status forever; `last_resolved_sequence_number` never increments.
5. All subsequent proposals (#2, #3, etc.) are blocked, freezing the multisig permanently.
