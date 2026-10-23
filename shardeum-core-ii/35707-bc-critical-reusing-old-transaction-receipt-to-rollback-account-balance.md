# #35707 \[BC-Critical] Reusing old transaction receipt to rollback account balance

**Submitted on Oct 4th 2024 at 04:43:18 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35707
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Absence of transaction timestamp verification in the binary\_repair\_oos\_accounts handler allows an attacker to reuse of old receipt to rollback account balance to a previous value.

## Vulnerability Details

The binary\_repair\_oos\_accounts P2P handler does not verify whether a transaction receipt has already been applied in the past. The timestamp is only checked in accountData, which is not signed by the consensus group. This oversight allows an attacker to use any old receipt for the same account to revert the account balance to its previous state after that receipt was initially applied. Consequently, the old receipt will be successfully pass verification, and the account update will be successfully propagated across the network.

## Impact Details

An attacker can set the account balance to any previous value from old transaction receipts. The updated balance will be accepted as valid across the entire network.

## References

<https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/state-manager/AccountPatcher.ts#L490>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/39dc63dc1124e2975fd6efea732e4bbf>

## Proof of Concept

Apply the patch from the gist to the attacker's node, wait for it to become active, and then send two transactions from the same account (as shown in \`poc.js\` from the documentation). Also apply server\_genesis.patch from documentation to shardeum repository. To launch the attack, run:

\`\`\`bash curl <http://ATTACKER\\_IP:ATTACKER\\_PORT/attack?accountId=e0291324263d7ec15fa3494bfdc1e902d8bd5d3d000000000000000000000000> \`\`\`

The exploit will roll back the account balance from the state after the second transaction to the balance after the first transaction by reusing the receipt from the first transaction. Run the attack only after you see the tx\_removed message for both transactions in the output log.
