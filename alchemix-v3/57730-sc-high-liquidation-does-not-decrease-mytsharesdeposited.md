# 57730 sc high liquidation does not decrease mytsharesdeposited

**Submitted on Oct 28th 2025 at 14:13:56 UTC by @MentemDeus28 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57730
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

During liquidation flows `forceRepay`, `liquidate`, and `_doLiquidation`, the protocol properly transfers the corresponding MYT tokens from the user’s position and adjusts their collateral and debt balances.

However, the accounting variable `mytSharesDeposited` which tracks the total amount of MYT shares a user has deposited into the system is not updated or reduced when a liquidation occurs.

This causes a state inconsistency between actual MYT balances and recorded accounting values. Even though the user’s MYT tokens are transferred out of their position to the transmuter, liquidator, mytSharesDeposited remains unchanged.

## Vulnerability Details

```solidity
/// @dev Total yield tokens deposited
    /// This is used to differentiate between tokens deposited into a CDP and balance of the contract
    uint256 private _mytSharesDeposited; //private


```

The `forceRepay`, `liquidate`, and `_doLiquidation` functions modify collateral and debt state but never decrement the corresponding user’s mytSharesDeposited or any equivalent share tracking variable after liquidating MYT tokens. This omission allows mytSharesDeposited to remain inflated despite the actual shares being removed from the system.

## Impact Details

this can lead to several potential issues:

Accounting drift: system assumes more collateralized MYT shares than actually exist.

Potential mis-calculations.

Misleading TVL.

## Recommended mitigation

update `mytSharesDeposited` aftera successful liquidation or forcerepay.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852>

## Proof of Concept

## Proof of Concept

**Consider changing the \_mytSharesDeposited variable to public for testing**

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

interface IFeeVault {
    function token() external view returns (address);
    function totalDeposits() external view returns (uint256);
    function withdraw(address to, uint256 amount) external;
}



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
contract MockFeeVault is IFeeVault {
    address public token;
    uint256 public deposits = 0;

    constructor(address _token) {
        token = _token;
    }

    function totalDeposits() external view returns (uint256) {
        return deposits;
    }

    function withdraw(address to, uint256 amount) external {
        deposits -= amount;
        IERC20(token).transfer(to, amount);
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

         MockFeeVault mockFeeVault = new MockFeeVault(address(underlyingToken));
        alchemist.setAlchemistFeeVault(address(mockFeeVault));
 
        
        // Fund tokens
        mytVault.mint(ALICE, 1e50);
        alToken.mint(ALICE, 1e50);
    
        //approve tokens
        vm.startPrank(ALICE);
        mytVault.approve(address(alchemist), type(uint256).max);
        
        alToken.approve(address(alchemist), type(uint256).max);
        vm.stopPrank();

    }

   
   function test_liquidation_does_not_deduct_mytSharesDeposited() public {
    uint256 tokenId = 1;

    // 1. Deposit and mint to open a CDP
    vm.startPrank(ALICE);
    alchemist.deposit(1e18, ALICE, 0);

    alchemist.mint(tokenId, 0.5e18, ALICE);
    vm.stopPrank();

    
    // 3. Simulate undercollateralization (drop rate)
    mytVault.setConversionRate(0.3e18);

    uint256 sharesBefore = mytVault.balanceOf(address(alchemist));
    uint256 depositedBefore = alchemist._mytSharesDeposited();


    console2.log("Before Liquidation:");
    emit log_named_uint("MYT Balance (shares) before", sharesBefore);
    emit log_named_uint("_mytSharesDeposited before", depositedBefore);
    
    // 4. Liquidate
    vm.prank(LIQUIDATOR);
    alchemist.liquidate(tokenId);

    // 5. Verify accounting drift
    // Check Accounting
    uint256 sharesAfter = mytVault.balanceOf(address(alchemist));
    uint256 depositedAfter = alchemist._mytSharesDeposited();

    console2.log("After Liquidation:");
    emit log_named_uint("MYT Balance (shares) after", sharesAfter);
    emit log_named_uint("_mytSharesDeposited after", depositedAfter);
    
    assertEq(depositedAfter, depositedBefore, "BUG: _mytSharesDeposited not decremented!");
}

}

```

**Expected Output** Ran 1 test for src/test/MyPoc.t.sol:AlchemistV3POCGROUND \[PASS] test\_liquidation\_does\_not\_deduct\_mytSharesDeposited() (gas: 554686)

Logs: Before Liquidation:

MYT Balance (shares) before: 1000000000000000000

\_mytSharesDeposited before: 1000000000000000000

After Liquidation:

MYT Balance (shares) after: 0

\_mytSharesDeposited after: 1000000000000000000
