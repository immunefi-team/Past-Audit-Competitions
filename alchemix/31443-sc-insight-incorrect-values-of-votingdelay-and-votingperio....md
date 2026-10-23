# 31443 - \[SC - Insight] Incorrect values of votingDelay and votingPerio...

Submitted on May 19th 2024 at 11:50:41 UTC by @OxRizwan for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31443

Report type: Smart Contract

Report severity: Insight

Target: <https://immunefi.com>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Incorrect values of `votingDelay` and `votingPeriod` would break intended design of governance

## Vulnerability Details

`AlchemixGovernor` contract has inherited `L2Governor` contract which is governance functionalities like propose, cancel, execute and vote of proposals.

One of the important aspects of any governance proposals are `votingDelay` and `votingPeriod` and these are used as below in abstract `L2Governor`:

```solidity
    uint256 public votingDelay = 2 days;
    uint256 public votingPeriod = 3 days;
```

These values have been used in `propose()` function. The issue is that, the values used for both `votingDelay` and `votingPeriod` is not correct and deviate from the Alchemix-V2 documentation. The governance documentation specifically states:

> votingDelay- 72 hours (time delay from proposal being proposed to when it is voted on votingPeriod)

> 1 epoch- (2 weeks) (period of time when a proposal can be voted on)

Documentation link: <https://alchemixdao.notion.site/veALCX-Launch-Parameters-Proposal-60113919e018424db7fc03c346c34386>

Therefore, the proposals created via propose() would not be as per intended design of Alchemix-V2 governance. This would break the intended design of governance.

However, it should be noted that, both `votingDelay` and `votingPeriod` are not constant and can be changed after knowing this issue by calling setter function setVotingDelay() and setVotingPeriod() function.

Therfore, the issue is being identified as low severity since it fails return promised returns as per governance specification. Both `votingDelay` and `votingPeriod` are public so they are getter function by default means their return value can be called. Additionally governace interface has getter functions which can be checked [here](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/interfaces/IGovernor.sol#L133-L147).

## Impact Details

The intended governance specification i.e promised values of `votingDelay` and `votingPeriod` would be returned incorrect thereby breaking Alchemix-V2 governance design.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2Governor.sol#L46-L47>

## Recommendation to fix

Use the correct values of `votingDelay` and `votingPeriod` as per Alchemix-V2 governance documentation.

```diff
-    uint256 public votingDelay = 2 days;
+    uint256 public votingDelay = 3 days;        @audit // 72 hours as per docs
-    uint256 public votingPeriod = 3 days;
+    uint256 public votingPeriod = 2 weeks;        @audit // 1 epoch (2 weeks) as per docs
```

## Proof of Concept

The issue is about incorrect values used in governance contracts which is deviating from Alchemix-V2 governance design and documentation. The value are used in contracts as a part of contract so highlighting the issue is important since it returns incorrect values which is against intended governance design.

Please check the Recommendation to fix above and further description to understand the issue.

This can be easily understood as its not complex issue so there is no need for coded POC.

Thanks for your understanding.
