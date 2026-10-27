# #37577 \[BC-Insight] \`tx.origin\` Usage in Group Management Contract Allows Phishing Attack for Unauthorized Actions

**Submitted on Dec 9th 2024 at 16:37:44 UTC by @cheems for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37577
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/hyperledger/besu>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

A critical vulnerability exists in the on-chain group management contract, introduced in [PR #1288](https://github.com/hyperledger/besu/pull/1288), where `tx.origin` was used to verify the contract owner. This allows a malicious contract to impersonate the owner and bypass ownership checks, enabling unauthorized users to lock/unlock the contract or modify group membership. The issue arises from the addition of `tx.origin` as a check for ownership, which was initially introduced as part of removing the enclave key and ensuring only the creator could update the contract.

## Vulnerability Details

In this PR (<https://github.com/hyperledger/besu/pull/1288>), the intention was to ensure that only the creator's account could add/remove members and update the management contract. However, using `tx.origin` to validate ownership exposes the contract to phishing attacks, where a malicious contract can call the vulnerable contract on behalf of the attacker. Since `tx.origin` refers to the original transaction sender, the attacker only needs to initiate the call (not execute the exploit directly), allowing them to bypass the ownership checks.

## Impact Details

Provide a detailed breakdown of possible losses from an exploit, especially if there are funds at risk. This illustrates the severity of the vulnerability, but it also provides the best possible case for you to be paid the correct amount. Make sure the selected impact is within the program’s list of in-scope impacts and matches the impact you selected.

## References

Add any relevant links to documentation or code

## Link to Proof of Concept

<https://gist.github.com/SamruddhiNavale/100b48157a7101d53bee9056b2a7dbb7>

## Proof of Concept

## Proof of Concept

```
pragma solidity ^0.8.0;

interface IVulnerableContract {
    function lock() external;
    function unlock() external;
}

contract MaliciousContract {
    IVulnerableContract public vulnerableContract;

    constructor(address _vulnerableContract) {
        vulnerableContract = IVulnerableContract(_vulnerableContract);
    }

    function attackLock() public {
        vulnerableContract.lock(); // Attack via tx.origin impersonation
    }

    function attackUnlock() public {
        vulnerableContract.unlock(); // Attack via tx.origin impersonation
    }
}
```

#### **Steps to Reproduce:**

1. **Deploy the Vulnerable Contract** on a local test network.
2. **Deploy the Malicious Contract**, passing the vulnerable contract address.
3. **Call `attackLock()` or `attackUnlock()`** from the malicious contract.
4. **Verify** the contract state is locked/unlocked by the attacker.

***

#### **Expected Result:**

The contract should only allow the actual owner to lock/unlock or modify its state.

#### **Actual Result:**

The malicious contract can lock/unlock or modify the contract state due to the `tx.origin` vulnerability.

***

#### **Impact:**

This vulnerability allows attackers to impersonate the contract owner and perform unauthorized actions, bypassing the intended ownership checks. It can lead to unauthorized updates to the contract and loss of control over group management.

***

#### **Mitigation Recommendations:**

1. **Avoid Using `tx.origin`** for ownership checks.
2. **Use `msg.sender`** for proper access control.
3. **Implement proper access control mechanisms** like `onlyOwner`.

***

#### **Conclusion:**

This vulnerability, introduced in PR #1288, exposes the contract to phishing attacks by using `tx.origin`. The malicious contract only needs to initiate the call, not execute the exploit directly, to bypass ownership checks.
