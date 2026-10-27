# IOP \_ ThunderNFT 34963 - \[Smart Contract - Insight] Invalid orders persist in storage maps with no i

Submitted on Sun Sep 01 2024 22:14:58 GMT-0400 (Atlantic Standard Time) by @rbz for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34963

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The storage maps lack a mechanism to flag or identify orders that have become invalid.

## Vulnerability Details

There are few scenarios under which the sell order in the `sell_order` storage map will have a None value:

* the order has been either canceled by the exchange;
* the order was successfully executed (filled by a buyer);

In the case where `_is_valid_order` returns `false` - like expiration or failing the `_is_valid_order` checks, the `strategy_fixed_price_sale` code does not take any action based on the following `ExecutionResult` payload and keeps it the storage map

```
if (!_is_valid_order(maker_order)) {
            return ExecutionResult {
                is_executable: false,
                collection: ZERO_CONTRACT_ID,
                token_id: ZERO_B256,
                amount: 0,
                payment_asset: ZERO_ASSET_ID,
            }
        }
```

## Impact Details

The accumulation of invalid orders in storage could lead to storage bloat, leading to increased storage usage and potentially impacting contract performance. If other parts of the contract or external client code rely on the `sell_order` or `buy_order` maps, they need to be aware that these maps could contain invalid orders, ensuring the additional filtering is implemented.

Additionally, if the storage map (`sell_order` or `buy_order`) is designed to be a queue for processing purposes, and invalid orders are not removed from the queue, it can prevent valid orders from being picked up and processed.

Example: In a FIFO (First-In, First-Out) queue, elements are processed in the order they were added. The first element added is the first one removed and processed. If an invalid order is at the front of the queue (e.g., it was placed earlier than the valid orders), and it's not removed even after being skipped from execution due to its invalid status, it will block the processing of all consecutive orders in the queue.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/main/contracts-v1/execution\\_strategies/strategy\\_fixed\\_price\\_sale/src/main.sw#L136-L144>

## Proof of concept

## Proof of Concept

Please see gist
