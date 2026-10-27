# #39364 \[BC-Critical] Trusting heavily on "appData" enables infinite SHM duplication through double-spend exploit

**Submitted on Jan 28th 2025 at 15:09:46 UTC by @bountyhunter2048 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39364
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Over dependence on the `appData` and memory pattern which are created by a single node leads to double-spending and unlimited duplication of SHM coins by disabling the account-locking mechanism of the processing queue.

## Vulnerability Details

Shardeum network performs upfront pre-processing and basic validation on the first node at which the transaction is injected by the client. The name of the function that does that job is called `txPreCrackData`. Among other things, a memory pattern definition is created in this function on the injected node. Later, that memory pattern is used by all executing nodes to lock accounts in the queue and ensure transactions from the same accounts are processed synchronously. If the injected node is malicious, it can manipulate the `appData.memoryPattern` object to completely disable the locking mechanism and cause nodes to process the transactions from the same account in parallel. The attacker node just needs to mark the sender's account as `ro` (which I believe is read-only).

Here is the vulnerable code in the `crack()` which is run by all nodes processing the transaction:

```
if (ShardeumFlags.autoGenerateAccessList && appData.accessList) {
    shardusMemoryPatterns = appData.shardusMemoryPatterns //  should not trust it
    ...
}
```

<https://github.com/shardeum/shardeum/blob/bugbounty/src/index.ts#L5623>

In my opinion, I don't think the queue locking mechanism and memory patterns are at fault, but all the nodes in the execute group blindly trusting the `appData` from a single node is the real issue. It would be better if more than one node were involved in the `txPreCrackData` function.

## Impact Details

The impact of any double-spending bug in a blockchain is always massive. It is a critical problem because now the attacker has a way to create (or duplicate) SHM without any limit (as far as I can think of). They can steal SHM, move it to an exchange and sell it while inflating the SHM supply and causing the SHM price to go down.

## Link to Proof of Concept

<https://gist.github.com/bountyhunter2048/442bb6be2d65eeaedda480e3916c2e67>

## Proof of Concept

## Proof of Concept

To exploit this vulnerability, I prepared a script that sends 5 SHM to 2 different accounts using the same nonce very quickly. In a normal network, one of these 2 transactions should fail no matter how quickly they are injected because they should be processed sequentially and the EVM will throw an error for nonce mismatch. But, if an attacker can get their malicious node into the network by modifying the `txPreCrackData` as provided in the GitHub gist below, he can inject coin transfer transactions to his malicious node and double-spend his coins. If he sends 5 SHM to 2 new addresses he controls, both of the transactions will be successful and they will get 5 SHM each. This means that there is an extra 5 SHM created out of nowhere.

I demonstrated this exploit from start to finish in this video:\
<https://youtu.be/GkmmNvExCgY?si=HM7XGBA6zI4P4xRA>

GitHub gist for exploit code:\
<https://gist.github.com/bountyhunter2048/442bb6be2d65eeaedda480e3916c2e67>
