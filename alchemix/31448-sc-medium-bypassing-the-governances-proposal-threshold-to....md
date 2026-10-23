# 31448 - \[SC - Medium] Bypassing the Governances proposal threshold to...

Submitted on May 19th 2024 at 13:27:32 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31448

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Details

The `proposalThreshold()` is the number of votes required for a voter to become a proposer.

```javascript
    /**
     * @dev Part of the Governor Bravo's interface: _"The number of votes required in order for a voter to become a proposer"_.
     */
    function proposalThreshold() public view override(L2Governor) returns (uint256) {
        return (token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR;
    }
```

> Link to code snippet: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol#L42-L47>

The threshold is calculated based on the total supply of veALCX.

Therefore, after the `VotingEscrow.sol` smart contract is deployed, and before anyone creates a lock by calling `VotingEscrow.createLock(...)`, the `proposalThreshold()` *is equal to 0* and *anybody* can spam any amount of malicious proposals in `AlchemixGovernor` with 0 voting power (not owning any veALCX at all).

Alchemix's team can manually cancel a malicious proposal, preventing this attack vector from becoming a critical vulnerability.

We are submitting it as griefing, as their team has to spend gas canceling all the malicious proposals.

After we mentioned this flag in a Discord thread, *Ov3rkoalafied* from Alchemix's team commented:

> Agreed, any proposal can be canceled instantly. It is griefing to the extent that they have to spend gas to create 'em and we'd have to spend gas to cancel 'em.

## Impact Details

A Griefing attack with no profit motive.

## Proof of Concept

In the "AlchemixGovernorTest.sol" smart contract go to the function "setUp()" and comment the two calls to "createVeAlcx(...)", to simulate that nobody has yet deposited (akka. total supply is 0).

Then include this test and run it with foundry - it will not revert.

```javascript
function testBypassProposalThreshold() public {
    address random_happy_user = address(0x123);
    hevm.startPrank(random_happy_user);

    (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();
    governor.propose(t, v, c, d, MAINNET);

    hevm.stopPrank();
}
```
