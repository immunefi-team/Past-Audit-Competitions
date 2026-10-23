# 58759 sc high yield stuck in adapter contracts forever

**Submitted on Nov 4th 2025 at 12:10:01 UTC by @Django for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58759
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The adapter contracts allow underlying tokens to be allocated to other vaults to accumulate rewards, however, the rewards can never be claimed from the adapter contract. The adapter's `realAssets()` will show growth, but the growing capital can not be deallocated.

## Vulnerability Details

Quite simply, there is no way to withdraw accumulated yield out of the adapter contracts. The adapters' `realYield()` amount will continue to grow as rewards are earned by the adapter, however, Alchemix will only be able to deallocate the amount that was previously allocated, not the yield.

For example, take the following strategy:

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
        vault.withdraw(amount, address(this), address(this));
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
        uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
        uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
        if (usdcRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        return amount;
    }
```

We see that upon deallocation, the exact amount requested is approved to the calling contract (VaultV2), however any yield generated is not returned.

## Impact Details

* All yield (rewards from adapter vaults) will remain stuck in adapter contracts.

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import "../libraries/SafeCast.sol";
import "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {EulerUSDCAdapter} from "../adapters/EulerUSDCAdapter.sol";
import {Transmuter} from "../Transmuter.sol";
import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {IAlchemicToken} from "../interfaces/IAlchemicToken.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {AlchemistETHVault} from "../AlchemistETHVault.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";

// Tests for integration with Euler V2 Earn Vault
contract IntegrationTest is Test {
    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;
    address weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;
    uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;
    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e18;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;
    // Fee receiver
    address receiver = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    address alUSD = 0xBC6DA0FE9aD5f3b0d58160288917AA56653660E9;

    address USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    address EULER_USDC = 0x797DD80692c3b2dAdabCe8e30C07fDE5307D48a9;

    // MYT variables
    VaultV2 vault;
    MockAlchemistAllocator allocator;
    MockMYTStrategy mytStrategy;
    address public operator = address(20); // default operator
    address public admin = address(21); // DAO OSX
    address public curator = address(22);
    address public mockVaultCollateral;
    address public mockStrategyYieldToken;
    uint256 public defaultStrategyAbsoluteCap = 2_000_000_000e18;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%

    event TestIntegrationLog(string message, uint256 value);

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        setUpMYT(6); // 6 decimals for USDC underlying token
        addDepositsToMYT();

        vm.startPrank(caller);

        /*         deal(EULER_USDC, address(0xbeef), 100_000e18);
        deal(EULER_USDC, address(0xdad), 100_000e18); */
        deal(alUSD, address(0xdad), 100_000e18);
        deal(alUSD, address(0xdead), 100_000e18);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: alUSD,
            feeReceiver: receiver,
            timeToTransmute: 5_256_000,
            transmutationFee: 100,
            exitFee: 200,
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
            debtToken: alUSD,
            underlyingToken: USDC,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            transmuter: address(transmuterLogic),
            protocolFee: 100,
            protocolFeeReceiver: receiver,
            liquidatorFee: 300, // in bps? 3%
            repaymentFee: 100,
            myt: address(vault)
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        transmuterLogic.setDepositCap(uint256(type(int256).max));

        transmuterLogic.setAlchemist(address(alchemist));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        vm.stopPrank();

        vm.startPrank(0x8392F6669292fA56123F71949B52d883aE57e225);
        IAlchemicToken(alUSD).setWhitelist(address(alchemist), true);
        IAlchemicToken(alUSD).setCeiling(address(alchemist), type(uint256).max);
        vm.stopPrank();
    }

    function setUpMYT(uint256 alchemistUnderlyingTokenDecimals) public {
        vm.startPrank(admin);
        uint256 TOKEN_AMOUNT = 1_000_000; // Base token amount
        uint256 initialSupply = TOKEN_AMOUNT * 10 ** alchemistUnderlyingTokenDecimals;
        mockVaultCollateral = USDC;
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

    function addDepositsToMYT() public {
        uint256 shares = _magicDepositToVault(address(vault), address(0xbeef), 1_000_000e6);
        emit TestIntegrationLog("0xbeef shares", shares);
        shares = _magicDepositToVault(address(vault), address(0xdad), 1_000_000e6);
        emit TestIntegrationLog("0xdad shares", shares);

        // then allocate to the strategy
        vm.startPrank(address(admin));
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();
    }

    function _magicDepositToVault(address vault, address depositor, uint256 amount) internal returns (uint256) {
        deal(USDC, address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(USDC, vault, amount);
        uint256 shares = IVaultV2(vault).deposit(amount, depositor);
        vm.stopPrank();
        return shares;
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }

    function testRoundTrip() external {
        vm.startPrank(address(0xbeef));
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        alchemist.deposit(100_000e18, address(0xbeef), 0);
        // a single position nft would have been minted to address(0xbeef)
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        (uint256 collateral,,) = alchemist.getCDP(tokenId);
        assertEq(collateral, 100_000e18);
        assertEq(IERC20(address(vault)).balanceOf(address(alchemist)), 100_000e18);

        alchemist.withdraw(100_000e18, address(0xbeef), tokenId);
        vm.stopPrank();

        (collateral,,) = alchemist.getCDP(tokenId);
        assertEq(collateral, 0);
        assertEq(IERC20(address(vault)).balanceOf(address(0xbeef)), 1_000_000e18);
    }
}
```
