# Attackathon \_ Fuel Network 32378 - \[Smart Contract - Insight] Missing Zero-Check for Recipient Addre

Submitted on Wed Jun 19 2024 19:03:37 GMT-0400 (Atlantic Standard Time) by @bugtester for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32378

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-bridge/tree/623dc288c332b9d55f59b1d3f5e04909e2b4435d/packages/fungible-token>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

The withdraw function in the smart contract lacks a check to ensure that the recipient address (to parameter) is not a zero address (b256::zero()). This oversight can lead to potential loss of funds by sending tokens to an invalid address.

## Vulnerability Details

in the withdraw function, the recipient address parameter (to) is not validated to ensure it is not a zero address. Sending funds to a zero address is an invalid operation and could result in irreversible loss of tokens.

<https://github.com/FuelLabs/fuel-bridge/blob/623dc288c332b9d55f59b1d3f5e04909e2b4435d/packages/fungible-token/bridge-fungible-token/src/main.sw#L162>

## Impact Details

loss of funds

## Proof of concept

## Proof of Concept

```
    let sender = msg_sender().unwrap();
    send_message(
        BRIDGED_TOKEN_GATEWAY,
        encode_data(to, amount.as_u256().as_b256(), l1_address, token_id),
        0,
    );
    log(WithdrawalEvent {
        to: to,
        from: sender,
        amount: amount,
    });
}
```

## fix

```
require(to != b256::zero(), BridgeFungibleTokenError::InvalidRecipient);
```

\#\[payable] #\[storage(read, write)] fn withdraw(to: b256) { // Check if the recipient address is zero require(to != b256::zero(), BridgeFungibleTokenError::InvalidRecipient);

```
let amount: u64 = msg_amount();
require(amount != 0, BridgeFungibleTokenError::NoCoinsSent);

let asset_id = msg_asset_id();
let sub_id = _asset_to_sub_id(asset_id);
let token_id = _asset_to_token_id(asset_id);
let l1_address = _asset_to_l1_address(asset_id);

// Hexens Fuel1-4: Might benefit from a custom error message
storage
    .tokens_minted
    .insert(
        asset_id,
        storage
            .tokens_minted
            .get(asset_id)
            .read() - amount,
    );
burn(sub_id, amount);

// send a message to unlock this amount on the base layer gateway contract
let sender = msg_sender().unwrap();
send_message(
    BRIDGED_TOKEN_GATEWAY,
    encode_data(to, amount.as_u256().as_b256(), l1_address, token_id),
    0,
);
log(WithdrawalEvent {
    to: to,
    from: sender,
    amount: amount,
});
```

}
