# 58163 sc critical total loss of user funds in claim redemption&#x20;

**Submitted on Oct 31st 2025 at 02:37:26 UTC by @MentemDeus28 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58163
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

when all earmarked debt is redeemed, cumulativeEarmarked = 0.The `_earmark` function in `AlchemistV3` cancels valid transmutation signals from the transmuter when `queryGraph` returns a value less than or equal to coverInDebt. This causes cumulativeEarmarked to remain zero even when users have legitimately transmuted synthetic debt into redeemable yield. As a result, the `redeem` function is called with amount = 0, and `claimRedemption` pays only the transmuter’s local balance (often dust). The user’s position is then deleted, resulting in permanent loss of 99%+ of expected redemption value, a total loss of funds.

```solidity
uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
amount = amount > coverInDebt ? amount - coverInDebt : 0;  
```

## Vulnerability Details

The Transmuter `claimRedemption` function can finalize a user’s position and permanently delete it even when no yield has been earmarked from the Alchemist, resulting in no yield token payout (MYT) and a loss of future yield entitlement.

```solidity
// If the contract has a balance of yield tokens from alchemist repayments then we only need to redeem partial or none from Alchemist earmarked
        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;

        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

        // Cap to what we actually hold now (handles redeem() rounding shortfalls).
        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem; //this is the main issue 


```

However, under certain protocol states, `cumulativeEarmarked` can remain zero, especially periods when `earmark` conditions are not met. This leads to an empty `redeem` execution and a dust or zero distributable value.

```solidity
 function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;

```

However, the system still executes:

```solidity
TokenUtils.safeBurn(syntheticToken, amountTransmuted);
alchemist.reduceSyntheticsIssued(amountTransmuted);
delete _positions[id];



```

which erases the user’s redemption position and burns the synthetic tokens, even though they were never truly converted into yield. Position is deleted , irreversible.

## Impact Details

User receives little or no yield (MYT) even though some portion of their position or full position is considered transmuted.

## ## Recommended Mitigation

consider making these changes in `claimRedeemption` function

```diff

- uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;


+uint256 distributable = totalYield;
+if (distributable > balAfterRedeem) {
+    revert InsufficientTransmuterBalance();
}



```

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L191>

## Proof of Concept

## Proof of Concept

```solidity

// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import "forge-std/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {MockERC20} from "../test/mocks/MockERC20.sol"; 
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
    uint256 public deposits;

    constructor(address _token) {
        token = _token;
    }

    function totalDeposits() external view returns (uint256) {
        return deposits;
    }

    function deposit(uint256 amount) external {
        deposits += amount;
        IERC20(token).transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(address to, uint256 amount) external {
        require(deposits >= amount, "Insufficient deposits");
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
        mytVault.mint(address(alchemist), 1e50);
        
        underlyingToken.mint(address(alchemist), 1e50);
        //approve tokens
        vm.startPrank(ALICE);
        mytVault.approve(address(alchemist), type(uint256).max);
        
        alToken.approve(address(transmuter), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(address(alchemist));
        underlyingToken.approve(address(alchemist), type(uint256).max);

        underlyingToken.approve(alchemist.alchemistFeeVault(), type(uint256).max);
        vm.stopPrank();

    }


    function test_redeem_DoS_when_no_earmark() public {
    uint256 tokenId = 1;

    // user deposits + mints
    vm.startPrank(ALICE);
    alchemist.deposit(100e18, ALICE, 0);


    alchemist.mint(tokenId, 50e18, ALICE);
    vm.stopPrank();

    //. Trigger first earmark (1e18)
    vm.roll(block.number + 100);

     mytVault.mint(address(transmuter), 1e18);

    // . Mock queryGraph to return value
  vm.mockCall(
    address(transmuter),
    abi.encodeWithSelector(ITransmuter.queryGraph.selector),
    abi.encode(1e18)
);

  // Call a function that internally triggers _earmark (like another mint)
    vm.startPrank(ALICE);
    alchemist.mint(tokenId, 1e18, ALICE);
    vm.stopPrank();

     //   Confirm cumulativeEarmarked is still zero
    //after earmark cumulativeEarmarked is still zero
   uint256 cumulativeEarmarked = alchemist.cumulativeEarmarked();
   emit log_named_uint("cumulativeEarmarked:", cumulativeEarmarked);

    
    vm.startPrank(address(this));
    transmuter.setDepositCap(1e50); // increase deposit cap to allow redemption

    transmuter.setAlchemist(address(alchemist));
    vm.stopPrank();

    vm.prank(ALICE);
    alToken.approve(address(transmuter), type(uint256).max);

    //  User creates redemption in transmuter using his synthetic token
    vm.startPrank(ALICE);
    uint256 positionId = 1;
    transmuter.createRedemption(50e18);
    vm.stopPrank();

    //  Advance to maturity
    vm.roll(block.number + 1000);

    

    //  Claim should get MYT
    uint256 mytBefore = mytVault.balanceOf(ALICE);

    vm.prank(ALICE);
    transmuter.claimRedemption(positionId);

    uint256 mytAfter = mytVault.balanceOf(ALICE);

    //after full transmutation user should get 50e18 MYT
    uint256 Expected = 50e18;
     
    emit log_named_uint("Expected:", Expected);
    emit log_named_uint("MYT Received:", mytAfter - mytBefore);
    assertGt(mytAfter - mytBefore, 0, "User got nothing DoS");
}
}


```

**Expected Output**

Ran 1 test for src/test/MyPoc.t.sol:AlchemistV3POCGROUND \[PASS] test\_redeem\_DoS\_when\_no\_earmark() (gas: 1879526) Logs: cumulativeEarmarked:: 0

Expected:: 50000000000000000000

MYT Received:: 1990000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 63.48ms (21.57ms CPU time)

**User staked 50e18 and after a successful transmutation he received 1.99e18 Losing 99% of his funds**
