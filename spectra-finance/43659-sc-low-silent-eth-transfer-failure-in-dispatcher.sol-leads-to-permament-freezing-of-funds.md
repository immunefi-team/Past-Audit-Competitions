# #43659 \[SC-Low] Silent ETH transfer failure in \`Dispatcher.sol\` leads to permament freezing of funds

**Submitted on Apr 9th 2025 at 16:08:11 UTC by @DSbeX for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43659
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `Dispatcher.sol` contract doesn't check for success after transferring ETH using the TRANSFER\_NATIVE command. This issue causes ETH sent to the contract to become locked if the transfer to the recipient fails, without any notification or error handling. The funds are effectively "stuck" in the contract, which could lead to a poor user experience and the permanent freezing of funds if not detected and addressed.

## Vulnerability Details

The issue arises from the lack of success checking after the contract attempts to transfer ETH to the recipient. Specifically, the contract uses a low-level call to transfer ETH via:

```javascript
(bool success, ) = payable(recipient).call{value: amount}("");
```

However, the success variable is not checked. If the transfer fails (for example, if the recipient's address is invalid, the contract cannot process the transfer, reverting fallback, gas griefing, etc.), the funds are not returned or reattempted, and no feedback is provided to the user. This results in the ETH being "locked" in the contract, and the transaction silently fails.

This vulnerability occurs in the `_dispatch` function when handling the TRANSFER\_NATIVE command, and can be reproduced when a user sends ETH to the contract and the transfer to the recipient fails without any indication of the failure.

## Impact Details

Permanent Freezing of Funds: If a user sends ETH to the `Router.sol` and the `TRANSFER_NATIVE` fails silently, the ETH will remain locked in the contract. Since the failure is not detected and there is no mechanism to recover or return the funds, they are permanently frozen unless manually addressed.

User Inconvenience: Affected users may not realize that their funds are locked, leading to confusion and unsatisfaction. This issue could potentially cause users to lose trust in the protocol, especially if they are unaware of the failure and are unable to retrieve their funds.

Reputation Damage: If this issue is not addressed, it could negatively impact the reputation of the Spectra protocol. Users may perceive the platform as unreliable, especially in cases where significant ETH is involved and the failure to transfer goes unnoticed.

## References

The missing check - <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

1. User Attempts to Send ETH to the Router Contract via the TRANSFER\_NATIVE Command

* The user initiates a transaction to send ETH from their wallet to the contract with the intention of transferring the ETH to another address (via the TRANSFER\_NATIVE function). However, the recipient address provided is either invalid (e.g., a non-contract address or an address that cannot accept ETH) or some other condition occurs causing the transfer to fail. At this point: 1.1. The user is under the impression that the transaction will successfully transfer 10 ETH to the specified recipient. 1.2. The TRANSFER\_NATIVE command is triggered in the Router contract to transfer the specified amount of ETH.

2. The TRANSFER\_NATIVE command in the Dispatcher.sol executes the ETH transfer without checking success. Since no success check is done, the success variable is not evaluated, and the transaction doesn't revert or throw an error, even if the transfer fails.
3. Silent Failure of the ETH Transfer The transfer attempt fails, but the lack of error handling means the failure is never reported to the user. The user assumes the transaction was successful because there is no notification or revert to indicate any issue. The funds (ETH) are now "stuck" in the Router contract, as there is no fail-safe mechanism, and no further steps in the code are taken to recover or notify the user about the failure.
