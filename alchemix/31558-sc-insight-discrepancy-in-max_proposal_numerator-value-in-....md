# 31558 - \[SC - Insight] Discrepancy in MAX\_PROPOSAL\_NUMERATOR Value in ...

Submitted on May 21st 2024 at 09:40:57 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31558

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Introduction

The `AlchemixGovernor` contract contains a discrepancy between the implemented `MAX_PROPOSAL_NUMERATOR` value and the value specified in the official `veALCX Launch Parameters Proposal` document. The contract currently uses a value of `5000` (50%), while the documentation specifies a value of `6600` (66%).

## Vulnerability Details

### Current Implementation

The `AlchemixGovernor` contract defines the `MAX_PROPOSAL_NUMERATOR` as follows:

```solidity
src/AlchemixGovernor.sol:
  19:     uint256 public constant MAX_PROPOSAL_NUMERATOR = 5000; // 50% of total supply to create a proposal
```

This means that currently, setting the % of total supply required to create a proposal (`proposalNumerator`) to a value more than 50% and less than 66% will revert with 'numerator too high' error.

```solidity
src/AlchemixGovernor.sol:
  64:     /**
  65:      * @dev Set the % of total supply required to create a proposal
  66:      * @param numerator The new numerator value for the quorum fraction
  67:      */
  68:     function setProposalNumerator(uint256 numerator) external {
  69:         require(msg.sender == admin, "not admin");
@>70:         require(numerator <= MAX_PROPOSAL_NUMERATOR, "numerator too high");
  71:         proposalNumerator = numerator;
  72:         emit ProposalNumberSet(numerator);
  73:     }
```

### Documented Parameters

According to the `veALCX Launch Parameters Proposal` document, the `MAX_PROPOSAL_NUMERATOR` should be set to `6600` (66%):

> MAX\_PROPOSAL\_NUMERATOR, 6600 (66%), Maximum threshold for quorum, hard coded, set up to never be changed except by an upgrade

## Impact Details

The discrepancy between the documented and implemented values can have the following impacts:

* **Governance Integrity**: The governance process may not function as intended, as the max threshold for creating proposals is lower than what was documented.
* **Community Trust**: Community members who refer to the documentation may be confused by the mismatch, leading to mistrust in the governance system.

## References

* [Quote from “veALCX Launch Parameters Proposal”](https://arc.net/l/quote/itavdnan)

## Proof of Concept

Here is a simple test case that you can use to verify the correct discrepancy behavior

```solidity
// SPDX-License-Identifier: UNLICENSED
import "./BaseTest.sol";

contract AlchemixGovernorPoCTest is BaseTest {

    function setUp() public {
        setupContracts(block.timestamp);
    }


    function testMaxProposalNumerator() public {
        assertTrue(governor.MAX_PROPOSAL_NUMERATOR() != 6600, "MAX_PROPOSAL_NUMERATOR is not 6600");
        
        hevm.prank(admin);
        hevm.expectRevert(abi.encodePacked("numerator too high"));
        governor.setProposalNumerator(6000);
    }
}
```
