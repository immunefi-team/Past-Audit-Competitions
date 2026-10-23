# #44131 \[SC-Low] An attacker can steal frozen user ETH due to Dispatcher error

**Submitted on Apr 17th 2025 at 08:12:43 UTC by @din for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44131
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

This is not an user error, because the user cannot be aware if a recipeint contract fallback/receive will revert at a specific moment. More details why this will happen inside the POC.

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

An attacker will steal frozen user's ETH due to a bug inside Dispatcher.sol

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol?utm\\_source=immunefi#L483C1-L486C11>

## Proof of Concept

## Proof of Concept

1. A honest user calls Dispatcher::execute() with command: TRANSFER\_NATIVE and sends 100 eth to a recepient contract
2. The recipient's fallback/receive are temporary frozen by the admins of the recepient address due to a hack. The recipient contract reverts intentionally until the recipient contract team troubleshoots and fixes the vulnerability. This is outside of the Spectra user's control and there is no way for them to know it before executing the Spectra command. // Recipient's fallback

   ```
          fallback(){
                if(isPaused) > revert("Cannot accept ETH at this moment. Try again later")
         }
   ```
3. The user's ETH remains stuck in Dispatcher.sol due to the lack of low lv call status.
4. A malicious user immediately calls Dispatcher::execute() TRANSFER\_NATIVE and steals the funds.
