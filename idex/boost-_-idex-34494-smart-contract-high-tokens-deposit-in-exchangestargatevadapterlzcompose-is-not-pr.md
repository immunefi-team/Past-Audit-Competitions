# Boost \_ IDEX 34494 - \[Smart Contract - High] Tokens deposit in ExchangeStargateVAdapterlzCompose is

Submitted on Tue Aug 13 2024 23:08:39 GMT-0400 (Atlantic Standard Time) by @Paludo0x for [Boost | IDEX](https://immunefi.com/bounty/boost-idex/)

Report ID: #34494

Report type: Smart Contract

Report severity: High

Target: <https://github.com/idexio/idex-contracts-ikon/blob/main/contracts/bridge-adapters/ExchangeStargateV2Adapter.sol>

Impacts:

* Temporary freezing of funds

## Description

## Brief/Intro

In `ExchangeStargateV2Adapter::lzCompose()` the `IExchange(custodian.exchange()).deposit();` function is called, if this is disabled for any reason the deposit will fail.

## Vulnerability Details

The tokens are bridged from L1 to L2 by means of Stargate protocol.

In order to transfer the tokens to the final contract the function `lzCompose` shall be implemented.

This is the implementation of the `ExchangeStargateV2Adapter` contract:

```
  function lzCompose(
    address _from,
    bytes32 /* _guid */,
    bytes calldata _message,
    address /* _executor */,
    bytes calldata /* _extraData */
  ) public payable override {
  ...
    // https://github.com/LayerZero-Labs/LayerZero-v2/blob/1fde89479fdc68b1a54cda7f19efa84483fcacc4/oapp/contracts/oft/libs/OFTComposeMsgCodec.sol#L52
    uint256 amountLD = OFTComposeMsgCodec.amountLD(_message);

    // https://github.com/LayerZero-Labs/LayerZero-v2/blob/1fde89479fdc68b1a54cda7f19efa84483fcacc4/oapp/contracts/oft/libs/OFTComposeMsgCodec.sol#L61
    address destinationWallet = abi.decode(OFTComposeMsgCodec.composeMsg(_message), (address));
    require(destinationWallet != address(0x0), "Invalid destination wallet");

    IExchange(custodian.exchange()).deposit(amountLD, destinationWallet); 
  } 

```

The call to `deposit` function is not inside a try/catch block as it is usual when tokens are transferred via a token/messagge bridge layer.

An example of implementation is reported in **Stargate** docs. <https://stargateprotocol.gitbook.io/stargate/v/v2-developer-docs/integrate-with-stargate/composability#receive-1>

The suggestion is to implement the `catch` block with a transfer to the destination wallet or to a permissioned wallet with transfering functionality implemented.

## Impact Details

The impact is that the funds can be stucked if the exchange deposits are disabled permanently for any reason, even if LayerZero and Stargate implements functionalities to retry sending a message

## Proof of concept

## Proof of Concept

In the test provided by the contract there's the POC that demonstrates that the call will revert if exchange deposits are disabled.

```
    it.only('should revert when deposits are disabled', async () => {
      await expect(
        stargateRouterMock.sgReceive(
          await adapter.getAddress(),
          1,
          '0x',
          0,
          await usdc.getAddress(),
          10000000000,
          ethers.AbiCoder.defaultAbiCoder().encode(
            ['address'],
            [ownerWallet.address],
          ),
        ),
      ).to.eventually.be.rejectedWith(/deposits disabled/i);
    });
```
