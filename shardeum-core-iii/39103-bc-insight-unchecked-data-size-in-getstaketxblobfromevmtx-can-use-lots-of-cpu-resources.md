# #39103 \[BC-Insight] Unchecked data size in "getStakeTxBlobFromEVMTx()" can use lots of CPU resources

**Submitted on Jan 22nd 2025 at 09:39:16 UTC by @bountyhunter2048 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39103
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

Shardeum embeds staking info in tx.data field of an ethereum tx but it never checks the size of the data field and parse the whatever data attached.

## Vulnerability Details

This parsing of staking info `const stakeTxString = toAscii(bytesToHex(transaction.data))` never checks the size of `transaction.data` and allows attacker to inject huge stake txs at constant "5 TPS" and causes nodes to use lots of CPU resources during the parsing.

If the attacker can use powerful machine or execute the script from different machine, he can maintain this attack without time limit because it does not cost him money as these bad txs are destined to fail (due to bad data field).

But, this attack is less dangerous if the attacker uses a single weak machine and unable to keep injecting at least 5 tps.

Here is the vulnerable function: <https://github.com/shardeum/shardeum/blob/bugbounty/src/index.ts#L839>

## Impact Details

Significant increase in the CPU usage as shown in the Demo video. <https://youtu.be/7ZHzl-\\_GxQc>

## References

Attack script: <https://gist.github.com/bountyhunter2048/47a63a2566fb4600e900c0a2be442246>

Demo attack: <https://youtu.be/7ZHzl-\\_GxQc>

## Link to Proof of Concept

<https://gist.github.com/bountyhunter2048/47a63a2566fb4600e900c0a2be442246>

## Proof of Concept

This simple script creates huge stake tx and injects it repeatedly into the network. The CPU usage of nodes goes up a lot. Attacker can execute it free of charge. Please see the demo video for detail.

<https://gist.github.com/bountyhunter2048/47a63a2566fb4600e900c0a2be442246>

Demo attack: <https://youtu.be/7ZHzl-\\_GxQc>
