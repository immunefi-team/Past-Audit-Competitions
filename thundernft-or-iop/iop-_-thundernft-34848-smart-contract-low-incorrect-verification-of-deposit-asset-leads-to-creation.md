# IOP \_ ThunderNFT 34848 - \[Smart Contract - Low] Incorrect verification of deposit asset leads to cre

Submitted on Thu Aug 29 2024 09:23:45 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34848

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a buyer places a *Buy* order the exchange verifies if enough payment asset was placed within the `Pool` contract by the buyer. The verification only takes into account `price` submitted in the order, but not the `amount` hence if `amount` is greater than 1 there would be not enough assets to cover the order.

## Vulnerability Details

When a user places a *Buy* order in `ThunderExchange` by calling `place_order()` or wants to update an existing order by calling `update_order()` the `Pool` balance is checked for the necessary amount of `payment_asset` deposited prior to placing/updating of the said order:

```
let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);
```

We can see here that only `order.price` is validated, however user can also submit the `order.amount` value which could be greater than 1. In this case the required balance will not be enough and hence the order placed could not be filled by potential sellers as there would be not enough assets within the `Pool`.

## Impact Details

The impact of this issue is low, as no assets are being lost or stolen, however the thunder design would like at least ensure that initially when the order is placed there is enough assets in the `Pool` to fill it. This validation is not working properly when `amount` is higher than 1 (and it could be more than 1 as per the design ERC1155 tokens should also be supported) and so the contract fails to deliver promised results.

This situation could lead to having registered orders which are in-executable and hence are putting risk on potential sellers where their transactions would get reverted and loss of gas ensues.

NOTE: The bug reported in #34736 should also be fixed for this fix to make sense.

## Solution Proposal

Change the validation statement to also include the `order.amount`:

```
let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
require(order.price * order.amount <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);
```

## References

Problematic line in `place_order()`: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L94>

Problematic line in `update_order()`: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L122>

## Proof of concept

## Proof of Concept

PoC available in the following gist: <https://gist.github.com/jecikpo/f0387ae4bf85a9ef1b722d279b5888e3>
