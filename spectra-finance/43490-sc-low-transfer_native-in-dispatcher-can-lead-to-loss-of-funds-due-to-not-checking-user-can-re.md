# #43490 \[SC-Low] TRANSFER\_NATIVE in Dispatcher can lead to loss of funds due to not checking user can receive ETH

**Submitted on Apr 7th 2025 at 09:09:16 UTC by @holydevoti0n for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43490
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `TRANSFER_NATIVE` command in the `Dispatcher` contract performs a low-level ETH transfer without verifying if the transfer was successful. This can lead to permanent loss of funds when sending ETH to addresses that cannot accept it.

## Vulnerability Details

In the Dispatcher contract, the TRANSFER\_NATIVE command is implemented as follows: <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L483-L485>

```solidity
if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
}
```

The issue is that the function retrieves the success boolean from the low-level call **but does not verify that the transfer was successful**. If the recipient is a contract without a receive() or fallback() function, or if these functions revert, the transfer will fail but the transaction will still proceed as if it succeeded.

Unlike ERC20 token transfers which revert on failure, low-level ETH transfers with `.call{value}()` only return a boolean indicating success. Without checking this boolean and reverting on failure, the system incorrectly assumes the transfer was successful, but the ETH never actually reaches the recipient.

## Impact Details

* Transaction will succeed when in fact it failed to send ETH to the user. This will cause the user to leave ETH in the contract which can be stolen later by anyone who calls the Router with `TRANSFER_NATIVE` setting himself as the recipient.

## Recommendation

Modify the `TRANSFER_NATIVE` command to check the success boolean and revert if the transfer fails:

```diff
if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
+    if (!success) {
+        revert CallFailed();
+    }
}
```

## Proof of Concept

1. Alice deposits ETH into the Router contract.
2. Alice attempts to transfer her ETH to a contract without receive/fallback functions using TRANSFER\_NATIVE. The transfer fails but the transaction succeeds.
3. Alice's ETH remains trapped in the Router contract as the code doesn't check the success boolean.
4. Bob calls TRANSFER\_NATIVE with themselves as recipient and steals Alice's trapped ETH.
