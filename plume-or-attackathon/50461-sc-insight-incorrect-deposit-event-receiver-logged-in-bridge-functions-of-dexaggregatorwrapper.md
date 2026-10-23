# 50461 sc insight incorrect deposit event receiver logged in bridge functions of dexaggregatorwrapperwithpredicateproxy sol&#x20;

**Submitted on Jul 25th 2025 at 03:35:14 UTC by @Rhaydden for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50461
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Any off-chain system that attributes shares to addresses based on the emitted `Deposit` event may incorrectly attribute shares to the wrong address.

## Description

### Brief / Intro

The two bridge helper functions `depositAndBridgeOneInch()` and `depositAndBridgeOkxUniversal()` emit a `Deposit` event that uses `msg.sender` as the share receiver. In practice, shares are burned locally and recreated for `bridgeData.destinationChainReceiver` on the destination chain, so the event’s `receiver` field is incorrect.

### Vulnerability Details

Relevant code:

```solidity
function _calcSharesAndEmitEvent(
        ERC20 supportedAsset,
        CrossChainTellerBase teller,
        address fromToken,
        uint256 fromTokenAmount,
        uint256 supportedAssetAmount
) internal {
    ...
    uint256 shares = supportedAssetAmount.mulDivDown(
        10 ** teller.vault().decimals(),
        AccountantWithRateProviders(teller.accountant()).getRateInQuoteSafe(supportedAsset)
    );
    emit Deposit(
        fromToken,          // depositAsset
        msg.sender,         // ← incorrect when bridging
        address(supportedAsset),
        fromTokenAmount,
        supportedAssetAmount,
        shares,
        address(teller),
        address(teller.vault())
    );
}
```

Both bridge wrappers call this helper after performing the bridge:

```solidity
teller.depositAndBridge(...);
_calcSharesAndEmitEvent(...);
```

During a bridge:

* `msg.sender` is the EOA that calls the wrapper.
* `teller.depositAndBridge` mints shares to the wrapper, immediately burns them, and bridges the same amount to `bridgeData.destinationChainReceiver`.
* The emitted `Deposit` event thus claims that `msg.sender` received `shares`, which never happens.

### Impact Details

Off-chain systems that rely on this event to determine who received shares will attribute them to the caller (`msg.sender`) instead of the actual recipient (`bridgeData.destinationChainReceiver`).

## Fix

Pass the real receiver into `_calcSharesAndEmitEvent` so the `Deposit` event correctly records the true destination receiver.

## References

<https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/helper/DexAggregatorWrapperWithPredicateProxy.sol#L407>

## Proof of Concept

{% stepper %}
{% step %}

### Step: Setup / Deploy

1. Deploy the relevant contracts.
2. Choose two EOAs:
   * Alice (caller)
   * Bob (destinationChainReceiver)
     {% endstep %}

{% step %}

### Step: Execute and Observe

1. Alice calls `depositAndBridgeOneInch` with `bridgeData.destinationChainReceiver = Bob`.
2. Observe the `Deposit` event on the source chain: it reports `receiver = Alice`.
3. After the bridge finalizes, Bob holds the bridged shares on the destination chain, while Alice never held them — demonstrating the event reports the wrong receiver.
   {% endstep %}
   {% endstepper %}
