# IOP \_ ThunderNFT 34642 - \[Smart Contract - High] strategy de-listing causes sellers NFTs locked on T

Submitted on Mon Aug 19 2024 07:34:18 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34642

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Temporary freezing of NFTs for at least 1 hour

## Description

## Brief/Intro

A Sell order can be cancelled through the `ThunderExchange` contract. If the order was created within a strategy that is delisted by the protocol team, the `cancel_order()` reverts and the seller cannot receive their NFTs (if they were not sold already)

## Vulnerability Details

Once a Sell order is created within the `ThunderExchange` the NFT to be sold is transferred to the `ThunderExchange` contract. A seller may decide to cancel their order by calling `cancel_order()` at the `ThunderExchange` contract. The method will then transfer the remaining NFT back to the seller, however before this can happen there is a `require` statement that is checking if the strategy used by the seller is whitelisted within the `ExecutionManager` contract which acts as a register of valid strategies.

The snippet below shows the check:

```
fn cancel_order(
        strategy: ContractId,
        nonce: u64,
        side: Side
    ) {
        [...]
        require(execution_manager.is_strategy_whitelisted(strategy), ThunderExchangeErrors::StrategyNotWhitelisted);
    [...]
    }
```

The protocol team can de-list a strategy at their own discretion even if there are standing Sell orders within it. If this happens (it could happen because of a bug discovered within the strategy) the NFT seller won't be able to withdraw their NFT using the above function should such an event occur.

It is worth noting that this is not an "attack" caused by the contract owner, but rather an unintended consequence of an action which the protocol owner is somehow forced to do.

## Impact Details

if an NFT owner has a standing Sell order within a de-listed strategy, the only way would be if the protocol team whitelists again the strategy. This could easily be more than 1 hour depending on the protocol's reaction times, hence the severity is High.

It could also be not possible at all if the strategy has been de-listed due to a bug found or an ongoing hack, hence the protocol could not be able to re-whitelist the strategy at all if it jeopardises the entire protocol.

## Solution Proposal

The solution could be to introduce pausing of a strategy, which would allow it to exist in an intermediate state between being whitelisted and de-listed. In that state e.g. only cancelling of orders would be possible.

## References

Add any relevant links to documentation or code The line involving the issue: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L145>

## Proof of concept

## Proof of Concept

PoC and its description can be found here: <https://gist.github.com/614fe5ec928f8b816f63e9ec4f78f717.git>
