# #44091 \[SC-Low] Lack of ETH transfer check leads to stolen funds

**Submitted on Apr 16th 2025 at 20:12:20 UTC by @din for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44091
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Dispatcher.sol does not check if the low lvl call for sending ETH is successfull. If the call fails, the tx does not revert and the users funds remain stuck in the contract, without any way for the user to know that his funds did not reach the destination address. A malicious user can steal those stuck funds immedeately.

## Vulnerability Details

The TRANSFER\_NATIVE command does not check if the low lvl call is successfull.

```
     else if (command == Commands.TRANSFER_NATIVE) {
        (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
        (bool success, ) = payable(recipient).call{value: amount}(""); <@
    }
```

As a result, all failed calls will NOT cause the execute() call to revert. ETH sent from the user will be stuck in the Router/Dispatcher contracts

## Impact Details

Stolen user funds from their failed ETH transfers. An attacker can call execute and transfer their ETH to his address

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol?utm\\_source=immunefi#L483C1-L486C11>

## Proof of Concept

## Proof of Concept

1. A honest user calls Dispatcher::execute() with command: TRANSFER\_NATIVE and sends 10 eth to a recepient contract
2. The recepient contract fallback() reverts due to logic which freezes any transfers to it due to custom conditions met. For example: Admin has paused any transfers between 10pm-12pm . Or the recipient is a malicious contract that reverts when the fallback() is called.
3. The honest user Dispatcher::execute() tx is successful, however his funds remain stuck in the contract, without him knowing it.
4. A malicious user immediately calls Dispatcher::execute() TRANSFER\_NATIVE and sends the stuck funds to his address.
