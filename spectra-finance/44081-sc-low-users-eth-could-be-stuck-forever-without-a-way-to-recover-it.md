# #44081 \[SC-Low] Users ETH could be stuck forever without a way to recover it

**Submitted on Apr 16th 2025 at 18:42:01 UTC by @MrMorningstar for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44081
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Whenever native token is transfered from the contract it is done throught `execute` which eventually invoke `_dispatch` function. For the purpose of this finding we would focus on this part of the function:

```
} else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            (bool success, ) = payable(recipient).call{value: amount}("");
        }
```

## Vulnerability Details

The issue here is that the protocol does not check if the call was successful or not, which could lead that protocol funds be stuck to the contract forever.

Check PoC section for very common scenario.

## Impact Details

The mentioned scenario is very common and likelihood it to happen is High which will lead the funds to be permanently stuck/froze forever.

## Recommendation

Make sure the protocol check if call is successful:

```diff
else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
+    require(success, "Native transfer failed");
}
```

## Proof of Concept

## Proof of Concept

The common (but not the only) scenario I will describe is this:

Transfering wrapped native token to the protocol and send out native token to the recipient. So the command execution would look like this:

1. `TRANSFER_FROM` - transfer wrapped native to the protocol
2. `WITHDRAW_NATIVE_FROM_WRAPPER` - withdraw native from wrapped native to the protocol
3. `TRANSFER_NATIVE` - from the protocol to the recipient

So the user send wrapped native to the protocol expecting to sent native token to the recipient, but for some reason (it could be grieving or the address is not able to receive native token, there are many reasons why it would fail) the transfer is not successful. The issue is that then that these native tokens would not be sent back to the user, but to the protocol because protocol sent them and there is no way for them to be recovered by our user which will make his funds stuck and lost forever.
