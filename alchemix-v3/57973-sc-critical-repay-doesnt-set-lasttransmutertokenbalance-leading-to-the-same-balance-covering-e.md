# 57973 sc critical repay doesnt set lasttransmutertokenbalance leading to the same balance covering earmark twice&#x20;

**Submitted on Oct 29th 2025 at 18:20:24 UTC by @silver\_eth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57973
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

A core invariant of the system could be defined as `transmutted debt might either be included in earmark or there must be subsequent collateral backing it already in the transmutter contract` any break in this invariant would constitute an issue any contract that sends collateral into the transmuter and updates earmarked based on this collateral without setting lastTransmuterTokenBalance breaks this invariant

## Vulnerability Details

one of the ways this invariant breaks that occurs in both repay and \_forceRepay is that the functions decrease earmark (using the funds to cover current earmark) but doesnt set lastTransmuterTokenBalance (meaning those same funds can be used to cover future transmutations when \_earmark is called next )

the example described in the poc demonstrates this

1. bob completes a borrow
2. bob transmutes some debt
3. after half the time, half of the original redemption amount has been transmuted
4. bob repays part of his debt which clears previous earmark (collateral now backs previous earmark)
5. the other half of the transmutation time passes
6. bobs position is poked and \_earmark is called, it sees the collateral that is in the transmuter and uses it to back all the newly transmuted debt meaning necessary state updates dont happen
7. bob claims his redemption but only claims half the the total value of the deb the transmutted
8. the remaining half of the transmuted debt is also burned without clearing any debt, confirmed by the last assertion which could be counted as either a loss for the redemption owner or protocol

## Impact Details

direct and complete loss of both some collateral and debt tokens belonging to redeemers

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L526-L539> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1103-L1110>

## Proof of Concept

## Proof of Concept

// SPDX-License-Identifier: GPL-3.0-or-later pragma solidity 0.8.28;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol"; import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol"; import {SafeCast} from "../libraries/SafeCast.sol"; import {Test} from "../../lib/forge-std/src/Test.sol"; import {SafeERC20} from "../libraries/SafeERC20.sol"; import {console} from "../../lib/forge-std/src/console.sol"; import {AlchemistV3} from "../AlchemistV3.sol"; import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol"; import {Transmuter} from "../Transmuter.sol"; import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol"; import {TestERC20} from "./mocks/TestERC20.sol"; import {TestYieldToken} from "./mocks/TestYieldToken.sol"; import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol"; import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol"; import {ITransmuter} from "../interfaces/ITransmuter.sol"; import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol"; import {InsufficientAllowance} from "../base/Errors.sol"; import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol"; import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol"; import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol"; import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol"; import {TokenUtils} from "../libraries/TokenUtils.sol"; import {AlchemistTokenVault} from "../AlchemistTokenVault.sol"; import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol"; import {MYTTestHelper} from "./libraries/MYTTestHelper.sol"; import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol"; import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol"; import {IMockYieldToken} from "./mocks/MockYieldToken.sol"; import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol"; import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol"; import {MockYieldToken} from "./mocks/MockYieldToken.sol";

