# 58606 sc high missing collateral accounting in liquidation leads to inflated bad debt calculations

## #58606 \[SC-High] Missing collateral accounting in liquidation leads to inflated bad debt calculations

**Submitted on Nov 3rd 2025 at 14:13:42 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58606
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

### Description

##

### Brief/Intro

During liquidations in `AlchemistV3::liquidate()`, yield tokens transferred to the transmuter and liquidator fees are not properly accounted for in the `_mytSharesDeposited` tracking variable. This creates a discrepancy between the actual and reported total underlying value, leading to incorrect bad debt ratio calculations that affect the transmuter's loss distribution mechanism.

### Vulnerability Details

The root cause lies in the inconsistent accounting of collateral balance changes in the `AlchemistV3::liquidate()` function. When examining other functions that modify user collateral balances, such as `repay()` and `burn()`, we observe proper accounting:

**In `AlchemistV3::burn()`:**

```solidity
// Debt is subject to protocol fee similar to redemptions
_accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;
TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
_mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;
```

**In `AlchemistV3::repay()`:**

```solidity
account.collateralBalance -= creditToYield * protocolFee / BPS;
// Transfer the repaid tokens to the transmuter.
TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
_mytSharesDeposited -= creditToYield * protocolFee / BPS;
```

However, in the `liquidate()` and `batchLiquidate()` functions, there is no corresponding adjustment to `_mytSharesDeposited`.

The `_mytSharesDeposited` variable is used in `_getTotalUnderlyingValue()` to calculate the system's total collateral value:

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

This inflated total collateral value is then used by the transmuter to calculate the bad debt ratio in `Transmuter::claimRedemption()`:

```solidity
uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
```

### Impact Details

The `_getTotalUnderlyingValue()` function returns values higher than the actual collateral remaining in the system. The transmuter calculates `badDebtRatio = totalSyntheticsIssued / totalUnderlyingValue`, where an inflated denominator results in artificially low bad debt ratios

When the system experiences bad debt (ratio > 1e18), the transmuter should scale down redemptions proportionally. However, due to underreported bad debt ratios, this mechanism fails to trigger. Early redeemers receive full payouts while later redeemers bear disproportionate losses when the alchemist reserves are depleted or will be unable to claim their redemptions due to lack of funds

### References

