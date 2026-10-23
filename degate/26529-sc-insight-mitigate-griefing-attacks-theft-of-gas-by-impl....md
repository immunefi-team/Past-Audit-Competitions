# 26529 - \[SC - Insight] Mitigate Griefing Attacks Theft of Gas by Impl...

Submitted on Dec 4th 2023 at 14:47:40 UTC by @Breeje for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26529

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Theft of gas

## Description

## Description

There are couple of issues related to External Call in `executeTransaction`.

```solidity
File: Timelock.sol

102:     (bool success, bytes memory returnData) = target.call{value: value}(callData);
102:     require(success, "Timelock::executeTransaction: Transaction execution reverted.");

```

Checkout PoC for detail.

## Recommendation

* To address the issue to prevent potential Griefing Attacks, it is recommended to implement protection against Return Data Bombs. Consider using [ExcessivelySafeCall](https://github.com/nomad-xyz/ExcessivelySafeCall) when interacting with untrusted contracts. This solution aims to safeguard against excessive gas costs incurred during memory allocation for large return data payloads, enhancing the security of the `Timelock` contract.
* To address the issue to prevent Gas Theft, cap the amount of gas used in the external call by passing it as a parameter.

## Proof of concept

The Timelock functionality works as:

1. The `admin` adds a transaction to a queue.
2. After a specified `eta`, the transaction can be triggered by the `admin` using the `executeTransaction` function.
3. A potential vulnerability arises when the `Timelock` is employed for an external target with specific execution, as it opens the door for a malicious return of a Gas Bomb via the fallback function.

In `executeTransaction` function, which interacts with an external contract (target), there is no protection against such large return data in the form of a Gas Bomb.

Actually, the way it works is the `bytes memory returnData` that was returned from the `target` will be copied to memory. Memory allocation becomes very costly if the payload is big, so this means that if a `target` implements a fallback function that returns a huge payload, then the `msg.sender` of the transaction, in our case the `admin`, will have to pay a huge amount of gas for copying this payload to memory.

Secondly, there is no cap on the amount of gas passed to the external call.

This means that the target address that has been called is now able to spend unbounded gas (up to 63/64% of the gas provided by the caller) on other activities like gas token minting.

If you are unfamiliar with gas token minting then I think this is a good article that talks about the topic in more depth: <https://medium.com/hackernoon/potential-attack-on-ethereum-network-to-mint-gastokens-5cf05a7e0303>
