# 26095 - \[SC - Insight] ID Uniqueness Violations

Submitted on Nov 24th 2023 at 18:13:41 UTC by @SentientX for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26095

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Theft of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.

## Description

## Bug Description

The scheme relies on a straightforward incrementing of integer to assign proposal IDs and doesn't incorporate a check for uniqueness, there's a possibility that two proposals submitted around the same time or in rapid succession might end up with the same ID, resulting in collision. This could lead to loss of funds if outcome of proposal is in favour of a Degate operator and involves funds.

***Currently the scheme has two design limitations:***

1. Proposal parameters submitted are not checked for uniqueness
2. Return id value of proposal submitted is not checked for uniqueness

There is likely hood that two proposals maybe submitted in rapid succession due to `double click` admin error or intentional error from operator. This may cause same proposal to be confirmed twice, if careful review is not taken during approval through to confirmation stage.

***Consider following scenario:***

1. Protocol has decided to approve the refund of large number of user funds or for specific user.
2. On the same time there are several other proposals that have to be approved.
3. Refund proposal id and parameters are approved twice and submitted for confirmation
4. Multi-sig confirms proposals without careful review since there is no uniqueness checks.
5. Users receive more than is due to them at the expense of protocol.

This error could be exploited by Malicious Degate Operator who is gaming Governance outcomes. Additionally,

***Here are additional technical scenario's where proposal could result in same id:***

1. Reorgs (propabilisitc event that results in re-organizing of blocks in a parent chain)
2. Miner manupilation attacks that may result in block timestamp manipulations.
3. Compiler errors
4. EVM glitches

## Impact

1.***ID Uniqueness:*** If a proposal is submitted for example to issue large scale refunds, protocol may issue refunds twice to users leading to Governance triggerred insolvency.

2.***Race Conditions:*** In a scenario where multiple submissions occur simultaneously or within a very short timeframe, and the uniqueness check isn't robust enough, there could be a race condition. This might lead to proposals getting the same or conflicting IDs due to concurrent processing.

3. ***Insufficient Uniqueness Checks:*** If the uniqueness check is based on limited proposal details or lacks comprehensive validation, it might not adequately prevent duplicates.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

```
    function submitTransaction(address, uint, bytes memory)
        public
        returns (uint transactionId)
    {
        transactionId = addTransaction(destination, value, data);
        confirmTransaction(transactionId);

        require(!usedTransactionIds[transactionId], "Transaction ID already used");
        usedTransactionIds[transactionId] = true;
    }

```

with the mapping `mapping(uint => bool) public usedTransactionIds;`

## References

## Proof of concept

This POC is just to show parameter uniqueness is not validated and should be known information to Degate Devs:

1. mkdir degatePOC
2. cd degatePOC
3. forge init
4. delete test folder, Counter.sol and Counter.t.sol
5. add POC and interfaces to src folder run with

`forge test --contracts ./src/boostPOC-2.sol -vv` Non check of id uniqueness is evident from code review.
