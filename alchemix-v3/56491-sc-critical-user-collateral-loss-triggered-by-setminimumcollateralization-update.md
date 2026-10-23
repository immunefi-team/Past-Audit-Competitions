# 56491 sc critical user collateral loss triggered by setminimumcollateralization update

**Submitted on Oct 16th 2025 at 19:07:29 UTC by @magtentic for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56491
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In `AlchemistV3`, an admin can update `minimumCollateralization` via `setMinimumCollateralization` as part of their normal operations.\
If a user deposits and borrows before this update and leaves their position open, normal protocol operations (withdrawals, redemptions) after the update can cause unexpected reduction of their collateral, resulting in a permanent loss.\
This occurs because `_collateralWeight` is updated globally, but the user’s account does not automatically sync with the new weight, leading to over-removal of collateral during withdrawal/redemption calculations.

From tests, the calculations mostly behaves as most functions use `minimumCollateralization` as a percentage :

> `minimumCollateralization / FIXED_POINT_SCALAR`\
> `FIXED_POINT_SCALAR / minimumCollateralization`

## Vulnerability Details

As a user does deposits before a `minimumCollateralization` and then initiates a borrow without withdrawing their funds before any further interactions happen from other users results in their funds been safe and they are able to withdraw the correct collateral.\
When they leave the debt and let normal operations occur on the contract, this can lead to a scenario where the user then loses their collateral which they wouldn't have lost had they withdrawn before other interactions.\
The reason for this is that normal interactions affect `_collateralWeight` which is not stored on the user's account level as the account had no syncing occuring since prior to the update happening.

## Impact Details

* Permanent loss of user funds.
* No action is required from an attacker as this happens in normal protocol usage.
* Could affect multiple users over time.
* Admin action triggers it indirectly, so it’s a critical design flaw.

## References

[Alchemist V3](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol)

## Proof of Concept

## Proof of Concept

For this test, I did add the below lines of code to display the specific values which result in the loss.

```
pragma solidity 0.8.28;

+import {console} from "../lib/forge-std/src/console.sol";

    function _calculateUnrealizedDebt(uint256 tokenId)
...
        // Collateral from fees and redemptions
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
+        console.log("Account raw locked : ", account.rawLocked);
+        console.log("_collateralWeight : ", _collateralWeight);
+        console.log("Account last collateral weight : ",account.lastCollateralWeight);
+        console.log("Collateral to remove : ", collateralToRemove);
        uint256 newCollateral = account.collateralBalance - collateralToRemove;

        return (newDebt, newEarmarked, newCollateral);
    }
```

Code run to show the loss of funds from the user's collateral.

