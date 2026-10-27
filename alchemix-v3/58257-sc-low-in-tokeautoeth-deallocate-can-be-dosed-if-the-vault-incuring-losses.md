# 58257 sc low in tokeautoeth deallocate can be dosed if the vault incuring losses

**Submitted on Oct 31st 2025 at 18:58:27 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58257
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`TokeAutoEthStrategy` and `TokeAutoUSDStrategy` can revert during deallocation after a loss because they compute `shareDiff = actualSharesHeld - sharesNeeded` without guarding `sharesNeeded > actualSharesHeld`. If the vault price per share has fallen (loss), `sharesNeeded` can exceed the strategy’s share balance, causing an immediate underflow revert. Even if that check were bypassed, the strategies require returning exactly the requested `amount` of underlying, which can be impossible post‑loss, forcing top‑ups to succeed.

## Vulnerability Details

Both strategies of tokeAuto convert the requested `amount` of underlying to shares, then subtract from the staked share balance before redeeming. The subtraction is unchecked for ordering and will underflow if the vault suffered losses.

TokeAutoETH:

```solidity
// src/strategies/mainnet/TokeAutoEth.sol
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    // @audit underflow if sharesNeeded > actualSharesHeld
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        // account for vault rounding up
        sharesNeeded = actualSharesHeld;
    }
    rewarder.withdraw(address(this), sharesNeeded, true);
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    autoEth.redeem(sharesNeeded, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

TokeAutoUSD:

```solidity
// src/strategies/mainnet/TokeAutoUSDStrategy.sol
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoUSD.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    // @audit underflow if sharesNeeded > actualSharesHeld
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        sharesNeeded = actualSharesHeld;
    }
    rewarder.withdraw(address(this), sharesNeeded, true);
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    autoUSD.redeem(sharesNeeded, address(this), address(this));
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    return amount;
}
```

Additionally, the base interface enforces that deallocation must return exactly the requested `amount`, leaving no room to return less when the vault has losses:

```solidity
// src/MYTStrategy.sol
/// @notice uint56 amount returned must be equal to the amount parameter passed in
function _deallocate(uint256 amount) internal virtual returns (uint256) {}
```

As a result:

* If `sharesNeeded > actualSharesHeld`, the subtraction underflows and reverts immediately.
* Even if clamped later, the `require(balance >= amount)` will revert after redeeming fewer assets due to losses.
* An admin could “make it pass” by deallocated a smaller amount but it will however break the accounting since the funds allocated will be inflated.

## Impact Details

* Liveness risk: deallocation can brick under loss scenarios (can’t withdraw), blocking rebalancing/offboarding.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L68-L70>

## Proof of Concept

## Proof of Concept

You can run the POC by copy pasting this code in a new test file and run `forge test --mt test_Deallocate_poc`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "forge-std/Test.sol";
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {VaultV2} from "lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "lib/vault-v2/src/interfaces/IVaultV2.sol";
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
    constructor(IERC20 _asset) {
        asset = _asset;
    }
    function depositMax(ERC4626 vault, address to, uint256 minSharesOut) external returns (uint256 sharesOut) {
        uint256 amount = asset.balanceOf(msg.sender);
        asset.approve(address(vault), amount);
        asset.transferFrom(msg.sender, address(this), amount);
        return vault.deposit(amount, to);
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
    function test_Deallocate_poc() public {
        deal(address(asset), address(this), 10e18);
        asset.approve(address(vault), 100e6);
        // deposit 100e6 into the vault
        vault.deposit(100e6, address(this));
        vm.prank(owner);
        // allocate 100e6 to the strategy
        allocator.allocate(address(strat), 100e6);
        assertEq(rewarder.balanceOf(address(strat)), 100e6);

       uint256 balanceVault= asset.balanceOf(address(autoETHVault));
       // simulate a loss of 25% of the vault
       deal(address(asset), address(autoETHVault), balanceVault*3e18/4e18);
       vm.startPrank(owner);
       vm.expectRevert(stdError.arithmeticError);
       allocator.deallocate(address(strat), 100e6);
       vm.stopPrank();
    }
}
```
