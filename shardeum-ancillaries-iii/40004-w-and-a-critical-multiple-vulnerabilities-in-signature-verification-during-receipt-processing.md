# #40004 \[W\&A-Critical] Multiple vulnerabilities in signature verification during receipt processing o

**Submitted on Feb 12th 2025 at 15:21:19 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #40004
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief/Intro

Multiple vulnerabilities in receipt signature verification allow an attacker with an active node connected to the archiver to modify any account balance.

### **Vulnerability Details**

There are two vulnerabilities in the receipt signature verification process on the archiver side. While I couldn't find a way to exploit them independently, combining them results in a critical impact.

#### **First Vulnerability: Bypassing Required Signature Verification**

The first issue is in the `verifyAppliedReceiptSignatures` function:

```js
const goodSignatures = new Map();
for (const [index, signature] of signaturePack.entries()) {
  if (Crypto.verify({ ...appliedVoteHash, sign: signature, voteTime: voteOffsets.at(index) })) {
    goodSignatures.set(signature.owner, signature);
    // Break the loop if the required number of good signatures are found
    if (goodSignatures.size >= requiredSignatures) break;
  } else {
    // Non-relevant code
  }
}
```

The function verifies the first `requiredSignatures` entries and stores valid signatures in a `Map`, using the `owner` field (the signer's public key) as the key. However, the `owner` field is case-sensitive, which allows an attacker to exploit this by:

* Using a single key pair to generate a valid signature.
* Cloning the same signature multiple times while modifying the letter case in the `owner` field.
* Ensuring that all forged signatures appear at the beginning of the `signaturePack` list.

Since there are **no checks** for whether the key belongs to a legitimate node or whether it corresponds to an execution shard, an attacker can effectively bypass proper verification.

#### **Second Vulnerability: Lack of Signature Verification in Node Checks**

The second issue is in the `verifyReceiptData` function:

```js
const uniqueSigners = new Set();
for (const signature of signaturePack) {
  const { owner: nodePubKey } = signature;
  
  // Get the node ID from the public key
  const node = cycleShardData.nodes.find((node) => node.publicKey === nodePubKey);
  if (node == null) {
    // Non-relevant code
    continue;
  }

  // Check if the node is in the execution group
  if (!cycleShardData.parititionShardDataMap.get(homePartition).coveredBy[node.id]) {
    // Non-relevant code
    continue;
  }

  uniqueSigners.add(nodePubKey);
}
```

This function performs two checks:

1. **Node existence validation** – Ensures that the `owner` field corresponds to a node in `cycleShardData.nodes`.
2. **Execution group check** – Ensures that the node is part of the execution group for the transaction.

However, **no actual signature verification is performed**. The function only checks whether the `owner` field exists in the execution group, meaning an attacker can:

* Extract a valid list of nodes from the execution group.
* Generate **completely fake signatures** with their `owner` fields set to any valid execution group node.
* Include a mix of valid and fake signatures in the `signaturePack`, ensuring that any invalid entries are simply skipped without triggering an error.

By carefully crafting the `signaturePack`, the attacker can pass both verification steps without having any legitimate signatures.

### **Impact**

By combining these vulnerabilities, an attacker can:

* Forge any receipt.
* Bypass signature verification on the archiver side.
* Update **any** account balance or state without proper authorization.

This effectively allows an attacker with an active node connected to the archiver to **manipulate account balances at will**.

## References

<https://github.com/shardeum/archiver/blob/a55860da3416ca0433579f0e180629d18c9e4c09/src/Data/Collector.ts#L823\\>
<https://github.com/shardeum/archiver/blob/a55860da3416ca0433579f0e180629d18c9e4c09/src/Data/Collector.ts#L574>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/15e3e6a880072e7947696fa69c9fd587>

## Proof of Concept

### **Proof of Concept (PoC)**

To reproduce the exploit, follow these steps:

1. **Apply the Patch**
   * Apply the patch from the provided Gist to the core repository to enable testing.
2. **Start the Local Network**
   * Launch a local test network and wait until it reaches the **processing state**.
   * Start the RPC server.
3. **Create a Coin Transfer Transaction**
   * Send any coin transfer transaction to the network.
4. **Check the Account State on the Archiver**
   * Access the archiver node.
   * Open the `accounts` database using SQLite.
   * Verify that the targeted account balance has been updated, even though the transaction was not properly signed.
5. **Verify the Forged Receipt in Validator Logs**
   * Check the logs of a connected validator.
   * Find the receipt that was sent to the archiver.
   * Confirm that it contains:
     * **Fake signatures** from nodes in the execution group.
     * **Cloned signatures** from the attacker's node.
