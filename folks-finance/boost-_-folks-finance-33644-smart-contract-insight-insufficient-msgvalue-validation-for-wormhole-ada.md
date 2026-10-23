# Boost \_ Folks Finance 33644 - \[Smart Contract - Insight] Insufficient msgvalue validation for Wormho

Submitted on Thu Jul 25 2024 12:18:48 GMT-0400 (Atlantic Standard Time) by @bbl4de for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33644

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0x8F27355662D6de024FEE83b176dD8DB1F2CA1585/>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

According to the Wormhole documentation, the sending function implementation must ensure the correct amount of `msg.value` was passed in to send the token and the payload. However, both in `WormholeDataAdapter` and `WormholeCCTPAdapter`do not check this properly.

## Vulnerability Details

The `getSendFee()` function in both Wormhole adapters uses `quoteEVMDeliveryPrice()` function to determine the cost of sending the message.

As stated in [this](https://docs.wormhole.com/wormhole/quick-start/tutorials/hello-token#implement-sending-function) article from the official Wormhole documentation the `wormhole.messageFee()` MUST be accounted for as it may change from 0 in the future.

```solidity
   function getSendFee(Messages.MessageToSend memory message) external view override returns (uint256 fee) {
        // get chain adapter if available
        (uint16 wormholeChainId, , ) = getChainAdapter(message.destinationChainId);

        // get cost of message to be sent
        (fee, ) = wormholeRelayer.quoteEVMDeliveryPrice(
            wormholeChainId,
            message.params.receiverValue,
            message.params.gasLimit
        );
    }
```

The `BridgeRouter` contract sends `fee` amount of gas tokens, even if `msg.value > fee`:

```solidity
 uint256 fee = adapter.getSendFee(message);

        // check if have sufficient funds to pay fee (can come from existing balance and/or msg.value)
        bytes32 userId = _getUserId(Messages.decodeActionPayload(message.payload));
        uint256 userBalance = balances[userId];

        if (msg.value + userBalance < fee) revert NotEnoughFunds(userId);

        // update user balance considering fee and msg.value
        balances[userId] = userBalance + msg.value - fee;

        // call given adapter to send message
@>      adapter.sendMessage{ value: fee }(message);
```

Because of this, sending more `msg.value` to account for this additional fee is impossible - it'll be always the value returned by `getSendFee()`.

## Impact Details

When Wormhole updates the message fee to be != 0, the `fee` sent to the Wormhole Adapters will be insufficient leading to all Wormhole-based cross-chain messages failing ( both data-only and CCTP token transfers).

Although this vulnerability strictly requires Wormhole protocol to change a parameter, their docs explicitly warn any integrations that this might happen and present the correct way of calculating the delivery fee:

```
// Total cost: delivery cost +
    // cost of publishing the 'sending token' wormhole message
    cost = deliveryCost + wormhole.messageFee();
```

Therefore, critical severity is justified because in case of this *predictable* change, all cross-chain messages from spoke chains compatible with only the Wormhole adapter will fail.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/WormholeCCTPAdapter.sol#L89C2-L99>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/WormholeDataAdapter.sol#L61-L71>

## Proof of concept

## Proof of Concept

Considering that the Wormhole Relayer is responsible for paying the fee to the Delivery Provider, to avoid complex integrations with cross-chain messaging `sendToEvm()` function in the `MockWormholeRelayer` will be changed to simulate this process:

```diff
function sendToEvm(
       uint16 targetChain,
       address targetAddress,
       bytes memory payload,
       uint256 receiverValue,
       uint256 paymentForExtraReceiverValue,
       uint256 gasLimit,
       uint16 refundChain,
       address refundAddress,
       address deliveryProviderAddress,
       VaaKey[] memory vaaKeys,
       uint8 consistencyLevel
   ) external payable override returns (uint64 sequence) {
+      uint deliveryFee = msg.value + 0.01e18;
+      require(msg.value >= deliveryFee, "insufficient payment");
       emit WormholeSendVaaKey(
           targetChain,
           targetAddress,
           payload,
           receiverValue,
           paymentForExtraReceiverValue,
           gasLimit,
           refundChain,
           refundAddress,
           deliveryProviderAddress,
           vaaKeys,
           consistencyLevel
       );
       return _sequence;
   }
```

where the `deliveryFee` is increased by `0.01e18` - arbitrary value for `wormhole.messageFee`. Obviously, calling this function will always fail, that's a simplification of what would be checked when the delivery fee is paid to the delivery provider.

To call this function run:

```shell
npx hardhat test --grep "Should successfuly send immediate finality message"
```

which will run a test case from `WormholeDataAdapter.test.ts`. It'll always fail, note that this "ever-failing" check makes sense because `msg.value` is *always* equal to the return value of `getSendFee()`.
