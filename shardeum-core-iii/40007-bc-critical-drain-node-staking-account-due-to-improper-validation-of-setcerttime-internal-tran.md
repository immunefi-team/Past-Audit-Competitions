# #40007 \[BC-Critical] Drain node staking account due to improper validation of SetCertTime internal transaction

**Submitted on Feb 12th 2025 at 16:57:21 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #40007
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

An attacker can bypass full validation of the SetCertTime transaction and drain the node's staking account balance.

### **Vulnerability Details**

The `validateTxnFields` function, responsible for validating transactions, relies on the `isSetCertTimeTx` function to enforce full validation on **SetCertTime** internal transactions. However, an attacker can bypass this validation by exploiting a condition in `isSetCertTimeTx`:

```ts
export function isSetCertTimeTx(tx): boolean {
  if (tx.isInternalTx && tx.internalTXType === InternalTXType.SetCertTime) {
    return true;
  }
  return false;
}
```

#### **Bypassing the Validation Check**

The `applySetCertTime` function is executed through `applyInternalTx`, which is triggered when `isInternalTx` returns `true`:

```ts
export function isInternalTx(timestampedTx: any): boolean {
  if (timestampedTx && timestampedTx.raw) return false;
  if (timestampedTx && timestampedTx.isInternalTx) return true;
  if (timestampedTx && timestampedTx.tx && timestampedTx.tx.isInternalTx) return true;
  return false;
}
```

As seen in the last condition, **`isInternalTx` can still return `true` even if `tx.isInternalTx` is `false`, as long as `tx.tx.isInternalTx` is `true`**. This loophole allows an attacker to craft a **SetCertTime** transaction that:

* **Bypasses full validation**, including signature verification of the proper nominee.
* **Is processed successfully** even though it lacks legitimate authorization.

#### **Impact: Draining the Staking Account**

By repeatedly submitting the forged **SetCertTime** transaction, the attacker can:

* **Avoid proper validation** while still being processed as an internal transaction.
* **Incur transaction fees with each execution**, leading to a gradual depletion of the node’s staking account balance.

Since **constant fees** are applied every time this transaction is executed, the attacker can systematically drain the staking balance over multiple iterations.

## References

<https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/setup/validateTxnFields.ts#L79>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/d2090cae2e9888feff9c8b156bac406a>

## Proof of Concept

### **Proof of Concept (PoC)**

To reproduce the exploit, follow these steps:

1. **Apply the Bug Bounty Patch**
   * Apply the **bug bounty patch** for the attacker's node from the **audit competition documentation**.
2. **Start the Local Network**
   * Launch a local test network and wait until it reaches the **processing state**.
3. **Stake the Nodes**
   * Stake the nodes as required for normal network operation.
4. **Run the Exploit Script**
   * Execute `poc.js` from the provided Gist, supplying:
     * The **attacker's IP and port**.
     * The **victim's IP and port**.
5. **Verify the Results**
   * Observe the execution logs and check the staking account balance.
   * If successful, the victim's staking balance should be drained due to repeated transaction fees.
