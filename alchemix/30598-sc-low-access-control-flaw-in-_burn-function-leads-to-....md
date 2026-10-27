# 30598 - \[SC - Low] Access Control Flaw in \_burn Function Leads to ...

Submitted on May 1st 2024 at 19:08:15 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30598

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Temporary freezing of NFTs

## Description

## Brief/Intro

The vulnerability in the VotingEscrow contract arises due to an access control flaw within the \_burn function, which inappropriately handles approval resetting, requiring broader permissions than necessary. This flaw disrupts operations such as token merging and withdrawals, as transactions initiated by users who are only approved for specific token IDs (and not globally) revert unexpectedly. On mainnet, this could lead to operational disruptions, preventing users from consolidating voting power or accessing their funds post-lock period, thereby undermining user trust and the functionality of the contract. Such issues could significantly impact user engagement and the platform's overall reliability.

## Vulnerability Details

### Lines of Code

alchemix-v2-dao/src/VotingEscrow\.sol #L1567\[^1]

```solidity
src/VotingEscrow.sol:
1558      function _burn(uint256 _tokenId, uint256 _value) internal {
..SNIP..
1566:         // Clear approval
1567:         approve(address(0), _tokenId);
```

In `VotingEscrow.sol` contracts, there are multiple functions that allow whether the `msg.sender` is approved for the given token ID, is an operator of the owner, or is the owner of the token by utilizing `_isApprovedOrOwner` function.

```solidity
src/VotingEscrow.sol:
618      function merge(uint256 _from, uint256 _to) external {
..SNIP..
621:         require(_isApprovedOrOwner(msg.sender, _from), "not approved or owner");
622:         require(_isApprovedOrOwner(msg.sender, _to), "not approved or owner");
..SNIP..
714      function updateUnlockTime(uint256 _tokenId, uint256 _lockDuration, bool _maxLockEnabled) external nonreentrant {
715:         require(_isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
..SNIP..
741      function withdraw(uint256 _tokenId) public nonreentrant {
742:         require(_isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
..SNIP..
778      function startCooldown(uint256 _tokenId) external {
779:         require(_isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
..SNIP..
826:     function _isApprovedOrOwner(address _spender, uint256 _tokenId) internal view returns (bool) {
827          address owner = idToOwner[_tokenId];
828          bool spenderIsOwner = owner == _spender;
829          bool spenderIsApproved = _spender == idToApprovals[_tokenId];
830          bool spenderIsApprovedForAll = (ownerToOperators[owner])[_spender];
831          return spenderIsOwner || spenderIsApproved || spenderIsApprovedForAll;
832      }
..SNIP..
928      function _transferFrom(address _from, address _to, uint256 _tokenId, address _sender) internal {
..SNIP..
931:         require(_isApprovedOrOwner(_sender, _tokenId));
932:         require(idToOwner[_tokenId] == _from, "from address is not owner");
```

However, two of these functions are in question in this report; `merge` and `withdraw`, since the `_burn` function is used.

The `_burn` function within the `VotingEscrow` contract is designed to remove tokens from circulation by clearing approvals and performing other state updates. However, due to the way access control is implemented, only the token owner or an entity approved for all user tokens can successfully execute this function without causing a revert. This is because the function internally calls `approve(address(0), _tokenId)`, which checks for broader permissions than those granted to entities approved for a single token.

## Impact Details

This results in operational disruptions for users who are legitimately authorized to perform actions (like `merge` and `withdraw`) that rely on `_burn`. They may encounter transaction reverts, leading to:

* Inability to merge tokens for voting power consolidation or management.
* Failed attempts to withdraw tokens post-lock period, leading to user dissatisfaction and potential disruption in planned economic activities within the platform.

## Recommended Mitigation

* Consider refactoring approval logic in `_burn`, Modify the internal logic of the `_burn` function to handle cases where the caller is only approved for the specific token ID being burned. This could involve bypassing the approval reset or adjusting the approval check to recognize and allow this scenario.
* Consider standardizing the approval reset process to use `_clearApproval`, especially in contexts where specific token approval is sufficient. This would align the behavior across different contract functions and improve the reliability of operations involving token transfers, burns, or modifications.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1567>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L501C1-L514C6>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L912C5-L919C6>

## Proof of Concept

Consider a scenario where a user has received specific approval for token ID `123` to facilitate a token merge or withdrawal. Under the current implementation, if this user attempts to initiate these actions, the transaction will revert during the `_burn` call due to the internal [`approve`](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L501C1-L514C6) function not recognizing their limited approval as sufficient.

```solidity
src/VotingEscrow.sol:
  501      function approve(address _approved, uint256 _tokenId) public {
..SNIP..
  507          // Check requirements
  508          bool senderIsOwner = (owner == msg.sender);
  509          bool senderIsApprovedForAll = (ownerToOperators[owner])[msg.sender];
  510:         require(senderIsOwner || senderIsApprovedForAll, "sender is not owner or approved");
```

This scenario can be replicated in a test environment to demonstrate the failure mechanism and its impact on contract usability.

### Test Case (Foundry)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract VotingEscrowPoC is BaseTest {
    uint256 internal constant THREE_WEEKS = 3 weeks;

    function setUp() public {
        setupContracts(block.timestamp);
    }

    // Withdraw reverts for token approved address
    function testWithdrawReversForTokenApprovedAddress() public {
        // Create address for Alice
        address alice = address(0x00001);

        // Create new token for Alice
        uint256 tokenId = createVeAlcx(alice, TOKEN_1, THREE_WEEKS, false);

        // alice start interacions
        hevm.startPrank(alice);

        // Reset the token status 
        voter.reset(tokenId);
        // Finish the epoch
        hevm.warp(newEpoch());
        voter.distribute();

        // Start cooldown once lock is expired
        veALCX.startCooldown(tokenId);
        // Go to the next epoch
        hevm.warp(newEpoch());

        // Now the token is withdrawable
        // Alice approve admin address on `tokenId`
        veALCX.approve(admin, tokenId);

        hevm.stopPrank();

        hevm.startPrank(admin);
        // Make sure admin is approved for alice token
        assertTrue(veALCX.isApprovedOrOwner(admin, tokenId));

        // Now if Admin try to withdraw, 
        // the call will revert with message that suggest the caller is not owner or approved.
        // While Admin is approved for the specific tokenId.
        hevm.expectRevert(abi.encodePacked("sender is not owner or approved"));
        veALCX.withdraw(tokenId);

        // This happen because the underline function `_burn` call `approve` with zero address to clear specific token approvals,
        // which revert if the caller not owner nor approved for all.
        hevm.expectRevert(abi.encodePacked("sender is not owner or approved"));
        veALCX.approve(address(0), tokenId);

        // However, Admin can use transferFrom function and then withdraw 
        veALCX.transferFrom(alice, admin, tokenId);
        veALCX.withdraw(tokenId);

        hevm.stopPrank();
    }
}
```

#### Test Output

```bash
alchemix-v2-dao main 1m38s
❯ make test_file_test
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/*** --match-path src/test/VotingEscrowPoC.t.sol --match-test testWithdrawReversForTokenApprovedAddress -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/VotingEscrowPoC.t.sol:VotingEscrowPoC
[PASS] testWithdrawReversForTokenApprovedAddress() (gas: 7657391)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 90.57s (77.12s CPU time)

Ran 1 test suite in 91.85s (90.57s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
