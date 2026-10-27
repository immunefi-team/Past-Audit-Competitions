# 29145 - \[SC - High] zeroLendToken is bricked to use for whitelisted...

Submitted on Mar 8th 2024 at 16:04:07 UTC by @oxumarkhatab for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29145

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The protocol intends to only allow whitelisted users to use the zeroLend Token. While its current implementation successfully stops the blacklisted address, it altogether stops the whitelisted users to transfer their tokens or anything with the token that includes \_update method to be called in its underlying execution.

however, counter-intuitively, the non-whitelisted people are allowed to interact with the token due to a logical flaw in the code.

This will make users feel a loss of control of their NFTs.

And if the airdropped zeroLend Token is worth much, this is essentially a loss of funds & NFTs scenario for whitelisted addresses too due to not being able to transact it .

Loss of protocol because there won't be enough activity on the platform and users might leave the protocol.

## Vulnerability Details

The zeroLend Token is going to be the incentive mechanism for people contributing to the zeroLend ecosystem. It has this custom functionality of blacklisting and whitelisting addresses. When a person is whitelisted , and they behave maliciously , they are put into blacklist using

```
function toggleBlacklist(
        address who,
        bool what
    ) public onlyRole(RISK_MANAGER_ROLE) {
        blacklisted[who] = what;
    }
```

And can become whitelisted using

```
   function toggleWhitelist(
        address who,
        bool what
    ) public onlyRole(RISK_MANAGER_ROLE) {
        whitelisted[who] = what;
    }
```

Upon some kind of hack or some incident , the protocol also has the functionality of pause

```
 function togglePause(bool what) public onlyRole(RISK_MANAGER_ROLE) {
        paused = what;
    }
```

But where do we enforce these checks ? Before understanding that , let's understand stucture of code :

The protocol's zeroLendToken is an ERC20Permit standard token from openzeppelin that inherits from ERC20 .

Which contains all the standard ERC20 functions . Among them most popular are \_mint,\_burn and transfer.

Now whenever a user does something with their tokens , inside the implementation of OZ ERC20 ,wer have an internal \_update method that does update the internal mappings

where it is used ?

```
 function _transfer(address from, address to, uint256 value) internal {
           // remaining code
->        _update(from, to, value);
    }
  function _mint(address account, uint256 value) internal {
        if (account == address(0)) {
            revert ERC20InvalidReceiver(address(0));
        }
->        _update(address(0), account, value);
    }
   function _burn(address account, uint256 value) internal {
        if (account == address(0)) {
            revert ERC20InvalidSender(address(0));
        }
        _update(account, address(0), value);
    }
```

So you see update is used in the core functions , so anyone that is trying to send , mint or burn ERC20 Tokens, it has to go through \_update method

what does \_update does ? it just updates the internal accounting based on passed parametrs

```
function _update(address from, address to, uint256 value) internal virtual {
        if (from == address(0)) {
            // Overflow check required: The rest of the code assumes that totalSupply never overflows
            _totalSupply += value;
        } else {
            uint256 fromBalance = _balances[from];
            if (fromBalance < value) {
                revert ERC20InsufficientBalance(from, fromBalance, value);
            }
            unchecked {
                // Overflow not possible: value <= fromBalance <= totalSupply.
                _balances[from] = fromBalance - value;
            }
        }

        if (to == address(0)) {
            unchecked {
                // Overflow not possible: value <= totalSupply or value <= fromBalance <= totalSupply.
                _totalSupply -= value;
            }
        } else {
            unchecked {
                // Overflow not possible: balance + value is at most totalSupply, which we know fits into a uint256.
                _balances[to] += value;
            }
        }

        emit Transfer(from, to, value);
    }
```

While you maybe familiar with \_update of ERC20 , zeroLendToken does something cool.

It overrides the parent's implementation (ERC20 \_update) , add custom logic , and also uses the sweet battle-tested implementation of the parent ERC20 contract from OZ.

```solidity

 function _update(
        address from,
        address to,
        uint256 value
    ) internal virtual override {
        require(!blacklisted[from] && !blacklisted[to], "blacklisted");
        require(!paused && !whitelisted[from], "paused");
        super._update(from, to, value);
    }

```

It implements three checks.

1. Addresses that are interacting with zeroLendToken are not blacklisted?

`require(!blacklisted[from] && !blacklisted[to], "blacklisted");`

2. Contract operation should not be paused

* hence the first part of the second require

`require(!paused && , "paused");`

3. The person who is using the token is whitelisted

`require( !whitelisted[from], "paused");` 2 & 3 are combined as `require(!paused && !whitelisted[from], "paused");`

However, note the mistake here, instead of checking `whitelisted[from]` and letting the require pass , the protocol checks invalid negation .. `whitelisted[from]` ,

which means whitelisted users will not be able to make transactions as the source of the transaction .

This also means Non-whitelisted users ( who are neiter whitelisted nor blacklisted ) are smoothly welcomed to interact with the token.

This breaks the logic of the protocol to only allow the whitelisted users to use zeroLendToken.

This will cause some serious issues if the attacker is creative.

Although you would argue that i've not illustrated the impact will have , but breaking the protocol logic can be itself an impact and Ryker will know it's impact more than we do because he would connect the dots looking forward to what disaster could this lead to.

## Impact Details

* Loss of user activity and functioning for protocol
* Loss of NFTs for whitelisted users
* UnAuthorized access of non-whitelisted addresses to the protocol

## References

see Poc for details

## Proof of Concept

Here's a sample PoC that can be used to visualize the bricking of the token.

```solidity

function test_BrickedZeroLendToken()public {
        address user1=0x17A8d3c727ec8fDc8BD859627F18ce89c31E1E8b;
        address user2=0x97A8d3c727ec8fDc8BD859627F18ce89c31E1E8c;
        address user3=0x37B8d3c727ec8fDc8BD859627F18ce89c31E1E9D;
        address user4=0x47C8d3c727ec8fDc8BD859627F18ce89c31E1E6e;
        vm.prank(Owner);
        ZeroLendToken zlToken=new ZeroLendToken();
        // Owner un-pause token
        zlToken.togglePause(false);
        zlToken.toggleWhitelist(user1);
        zlToken.toggleWhitelist(user2);
        // The owner is neither whitelisted , nor blacklisted ,
        // so current implementation allows the transfer 

        zlToken.transfer(user1,3 ether);
        zlToken.transfer(user2,3 ether);
        
        // protocol operates for some time
        // owner decides to blacklist user2 due to some suspicious behavior reported.
        zlToken.toggleBlacklist(user2);
        vm.stopPrank();

        vm.prank(user2);
        // This call will fail , because the user is blacklisted , so it can not transfer the tokens to any ther
        zlToken.transfer(user3,1 ether);
        vm.stopPrank();
        
        // but interestingly , 
        vm.prank(user1);
        // This call will also fail , because the user is whitelisted , so it can not transfer the tokens to any ther
        zlToken.transfer(user3,1 ether);
        vm.stopPrank();

        // but the door remains open for Non-whitelisted users
        // but there's 
        

    }
```
