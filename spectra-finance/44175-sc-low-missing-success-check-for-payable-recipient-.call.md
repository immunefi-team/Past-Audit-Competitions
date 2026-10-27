# #44175 \[SC-Low] Missing Success Check for payable(recipient).call

**Submitted on Apr 17th 2025 at 13:47:58 UTC by @Osuolale for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44175
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Dispatcher contract executes native token transfers (e.g., ETH) using a low-level .call but fails to check the success status returned by this call. This oversight means that if the native token transfer fails (for instance, if the recipient contract cannot receive Ether or reverts), the failure is silent. The transaction containing the command may still succeed, but the funds remain in the Dispatcher (router) contract, leading to inconsistent states, stranded assets, and potential disruption of batched operations that assume the transfer was successful.

## Vulnerability Details

Within the \_dispatch function, the Commands.TRANSFER\_NATIVE command type facilitates sending native currency to a specified recipient. This is implemented using a low-level .call at line 485:

```solidity
(bool success, ) = payable(recipient).call{value: amount}("");
```

This call attempts to transfer amount wei to the recipient. The success variable captures whether the call executed without reverting on the recipient's side. However, the contract code proceeds without verifying if success is true.

## Impact Details

The primary impact is the silent failure of native token transfers, leading to stranded assets and broken operational logic.

Trapped Funds: As demonstrated by the test case, native currency (e.g., ETH) sent to the router for a TRANSFER\_NATIVE command can become permanently stuck within the router contract if the intended recipient cannot receive the funds. The user loses access to these funds, as the router has no mechanism to return them in this failure scenario. Inconsistent State / Broken Atomicity: Users interacting with the execute function expect the batch of commands to either succeed atomically or fail entirely. This vulnerability breaks that expectation for TRANSFER\_NATIVE. Subsequent commands in the same batch might operate under the false assumption that the native transfer occurred, leading to incorrect calculations or state transitions. Loss of User Funds: The ETH sent by the user to the router for the transfer remains in the router. Since the transaction succeeded, the user has no direct recourse via transaction failure, and the funds are effectively lost to the user unless a separate recovery mechanism exists (which is unlikely for arbitrary trapped ETH). Undermined Protocol Reliability: The router fails to perform a core requested action (transferring native tokens) without any warning or reversion, undermining the reliability and predictability of the protocol's execution logic.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

A user (innocentUser) intends to execute a batch of commands via the router.execute function, sending amount ETH along with the call. One of the commands is TRANSFER\_NATIVE, instructing the router to send the received amount ETH to a specific recipient contract (VulnerableContract). The VulnerableContract is designed such that it cannot receive ETH (e.g., lacks a payable receive/fallback or its receiving function reverts). When the Dispatcher executes the .call{value: amount}("") to VulnerableContract, the call fails internally, and success becomes false. Crucially, the Dispatcher does not check the value of success. Execution continues, and the router.execute transaction completes successfully from the perspective of the innocentUser and the blockchain. However, the amount ETH was not transferred to VulnerableContract. Instead, it remains held within the Dispatcher/router contract.
