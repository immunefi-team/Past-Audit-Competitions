# 28890 - \[SC - Insight] EBTCTokensol mint function lack of checks allow\...

Submitted on Mar 1st 2024 at 01:09:47 UTC by @cryptonoob2k for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28890

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/EBTCToken.sol>

Impacts:

* Permanent freezing of funds

## Description

## Bug Description

EBTCToken.sol mint function logic is incompatible with restrictions implemented in EBTCToken.sol::transfer and EBTCToken.sol::transferFrom methods that prevents EBTCToken holding EBTC tokens breaking EBTCToken balance restriction and leading to EBTC tokens funds stuck in contract unable to recover

## Brief/Intro

EBTCToken.sol::transfer and EBTCToken.sol::transferFrom methods implements restrictions to block users to send EBTC tokens to EBTCToken contract, thus ensuring EBTCToken contract EBTC balance always remains 0.\
However this restriction can be bypassed using mint function.

## Vulnerability Details

The restriction inside transfer and transferFrom are implemented using the internal function `_requireValidRecipient`:

```js
function _requireValidRecipient(address _recipient) internal view {
    require(
        _recipient != address(0) && _recipient != address(this),	// <@ block
        "EBTC: Cannot transfer tokens directly to the EBTC token contract or the zero address"
    );
    //...
}
```

This function ensures that EBTCToken's EBTC balance remains 0 because it blocks transfer to EBTCToken address:

```js
contract EBTCToken is IEBTCToken, AuthNoOwner, PermitNonce {
	//...
    function transfer(address recipient, uint256 amount) external override returns (bool) {
        _requireValidRecipient(recipient);
        _transfer(msg.sender, recipient, amount);
        return true;
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external override returns (bool) {
    	_requireValidRecipient(recipient);
        _transfer(sender, recipient, amount);
        //...
    }
```

However this restriction doesnt hold if a user mints tokens directly to this contract, because in mint function there isnt this check in place:

```js
    function mint(address _account, uint256 _amount) external override {
        _requireCallerIsBOorCdpMOrAuth();	// <@ no restriction 
        _mint(_account, _amount);
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "EBTCToken: mint to zero recipient!");

        _totalSupply = _totalSupply + amount;
        _balances[account] = _balances[account] + amount;
        emit Transfer(address(0), account, amount);
    }
```

## Impact Details

By using mint function to directly issue EBTC tokens to EBTCToken contract the restrictions implemented in transfer and transferFrom functions to keep EBTCToken balance to 0 are bypassed allowing EBTCToken contract to hold tokens and have EBTC tokens stuck in contract

## Risk Breakdown

The vulnerability is easy to exploit, however to exploit it mint capability is needed leading to stuck tokens in EBTCToken contract and balance restriction bypass

## Recommendation

Implement a restriction in mint function like the ones implemented in transfer and transferFrom function such as

```
    function mint(address _account, uint256 _amount) external override {
        _requireCallerIsBOorCdpMOrAuth();
       _requireValidRecipient(recipient);
        _mint(_account, _amount);
    }
```

## Proof of Concept

Here is a foundry test file, save it in packages/contracts/foundry\_test subdir and run it with:

```bash
forge test -vvv --match-contract EBTCTokenMintToItself
```

Code:

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;
import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseFixture} from "./BaseFixture.sol";

contract EBTCTokenMintToItself is eBTCBaseFixture {
    uint256 public mintAmount = 1e18;

    function setUp() public override {
        eBTCBaseFixture.setUp();
        eBTCBaseFixture.connectCoreContracts();
        eBTCBaseFixture.connectLQTYContractsToCore();
    }

    function testEBTCUserWithMintingPermisionCanMint() public {
        address user = _utils.getNextUserAddress();
        // Grant mint permissions to user
        vm.prank(defaultGovernance);
        authority.setUserRole(user, 1, true);

        // Starting balance
        uint256 totalSupply0 = eBTCToken.totalSupply();
        uint256 balanceOfeBTCToken0 = eBTCToken.balanceOf(address(eBTCToken));
        console.log("\n********** Starting balance ************");
        console.log("eBTCToken.balanceOf(address(eBTCToken)) ",eBTCToken.balanceOf(address(eBTCToken)));

        vm.startPrank(user);
        vm.deal(user, type(uint96).max);

        // User can mint to eBTCToken
        eBTCToken.mint(address(eBTCToken), mintAmount);

        vm.stopPrank();

        // Check balance
        uint256 totalSupply1 = eBTCToken.totalSupply();
        uint256 balanceOfeBTCToken1 = eBTCToken.balanceOf(address(eBTCToken));

        console.log("\n********** Final balance ************");
        console.log("eBTCToken.balanceOf(address(eBTCToken)) ",eBTCToken.balanceOf(address(eBTCToken)));

        assertEq(totalSupply1 - totalSupply0, mintAmount);
        assertEq(balanceOfeBTCToken1 - balanceOfeBTCToken0, mintAmount);
        assertGt(balanceOfeBTCToken1,0);
    }
}
```
