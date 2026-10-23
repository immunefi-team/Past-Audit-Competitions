# IOP \_ ThunderNFT 34839 - \[Smart Contract - Low] Royalty Fee limit is not enforced for registered col

Submitted on Wed Aug 28 2024 21:35:16 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34839

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/royalty\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

NFT sellers can provide royalty fee for their collections. The protocol owner can set the royalty fee limit. The royalty fee limit however is only enforced for collections which register after the new royalty fee limit was set.

## Vulnerability Details

`RoyaltyManager` contract provides `register_royalty_info()` method where the sellers can provide their desired royalty fee. The method reverts if the provided fee is higher than the stored `fee_limit`:

```
require(fee <= storage.fee_limit.read(), RoyaltyManagerErrors::FeeHigherThanLimit);
```

`fee_limit` is set by the `set_royalty_fee_limit()` method. The problem is that the `fee_limit` can be set at a time where there are already some royalty information registered for different collections. If the protocol owner desired to lower the maximum royalty fee it will have an effect only on the newly registered collections.

Existing collection fee is unaffected as we can see in the method which obtains the fee (the `get_royalty_info()` takes the `RoyaltyInfo` object stored and returns it "as is"). Here is the relevant code snippet:

```
    fn get_royalty_info(collection: ContractId) -> Option<RoyaltyInfo> {
        let none: Option<RoyaltyInfo> = Option::None;
        let status = storage.royalty_info.get(collection).try_read();
        match status {
            Option::Some(royalty_info) => royalty_info,
            Option::None => none,
        }
    }
```

## Impact Details

The impact is that for certain collections higher fees might be charged to the buyers and sellers of the collections than the protocol owner desires, hence the contract doesn't lose value, yet fails to deliver promised results. The severity is hence low.

## Solution Proposal

The `get_royalty_info()` method should validate the stored royalty fee agains the `fee_limit` every time it is called. If the collection's fee is higher than the `fee_limit`, `fee_limit` value should be returned instead.

## References

The following is the problematic function: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/royalty\\_manager/src/main.sw#L85>

## Proof of concept

## Proof of Concept

PoC available through gist: <https://gist.github.com/jecikpo/aa2b5dcc027f736c8523fb6eb00a0dc7>
