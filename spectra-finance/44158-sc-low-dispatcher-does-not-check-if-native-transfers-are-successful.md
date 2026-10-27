# #44158 \[SC-Low] Dispatcher does not check if native transfers are successful

**Submitted on Apr 17th 2025 at 12:01:36 UTC by @hgrano for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44158
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`Dispatcher` allows for native transfers but the call status is not checked i.e. if called on a contract which reverts, then the funds will be left in the `Router`. An attacker could then use the native transfer functionality to take the native tokens for themselves.

## Vulnerability Details

On Dispatcher.sol:485, the `success` flag isn't checked. If the `recipient` is unable to handle the native transfer, then this will silently fail and the native token is left in the `Router`. This would generally be caused by user error, but regardless, an attacker/bot could notice the failed transfer and use the `TRANSFER_NATIVE` command to send the tokens to their own wallet.

## Impact Details

The impact is potential permanent loss of funds as described above.

## References

Dispatcher.sol:485 <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

The issue could be exploited by an attacker through:

1. Creating a bot to monitor the mempool for transactions that may cause the native transfer to fail.
2. If a transaction is detected, backrun it, transferring the native token back to the attacker's wallet.
