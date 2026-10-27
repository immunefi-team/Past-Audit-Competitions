# 28646 - \[SC - Insight] Resubmission with Pause Bypass Potential Exploi...

Submitted on Feb 22nd 2024 at 23:05:17 UTC by @offside0011 for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28646

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Theft of unclaimed yield

## Description

## This is a resubmission of #28611 and #28614

Please refer to the previous submissions for technical details: <https://bugs.immunefi.com/dashboard/submission/28611> <https://bugs.immunefi.com/dashboard/submission/28614>

## Vulnerability

Dust stETH could still be stolen from the current depositor.

These issues are already documented in the audit reports from BlockSec and Quantstamp, and the Puffer Team has been notified.

## Explanation

In response to the previous issue, the team confirmed that:

> Puffer Depositor swap functions are in scope, but due to them being paused bugs will only be considered if they can bypass the pause mechanism.

However, these swap functions are still accessible on the mainnet. We have already demonstrated the vulnerability (the misconfiguration) in our previous POC, which can be found at <https://bugs.immunefi.com/dashboard/submission/28614>. The POC successfully works on block number 19281954.

## Recommendation

Currently, only a small amount of dust (approximately $46) is vulnerable. The functions should be paused.

## Proof of Concept

We demonstrated the accessibility of the swap functions by checking the authority (AccessManager) of the depositor.

```
# cast bn
19284511
# cast sig 'swapAndDeposit1Inch(address,uint256,bytes)'
0x9b207f4a

# cast call 0x8c1686069474410E6243425f4a10177a94EBEE11 'canCall(address,address,bytes4)(boo
l,uint32)' 0x1111111111111111111111111111111111111111 0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF 0x9b207f4a
true
0
```

The result indicates that an arbitrary address (0x1111111111111111111111111111111111111111) can call the depositor (0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF) using the selector of the swap function, *swapAndDeposit1Inch(address,uint256,bytes)* at block number 19284511.
