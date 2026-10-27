# 29270 - \[SC - High] The main functionality of the contract EarlyZER...

Submitted on Mar 12th 2024 at 22:40:12 UTC by @stiglitz for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29270

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Protocol insolvency
* Protocol contract does not work

## Description

## Brief/Intro

The function `EarlyZEROVesting::startVesting` is broken due to missing allowances between contracts.

## Vulnerability Details

The problem is in the following function call in the `startVesting` function:

```solidity
uint256 id = vesting.mint(
            stake ? address(this) : msg.sender, // address _who,
            (amount * 75) / 100, // uint256 _pending,
            (amount * 25) / 100, // uint256 _upfront,
            86400 * 30 * 3, // uint256 _linearDuration,
            86400 * 30, // uint256 _cliffDuration,
            block.timestamp, // uint256 _unlockDate,
            false, // bool _hasPenalty
            IVestedZeroNFT.VestCategory.EARLY_ZERO
);
```

Inside the `VestedZeroNFT::mint`, almost the last line contains the following call:

```solidity
zero.transferFrom(msg.sender, address(this), _pending + _upfront);
```

Where `msg.sender == EarlyZEROVesting`.

Because there is no allowance set `from == EarlyZEROVesting` to `spender == VestedZeroNFT`, the **TX will always revert** (`IERC20Errors.ERC20InsufficientBalance`)

***

This approval is necessary for the function `EarlyZEROVesting::startVesting`:

```solidity
function startVesting(uint256 amount, bool stake) external { 
    require(enableVesting || stake, "vesting not enabled; staking only");
    earlyZERO.burnFrom(msg.sender, amount);
    
    // Approve call here is necessary
    earlyZERO.approve(address(vesting),amount);
    ...
```

## Impact Details

The main functionality of the contract `EarlyZEROVesting`, which is `startVesting` is broken and always reverts

## References

PoC and .png shows the problem

## Proof of Concept

### Test

#### tests/test\_vested.py

```python
from wake.testing import *

from pytypes.openzeppelin.contracts.proxy.ERC1967.ERC1967Proxy import ERC1967Proxy
from pytypes.contracts.locker.OmnichainStaking import OmnichainStaking
from pytypes.contracts.locker.LockerToken import LockerToken
from pytypes.contracts.locker.LockerLP import LockerLP
from pytypes.tests.VeToken import VeToken


from pytypes.contracts.vesting.earlyzero.EarlyZEROVesting import EarlyZEROVesting
from pytypes.contracts.vesting.earlyzero.EarlyZERO import EarlyZERO
from pytypes.contracts.vesting.VestedZeroNFT import VestedZeroNFT
from pytypes.contracts.vesting.StakingBonus import StakingBonus

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

Go to wake `tests` folder and paste this code in tests/test_vested.py and run
    $ wake test tests/test_vested.py

'''


def deploy_with_proxy(contract):
    impl = contract.deploy()
    proxy = ERC1967Proxy.deploy(impl, b"")
    return contract(proxy)

# Print failing tx call trace
def revert_handler(e: TransactionRevertedError):
    if e.tx is not None:
        print(e.tx.call_trace)
        print(e.tx.events)

@default_chain.connect()
@on_revert(revert_handler)
def test_vested():
    # ======================DEPLOY========================= #
    random  = default_chain.accounts[9]
    owner   = default_chain.accounts[0]
    bob     = default_chain.accounts[1]
    
    # Deploy mock token
    zero_token = EarlyZERO.deploy(from_=owner)
    ve_token = VeToken.deploy(100*10**18, from_=bob)
    
    # Proxy deployment
    zero_vesting = deploy_with_proxy(EarlyZEROVesting)
    omnichain = deploy_with_proxy(OmnichainStaking)
    staking_bonus = deploy_with_proxy(StakingBonus)
    vested_zero_nft = deploy_with_proxy(VestedZeroNFT)
    locker_lp = deploy_with_proxy(LockerLP)
    locker_token = deploy_with_proxy(LockerToken)
    
    # Init deployment
    zero_vesting.init(zero_token, vested_zero_nft, staking_bonus, from_=owner)
    omnichain.init(random, locker_token, locker_lp, from_=owner)
    staking_bonus.init(zero_token, locker_token, vested_zero_nft, 100, from_=owner)
    vested_zero_nft.init(zero_token, staking_bonus, from_=owner)
    locker_lp.init(ve_token, omnichain, random, from_=owner)
    locker_token.init(ve_token, omnichain, random, from_=owner)
    # Send something to bob
    zero_token.transfer(bob, 100*10**18, from_=owner)
    zero_token.transfer(zero_vesting, 100*10**18, from_=owner)
    # Disable whitelist and blacklist
    zero_token.toggleWhitelist(False, False, from_=owner)
    zero_token.approve(zero_vesting, 100*10**18, from_=bob)
    print(vested_zero_nft.address)
    print(zero_vesting.address)
    #zero_token.approve(vested_zero, 100*10**18, from_=zero_vesting)

    zero_vesting.toggleVesting(from_=owner)
    zero_vesting.startVesting(100*10**18, True, from_=bob)

    # Just simply run test
    # On-revert handler will print call trace and errors of the reverting TX


```
