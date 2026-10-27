# 31503 - \[SC - Insight] Incorrect value of MAX\_PROPOSAL\_NUMERATOR in Al...

Submitted on May 20th 2024 at 18:26:21 UTC by @OxRizwan for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31503

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Incorrect value of `MAX_PROPOSAL_NUMERATOR` in `AlchemixGovernor.sol`

## Vulnerability Details

AlchemixGovernor contract has `MAX_PROPOSAL_NUMERATOR` which is implemented as:

```solidity
    uint256 public constant MAX_PROPOSAL_NUMERATOR = 5000; // 50% of total supply to create a proposal
```

This constant variable acts as a max limit of proposal numerator and has been used in `setProposalNumerator()` function where it restricts the `proposalNumerator` to be within max limit while setting the value of `proposalNumerator`.

```solidity
    function setProposalNumerator(uint256 numerator) external {
        require(msg.sender == admin, "not admin");
@>      require(numerator <= MAX_PROPOSAL_NUMERATOR, "numerator too high");
        proposalNumerator = numerator;
        emit ProposalNumberSet(numerator);
    }
```

The issue is that, the values used for `MAX_PROPOSAL_NUMERATOR` is not correct and deviate from the Alchemix-V2 documentation. The governance documentation specifically states:

```solidity
MAX_PROPOSAL_NUMERATOR- 6600 (66%) 

Maximum threshold for quorum, hard coded, set up to never be changed except by an upgrade
```

Documentation link: <https://alchemixdao.notion.site/veALCX-Launch-Parameters-Proposal-60113919e018424db7fc03c346c34386>

Therefore, with the use of `MAX_PROPOSAL_NUMERATOR`, the max limit of current `proposalNumerator` can be bypassed. This would not be as per intended design of Alchemix-V2 governance. This would break the intended design of governance.

`proposalNumerator` is used to calculate the `proposalThreshold`

```solidity
    function proposalThreshold() public view override(L2Governor) returns (uint256) {
        return (token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR;
    }
```

With this issue, the calculation of `proposalThreshold` will be hugely varied.

The issue is being identified as low severity since it fails return promised returns as per governance specification. `MAX_PROPOSAL_NUMERATOR` is public so this is getter function by default means their return value can be called.

## Impact Details

The intended governance specification i.e promised values of `MAX_PROPOSAL_NUMERATOR` would be returned incorrect thereby breaking Alchemix-V2 governance design.

The calculations of `proposalThreshold` would be affected as the current implementation allows to bypass the max limit of `MAX_PROPOSAL_NUMERATOR` since the actual limit is 6600 as per documentation but implementation has hardcoded to 5000

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/AlchemixGovernor.sol#L19>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/AlchemixGovernor.sol#L70>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/AlchemixGovernor.sol#L46>

## Recommendation to fix

Use the correct values of `MAX_PROPOSAL_NUMERATOR` as per Alchemix-V2 governance documentation.

```diff
-    uint256 public constant MAX_PROPOSAL_NUMERATOR = 5000; // 50% of total supply to create a proposal
+    uint256 public constant MAX_PROPOSAL_NUMERATOR = 6600; // 66% of total supply to create a proposal
```

## Proof of Concept

The issue is about incorrect value of `MAX_PROPOSAL_NUMERATOR` which is CONSTANT variable used in governance contracts which is deviating from Alchemix-V2 governance design and documentation. The value are used in contracts as a part of contract so highlighting the issue is important since it returns incorrect values which is against intended governance design. The constant variable can not be changed and there is no setter function so impact is severe.

Please check the Recommendation to fix above and further description to understand the issue.

This can be easily understood as its not complex issue so there is no need for coded POC.

Thanks for your understanding.
