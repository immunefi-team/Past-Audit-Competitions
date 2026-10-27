# 57464 sc high incorrect accounting in stargate strategy causes protocol insolvency and user liquidations

**Submitted on Oct 26th 2025 at 13:16:49 UTC by @terrah for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57464
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Bug Description

The StargateEthPoolStrategy contract uses pool.redeemable() to report its assets instead of the actual LP token balance. In Stargate V2, redeemable() returns instantly available credits which can be significantly lower than the actual LP token value during normal AIPM operations.

## Brief/Intro

StargateEthPoolStrategy.realAssets() reports pool.redeemable(address(this)) instead of lp.balanceOf(address(this)). During regular Stargate V2 AIPM credit rebalancing operations, redeemable() can drop to 20% of actual LP value while the strategy still holds 100% of LP tokens. This false underreporting flows through the Morpho vault conversion rate into AlchemistV3's collateral calculations, causing healthy user positions to appear undercollateralized and get liquidated with permanent loss of funds.

It's worth noting that this is not a "black-swan" type of event on Stargate part, but normal protocol operations.

## Details

In StargateEthPoolStrategy.sol line 98-101:

```solidity
function realAssets() external view override returns (uint256) {
    return pool.redeemable(address(this));
}
```

The Stargate V2 documentation states: "The AIPM dynamically adjusts the distribution of credits across pathways in response to observed changes in volume and user behavior, optimizing liquidity, slippage, and fee structures for the protocol." (Source: Stargate documentation)

I'm including Stargate code as redeemable function is not well documented in the docs. From StargatePool.sol lines 153-165:

```solidity
/// @notice Get how many LP tokens can be redeemed by a given account.
/// @dev Use 0x0 to get the total maximum redeemable (since its capped to the local credit)
/// @param _owner The account to check for
/// @return amountLD The max amount of LP tokens redeemable by the account
function redeemable(address _owner) external view returns (uint256 amountLD) {
    uint256 cap = _sd2ld(paths[localEid].credit);
    if (_owner == address(0)) {
        amountLD = cap;
    } else {
        uint256 userLp = lp.balanceOf(_owner);
@>      amountLD = cap > userLp ? userLp : cap;
    }
}
```

This means pool.redeemable() can return much less than the actual LP token value held. The strategy may hold 100 ETH worth of LP tokens, but pool.redeemable() returns only 20 ETH during credit rebalancing.

The flow of the bug:

1. Strategy holds 100 ETH worth of LP tokens (lp.balanceOf = 100 ETH)
2. AIPM moves credits to another pathway (normal operations)
3. pool.redeemable() now returns 20 ETH (only instant liquidity available)
4. realAssets() returns 20 ETH instead of 100 ETH
5. MYT vault (Morpho V2) aggregates: vault.totalAssets() uses strategy.realAssets()
6. AlchemistV3 converts: convertYieldTokensToUnderlying() uses vault conversion rate
7. User with 100 MYT shares and 80 ETH debt appears to have only 20 ETH collateral
8. Collateralization ratio = 20/80 = 0.25 (25%) instead of 100/80 = 1.25 (125%)
9. AlchemistV3.\_liquidate() executes because ratio is below collateralizationLowerBound
10. User permanently loses collateral to transmuter and pays liquidation fees

From AlchemistV3.sol:

```solidity
function _liquidate(uint256 accountId) internal returns (...) {
    uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;
    
    if (collateralizationRatio <= collateralizationLowerBound) {
        // Liquidation executes - permanent loss
        account.collateralBalance -= amountLiquidated;
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated);
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    }
}
```

The Stargate documentation confirms this is expected behavior: "At its most malicious, the AIPM could cause liveness issues for certain pathways if it, for example, withdrew 100% of credits on a pathway meaning no volume can move between those two pools. This mechanism can also be protective in case of chain exploit, or asset depeg." This means credit movements are not rare events but active protocol operations.

## Impact

This causes protocol insolvency. The protocol reports holding less collateral than it actually has, leading to false liquidations of healthy positions. The accounting discrepancy is not theoretical - during AIPM credit rebalancing (which happens during normal operations to optimize liquidity), pool.redeemable() can report 20-30% of actual LP value while the LP tokens remain fully owned by the strategy.

Example scenario with real numbers:

Strategy holds 1000 ETH in LP tokens. AIPM moves credits to high-demand pathway. pool.redeemable() drops to 200 ETH. All users with Stargate collateral now show 80% less collateral value. Any user with debt ratio above 125% (using real 1000 ETH value) now appears to have ratio below 25% (using false 200 ETH value) and gets liquidated.

A user with 100 ETH collateral and 80 ETH debt (healthy 125% ratio) loses their entire position plus liquidation fees because the system thinks they only have 20 ETH collateral (false 25% ratio). When credits return hours or days later, the user's collateral is already gone - sent to transmuter and paid as liquidation fees.

