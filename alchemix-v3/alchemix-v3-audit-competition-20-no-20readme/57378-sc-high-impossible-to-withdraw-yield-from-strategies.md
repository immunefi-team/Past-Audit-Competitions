# 57378 sc high impossible to withdraw yield from strategies

**Submitted on Oct 25th 2025 at 16:31:30 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57378
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Deallocation currently withdraws only principal and reduces the tracked allocation accordingly. If an admin brings an adapter’s allocation down to zero, any accrued interest (e.g., Aave aToken yield) remaining on the startegy cannot be pulled by subsequent deallocations because the vault blocks deallocation when allocation is zero and the call revert if the deallocated amount is higher than the allocation. Although interest still contributes to the vault’s total assets via `realAssets()`, it will not be able to be withdrawn by the strategy.

## Vulnerability Details

We will take the exemple of the Aave strategy’s deallocation path withdraws exactly the requested `amount` and returns exactly that amount to the vault, never more:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    // withdraw exact underlying amount back to this adapter
    pool.withdraw(address(usdc), amount, address(this));
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

The vault updates allocation based on the `change` returned by the strategy and rejects deallocations when allocation is zero:

```solidity
for (uint256 i; i < ids.length; i++) {
    Caps storage _caps = caps[ids[i]];
    require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
    _caps.allocation = (int256(_caps.allocation) + change).toUint256();
}
```

Meanwhile, the vault’s share price accounting includes adapter-level interest via `realAssets()`:

```solidity
uint256 realAssets = IERC20(asset).balanceOf(address(this));
for (uint256 i = 0; i < adapters.length; i++) {
    realAssets += IAdapter(adapters[i]).realAssets();
}
```

On Aave, `realAssets()` is the aToken balance (principal + interest):

```solidity
function realAssets() external view override returns (uint256) {
    // aToken balance reflects principal + interest in underlying units
    return aUSDC.balanceOf(address(this));
}
```

Result: if the admin deallocates exactly the principal until allocation hits zero, any residual interest remains at the adapter, and further deallocations are blocked by `ZeroAllocation`. There is no standard “harvest/skim” method to pull only interest without modifying allocation. Moreover after the deallocation the strategy compute the new allocation in the MYTStrategy contract:

```solidity
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountDeallocated = _deallocate(assets);
        // underflow here: 
        uint256 newAllocation = oldAllocation - amountDeallocated;

```

Because in the majority of the strategies, the deallocation function will return the amount used as parameter if the admin try to deallocate the principal + interest, the call will revert because of an underflow.

## Impact Details

* Interest may remain stuck on the strategy after principal is fully deallocated, making it unavailable.
* Although accounting includes the interest in total assets, the inability to repatriate can break assumptions about the value of MYT shares.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L128-L130>

## Proof of Concept

## Proof of Concept

This is a coded POC that demonstrate the issue, you can run it by copy pasting this code into a test file and run `forge test --mt test_StrandedInterest_WhenDeallocateToZero -vv`

In this POC:

1. user deposit 100 usdc in the vault
2. The owner allocate 100 usdc into aave
3. the vault earn 10 usdc as yield
4. if the admin deallocates 110 USDCs the call revert because of an underflow
5. if he deallocates 100 USDC, 10 aToken remain stuck in the strategy.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "forge-std/Test.sol";
import {IERC20} from "lib/vault-v2/src/interfaces/IERC20.sol";
import {VaultV2} from "lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "lib/vault-v2/src/interfaces/IVaultV2.sol";
import {AaveV3ARBUSDCStrategy} from "src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol";
import {AlchemistAllocator} from "src/AlchemistAllocator.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {IMYTStrategy} from "src/interfaces/IMYTStrategy.sol";
import {AlchemistCurator} from "src/AlchemistCurator.sol";

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

contract DeallocateInterestStrandingTest is Test {
    ERC20Mock usdc;
    ERC20Mock aToken;
    MockAavePool pool;
    VaultV2 vault;
    AaveV3ARBUSDCStrategy strat;
    AlchemistAllocator allocator;
    address owner = makeAddr("owner");
    address operator = makeAddr("operator");
    address permit2= makeAddr("permit2");

    function setUp() public {
      vm.warp(1524785992);
        vm.roll(4370000);
        usdc = new ERC20Mock();
        aToken = new ERC20Mock();
        pool = new MockAavePool(usdc, aToken);

        // Set up a minimal vault for illustration. In practice timelocks/roles must be set.
        vault = new VaultV2(owner, address(usdc));
        allocator = new AlchemistAllocator(address(vault), owner, operator);

        // Deploy strategy (constructor params abridged for the PoC)
        strat = new AaveV3ARBUSDCStrategy(address(vault),  IMYTStrategy.StrategyParams({
            owner: owner,
            name: "AaveV3ARBUSDC",
            protocol: "AaveV3ARBUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e6,
            globalCap: 1e18,
            estimatedYield: 100e6,
            additionalIncentives: false,
            slippageBPS: 1}), address(usdc), address(aToken), address(pool), permit2);
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

    function test_StrandedInterest_WhenDeallocateToZero() public {
        // Simulate adapter having principal 100 and accrued interest 10
        usdc.mint(address(this), 100e6);
        usdc.approve(address(vault), 100e6);
        vault.deposit(100e6, address(this));
        vm.prank(owner);
        allocator.allocate(address(strat), 100e6); // internal in real code; called by vault in prod
        usdc.mint(address(pool), 10e6);
        // Simulate +10e6 interest on aToken
        pool.addBoost(address(strat), 10e6);
        assertEq(aToken.balanceOf(address(strat)), 110e6, "principal + interest present");
        vm.expectRevert(stdError.arithmeticError);
        vm.prank(owner);
        // Deallocate  principal + interest this call will revert because of the underflow
       allocator.deallocate(address(strat), 110e6); // internal in real code; called by vault in prod
        vm.prank(owner);
        // Deallocate exactly principal
       allocator.deallocate(address(strat), 100e6); 

        // Residual interest remains on adapter
        assertEq(aToken.balanceOf(address(strat)), 10e6, "interest stranded on adapter");

    }
}
```
