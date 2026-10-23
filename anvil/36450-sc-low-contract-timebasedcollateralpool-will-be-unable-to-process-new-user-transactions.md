# #36450 \[SC-Low] contract timebasedcollateralpool will be unable to process new user transactions

## #36450 \[SC-Low] Contract TimeBasedCollateralPool will be unable to process new user transactions and user funds are temporary frozen if a user unstake transaction of TimeBasedCollateralPool execute...

**Submitted on Nov 3rd 2024 at 04:16:31 UTC by @perseverance for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36450
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Temporary freezing of funds within the TimeBasedCollateralPool for at least 48 hours

### Description

## Description

### Brief/Intro

The users can unstake from the TimeBasedCollateralPool by using the function unstake

\`\`\`solidity function unstake(IERC20 \_token, uint256 \_poolUnits) external {} \`\`\`

### The vulnerability

#### Vulnerability Details

The impact described in this bug report can happen in the Scenario:

If a victim user want to unstake his tokens from the pool, then he call ustake(). An user with RESETTER\_ROLE reset the pool (may be via Governance Proposal Execution) and this transaction got executed before the victim's transaction. The victim's unstake transaction got executed after reset pool.

But the scenario can happen because of the **race condition** in Ethereum and the reset pool transaction can get executed before the user's unstake transaction. **Because of decentralized and mempool, There is possibility that this scneario can happen.**

Step 1: So the pool is reset.

Step 2: Now the victim's unstake transaction got executed

\`\`\`solidity function unstake(IERC20 \_token, uint256 \_poolUnits) external { if (\_poolUnits == 0) revert UnstakeAmountZero();

```
    address tokenAddress &#x3D; address(_token);
```

701: if ( 702: accountTokenState\[msg.sender]\[tokenAddress].totalUnits - 703: getTotalAccountUnitsPendingUnstake(msg.sender, tokenAddress) < 704: \_poolUnits ) { revert InsufficientBalance( \_poolUnits, accountTokenState\[msg.sender]\[tokenAddress].totalUnits - getTotalAccountUnitsPendingUnstake(msg.sender, tokenAddress) ); }

```
    {
        // Release eligible account tokens. If a reset occurred, there is nothing left to unstake, so return.
```

715: bool accountWasReset = \_releaseEligibleAccountTokens(msg.sender, tokenAddress); if (accountWasReset) { // Do not revert because the user wanted tokens unstaked, and those were unstaked and released. return; } }

```
    _addToAccountPendingUnstakeNextEpoch(tokenAddress, msg.sender, _poolUnits);
    _addToContractPendingUnstakeNextEpoch(tokenAddress, _poolUnits);

    emit UnstakeInitiated(msg.sender, _token, _poolUnits, getEpochEndTimestamp(getCurrentEpoch() + 1));
}
```

\`\`\`

So after the reset pool event, when unstake(), at Line 701-704, the accountTokenState\[msg.sender]\[tokenAddress].totalUnits is still the totalUnits of the user before the reset pool event.

But in Line 715 the internal function \_releaseEligibleAccountTokens is called. In this internal function, if the pool was reset, then the tokens and units of the user are released.

\`\`\`solidity function \_releaseEligibleAccountTokens( address \_account, address \_tokenAddress ) internal returns (bool \_poolWasReset) { \_poolWasReset = \_unlockEligibleTokenContractPendingUnstakes(\_tokenAddress);

```
    (uint256 totalUnitsToRelease, uint256 totalTokensToRelease) &#x3D; _resetAccountTokenStateIfApplicable(
        _account,
        _tokenAddress
    );

    if (totalUnitsToRelease &#x3D;&#x3D; 0) {
        (uint256 units, uint256 tokens) &#x3D; _processAccountTokenUnstakes(_account, _tokenAddress);
        totalUnitsToRelease +&#x3D; units;
        totalTokensToRelease +&#x3D; tokens;
    }

    if (totalUnitsToRelease &#x3D;&#x3D; 0) {
        return _poolWasReset;
    }

    collateral.transferCollateral(_tokenAddress, totalTokensToRelease, _account);

    emit CollateralReleased(IERC20(_tokenAddress), totalTokensToRelease, totalUnitsToRelease, _account);
}
```

\`\`\`

After the the tokens and units of the user are released, the user's totalUnits is reset to 0.

Please note that \_releaseEligibleAccountTokens called inside unstake() still return **false** means that the pool was not reset because in this scenario, the reset happened before the unstake transaction.

Note: If the resetPool happened inside the \_releaseEligibleAccountTokens function, then \_releaseEligibleAccountTokens will return true.

But in the function unstake, the **accountWasReset** is false, so it still process unstake by calling

\`\`\`solidity \_addToAccountPendingUnstakeNextEpoch(tokenAddress, msg.sender, \_poolUnits); \_addToContractPendingUnstakeNextEpoch(tokenAddress, \_poolUnits);

```
    emit UnstakeInitiated(msg.sender, _token, _poolUnits, getEpochEndTimestamp(getCurrentEpoch() + 1));