This affects all users with Stargate strategy collateral during credit rebalancing periods. The loss is permanent and irreversible. Users lose principal funds, not unclaimed yield.

## Risk Breakdown

Difficulty: None (happens automatically during normal protocol operations) Likelihood: High (AIPM rebalances credits regularly based on bridge volume) Impact: Critical (permanent loss of user funds via false liquidations)

The AIPM actively manages credits across pathways. The Stargate documentation states this is core functionality, not an edge case. Credit rebalancing happens whenever there are volume changes across chains, which is daily or multiple times per day in production.

## Recommendation

Change realAssets() to return the actual LP token balance instead of instant redeemable amount:

```solidity
function realAssets() external view override returns (uint256) {
    return lp.balanceOf(address(this));
}
```

The strategy's real assets are the LP tokens it owns, not the instantly redeemable liquidity. Redemption timing is a liquidity constraint that should not affect accounting. The LP tokens retain their full value even when credits are temporarily moved to other pathways.

This change accurately represents the strategy's holdings for collateral calculations while redemption mechanics remain unchanged in the allocate/deallocate functions which already handle the redemption process correctly.

## References

Stargate V2 Documentation on AIPM: <https://stargateprotocol.gitbook.io/stargate/v2-user-docs/whats-new-in-stargate-v2/credit-allocation-system>

Stargate V2 FAQ on AIPM risks: <https://stargateprotocol.gitbook.io/stargate/v/v2-user-docs/faq>

StargateEthPoolStrategy.sol (lines 98-101) AlchemistV3.sol \_liquidate function MYT vault (Morpho V2) totalAssets calculation

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeERC20} from "../../src/libraries/SafeERC20.sol";
import {AlchemistV3} from "../../src/AlchemistV3.sol";
import {AlchemicTokenV3} from "../../src/test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../../src/Transmuter.sol";
import {AlchemistV3Position} from "../../src/AlchemistV3Position.sol";
import {AlchemistInitializationParams} from "../../src/interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../../src/interfaces/ITransmuter.sol";
import {TokenUtils} from "../../src/libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../../src/AlchemistTokenVault.sol";
import {MYTTestHelper} from "../../src/test/libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../../src/interfaces/IMYTStrategy.sol";
import {MockAlchemistAllocator} from "../../src/test/mocks/MockAlchemistAllocator.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MYTStrategy} from "../../src/MYTStrategy.sol";

interface IStargatePool {
    function deposit(address receiver, uint256 amountLD) external payable returns (uint256 amountLDOut);
    function redeem(uint256 lpAmount, address receiver) external returns (uint256 amountLDOut);
    function redeemable(address owner) external view returns (uint256 amountLD);
    function lpToken() external view returns (address);
    function tvl() external view returns (uint256);
}

contract MockStargatePoolWithAIPM {
    IStargatePool public realPool;
    uint256 public redeemableMultiplier = 10000;

    constructor(address _realPool) {
        realPool = IStargatePool(_realPool);
    }

    function setRedeemableMultiplier(uint256 _multiplier) external {
        redeemableMultiplier = _multiplier;
    }

    function deposit(address receiver, uint256 amountLD) external payable returns (uint256) {
        return realPool.deposit{value: msg.value}(receiver, amountLD);
    }

    function redeem(uint256 lpAmount, address receiver) external returns (uint256) {
        return realPool.redeem(lpAmount, receiver);
    }

    function redeemable(address owner) external view returns (uint256) {
        uint256 actualRedeemable = realPool.redeemable(owner);
        return actualRedeemable * redeemableMultiplier / 10000;
    }

    function lpToken() external view returns (address) {
        return realPool.lpToken();
    }

    function tvl() external view returns (uint256) {
        return realPool.tvl();
    }

    receive() external payable {}
}

contract MockStargateEthPoolStrategy is MYTStrategy {
    address public weth;
    MockStargatePoolWithAIPM public mockPool;
    address public lpToken;

    constructor(
        address _myt,
        StrategyParams memory _params,
        address _weth,
        address payable _mockPool,
        address _permit2Address
    ) MYTStrategy(_myt, _params, _permit2Address, _weth) {
        weth = _weth;
        mockPool = MockStargatePoolWithAIPM(_mockPool);
        lpToken = mockPool.lpToken();
    }

    function _allocate(uint256 amount) internal override returns (uint256) {
        IERC20(weth).approve(address(this), amount);
        (bool success,) = weth.call(abi.encodeWithSignature("withdraw(uint256)", amount));
        require(success, "WETH withdraw failed");

        uint256 amountToDeposit = (amount / 1e12) * 1e12;
        mockPool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
        return amount;
    }

    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 lpBalance = IERC20(lpToken).balanceOf(address(this));
        uint256 lpNeeded = amount;

        if (lpNeeded > lpBalance) {
            lpNeeded = lpBalance;
        }

        IERC20(lpToken).approve(address(mockPool), lpNeeded);
        uint256 ethBalanceBefore = address(this).balance;
        mockPool.redeem(lpNeeded, address(this));
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;

        (bool success,) = weth.call{value: ethRedeemed}(abi.encodeWithSignature("deposit()"));
        require(success, "WETH deposit failed");

        IERC20(weth).approve(msg.sender, amount);
        return amount;
    }

    function realAssets() external view override returns (uint256) {
        return mockPool.redeemable(address(this));
    }

    function _previewAdjustedWithdraw(uint256 amount) internal view override returns (uint256) {
        uint256 withSlippage = amount - (amount * slippageBPS / 10_000);
        uint256 divisibleAmount = (withSlippage / 1e12) * 1e12;
        return divisibleAmount;
    }

    receive() external payable {}
}

