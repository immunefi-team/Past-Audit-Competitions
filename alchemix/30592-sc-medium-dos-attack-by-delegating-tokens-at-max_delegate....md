# 30592 - \[SC - Medium] DOS attack by delegating tokens at MAX\_DELEGATE...

Submitted on May 1st 2024 at 15:20:17 UTC by @oxumarkhatab for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30592

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of NFTs
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In Alchemix v2 DAO's `VotingEscrow` contract, the `MAX_DELEGATES` limit is set to `1024`. This amount of delegates takes 25M gas to be processed,.However if the contracts are deployed on EVM chains having less than 25M gas block limit , Especially Optimism which has only 15M gas limit.There will be denial of service in system's core opeations especially during token transfer/withdrawal when there are 1024 delegated votes on a token.

## Resubmission

This report is a resubmission of Repoer#30549 but this one includes a runnable PoC whose output is also attached in the given secret gist.

## Vulnerability details

Any user can give their locked NFT balance to someone else using the "delegate" function. But in the "VotingEscrow" contract, there's a rule called MAX\_DELEGATES. It stops any address from having too many tokens.

here is the relevant code

VotingEscrow\.sol

```solidity

// state variable
 Line 34   uint256 public constant MAX_DELEGATES = 1024; // avoid too much gas
...
_moveTokenDelegates() method
L1040 :                require(dstTokensOld.length + 1 <= MAX_DELEGATES, "dst would have too many tokenIds");

...
_moveAllDelegates() method

 require(dstTokensOld.length + ownerTokenCount <= MAX_DELEGATES, "dst would have too many tokenIds");

```

This rule helps stop attacks that could slow down or stop the contract.

Right now, if a user has 1024 delegated tokens, it takes about 25 million gas to move, burn, or make new tokens.

The `_moveTokenDelegates` is invoked inside mint , burn & transferFrom

```solidity

  function _transferFrom(address _from, address _to, uint256 _tokenId, address _sender) internal {
        //snip
        _moveTokenDelegates(delegates(_from), delegates(_to), _tokenId);
        //snip
    }
      function _mint(address _to, uint256 _tokenId) internal returns (bool) {
         //snip
        _moveTokenDelegates(address(0), delegates(_to), _tokenId);
        //snip
    }
    
      function _burn(uint256 _tokenId, uint256 _value) internal {
         //snip
        _moveTokenDelegates(delegates(owner), address(0), _tokenId);
        //snip
        
    }
```

But the gas limit on some target chains might be less than 25M gas , Most importanly one of the optimism chain which is only 15 million.

As Alchemix already has deployed contracts across EVM chains like Arbitrum & Optimism

<https://alchemix-finance.gitbook.io/user-docs/contracts#optimism>

we see its a Critical concern.

Also, it's cheaper to give tokens from an address with fewer tokens to one with more.

This sets up a problem. An attacker could make a new address, lock tokens, and give them to someone else and cause DoS to them for spending their tokens.

## Impact

* Increased gas costs for token transfer/withdrawal when there are 1024 delegated votes on a token.
* Potential denial of service (DoS) attack on victims, preventing them from withdrawing/transferring/delegating.

## Verification

This finding is inspired and verified from

Spearbit's Velodrome Audit : <https://solodit.xyz/issues/dos-attack-by-delegating-tokens-at-max\\_delegates-1024-spearbit-none-velodrome-finance-pdf>

## Recommendation

1. **Adjust MAX\_DELEGATES:** Reduce MAX\_DELEGATES from 1024 to 128 to mitigate the risk of gas exhaustion during token transfer/withdrawal.
2. **Opt-out/Opt-in Mechanism:** Provide users with the option to opt-out/opt-in. Users should only accept delegated tokens if they opt-in. Alternatively, they can opt-out to refuse any uncommissioned delegated tokens.

## Proof of Concept

Please check the provided gist.