```
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";

contract AlchemistV3Test is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public constant BPS = 10_000;

    uint256 public protocolFee = 100;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds;

    // large amount to test with
    uint256 whaleSupply;

    // amount of yield/underlying token to deposit
    uint256 depositAmount;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    address public protocolFeeReceiver = address(10);

    // MYT variables
    VaultV2 vault;
    MockAlchemistAllocator allocator;
    MockMYTStrategy mytStrategy;
    address public operator = address(0x2222222222222222222222222222222222222222); // default operator
    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX
    address public curator = address(0x8888888888888888888888888888888888888888);
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    uint256 public defaultStrategyAbsoluteCap = 2_000_000_000e18;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%

    struct CalculateLiquidationResult {
        uint256 liquidationAmountInYield;
        uint256 debtToBurn;
        uint256 outSourcedFee;
        uint256 baseFeeInYield;
    }

    struct AccountPosition {
        address user;
        uint256 collateral;
        uint256 debt;
        uint256 tokenId;
    }

    function setUp() external {
        adJustTestFunds(18);
        setUpMYT(18);
        deployCoreContracts(18);
    }

    function adJustTestFunds(uint256 alchemistUnderlyingTokenDecimals) public {
        accountFunds = 200_000 * 10 ** alchemistUnderlyingTokenDecimals;
        whaleSupply = 20_000_000_000 * 10 ** alchemistUnderlyingTokenDecimals;
        depositAmount = 200_000 * 10 ** alchemistUnderlyingTokenDecimals;
    }

    function setUpMYT(uint256 alchemistUnderlyingTokenDecimals) public {
        vm.startPrank(admin);
        uint256 TOKEN_AMOUNT = 1_000_000; // Base token amount
        uint256 initialSupply = TOKEN_AMOUNT * 10 ** alchemistUnderlyingTokenDecimals;
        mockVaultCollateral = address(new TestERC20(initialSupply, uint8(alchemistUnderlyingTokenDecimals)));
        mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(idData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(idData, defaultStrategyRelativeCap);
        vm.stopPrank();
    }

    function _magicDepositToVault(address vault, address depositor, uint256 amount) internal returns (uint256) {
        deal(address(mockVaultCollateral), address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(address(mockVaultCollateral), vault, amount);
        uint256 shares = IVaultV2(vault).deposit(amount, depositor);
        vm.stopPrank();
        return shares;
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }

    function deployCoreContracts(uint256 alchemistUnderlyingTokenDecimals) public {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(vault.asset()),
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: liquidatorFeeBPS,
            repaymentFee: 100,
            myt: address(vault)
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(vault.asset()), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        _magicDepositToVault(address(vault), address(0xbeef), accountFunds);
        _magicDepositToVault(address(vault), address(0xdad), accountFunds);
        _magicDepositToVault(address(vault), externalUser, accountFunds);
        _magicDepositToVault(address(vault), yetAnotherExternalUser, accountFunds);
        _magicDepositToVault(address(vault), anotherExternalUser, accountFunds);
        vm.stopPrank();

        vm.startPrank(address(admin));
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();

        deal(address(alToken), address(0xdad), accountFunds);
        deal(address(alToken), address(anotherExternalUser), accountFunds);
        deal(address(vault.asset()), address(0xbeef), accountFunds);
        deal(address(vault.asset()), externalUser, accountFunds);
        deal(address(vault.asset()), yetAnotherExternalUser, accountFunds);
        deal(address(vault.asset()), anotherExternalUser, accountFunds);
        deal(address(vault.asset()), alchemist.alchemistFeeVault(), 10_000 * (10 ** alchemistUnderlyingTokenDecimals));

        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault.asset()), address(vault), accountFunds);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault.asset()), address(vault), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(vault), someWhale, whaleSupply);
        deal(address(vault.asset()), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(vault.asset()), address(mockStrategyYieldToken), whaleSupply);
        vm.stopPrank();
    }

    function testUpdatedMinCollateralization() public {
        /*Mo write up
            1) Set up contract on normal minimumCollateralization
            2) First user deposits funds to contract
            3) First user borrows half their deposit amount
            4) Second user does a deposit
            5) Second user borrows half their deposit amount increasing _totalLocked 
            6) Admin updates minimumCollateralization
            7) Second user pays back their loan at this point
            8) External user triggers a redemption
            9) External user pays claims their redemption after sufficient blocks have passed
            10) External user triggers another redemption
            11) External user pays claims their redemption after sufficient blocks have passed
            12) User one collateral is now below their expected value and below earmarked value
        */
        uint256 updatedMinimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 8e17;
        console.log(updatedMinimumCollateralization);
        console.log(minimumCollateralization);
        uint256 amount = 100e18;

        console.log("Start balance of 0xbeef ", vault.balanceOf(address(0xbeef)));
        // User one does deposit
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));
        vm.stopPrank();

        //User Two does a deposit
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xdad), 0);
        uint256 tokenIdTwo = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        //User two borrows on their deposit
        alchemist.mint(tokenIdTwo, amount / 2, address(0xdad));
        vm.stopPrank();

        //Admin increases minimumCollateralization
        vm.prank(address(alOwner)); 
        alchemist.setMinimumCollateralization(updatedMinimumCollateralization);

        //User two pays their loan
        vm.roll(block.number + 1);
        vm.prank(address(0xdad));
        alchemist.repay(100e18, tokenIdTwo);

        //User two withdraws
        vm.roll(block.number + 1);
        vm.prank(address(0xdad));
        alchemist.withdraw(amount / 2, address(0xdad), tokenIdTwo);

        vm.startPrank(address(anotherExternalUser));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        (uint256 collateral, uint256 userDebt,) = alchemist.getCDP(tokenId);
        console.log(collateral);

        assertEq(userDebt, (amount / 2));
        assertApproxEqAbs(collateral, amount, 0);

        uint256 maxBorrowZero = alchemist.getMaxBorrowable(tokenId);
        console.log("maxBorrowZero 0: ", maxBorrowZero);

        (uint256 collateralZero, uint256 userDebtZero, uint256 earmarkedZero) = alchemist.getCDP(tokenId);       
        console.log("User Debt Zero : ", userDebtZero);
        console.log("User Collateral Zero : ", collateralZero); 
        console.log("Earmarked Zero : " , earmarkedZero);

        console.log("Balance of 0xbeef before claim : ", vault.balanceOf(address(0xbeef)));

        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        (uint256 collateralOne, uint256 userDebtOne, uint256 earmarkedOne) = alchemist.getCDP(tokenId);       
        console.log("User Debt One : ", userDebtOne);
        console.log("User Collateral One : ", collateralOne); 
        console.log("Earmarked One : " , earmarkedOne);

        vm.startPrank(address(anotherExternalUser));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(2);
        vm.stopPrank();
        
        //Validate to see that the user is paid up
        (uint256 collateralTwo, uint256 userDebtTwo, uint256 earmarkedTwo) = alchemist.getCDP(tokenId);       
        console.log("User Debt Two : ", userDebtTwo);
        console.log("User Collateral Two : ", collateralTwo); 
        console.log("Earmarked Two : " , earmarkedTwo);

        //User One now withdraws their balance
        vm.roll(block.number + 1);
        vm.prank(address(0xbeef));
        alchemist.withdraw(collateralTwo, address(0xbeef), tokenId);

        console.log("End balance of 0xbeef ", vault.balanceOf(address(0xbeef)));
    }

    function testWithoutUpdatingMinCollateralization() public {
        /*Mo write up
            1) Set up contract on normal minimumCollateralization
            2) First user deposits funds to contract
            3) First user borrows half their deposit amount
            4) Second user does a deposit
            5) Second user borrows half their deposit amount increasing _totalLocked 
            6) Second user pays back their loan at this point
            7) External user triggers a redemption
            8) External user pays claims their redemption after sufficient blocks have passed
            9) External user triggers another redemption
            10) External user pays claims their redemption after sufficient blocks have passed
            11) User one collateral is now below their expected value and below earmarked value
        */
        uint256 updatedMinimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 8e17;
        console.log(updatedMinimumCollateralization);
        console.log(minimumCollateralization);
        uint256 amount = 100e18;

        console.log("Start balance of 0xbeef ", vault.balanceOf(address(0xbeef)));
        // User one does deposit
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));
        vm.stopPrank();

        //User Two does a deposit
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xdad), 0);
        uint256 tokenIdTwo = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        //User two borrows on their deposit
        alchemist.mint(tokenIdTwo, amount / 2, address(0xdad));
        vm.stopPrank();

        //User two pays their loan
        vm.roll(block.number + 1);
        vm.prank(address(0xdad));
        alchemist.repay(100e18, tokenIdTwo);

        //User two withdraws
        vm.roll(block.number + 1);
        vm.prank(address(0xdad));
        alchemist.withdraw(amount / 2, address(0xdad), tokenIdTwo);

        vm.startPrank(address(anotherExternalUser));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        (uint256 collateral, uint256 userDebt,) = alchemist.getCDP(tokenId);
        console.log(collateral);

        assertEq(userDebt, (amount / 2));
        assertApproxEqAbs(collateral, amount, 0);

        uint256 maxBorrowZero = alchemist.getMaxBorrowable(tokenId);
        console.log("maxBorrowZero 0: ", maxBorrowZero);

        (uint256 collateralZero, uint256 userDebtZero, uint256 earmarkedZero) = alchemist.getCDP(tokenId);       
        console.log("User Debt Zero : ", userDebtZero);
        console.log("User Collateral Zero : ", collateralZero); 
        console.log("Earmarked Zero : " , earmarkedZero);

        console.log("Balance of 0xbeef before claim : ", vault.balanceOf(address(0xbeef)));

        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        (uint256 collateralOne, uint256 userDebtOne, uint256 earmarkedOne) = alchemist.getCDP(tokenId);       
        console.log("User Debt One : ", userDebtOne);
        console.log("User Collateral One : ", collateralOne); 
        console.log("Earmarked One : " , earmarkedOne);

        vm.startPrank(address(anotherExternalUser));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(2);
        vm.stopPrank();
        
        //Validate to see that the user is paid up
        (uint256 collateralTwo, uint256 userDebtTwo, uint256 earmarkedTwo) = alchemist.getCDP(tokenId);       
        console.log("User Debt Two : ", userDebtTwo);
        console.log("User Collateral Two : ", collateralTwo); 
        console.log("Earmarked Two : " , earmarkedTwo);

        //User One now withdraws their balance
        vm.roll(block.number + 1);
        vm.prank(address(0xbeef));
        alchemist.withdraw(collateralTwo, address(0xbeef), tokenId);

        console.log("End balance of 0xbeef ", vault.balanceOf(address(0xbeef)));
    }
}
```
