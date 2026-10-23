# 29213 - \[SC - High] The function always revert if \_stakeNFT True d...

Submitted on Mar 10th 2024 at 20:34:07 UTC by @stiglitz for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29213

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of funds
* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

See the function `safeTransferFrom::_createLock`:

```solidity
if (_stakeNFT) {
        _mint(address(this), _tokenId);
        bytes memory data = abi.encode(_to);
        safeTransferFrom(address(this), address(staking), _tokenId, data);
} else _mint(_to, _tokenId);
```

if `_stakeNFT == True`, the tx always reverts because `msg.sender` is the user who called `createLock`, and the user does not have approval for moving NFT owned by someone else. As we can see in the code, the NFT is minted to `address(this)`, so the user is not the owner, and he needs approval from the contact.

However, the problem imho is the fact that the NFT is not even owned by a user but by the contract itself. Because if approval is implemented to make the transfer succeed, the user has no ability to get what is his.

## Vulnerability Details

Let's also look at the `OmnichainStaking::onERC721Received`:

```solidity
if (msg.sender == address(lpLocker)) {
        lpPower[tokenId] = lpLocker.balanceOfNFT(tokenId);
        _mint(from, lpPower[tokenId] * 4);
}
```

And also the function `OmnichainStaking::unstakeLP`:

```solidity
function unstakeLP(uint256 tokenId) external { /
        _burn(msg.sender, lpPower[tokenId] * 4);
        lpLocker.safeTransferFrom(address(this), msg.sender, tokenId);
}
```

Lets say we have the approval and we create a lock with `_stakeNFT == True`. The function `safeTransferFrom` is executed where `from == address(this)` which is `BaseLocker` (`LockerLP`). NOT A USER.

Who is gonna call `unstakeLP` ? The tokens are owner by the `LockerLP` contract, and only the contract can unstake them.

## Impact Details

The protocol does not work as it is supposed to, and if the function with `_stakeNFT == True` is called and it works (it does not right now) - The user will basically lose his invested money into veNFT and the voting power will be useless owned by the protocol contract.

I think the impact is very high and I would say critical. But because of the missing approval, the function does not work at all, so the impact is not there :D

## References

Add any relevant links to documentation or code

## Proof of Concept

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

### X contract (ERC721 compatible)

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

### Y contract (ERC721 compatible)

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

### Failing test

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

Go to wake `tests` folder and paste this code in tests/test_approval.py and run
    $ wake test tests/test_approval.py

If you are interest I would be happy to provide more examples + complete protocol deployment and fuzz testing
(yout tests are not good tbh)
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
def test_approval():
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
    locker.createLockFor(amount, two_weeks,x ,True, from_=bob)
    # This tx with _stakeNFT == True ALWAYS FAILS

    # ===================================================== #




```