contract Poc010IncorrectAccountingTest is Test {
    address public constant STARGATE_ETH_POOL = 0xe8CDF27AcD73a434D661C84887215F7598e7d0d3;
    address public constant WETH = 0x4200000000000000000000000000000000000006;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    AlchemistV3 public alchemist;
    Transmuter public transmuter;
    AlchemistV3Position public alchemistNFT;
    AlchemicTokenV3 public alToken;
    AlchemistTokenVault public alchemistFeeVault;

    VaultV2 public vault;
    MockAlchemistAllocator public allocator;
    MockStargateEthPoolStrategy public strategy;
    MockStargatePoolWithAIPM public mockPool;

    address public admin = address(0x4444);
    address public curator = address(0x8888);
    address public operator = address(0x2222);
    address public user = address(0xBEEF);
    address public liquidator = address(0xDEAD);
    address public protocolFeeReceiver = address(0xFEE);

    uint256 public constant FIXED_POINT_SCALAR = 1e18;
    uint256 public constant BPS = 10_000;
    uint256 public collateralizationLowerBound = 1_052_631_578_950_000_000;
    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;
    uint256 public liquidatorFeeBPS = 300;

    function setUp() public {
        vm.createSelectFork(vm.envString("OPTIMISM_RPC_URL"), 141_751_698);

        _setupMYT();
        _setupAlchemist();
        _fundUsers();
    }

    function _setupMYT() internal {
        vm.startPrank(admin);

        vault = MYTTestHelper._setupVault(WETH, admin, curator);

        mockPool = new MockStargatePoolWithAIPM(STARGATE_ETH_POOL);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: admin,
            name: "StargateEthPool",
            protocol: "StargateEthPool",
            riskClass: IMYTStrategy.RiskClass.HIGH,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strategy = new MockStargateEthPoolStrategy(
            address(vault),
            params,
            WETH,
            payable(address(mockPool)),
            OPTIMISM_PERMIT2
        );

        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();

        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        vault.addAdapter(address(strategy));
        bytes memory idData = strategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, 10_000e18)));
        vault.increaseAbsoluteCap(idData, 10_000e18);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, 1e18)));
        vault.increaseRelativeCap(idData, 1e18);
        vm.stopPrank();
    }

    function _setupAlchemist() internal {
        vm.startPrank(admin);

        alToken = new AlchemicTokenV3("", "", 0);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: admin,
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        transmuter = new Transmuter(transParams);

        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: admin,
            debtToken: address(alToken),
            underlyingToken: WETH,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: collateralizationLowerBound,
            globalMinimumCollateralization: 1_111_111_111_111_111_111,
            transmuter: address(transmuter),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: liquidatorFeeBPS,
            repaymentFee: 100,
            myt: address(vault)
        });

        AlchemistV3 alchemistLogic = new AlchemistV3();
        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        TransparentUpgradeableProxy proxyAlchemist = new TransparentUpgradeableProxy(
            address(alchemistLogic),
            admin,
            alchemParams
        );
        alchemist = AlchemistV3(address(proxyAlchemist));

        alToken.setWhitelist(address(alchemist), true);
        transmuter.setAlchemist(address(alchemist));
        transmuter.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(WETH, address(alchemist), admin);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        vm.stopPrank();
    }

    function _fundUsers() internal {
        deal(WETH, user, 1000e18);
        deal(WETH, liquidator, 1000e18);
        deal(WETH, address(alchemistFeeVault), 10_000e18);

        vm.startPrank(user);
        IERC20(WETH).approve(address(vault), type(uint256).max);
        vault.deposit(500e18, user);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(admin);
        deal(WETH, admin, 1000e18);
        IERC20(WETH).approve(address(vault), type(uint256).max);
        vault.deposit(500e18, admin);
        allocator.allocate(address(strategy), 500e18);
        vm.stopPrank();
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }
```
