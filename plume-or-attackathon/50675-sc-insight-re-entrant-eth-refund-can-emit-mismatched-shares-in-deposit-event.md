# 50675 sc insight re entrant eth refund can emit mismatched shares in deposit event

**Submitted on Jul 27th 2025 at 12:11:56 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50675
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Vulnerability Details

`TellerWithMultiAssetSupportPredicateProxy::depositAndBridge()` calculates the number of shares to log after calling `teller.depositAndBridge()`.

During that external call the Teller may refund surplus ETH to the proxy; the proxy’s `receive()` immediately forwards that ETH to `lastSender` with a raw call. This opens a re-entrancy window before the share amount is computed for the Deposit event.

A malicious user could manipulate the rate inside that window, causing the event to report a share value different from the amount actually minted.

### Impact Details

Severity is set to Low because there is no direct fund loss.

However, because this is an RWA protocol, many off-chain services may rely on emitted events. There's a secondary risk of mis-accounting, compliance violations, and potential business/legal exposure if RWA reporting relies solely on events.

## Recommended fix

{% stepper %}
{% step %}
Compute the `shares` value before the external call.
{% endstep %}

{% step %}
Have `Teller:depositAndBridge()` return the minted shares value, and use that returned value for the event emission.
{% endstep %}
{% endstepper %}

## Proof of Concept

<details>

<summary>Relevant proxy code and explanation (expand)</summary>

This is `TellerWithMultiAssetSupportPredicateProxy::depositAndBridge()` relevant code:

{% code title="TellerWithMultiAssetSupportPredicateProxy::depositAndBridge()" %}

```
```

{% endcode %}

```solidity
function depositAndBridge(
    ERC20 depositAsset,
    uint256 depositAmount,
    uint256 minimumMint,
    BridgeData calldata data,
    CrossChainTellerBase teller,
    PredicateMessage calldata predicateMessage
)
    external
    payable
    nonReentrant
{
...
    // mint shares
    teller.depositAndBridge{ value: msg.value }(depositAsset, depositAmount, minimumMint, data);
    lastSender = address(0);
    uint96 nonce = teller.depositNonce();
    //get the current share lock period
    uint64 currentShareLockPeriod = teller.shareLockPeriod();
    AccountantWithRateProviders accountant = AccountantWithRateProviders(teller.accountant());
    //get the share amount
    uint256 shares = depositAmount.mulDivDown(10 ** vault.decimals(), accountant.getRateInQuoteSafe(depositAsset));

    emit Deposit(
        address(teller),
        data.destinationChainReceiver,
        address(depositAsset),
        depositAmount,
        shares,
        block.timestamp,
        currentShareLockPeriod,
        nonce > 0 ? nonce - 1 : 0,
        address(vault)
    );
}
```

When calling `teller.depositAndBridge{ value: msg.value }(...)` the native tokens are sent to the teller, which will bridge the message and pass `msg.value` to the bridging function.

Excerpt from `MultiChainLayerZeroTellerWithMultiAssetSupport`:

{% code title="MultiChainLayerZeroTellerWithMultiAssetSupport::\_bridge" %}

```
```

{% endcode %}

```solidity
function _bridge(uint256 shareAmount, BridgeData calldata data) internal override returns (bytes32) {

...
    MessagingReceipt memory receipt = _lzSend(
        data.chainSelector,
        _payload,
        _options,
        // Fee in native gas and ZRO token.
        MessagingFee(msg.value, 0),
        // Refund address in case of failed source message.
        payable(msg.sender)
    );

    return receipt.guid;
}
```

LayerZero (and other bridges) can return excess native tokens to the sender. In this case the proxy's `receive()` forwards ETH immediately to `lastSender`:

{% code title="TellerWithMultiAssetSupportPredicateProxy::receive" %}

```
```

{% endcode %}

\`\`\`solidity receive() external payable { // If we have a lastSender and receive ETH, forward it if (lastSender != address(0) && msg.value > 0) { // Forward the ETH to the last sender (bool success,) = lastSender.call{ value: msg.value }(""); if (!success) revert TellerWithMultiAssetSupportPredicateProxy\_\_ETHTransferFailed(); } } \`\`\`

If `lastSender` is a smart contract, its `receive()` can re-enter and manipulate state (e.g., change a rate provider or another component) before the proxy calculates `shares` and emits the `Deposit` event, causing the event to contain an incorrect `shares` value.

```
    function depositAndBridge(...)
        external
        payable
        nonReentrant
    {
...
receive() IS TRIGGERED INSIDE THIS CALL:
        teller.depositAndBridge{ value: msg.value }(depositAsset, depositAmount, minimumMint, data);

HERE STARTS  THE CALCULATION OF VALUES TO BE EMITTED
        lastSender = address(0);
        uint96 nonce = teller.depositNonce();
        //get the current share lock period
        uint64 currentShareLockPeriod = teller.shareLockPeriod();
        AccountantWithRateProviders accountant = AccountantWithRateProviders(teller.accountant());
        //get the share amount
        uint256 shares = depositAmount.mulDivDown(10 ** vault.decimals(), accountant.getRateInQuoteSafe(depositAsset));

        emit Deposit(...);
    }
```

</details>

## Notes

* All links and code snippets are preserved as in the original report.
* No additional assertions or claims are added beyond the original content.
