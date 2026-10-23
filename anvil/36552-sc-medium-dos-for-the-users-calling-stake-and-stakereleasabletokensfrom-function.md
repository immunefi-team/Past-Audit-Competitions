# #36552 \[SC-Medium] DoS for the user's calling \`stake\` and \`stakeReleasableTokensFrom\` function

**Submitted on Nov 5th 2024 at 20:34:01 UTC by @savi0ur for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36552
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

In the \`TimeBasedCollateralPool\` contract, user can call \`stake\` and \`stakeReleasableTokensFrom\` function to stake their tokens. However, both of this functions uses signed signature to modify collateralizable token allowance using \`modifyCollateralizableTokenAllowanceWithSignature\` function, as shown below.

\`\`\`solidity function stake( IERC20 \_token, uint256 \_amount, bytes calldata \_collateralizableApprovalSignature ) external withEligibleAccountTokensReleased(msg.sender, address(\_token)) returns (uint256) { if (\_collateralizableApprovalSignature.length > 0) { collateral.modifyCollateralizableTokenAllowanceWithSignature( msg.sender, address(this), address(\_token), Pricing.safeCastToInt256(\_amount), \_collateralizableApprovalSignature ); // <==== }

```
return _stake(_token, _amount);
```

} \`\`\`

\`\`\`solidity function stakeReleasableTokensFrom( ITimeBasedCollateralPool \_pool, IERC20 \_token, uint256 \_amount, bytes calldata \_collateralizableApprovalSignature ) external withEligibleAccountTokensReleased(msg.sender, address(\_token)) returns (uint256) { if (address(\_pool) != address(this)) { IERC20\[] memory tokens = new IERC20; tokens\[0] = \_token; \_pool.releaseEligibleTokens(msg.sender, tokens); } if (\_collateralizableApprovalSignature.length > 0) { collateral.modifyCollateralizableTokenAllowanceWithSignature( msg.sender, address(this), address(\_token), Pricing.safeCastToInt256(\_amount), \_collateralizableApprovalSignature ); // <==== }

```
return _stake(_token, _amount);
```

} \`\`\`

Whenever users call this functions, attacker can frontrun their transaction and uses their signed signature to directly call \`modifyCollateralizableTokenAllowanceWithSignature\` function on \`CollateralVault\` contract using all the other details from user's tx. Thus user's tx will get reverted due to Invalid Signature error.

As we can see below, \`modifyCollateralizableTokenAllowanceWithSignature\` function is directly callable with all the details from victim's tx.

\`\`\`solidity function modifyCollateralizableTokenAllowanceWithSignature( address \_accountAddress, address \_collateralizableContractAddress, address \_tokenAddress, int256 \_allowanceAdjustment, bytes calldata \_signature ) external { if (\_allowanceAdjustment > 0 && !collateralizableContracts\[\_collateralizableContractAddress]) revert ContractNotApprovedByProtocol(\_collateralizableContractAddress);

```
_modifyCollateralizableTokenAllowanceWithSignature(
    _accountAddress,
    _collateralizableContractAddress,
    _tokenAddress,
    _allowanceAdjustment,
    _signature
);
```

}

function \_modifyCollateralizableTokenAllowanceWithSignature( address \_accountAddress, address \_collateralizableContractAddress, address \_tokenAddress, int256 \_allowanceAdjustment, bytes calldata \_signature ) private { { bytes32 hash = \_hashTypedDataV4( keccak256( abi.encode( COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH, \_collateralizableContractAddress, \_tokenAddress, \_allowanceAdjustment, \_useNonce(\_accountAddress, COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH) ) ) ); if (!SignatureChecker.isValidSignatureNow(\_accountAddress, hash, \_signature)) { revert InvalidSignature(\_accountAddress); // <=== } \`\`\`

### Attack Scenario

1. Alice sign the tx off-chain and submit it to perform \`stake\`
2. While the Alice tx is in mempool, Bob \`(Attacker)\` can see it, frontrun it, and execute \`modifyCollateralizableTokenAllowanceWithSignature\` directly on the collateral vault contract with all the necessary details and signature of Alice.
3. Alice's tx now executes after Bob's tx and it will get reverted as the signature is already used.

## Impact

Attacker can make user's \`stake\` and \`stakeReleasableTokensFrom\` tx to always failed.

I'm marking this issue as **Medium** according to [Immunefi's vulnerability classification system](https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/) : Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).

\> **Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol):** Griefing is when the attacker calls certain functions of the smart contract that would put it in a suboptimal state, thus blocking normal function execution for any user. This would cause the user to lose money for sending the transaction, but when the smart contract is back to normal, the user would be able to call the function once again to complete it. In this instance, the attacker damaged the user by requiring them to send another transaction. The attacker does not profit, but they do damage the users or the protocol.

## Recommendation

Make sure the caller (\`msg.sender\`) for \`modifyCollateralizableTokenAllowanceWithSignature\` function in \`CollateralVault\` contract is approved collateralizable contracts in \`collateralizableContracts\` mapping as shown below.

\`\`\`diff function modifyCollateralizableTokenAllowanceWithSignature( address \_accountAddress, address \_collateralizableContractAddress, address \_tokenAddress, int256 \_allowanceAdjustment, bytes calldata \_signature ) external {

* if (!collateralizableContracts\[msg.sender]) revert Unauthorized(msg.sender); if (\_allowanceAdjustment > 0 && !collateralizableContracts\[\_collateralizableContractAddress]) revert ContractNotApprovedByProtocol(\_collateralizableContractAddress);

  \_modifyCollateralizableTokenAllowanceWithSignature( \_accountAddress, \_collateralizableContractAddress, \_tokenAddress, \_allowanceAdjustment, \_signature ); } \`\`\`

## References

* CollateralVault: <https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f?utm\\_source=immunefi>
* TimeBasedCollateralPool: <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a?utm\\_source=immunefi>

## Proof of Concept

## Proof Of Concept

PoC Secret Gist: <https://gist.github.com/pratraut/1a66b395e862383be9945713b89ff035>
