# #43712 \[SC-Low] Silent ETH transfer failure in \`TRANSFER\_NATIVE\` command leads to permament locking of user funds

**Submitted on Apr 10th 2025 at 09:02:26 UTC by @DSbeX for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43712
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The Router contract includes a TRANSFER\_NATIVE command that attempts to send ETH to a recipient using a low-level .call{value: amount}("") statement. However, it does not check whether this transfer succeeds, and if it fails (e.g. if the recipient is a contract that rejects ETH), the funds are not returned to the user — instead, they remain permanently locked in the Router contract, with no mechanism for withdrawal or recovery.

## Vulnerability Details

In the `_dispatch()` function of the Dispatcher contract, the following command is handled:

```javascript
} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
    // success is not checked
}
```

If success is false, the transfer has failed — meaning the ETH remains in the Router. However, the failure is ignored, and the contract continues execution without reverting. This would be acceptable if the ETH was never received, but in this case, the user has already sent the ETH to the Router via msg.value, as part of a broader multi-command execute() call. The Router contract does not expose any method to recover ETH, and its normal operational path assumes transfers succeed. As a result, the user’s funds are: Accepted by the Router (via msg.value) Not delivered to the target (because of a failed .call) Locked forever in the Router with no recovery path.

## Impact Details

The direct impact is: Permanent loss of funds for the user, with no ability to recover. This applies to any user sending ETH through a TRANSFER\_NATIVE command to a recipient that rejects ETH or causes the .call to fail for some reason.

Examples of common failure scenarios:

1. Recipient is a contract with no receive() or fallback() function
2. Gas limits are too low for the recipient to handle the transfer
3. The recipient reverts intentionally Since the Router doesn’t track the failure, users are not even aware their ETH never arrived, and the UI or off-chain logic may assume success. This matches the “Permanent freezing of funds” severity category.

## References

`_dispatch()` in `Dispatcher.sol` <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

`Router.execute()` passes `msg.value` through to `_dispatch()` when ETH is being sent

```javascript
function execute(...) public payable override whenNotPaused {
    ...
    if (msgSender == address(0)) {
        msgSender = msg.sender;
        topLevel = true;
        msgValue = msg.value; // ETH is sent here
    }
    ...
    _dispatch(command, input); // Handles TRANSFER_NATIVE with msg.value
}
```

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Router.sol#L141>

## Proof of Concept

## Proof of Concept

1. User prepares a TRANSFER\_NATIVE command. The user wants to send 1 ETH to a recipient using the Router contract

```javascript
execute(
    abi.encodePacked(Commands.TRANSFER_NATIVE),
    [abi.encode(recipient, 1 ether)]
)
```

2. Recipient is a contract without a payable fallback or receive() function
3. The user sends the transaction to the Router contract and includes 1 ETH as msg.value.
4. Router attempts to forward ETH using a low-level .call
5. The .call fails silently because the recipient cannot receive ETH, the .call returns false. However, there is no check for success, so no revert occurs. Execution continues as if the ETH was successfully sent.
6. Router keeps the ETH. The ETH was taken from the user (via msg.value), but it was never delivered to the recipient. It remains in the Router contract's balance.
7. Funds are permanently stuck. There is no function in the Router contract to withdraw or recover stuck ETH. The contract was not designed to have a mechanism for manual recovery. The 1 ETH is now unrecoverable.
