# 56727 sc high underlying increase in forced repayments leads to insolvency

**Submitted on Oct 19th 2025 at 22:35:18 UTC by @madman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56727
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Protocol insolvency

## Description

## Brief/Intro

In `AlchemistV3::_liquidate`, undercollateralized accounts forcibly pay their debts before being liquidated. However, both the protocol fees and the liquidated amount transferred to the Transmuter are not deducted from `mySharesDeposited`, leading to an increase in perceived total underlying value. This increase causes the bad debt downscaling mechanism to not be triggered, leading to insolvency.

## Vulnerability Details

Forced payments during liquidations fail to deduct liquidated amounts from `mySharesDeposited`, as shown below in **Snippet 1**.

#### Snippet 1

```
        // _AlchemistV3::forceRepay...
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }


```

In the Transmuter, when claiming redemptions, we have:

#### Snippet 2

```
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
```

The bad debt ratio is skewed downwards, because it considers both the Transmuter's balance (which increased) as well as the Alchemist's total underlying value (unchanged). That is, the denominator goes up and `badDebtRatio` goes down.

#### Snippet 3

```
        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

A further decrease in MYT value will only offset this underestimation, e.g bringing the badDebtRatio from 0.5 to 0.7. As such, users can freely claim transmuted amounts with no downscaling.

## Impact Details

The bad debt handler mechanism fails to act after forced repayments occur because of incorrect internal accounting. Ultimately, it results in protocol insolvency: more MYT tokens will be up for claiming than the amount available.

## Proof of Concept

## Proof of Concept

We will craft the following scenario:

1. Users deposit and max mint.
2. Users create redemptions in the Transmuter.
3. MYT price drops slightly, triggering force repays only.
4. The perceived underlying value goes up, decreasing bad debt ratio.
5. MYT price drops more, increasing the bad debt ratio, but not enough for scaling to take place.
6. First users claim full amounts, others are left with nothing.

We include the PoC below in Foundry, our test is the very last function: `testForceRepay_Protocol_Insolvent()`.

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

contract AlchemistV3AuditorTest is Test {
    address[] users2;
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

        _depositToVaultAndAllocate(100);
    }

    function _depositToVaultAndAllocate(uint256 users) internal{
        uint256 n = users;
        users2 = _generateUsers(n);

        for(uint i=0; i < users2.length; i++){
            _magicDepositToVault(address(vault), users2[i], accountFunds);

        }

        vm.startPrank(address(admin));
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();
    }

    /// @notice Increases the MYT supply by `bps`.
    function _increaseYieldTokenSupply(uint256 bps) internal{
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * bps / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        console.log("Increased MYT supply from %s to %s", initialVaultSupply, modifiedVaultSupply);
    }

    /// @notice Gets protocol underlying value as calculated by Transmuter::claimRedemption.
    function _getProtocolUnderlyingValue() internal view returns(uint256 totalUnderlyingValue) {
        uint256 transmuterYieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(transmuterLogic));
        totalUnderlyingValue = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(transmuterYieldTokenBalance);
    }

    /// @notice Deposits `depositAmount` as `user`.
    function _depositAsUser(address user, string memory userString) internal returns (uint256 tokenId){
        vm.startPrank(user);

        SafeERC20.safeApprove(address(vault), address(alchemist), TokenUtils.safeBalanceOf(address(vault), user));
        alchemist.deposit(depositAmount, user, 0);

        vm.stopPrank();
        //console.log("%s deposited %s.", userString, depositAmount);
        tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
    }

    /// @notice Deposits `depositAmount` as `user`.
    function _depositAmountAsUser(address user, uint256 amount, string memory userString) internal returns (uint256 tokenId){
        vm.startPrank(user);

        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, user, 0);

        vm.stopPrank();
        //console.log("%s deposited %s.", userString, depositAmount);
        tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
    }

    /// @notice Max mints as `user`.
    function _maxMintAsUser(address user, string memory userString, uint256 tokenId) internal{
        vm.startPrank(user);

        alchemist.mint(tokenId, alchemist.getMaxBorrowable(tokenId), address(user));
        uint256 debtTokensFromBalanceOf = alToken.balanceOf(address(user));
        //console.log("%s maxminted %s alTokens.", userString, debtTokensFromBalanceOf);
        (, uint256 debtTokensFromCDP,) = alchemist.getCDP(tokenId);

        vm.stopPrank();
        assertEq(debtTokensFromBalanceOf, debtTokensFromCDP);
    }

    /// @notice Liquidates `tokenId` position as external user.
    function _liquidatePositionAsExternalUser(uint256 tokenId) internal{
        vm.startPrank(yetAnotherExternalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();
    }

    /// @notice Creates a maximum redemption as 0xbeef.
    function _createMaxRedemptionAsUser(address user) internal{ // INSIGHT: How will people find the position? createRedemption should return it.
        uint256 debt = alToken.balanceOf(user);
        vm.startPrank(user);
        
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debt);
        transmuterLogic.createRedemption(debt);

        vm.stopPrank();
    }

    function _claimRedemptionAsUser(address user, uint256 positionId) internal{
        vm.startPrank(user);
        transmuterLogic.claimRedemption(positionId);
        vm.stopPrank();
    }

    function _generateUsers(uint256 num) internal returns (address[] memory) {
        address[] memory addresses = new address[](num);
        for (uint256 i = 0; i < num; i++) {
            addresses[i] = vm.addr(uint256(keccak256(abi.encodePacked(i))));
        }
        return addresses;
    }

    function testForceRepay_Protocol_Insolvent() external{
        console.log("Depositing and max minting with %s users.", users2.length);
        for(uint i=0; i < users2.length; i++){
            uint256 tokenId = _depositAsUser(users2[i], "user");
            _maxMintAsUser(users2[i], "user", tokenId);
        }

        console.log("Users create full redemptions so debt can be earmarked.");
            for(uint i=0; i < users2.length; i++){
            _createMaxRedemptionAsUser(users2[i]);
        }

        vm.roll(block.number + 5_256_000 * 100 / 100);

        _increaseYieldTokenSupply(600);

        console.log("Triggering forced repayments...");

        for(uint i=0; i < users2.length; i++){
            _liquidatePositionAsExternalUser(i + 1);
        }

        console.log("Forced repayments triggered.");

        _increaseYieldTokenSupply(10_000);
        uint256 insolvents;
        for(uint i=0; i < users2.length; i++){
            _claimRedemptionAsUser(users2[i], i + 1);
            uint bal = TokenUtils.safeBalanceOf(alchemist.myt(), users2[i]);
            if (bal == 0) insolvents++;
        }
        console.log("Transmuter fully insolvent for %s/%s of users", insolvents, users2.length);
        return;
    }
}
```

As output, we get:

```
[PASS] testForceRepay_Protocol_Insolvent() (gas: 158893491)
Logs:
  Depositing and max minting with 100 users.
  Users create full redemptions so debt can be earmarked.
  Increased MYT supply from 20000000000000000000000000 to 21200000000000000000000000
  Triggering forced repayments...
  Forced repayment cleared debt, no liquidation necessary.
  Forced repayment cleared debt, no liquidation necessary.
  Forced repayment cleared debt, no liquidation necessary.
  Forced repayment cleared debt, no liquidation necessary.
  // .. added console logs to check no liquidations happened.
  Forced repayments triggered.
  Increased MYT supply from 20000000000000000000000000 to 40000000000000000000000000
  Transmuter fully insolvent for 46/100 of users
```
