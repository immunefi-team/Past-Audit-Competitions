# 58780 sc high weth yield will be locked on aaveweth pool on arbitrum&#x20;

**Submitted on Nov 4th 2025 at 13:19:35 UTC by @kenzo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58780
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

Rewards earned by AaveV3ARBWETHStrategy will not be withdrawable from Aave WETH pool on Arbitrum.

## Vulnerability Details

In the Alchemix strategy contracts, a strategy contract is deployed for every MYT contract. MYT contract is the VaultV2 contract which users deposit assets into. These assets can be deposited in strategies through allocations and then earn yield.

The problem is that in the `AaveV3ARBWETHStrategy` contract, only the initial full allocation of assets to Aave WETH on Arbitrum pool can be withdrawn because once allocation becomes zero, all deallocation call from VaultV2 will revert and `AaveV3ARBWETHStrategy` contract does not expose a function to claim earned yields from Aave. So, this yield will be locked and will not be sent to MYT contract for the users to claim

```solidity
contract AaveV3ARBWETHStrategy is MYTStrategy {
    IERC20 public immutable weth; // ARB WETH
    IAavePool public immutable pool; // Aave v3 Pool on ARB
    IAaveAToken public immutable aWETH; // aToken for WETH on ARB

    constructor(address _myt, StrategyParams memory _params, address _aWETH, address _weth, address _pool, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _weth)
    {
        weth = IERC20(_weth);
        pool = IAavePool(_pool);
        aWETH = IAaveAToken(_aWETH);
    }

    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(pool), amount);
        pool.supply(address(weth), amount, address(this), 0);
        return amount;
    }

    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        // withdraw exact underlying amount back to this adapter
        pool.withdraw(address(weth), amount, address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
}
```

This is the `AaveV3ARBWETHStrategy` and it inherits MYTStrategy. In MYT base contract there is claim rewards function but that is not enough because when we allocate assets to Aave, we use `asset` not `share` amounts. Since if we deposit 10k WETH into Aave and earn 13 WETH after 1 year, when we withdraw 10k WETH from Aave, allocation will become 0 and not all shares will be burned in Aave and we still have 13 WETH yield assets in Aave and shares corresponding to this amount.

Also, in the VaultV2 contract, we cannot force deallocation once the current `caps[id].allocation` is 0 since it will revert from there.

Issue steps is:

1. User deposits 10k WETH into VaultV2 contract
2. After sometime e.g 2 hours, operator or the admin calls the allocator contract to allocate 10k WETH from VaultV2 to the `AaveV3ARBWETHStrategy`
3. Aave mints shares to the `AaveV3ARBWETHStrategy` and the 10k WETH is deposited into Aave
4. After sometime e.g 1 month, user wants to withdraw back 10k WETH, the contract deallocates the 10k allocated to Aave
5. Allocation to Aave becomes 0, 10k WETH is sent to user.
6. 13.8 WETH is then locked in the Aave pool. This is the yield amount gained from the allocation amount of 10k WETH for some month. Only way the `AaveV3ARBWETHStrategy` contract withdraws from Aave is, if we trigger `_deallocate` which we cannot since allocation is 0.

The POC demonstrates the issue, please check the logs to see the stuck WETH in Aave.

Alchemix team needs to add a function to `AaveV3ARBWETHStrategy` contract to claim rewards/withdraw yields otherwise these yields will be stuck in Aave.

Note that I have also logged this issue in another one of my reports and this time it affects another asset in scope which is the `AaveV3ARBWETHStrategy` contract.

## Impact Details

Stuck yields in Aave that cannot be claimed since allocation is already depleted and since we work with assets in allocation instead of shares.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol>

## Proof of Concept

## POC

Paste the below codes into the AaveV3ARBWETHStrategy.t.sol file:

```solidity
import {IAllocator} from "../../interfaces/IAllocator.sol";
import {console} from "forge-std/console.sol";


interface IAavePool {
    function balanceOf(address user) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
}

interface IWETH {
    function balanceOf(address user) external view returns (uint256);
}

function getForkBlockNumber() internal pure override returns (uint256) {
        return 387_030_683;
    }

function test_aaveWETHYieldLocked() public {
    uint256 amountToAllocate = 10000e18;
    uint256 amountToDeallocate = amountToAllocate;
    bytes32 allocationID = IMYTStrategy(strategy).adapterId();

    vm.startPrank(vault);
    deal(testConfig.vaultAsset, address(vault), amountToAllocate);
    vm.stopPrank();

    // start prank admin to call allocator.allocate
    vm.startPrank(admin);
    IAllocator(allocator).allocate(address(strategy), amountToAllocate);

    vm.warp(1762435367);
    vm.roll(387246683);

    IAllocator(allocator).deallocate(address(strategy), amountToDeallocate);

    uint256 sharesLef = IAavePool(AAVE_V3_ARB_WETH_ATOKEN).balanceOf(address(strategy));
    console.log("Shares in Aave: ", sharesLef);

    uint256 wethInVault = IWETH(WETH).balanceOf(address(vault));
    console.log("WETH in the Vault: ", wethInVault);

    uint256 allocation = IVaultV2(vault).allocation(allocationID);
    console.log("Allocation: ", allocation);

    vm.expectRevert();
    IAllocator(allocator).deallocate(address(strategy), amountToDeallocate);

    vm.stopPrank();
}
```

In the logs below for this test, you will see that 13 WETH is locked in Aave and further deallocation calls (to attempt reclaiming back these WETH) will revert:

```js
Logs:
  Shares in Aave:  13184488084843127832
  WETH in the Vault:  10000000000000000000000
  Allocation:  0
```
