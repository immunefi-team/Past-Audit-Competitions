# 31451 - \[SC - Insight] MAX\_PROPOSAL\_NUMERATOR is incorrectly set

Submitted on May 19th 2024 at 14:34:45 UTC by @Kenzo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31451

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

In AlchemixGovernor contract, the `MAX_PROPOSAL_NUMERATOR` is used to determine the maximum threshold for quorum which is hardcoded and can be never changed except by an upgrade. The current implementation set the `MAX_PROPOSAL_NUMERATOR = 5000`. But the issue is according to the Alchemix [doc](https://alchemixdao.notion.site/veALCX-Launch-Parameters-Proposal-60113919e018424db7fc03c346c34386), The `MAX_PROPOSAL_NUMERATOR` should be equal to 6600(60%) instead of 5000.

## Impact Details

Due to adding the wrong value in `MAX_PROPOSAL_NUMERATOR` the protocol doesn't allows the admin to set the value of `MAX_PROPOSAL_NUMERATOR` above the 5000(50%) which makes the admin/protocol restricted to set maximum threshold for quorum above than 50% as intended by the protocol in the docs using the function below:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol?utm\\_source=immunefi#L68C1-L74C1>

```solidity
    function setProposalNumerator(uint256 numerator) external {
        require(msg.sender == admin, "not admin");
        require(numerator <= MAX_PROPOSAL_NUMERATOR, "numerator too high");
        proposalNumerator = numerator;
        emit ProposalNumberSet(numerator);
    }

```

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol?utm\\_source=immunefi#L19>

```solidity
    uint256 public constant MAX_PROPOSAL_NUMERATOR = 5000; // 50% of total supply to create a proposal
```

## Recommendation

Change the following according to the docs:

```solidity
    uint256 public constant MAX_PROPOSAL_NUMERATOR = 6600;  
```

## Proof of Concept

Run this test in `AlchemixGovernor.t.sol`:

```
    function testUpdateProposalNumerator() public {
        hevm.prank(admin);
        governor.setAdmin(devmsig);
        hevm.startPrank(devmsig);
        governor.acceptAdmin();
        hevm.expectRevert(abi.encodePacked("Unable to set MAX_PROPOSAL_NUMERATOR above than 5000"));
        
        governor.setProposalNumerator(6000);
        assertEq(governor.proposalNumerator(), 500);
        hevm.stopPrank();
    }
```