contract SilverAlchemistTest is Test { // ----- \[SETUP] Variables for setting up a minimal CDP -----

```
// Callable contract variables
AlchemistV3 alchemist;
Transmuter transmuter;
AlchemistV3Position alchemistNFT;
AlchemistTokenVault alchemistFeeVault;

// // Proxy variables
TransparentUpgradeableProxy proxyAlchemist;
TransparentUpgradeableProxy proxyTransmuter;

// Contract variables
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

address bob = address(0xbeef);

address alice = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

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
    mytStrategy = MYTTestHelper._setupStrategy(
        address(vault),
        mockStrategyYieldToken,
        admin,
        "MockToken",
        "MockTokenProtocol",
        IMYTStrategy.RiskClass.LOW
    );
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
    vm.prank(address(allocator));
    vault.setMaxRate(200e16 / uint256(365 days));
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

    _magicDepositToVault(address(vault), bob, accountFunds);
    _magicDepositToVault(address(vault), alice, accountFunds);
    // _magicDepositToVault(address(vault), externalUser, accountFunds);
    // _magicDepositToVault(address(vault), yetAnotherExternalUser, accountFunds);
    // _magicDepositToVault(address(vault), anotherExternalUser, accountFunds);
    vm.stopPrank();

    // vm.startPrank(address(admin));
    // allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
    // vm.stopPrank();

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
    // deal(address(vault.asset()), someWhale, whaleSupply);
    SafeERC20.safeApprove(address(vault.asset()), address(mockStrategyYieldToken), whaleSupply);
    vm.stopPrank();
}

modifier executeUnderPrank(address caller) {
    _startPrank(caller);
    _;
    _stopPrank();
}

function _startPrank(address caller) private {
    vm.startPrank(caller);
}

function _stopPrank() private {
    vm.stopPrank();
}

function _addCollateral(address caller, uint256 tokenId, uint256 amount) public executeUnderPrank(caller) {
    SafeERC20.safeApprove(address(vault), address(alchemist), amount);
    alchemist.deposit(amount, caller, tokenId);
}

function _removeCollateral(address caller, uint256 tokenId, uint256 amount) public executeUnderPrank(caller) {
    alchemist.withdraw(amount, caller, tokenId);
}

function _createRedemption(address caller, uint256 amount) public executeUnderPrank(caller) {
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amount);
    transmuterLogic.createRedemption(amount);
}

function _claimRedemption(address caller, uint256 tokenId) public executeUnderPrank(caller) {
    transmuterLogic.claimRedemption(tokenId);
}

function _mint(address caller, uint256 tokenId, uint256 amount) public executeUnderPrank(caller) {
    alchemist.mint(tokenId, amount, caller);
}

function _burn(address caller, uint256 tokenId, uint256 amount) public executeUnderPrank(caller) {
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint).max);
    alchemist.burn(amount, tokenId);
    SafeERC20.safeApprove(address(vault), address(alchemist), 0);
}

function _repay(address caller, uint256 tokenId, uint256 amount) public executeUnderPrank(caller) {
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint).max);
    alchemist.repay(amount, tokenId);
    SafeERC20.safeApprove(address(vault), address(alchemist), 0);
}

function _liquidate(address caller, uint256 tokenId) public executeUnderPrank(caller) {
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint).max);
    alchemist.liquidate(tokenId);
    SafeERC20.safeApprove(address(vault), address(alchemist), 0);
}

function _simulateVaultProfit(uint256 amount) public {
    deal(address(mytStrategy.token()), address(mytStrategy), mytStrategy.realAssets() + amount);
    deal(
        address(mytStrategy.token().underlyingToken()),
        address(mytStrategy.token()),
        TokenUtils.safeBalanceOf(mytStrategy.token().underlyingToken(), address(mytStrategy.token())) + amount
    );
    skip(200 days);

    vault.accrueInterest();
}

function _simulateVaultProfitPrice(uint256 percentDecrease) public {
    uint256 currentSupply = MockYieldToken(mockStrategyYieldToken).mockTokenSupply();
    uint256 newSupply = (currentSupply * percentDecrease) / 1e18;
    MockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);
    skip(100 days);
    vault.accrueInterest();
    deal(bob, 1e18);
}

function _simulateVaultLoss(uint256 amount) public {
    deal(address(mytStrategy.token()), address(mytStrategy), mytStrategy.realAssets() - amount);
    deal(
        address(mytStrategy.token().underlyingToken()),
        address(mytStrategy.token()),
        TokenUtils.safeBalanceOf(mytStrategy.token().underlyingToken(), address(mytStrategy.token())) - amount
    );
    skip(200 days);

    vault.accrueInterest();
}

function _simulateVaultLossPrice(uint256 percentIncrease) public {
    uint256 currentSupply = MockYieldToken(mockStrategyYieldToken).mockTokenSupply();
    uint256 newSupply = (currentSupply * percentIncrease) / 1e18;
    MockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);
    skip(100 days);
    vault.accrueInterest();
}

function _poke(uint256 tokenId) public {
    alchemist.poke(tokenId);
}

function _upperBoundMinimumCollaterization() public {
    vm.prank(alOwner);
    alchemist.setMinimumCollateralization(uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9.5e17);
}

function _lowerBoundMinimumCollaterization() public {
    vm.prank(alOwner);
    alchemist.setMinimumCollateralization(uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 8.5e17);
}

function skipBlock(uint256 blocks) private {
    vm.roll(vm.getBlockNumber() + blocks);
}

function testRepayDoubleEarmark() public {
    _addCollateral(bob, 0, 100e18);
    _mint(bob, 1, 90e18);
    _createRedemption(bob, 80e18);
    skipBlock(transmuterLogic.timeToTransmute() / 2);
    _repay(bob, 1, 40e18);
    skipBlock(transmuterLogic.timeToTransmute() / 2 + 1);
    _poke(1);
    uint256 bobsPreviousBalance = vault.balanceOf(address(bob));
    _claimRedemption(bob, 1);
    uint256 bobsNewBalance = vault.balanceOf(address(bob));
    console.log(bobsNewBalance - bobsPreviousBalance);
    assertEq(
        bobsNewBalance - bobsPreviousBalance,
        (40e18 * (10_000 - transmuterLogic.transmutationFee())) / 10_000
    );
    // should be 80e18, comment out the repay to check
     assertEq(alchemist.totalDebt(), alchemist.totalSyntheticsIssued() + 40e18);
}
```

}