```

\`\`\`

So now for the victim the state of accountTokenState is incorrect because

\`\`\`solidity accountTokenState.totalUnits = 0 accountTokenState.firstPendingUnstakeUnits != 0 \`\`\` The pool state is also incorrect

\`\`\` contractStateStorage.totalUnits = 0;

contractStateStorage.firstPendingUnstakeUnits != 0 and = unstake Units amount.

the ReserveCollateral of TimeBasedCollateralPool was also reset to 0

\`\`\`

So the victim and other user future transactions **after the willCompleteAtTimestampSeconds** time, will call the modifier withEligibleAccountTokensReleased

\`\`\`solidity modifier withEligibleAccountTokensReleased(address \_account, address \_tokenAddress) { \_releaseEligibleAccountTokens(\_account, \_tokenAddress);

```
    _;
}
```

\`\`\`

And this modifier will call function **\_unlockEligibleTokenContractPendingUnstakes**

\`\`\`solidity function \_releaseEligibleAccountTokens( address \_account, address \_tokenAddress ) internal returns (bool \_poolWasReset) { \_poolWasReset = \_unlockEligibleTokenContractPendingUnstakes(\_tokenAddress); //... } \`\`\`

\`\`\`solidity function \_unlockEligibleTokenContractPendingUnstakes(address \_tokenAddress) internal returns (bool) {

```
    // ... 

    if ((firstVestedUnits + secondVestedUnits) &#x3D;&#x3D; totalPoolUnits) {
        collateral.releaseAllCollateral(contractStateStorage.collateralReservationId);
        contractStateStorage.collateralReservationId &#x3D; 0;
    } else {
        uint256 tokensToRelease &#x3D; firstVestedTokens + secondVestedTokens;
        if (tokensToRelease &gt; 0) {
            try
                collateral.modifyCollateralReservation(
                    contractStateStorage.collateralReservationId,
                    -Pricing.safeCastToInt256(tokensToRelease)
                )
            {} catch (bytes memory reason) {
                if (bytes4(reason) &#x3D;&#x3D; ICollateral.ClaimableAmountZero.selector) {
                    // If we&#x27;re here, it means that the result of lowering the collateral amount is a reservation with
                    // 0 claimable balance. The only way to get this collateral out is to release less or more. Less
                    // invalidates the unstaking logic, so we choose to release more, resetting the dust that remains.
                    _resetPool(_tokenAddress);
                    return true;
                } else {
                    assembly {
                        revert(add(reason, 0x20), mload(reason))
                    }
                }
            }
        }
    }

    // ... 
}
```

\`\`\`

Since the collateral.modifyCollateralReservation will revert in **Line 748** in function \_modifyCollateralReservation with the check because the oldReservedAmount was reset and the transaction want to release unstake amount.

**if (byAmountUint >= oldReservedAmount) revert InsufficientCollateral(byAmountUint, oldReservedAmount); // @audit-issue**

**Note: In the POC, the victim or some other user stake small amount between the unstake transaction timestamp and the willCompleteAtTimestampSeconds, so the oldReservedAmount is not 0. But if no transactions happen during this time frame, then the oldReservedAmount is 0.**

\`\`\`solidity function \_modifyCollateralReservation( uint96 \_reservationId, int256 \_byAmount ) internal returns (uint256 \_reservedCollateral, uint256 \_claimableCollateral) { // Redacted

```
    if (_byAmount &lt; 0) {
        uint256 byAmountUint &#x3D; uint256(-_byAmount);
```

Line 748: if (byAmountUint >= oldReservedAmount) revert InsufficientCollateral(byAmountUint, oldReservedAmount); // @audit-issue

```
        _reservedCollateral &#x3D; oldReservedAmount - byAmountUint;
        reservationStorage.tokenAmount &#x3D; _reservedCollateral;

        address account &#x3D; reservationStorage.account;
        address tokenAddress &#x3D; reservationStorage.tokenAddress;

        CollateralBalance storage balanceStorage &#x3D; accountBalances[account][tokenAddress];
        balanceStorage.reserved -&#x3D; byAmountUint;
        balanceStorage.available +&#x3D; byAmountUint;
    } else {
       
}
```

\`\`\` So all the transactions will revert in collateral.modifyCollateralReservation with the reason **InsufficientCollateral**

**Since all user transactions will call the modifier withEligibleAccountTokensReleased, so all user transactions will revert.**

## Impacts

## About the severity assessment

If the scenario in this bugs occur, then **all of the victim and user future transactions after willCompleteAtTimestampSeconds of unstake transaction always revert**. This will cause denial of service of the protocol and temporary frozen of funds in TimeBasedCollateralPool contract.

This will result that in the owner or the protocol team need to manualy interveen to rescue the funds and fix the situation, maybe to propose a Governance Proposal and execute it.

Bug Severity: Low

Impact Category: Temporary freezing of funds within the TimeBasedCollateralPool for at least 48 hours

Probability: Medium, likely

Because the protocol team will need to investigate the root cause and pass a proposal to fix this situation that might take 3-4 days so the freezing of funds can be longer than 48 hours.

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/f7643506a8ed7cc80ec111ce111151da#file-testtimebasedcollateralpool-sol>

### Proof of Concept

## Proof of concept

Steps to reproduce the bug

Step 1: The user unstake his tokens

Step 2: the user with RESETTER ROLE reset the pool for some reasons and because of race condition, this transaction got executed before step 1

Test code to show: \`\`\`solidity function testUnStake() public {

```
    console.log(&quot;Get getAccountTokenState&quot;);
    TimeBasedCollateralPool.AccountState memory accountTokenState &#x3D; TimeBasedCollateralPool(TimeBasedCollateralPool_contract).getAccountTokenState(victim, USDC); 
    console.log(&quot;accountTokenState: &quot;, victim, accountTokenState.totalUnits, accountTokenState.firstPendingUnstakeEpoch);
    uint256 unstake_amount &#x3D; accountTokenState.totalUnits; 
    console.log(&quot;Victim can unstake amount: &quot;,unstake_amount); 
    
    scenario &#x3D; 1; 
    console.log(&quot;Scenario: &quot;,scenario);
    resetPool(scenario); 

    accountTokenState &#x3D; TimeBasedCollateralPool(TimeBasedCollateralPool_contract).getAccountTokenState(victim, USDC); 
    console.log(&quot;accountTokenState before unstake victim: &quot;,victim);
    console.log(&quot;accountTokenState before unstake accountTokenState.totalUnits: &quot;,accountTokenState.totalUnits);
    console.log(&quot;accountTokenState before unstake accountTokenState.firstPendingUnstakeUnits: &quot;,accountTokenState.firstPendingUnstakeUnits);

    vm.startPrank(victim); 
    console.log(&quot;Step : Victim Unstake from TimeBasedCollateralPool&quot;);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).unstake(IERC20(USDC), unstake_amount);
    vm.stopPrank(); 
   
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(VictimPrivateKey, amount, victim_nonce++);
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool, stake amount: &quot;,amount);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
    
    
    vm.warp(1730605189); // Sun Nov 03 2024 03:39:49 GMT+0000
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(VictimPrivateKey, amount, victim_nonce);
   
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool&quot;);
    console.log(&quot;Victim transaction expected to revert&quot;); 
    vm.expectRevert();
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(PrivateKey_2, amount, user2_nonce++);
   
    vm.startPrank(user2);
    console.log(&quot;user2 stake to TimeBasedCollateralPool&quot;);
    console.log(&quot;user2 transaction expected to revert&quot;); 
    vm.expectRevert();
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
   

} 
```

\`\`\`

Test Log:

<https://gist.github.com/Perseverancesuccess2021/f7643506a8ed7cc80ec111ce111151da#file-testunstake\\_241103\\_0850-log>

\`\`\` \[PASS] testUnStake() (gas: 508557) Logs: Setup Precondition for the test case Deploying TimeBasedCollateralPool Proxy contract that is VisibleBeaconProxy points to TimeBasedCollateralPool\_beacon Initialize TimeBasedCollateralPool contract Approve the TimeBasedCollateralPool\_contract as the collateralizable contract Balance of USDC before depositing: 10000000000 Approve the CollateralVault\_contract to spend USDC to deposit USDC Step: Deposit 10000 USDC to CollateralVault for the user Balance of USDC before depositing: 10000000000 Approve the CollateralVault\_contract to spend USDC to deposit USDC Step: Deposit 10000 USDC to CollateralVault for the user Victim stake to TimeBasedCollateralPool. Stake amount: 100000000 user2 stake to TimeBasedCollateralPool, Stake amount: 100000000 Get getAccountTokenState accountTokenState: 0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7 100000000 0 Victim can unstake amount: 100000000 Scenario: 1 Step: reset pool Pool was reset accountTokenState before unstake victim: 0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7 accountTokenState before unstake accountTokenState.totalUnits: 100000000 accountTokenState before unstake accountTokenState.firstPendingUnstakeUnits: 0 Step : Victim Unstake from TimeBasedCollateralPool Victim stake to TimeBasedCollateralPool, stake amount: 50000000 Victim stake to TimeBasedCollateralPool Victim transaction expected to revert user2 stake to TimeBasedCollateralPool user2 transaction expected to revert

\`\`\`

**Explanation:**

Before the ResetPool transaction, the victim state

\`\`\` Get getAccountTokenState accountTokenState: 0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7 100000000 0 Victim can unstake amount: 100000000 \`\`\`

When user unstake transaction, can see from the log that his tokens was released and still the event UnstakeInitiated was emitted

\`\`\` ├─ \[92608] VisibleBeaconProxy::unstake(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, 100000000 \[1e8]) │ ├─ \[307] 0x1f00D6f7C18a8edf4f8Bb4Ead8a898aBDd9c9E14::implementation() \[staticcall] │ │ └─ ← \[Return] 0xd042C267758eDDf34B481E1F539d637e41db3e5a │ ├─ \[91594] 0xd042C267758eDDf34B481E1F539d637e41db3e5a::unstake(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, 100000000 \[1e8]) \[delegatecall] │ │ ├─ emit AccountResetNonceUpdated(account: victim: \[0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7], oldNonce: 0, newNonce: 1) │ │ ├─ \[8527] 0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f::transferCollateral(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, 100000000 \[1e8], victim: \[0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7]) │ │ │ ├─ emit CollateralTransferred(fromAccount: VisibleBeaconProxy: \[0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenAddress: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, toAccount: victim: \[0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7], tokenAmount: 100000000 \[1e8]) │ │ │ └─ ← \[Stop] │ │ ├─ emit CollateralReleased(token: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, tokenAmount: 100000000 \[1e8], poolUnits: 100000000 \[1e8], destinationAccount: victim: \[0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7]) │ │ ├─ emit UnstakeInitiated(account: victim: \[0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7], token: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, unitsToUnstake: 100000000 \[1e8], willCompleteAtTimestampSeconds: 1730605188 \[1.73e9]) │ │ └─ ← \[Stop] │ └─ ← \[Return] ├─ \[0] VM::stopPrank()

\`\`\`

Then after the time willCompleteAtTimestampSeconds then all user transactions will revert

\`\`\`solidity

```
     vm.warp(1730605189); // Sun Nov 03 2024 03:39:49 GMT+0000 // willCompleteAtTimestampSeconds + 1
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(VictimPrivateKey, amount, victim_nonce);
   
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool&quot;);
    console.log(&quot;Victim transaction expected to revert&quot;); 
    vm.expectRevert();
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(PrivateKey_2, amount, user2_nonce++);
   
    vm.startPrank(user2);
    console.log(&quot;user2 stake to TimeBasedCollateralPool&quot;);
    console.log(&quot;user2 transaction expected to revert&quot;); 
    vm.expectRevert();
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
```

\`\`\`

Note that:

In the POC, I have created the code with more contract log to make sure my bug is valid.\
To turn on the log, just modify the code and re-run the test

\`\`\`solidity uint256 contract\_log = 1; \`\`\` Log: <https://gist.github.com/Perseverancesuccess2021/f7643506a8ed7cc80ec111ce111151da#file-testunstake\\_241103\\_0900-log>

Full POC:

<https://gist.github.com/Perseverancesuccess2021/f7643506a8ed7cc80ec111ce111151da#file-testtimebasedcollateralpool-sol>

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
address public victim;
uint256 public VictimPrivateKey;
address public user2; 
uint256 public PrivateKey_2;
uint256 amount; 
bytes public signature; 
uint256 public scenario; 
uint256 contract_log &#x3D; 0; 
uint256 impact &#x3D; 1;
uint256 user2_nonce &#x3D; 0; 
uint256 victim_nonce &#x3D; 0; 

function setUp() public {
    vm.createSelectFork(&quot;https://rpc.ankr.com/eth&quot;, 21104482); // Nov-03-2024 03:39:47 AM +UTC
    VictimPrivateKey &#x3D; 0xa11ce; // This is just a sample, not important 
    victim &#x3D; vm.addr(VictimPrivateKey);
    PrivateKey_2 &#x3D; 0xa11ca; // This is just a sample, not important 
    user2&#x3D; vm.addr(PrivateKey_2);
    deal(USDC,victim, 10000*10**6); 
    deal(USDC,user2, 10000*10**6); 
    setup_precondition(); 
    vm.label(victim, &quot;victim&quot;);
    vm.label(user2, &quot;user2&quot;);
    
}

function deployContract() public {
    console.log(&quot;Deploying TimeBasedCollateralPool Proxy contract that is VisibleBeaconProxy points to TimeBasedCollateralPool_beacon&quot;);
    bytes memory data &#x3D; &quot;&quot;; 
    TimeBasedCollateralPool_contract  &#x3D;  address(new VisibleBeaconProxy(TimeBasedCollateralPool_beacon,data));
    
}

function deployTimeBasedCollateralPoolContract() public {
    console.log(&quot;Deploying TimeBasedCollateralPool contract&quot;);
    address TimeBasedCollateralPool_contract_new  &#x3D;  address(new TimeBasedCollateralPool());
    vm.etch(TimeBasedCollateralPool_contract, address(TimeBasedCollateralPool_contract_new).code);
    
}


function initialize() public {
    console.log(&quot;Initialize TimeBasedCollateralPool contract&quot;);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).initialize(CollateralVault(CollateralVault_contract), 1, Owner, Owner, Owner, Owner, Owner); 
}

function setup_precondition() public {
    
    console.log(&quot;Setup Precondition for the test case&quot;);
    deployContract(); 
    if (contract_log &#x3D;&#x3D; 1 ) 
    {
        deployTimeBasedCollateralPoolContract();
    }
    
    initialize(); 
    console.log(&quot;Approve the TimeBasedCollateralPool_contract as the collateralizable contract&quot;);
    CollateralVault.CollateralizableContractApprovalConfig[] memory _collateralizableContractApprovals &#x3D; new CollateralVault.CollateralizableContractApprovalConfig[](1);
    _collateralizableContractApprovals[0] &#x3D; CollateralVault.CollateralizableContractApprovalConfig(TimeBasedCollateralPool_contract, true);
    vm.startPrank(CollateralVault_owner);
    CollateralVault(CollateralVault_contract).upsertCollateralizableContractApprovals(_collateralizableContractApprovals);
    vm.stopPrank(); 
    depositToCollateralVault(victim); 
    depositToCollateralVault(user2); 
    
    amount &#x3D; 100*10**6; 
    signature &#x3D; CreateSignature(VictimPrivateKey, amount, victim_nonce++);
   
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool. Stake amount: &quot;,amount);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank();       


    signature &#x3D; CreateSignature(PrivateKey_2,amount, user2_nonce++);
    vm.startPrank(user2);
    console.log(&quot;user2 stake to TimeBasedCollateralPool, Stake amount: &quot;,amount);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
    

} 

// forge test --match-test testUnStake -vvvvv | format &gt; testUnStake_241103_0800.log &amp;&amp; cat test/testTimeBasedCollateralPool.sol &gt;&gt; testUnStake_241102_0800.log
function testUnStake() public {      
  
    console.log(&quot;Get getAccountTokenState&quot;);
    TimeBasedCollateralPool.AccountState memory accountTokenState &#x3D; TimeBasedCollateralPool(TimeBasedCollateralPool_contract).getAccountTokenState(victim, USDC); 
    console.log(&quot;accountTokenState: &quot;, victim, accountTokenState.totalUnits, accountTokenState.firstPendingUnstakeEpoch);
    uint256 unstake_amount &#x3D; accountTokenState.totalUnits; 
    console.log(&quot;Victim can unstake amount: &quot;,unstake_amount); 
    
    scenario &#x3D; 1; 
    console.log(&quot;Scenario: &quot;,scenario);
    resetPool(scenario); 

    accountTokenState &#x3D; TimeBasedCollateralPool(TimeBasedCollateralPool_contract).getAccountTokenState(victim, USDC); 
    console.log(&quot;accountTokenState before unstake victim: &quot;,victim);
    console.log(&quot;accountTokenState before unstake accountTokenState.totalUnits: &quot;,accountTokenState.totalUnits);
    console.log(&quot;accountTokenState before unstake accountTokenState.firstPendingUnstakeUnits: &quot;,accountTokenState.firstPendingUnstakeUnits);

    vm.startPrank(victim); 
    console.log(&quot;Step : Victim Unstake from TimeBasedCollateralPool&quot;);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).unstake(IERC20(USDC), unstake_amount);
    vm.stopPrank(); 
   
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(VictimPrivateKey, amount, victim_nonce++);
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool, stake amount: &quot;,amount);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
    
    
    vm.warp(1730605189); // Sun Nov 03 2024 03:39:49 GMT+0000
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(VictimPrivateKey, amount, victim_nonce);
   
    vm.startPrank(victim);
    console.log(&quot;Victim stake to TimeBasedCollateralPool&quot;);
    console.log(&quot;Victim transaction expected to revert&quot;); 
    vm.expectRevert();
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
    
    amount &#x3D; 50*10**6; 
    signature &#x3D; CreateSignature(PrivateKey_2, amount, user2_nonce++);
   
    vm.startPrank(user2);
    console.log(&quot;user2 stake to TimeBasedCollateralPool&quot;);
    console.log(&quot;user2 transaction expected to revert&quot;); 
    vm.expectRevert();
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).stake(IERC20(USDC), amount, signature); 
    vm.stopPrank(); 
   

} 

    function resetPool(uint256 _scenario) public {
  
    IERC20[] memory _tokens &#x3D; new IERC20[](1); 
    _tokens[0] &#x3D; IERC20(USDC); 
    
    console.log(&quot;Step: reset pool&quot;);
        
    vm.startPrank(Owner);
    TimeBasedCollateralPool(TimeBasedCollateralPool_contract).resetPool(_tokens);
    vm.stopPrank(); 
    console.log(&quot;Pool was reset&quot;);
  
}    

function depositToCollateralVault(address user ) public {
    vm.startPrank(user); 
    console.log(&quot;Balance of USDC before depositing: &quot;,IERC20(USDC).balanceOf(user));
    console.log(&quot;Approve the CollateralVault_contract to spend USDC to deposit USDC&quot;);
    IERC20(USDC).approve(CollateralVault_contract, 10000*10**6); 

    address[] memory _tokenAddresses &#x3D; new address[](1);
    _tokenAddresses[0] &#x3D; USDC;
    uint256[] memory _amounts &#x3D; new uint256[](1);
    _amounts[0] &#x3D; 10000*10**6; 
    console.log(&quot;Step: Deposit 10000 USDC to CollateralVault for the user&quot;);
    CollateralVault(CollateralVault_contract).depositToAccount(user, _tokenAddresses, _amounts);
    vm.stopPrank();

}

bytes32 public constant COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH &#x3D;
    keccak256(
        &quot;CollateralizableTokenAllowanceAdjustment(address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)&quot;
    );

bytes32 private constant TYPE_HASH &#x3D;
    keccak256(&quot;EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)&quot;);

function CreateSignature(uint256 PrivateKey, uint256 _allowanceAdjustment, uint256 nonce) public returns ( bytes memory _signature){
    
    address _collateralizableContractAddress &#x3D; TimeBasedCollateralPool_contract;         
    (, string memory name, string memory  version, , , ,   ) &#x3D; CollateralVault(CollateralVault_contract).eip712Domain(); 
    
    bytes32 _buildDomainSeparator &#x3D; keccak256(abi.encode(TYPE_HASH, keccak256(bytes(name)),  keccak256(bytes(version)), block.chainid, CollateralVault_contract));

    bytes32 structHash &#x3D; keccak256(
        abi.encode(
            COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH,
            _collateralizableContractAddress,
            USDC,
            _allowanceAdjustment,
            nonce
        )
    ); 

    bytes32 message &#x3D; MessageHashUtils.toTypedDataHash(_buildDomainSeparator, structHash); 

    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(PrivateKey, message); 

    _signature &#x3D; abi.encodePacked(r, s, v);

}
     
```

}

\`\`\`

Just download the zip file:

<https://drive.google.com/file/d/1eaO4\\_BvlBjyFhWe22rwnWAL2JzOKhE9m/view?usp=sharing>

The test code uses Foundry. Just Unzip and run the test case:

\`\`\`bash forge test --match-test testUnStake -vvvvv > testUnStake\_241103\_0850.log

\`\`\`