AlchemistV3::repay(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L541>

AlchemistV3::burn(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L485>

Transmuter::claimRedemption(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L217-L226>

### Proof of Concept

### Proof of Concept

```solidity
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

contract PoC_LiquidationDoesNotUpdateMytSharesDeposited is Test {
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

    function _magicDepositToVault(address vaultAddr, address depositor, uint256 amount) internal returns (uint256) {
        deal(address(mockVaultCollateral), address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(address(mockVaultCollateral), vaultAddr, amount);
        uint256 shares = IVaultV2(vaultAddr).deposit(amount, depositor);
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

    function test_PoC_LiquidationDoesNotUpdateMytSharesDeposited() external {
        console.log("\n=== PoC: Liquidation Does Not Update _mytSharesDeposited ===\n");

        // Setup: Mint whale supply
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // Setup: Create healthy position to maintain global collateralization
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Setup: Create position that will be liquidated
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // Capture state BEFORE liquidation
        uint256 totalUnderlyingValueBefore = alchemist.getTotalUnderlyingValue();
        uint256 totalDebtBefore = alchemist.totalDebt();
        uint256 totalSyntheticsIssuedBefore = alchemist.totalSyntheticsIssued();
        uint256 mytBalanceInAlchemistBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("--- STATE BEFORE LIQUIDATION ---");
        console.log("Total Underlying Value (from getTotalUnderlyingValue()):", totalUnderlyingValueBefore);
        console.log("MYT balance in Alchemist contract:", mytBalanceInAlchemistBefore);
        console.log("Total Debt:", totalDebtBefore);
        console.log("Total Synthetics Issued:", totalSyntheticsIssuedBefore);

        // Calculate bad debt ratio BEFORE liquidation
        uint256 yieldTokenBalanceInTransmuterBefore = TokenUtils.safeBalanceOf(alchemist.myt(), address(transmuterLogic));
        uint256 denominatorBefore = totalUnderlyingValueBefore + alchemist.convertYieldTokensToUnderlying(yieldTokenBalanceInTransmuterBefore);
        if (denominatorBefore == 0) denominatorBefore = 1;
        uint256 badDebtRatioBefore = totalSyntheticsIssuedBefore * 1e18 / denominatorBefore;
        console.log("Bad Debt Ratio BEFORE:", badDebtRatioBefore);
        console.log("(Ratio > 1e18 means bad debt)\n");

        // Make position undercollateralized by manipulating yield token price
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Increase yield token supply by 5.9% while keeping underlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        
        console.log("--- Price drops ---");
        totalUnderlyingValueBefore = alchemist.getTotalUnderlyingValue();
        console.log("Total Underlying Value (from getTotalUnderlyingValue()):", totalUnderlyingValueBefore);

        (uint256 collateralBefore, uint256 debtBefore,) = alchemist.getCDP(tokenIdFor0xBeef);
        console.log("--- POSITION TO BE LIQUIDATED ---");
        console.log("Collateral (in MYT):", collateralBefore);
        console.log("Debt:", debtBefore);
        console.log("Collateral Value (in debt tokens):", alchemist.convertYieldTokensToDebt(collateralBefore));
        console.log("Collateralization Ratio:", alchemist.convertYieldTokensToDebt(collateralBefore) * FIXED_POINT_SCALAR / debtBefore);
        console.log("(Should be below lower bound of ~1.05e18)\n");

        // Execute liquidation
        vm.startPrank(externalUser);
        (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        console.log("--- LIQUIDATION EXECUTED ---");
        console.log("Amount Liquidated (MYT sent to transmuter):", amountLiquidated);
        console.log("Fee in Yield (MYT sent to liquidator):", feeInYield);
        console.log("Fee in Underlying:", feeInUnderlying);

        uint256 totalMytTransferredOut = (amountLiquidated - feeInYield) + feeInYield; // transmuter + liquidator
        console.log("Total MYT transferred out of Alchemist:", totalMytTransferredOut, "\n");

        // Capture state AFTER liquidation
        uint256 totalUnderlyingValueAfter = alchemist.getTotalUnderlyingValue();
        uint256 totalDebtAfter = alchemist.totalDebt();
        uint256 totalSyntheticsIssuedAfter = alchemist.totalSyntheticsIssued();
        uint256 mytBalanceInAlchemistAfter = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("--- STATE AFTER LIQUIDATION ---");
        console.log("Total Underlying Value (from getTotalUnderlyingValue()):", totalUnderlyingValueAfter);
        console.log("MYT balance in Alchemist contract:", mytBalanceInAlchemistAfter);
        console.log("Total Debt:", totalDebtAfter);
        console.log("Total Synthetics Issued:", totalSyntheticsIssuedAfter);

        // THE BUG: getTotalUnderlyingValue should have decreased, but it hasn't!
        console.log("\n");
        console.log("Expected Total Underlying Value After Liquidation:");
        console.log("  Should be: ~", totalUnderlyingValueBefore - alchemist.convertYieldTokensToUnderlying(totalMytTransferredOut));
        console.log("  Actually is:", totalUnderlyingValueAfter);
        console.log(
            "  Difference (inflated by):",
            totalUnderlyingValueAfter - (totalUnderlyingValueBefore - alchemist.convertYieldTokensToUnderlying(totalMytTransferredOut))
        );

        // The actual MYT balance decreased correctly
        console.log("\nMYT Balance Check:");
        console.log("  MYT Balance Before:", mytBalanceInAlchemistBefore);
        console.log("  MYT Balance After:", mytBalanceInAlchemistAfter);
        console.log("  Actual Decrease:", mytBalanceInAlchemistBefore - mytBalanceInAlchemistAfter);
        console.log("  Expected Decrease:", totalMytTransferredOut);

        // But getTotalUnderlyingValue() didn't change!
        console.log("\ngetTotalUnderlyingValue() Check:");
        console.log("  Before:", totalUnderlyingValueBefore);
        console.log("  After:", totalUnderlyingValueAfter);
        console.log(
            "  Change:",
            totalUnderlyingValueAfter > totalUnderlyingValueBefore
                ? int256(totalUnderlyingValueAfter - totalUnderlyingValueBefore)
                : -int256(totalUnderlyingValueBefore - totalUnderlyingValueAfter)
        );
        console.log("  Expected Change: ~", -int256(alchemist.convertYieldTokensToUnderlying(totalMytTransferredOut)));

        // Calculate bad debt ratio AFTER liquidation (using the INCORRECT inflated value)
        uint256 yieldTokenBalanceInTransmuterAfter = TokenUtils.safeBalanceOf(alchemist.myt(), address(transmuterLogic));
        uint256 denominatorAfter = totalUnderlyingValueAfter + alchemist.convertYieldTokensToUnderlying(yieldTokenBalanceInTransmuterAfter);
        if (denominatorAfter == 0) denominatorAfter = 1;
        uint256 badDebtRatioAfterIncorrect = totalSyntheticsIssuedAfter * 1e18 / denominatorAfter;

        // Calculate what the bad debt ratio SHOULD be (using correct value)
        uint256 correctTotalUnderlyingValue = totalUnderlyingValueBefore - alchemist.convertYieldTokensToUnderlying(totalMytTransferredOut);
        uint256 denominatorCorrect = correctTotalUnderlyingValue + alchemist.convertYieldTokensToUnderlying(yieldTokenBalanceInTransmuterAfter);
        if (denominatorCorrect == 0) denominatorCorrect = 1;
        uint256 badDebtRatioAfterCorrect = totalSyntheticsIssuedAfter * 1e18 / denominatorCorrect;

        console.log("\n--- BAD DEBT RATIO CALCULATION ---");
        console.log("Bad Debt Ratio AFTER (using INCORRECT inflated value):", badDebtRatioAfterIncorrect);
        console.log("Bad Debt Ratio AFTER (using CORRECT value):", badDebtRatioAfterCorrect);
        console.log("Difference:", badDebtRatioAfterCorrect > badDebtRatioAfterIncorrect ? badDebtRatioAfterCorrect - badDebtRatioAfterIncorrect : 0);
       
        // Assertions to prove the bug
        // 1. MYT balance decreased correctly
        assertApproxEqAbs(
            mytBalanceInAlchemistBefore - mytBalanceInAlchemistAfter, totalMytTransferredOut, 1e18, "MYT balance should have decreased by liquidated amount"
        );

        // 2. But getTotalUnderlyingValue() did NOT decrease (BUG!)
        // It should have decreased, but it stays the same because _mytSharesDeposited wasn't updated
        assertEq(totalUnderlyingValueAfter, totalUnderlyingValueBefore, "BUG: getTotalUnderlyingValue() should have decreased but didn't!");

        // 3. This causes incorrect bad debt ratio calculation
        assertTrue(badDebtRatioAfterIncorrect < badDebtRatioAfterCorrect, "BUG: Bad debt ratio is underestimated due to inflated getTotalUnderlyingValue()");
    }
}
```
