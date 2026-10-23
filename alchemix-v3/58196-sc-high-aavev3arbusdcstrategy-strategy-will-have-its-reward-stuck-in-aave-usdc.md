# 58196 sc high aavev3arbusdcstrategy strategy will have its reward stuck in aave usdc

**Submitted on Oct 31st 2025 at 10:20:57 UTC by @kenzo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58196
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Permanent freezing of unclaimed royalties

## Description

## Description

In the Alchemix strategy contracts, a strategy is deployed for a MYT contract. MYT contract is the VaultV2 contract which users deposit assets into. These assets can be deposited in strategies through allocations and then earn yield.

The problem is that in the `AaveV3ARBUSDCStrategy` contract, only the initial full allocation of assets to Aave USDC on Arbitrum pool can be withdrawn because once allocation becomes zero, all deallocation call from VaultV2 will revert and `AaveV3ARBUSDCStrategy` contract does not expose a function to claim earned yields from Aave. So, this yield will be locked and will not be sent to MYT contract for the users to claim

```solidity
contract AaveV3ARBUSDCStrategy is MYTStrategy {
    IERC20 public immutable usdc; // ARB USDC
    IAavePool public immutable pool; // Aave v3 Pool on ARB
    IAaveAToken public immutable aUSDC; // aToken for USDC on ARB

    constructor(address _myt, StrategyParams memory _params, address _usdc, address _aUSDC, address _pool, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _usdc)
    {
        usdc = IERC20(_usdc);
        pool = IAavePool(_pool);
        aUSDC = IAaveAToken(_aUSDC);
    }

    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(pool), amount);
        pool.supply(address(usdc), amount, address(this), 0);
        return amount;
    }

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

    function _previewAdjustedWithdraw(uint256 amount) internal view override returns (uint256) {
        // amount in USDC is 1:1 with aUSDC.
        // which differs from actual balance of aUSDC which includes interest.
        return amount - (amount * slippageBPS / 10_000);
    }

    function realAssets() external view override returns (uint256) {
        // aToken balance reflects principal + interest in underlying units
        return aUSDC.balanceOf(address(this));
    }
}
```

This is `AaveV3ARBUSDCStrategy` and inherits MYTStrategy. In MYT base contract there is claim rewards function but that is not enough because when we allocate assets to Aave, we use `asset` not `share` amounts. Since if we deposit 1 USDC into Aave and earn 0.15 USDC after 1 year, when we withdraw 1 USDC from Aave, allocation will become 0 and about 0.9 shares will be burned in Aave but we still have 0.25 USDC assets in Aave and shares corresponding to this amount.

Also, in the VaultV2 contract, we cannot force deallocation once the current `caps[id].allocation` is 0 since it will revert from there.

### Attack Path:

1. User deposits 10k USDC into VaultV2 contract
2. After sometime e.g 2 hours, operator or the admin calls the allocator contract to allocate 10k USDC from VaultV2 to the `AaveV3ARBUSDCStrategy`
3. Aave mints shares to the `AaveV3ARBUSDCStrategy` and the 10k USDC is deposited into Aave
4. After sometime e.g 1 month, user wants to withdraw back 10k USDC, the contract deallocates the 10k allocated to Aave
5. Allocation to Aave becomes, 10k USDC is sent to user.
6. 36.6 USDC is locked in Aave pool. This is the yield amount gained from the allocation amount of 10k for 1 month. Only way the `AaveV3ARBUSDCStrategy` contract withdraws from Aave is, if we trigger `_deallocate` which we cannot since allocation is 0.

The POC demonstrates the issue, please check the logs to see the stuck yields in Aave.

## Impact

Stuck yields in Aave that cannot be claimed since allocation is already depleted and since we work with assets in allocation instead of shares.

## Recommendation

Add a function to `AaveV3ARBUSDCStrategy` contract to claim rewards/withdraw yields otherwise these yields will be stuck in Aave.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>

## Proof of Concept

## Proof of Concept

Add this to `AaveV3ARBUSDCStrategy.t.sol` test file

```solidity
import {IAllocator} from "../../interfaces/IAllocator.sol";
import {console} from "forge-std/console.sol";

interface IAavePool {
    function balanceOf(address user) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
}

interface IUSDC {
    function balanceOf(address user) external view returns (uint256);
}

function test_aaveUSDCYieldLocked() public {
        uint256 amountToAllocate = 10000e6;
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

        uint256 sharesLef = IAavePool(AAVE_V3_USDC_ATOKEN).balanceOf(address(strategy));
        console.log("Shares locked in Aave: ", sharesLef);

        uint256 usdcInVault = IUSDC(USDC).balanceOf(address(vault));
        console.log("USDC in the Vault: ", usdcInVault);

        uint256 allocation = IVaultV2(vault).allocation(allocationID);
        console.log("Allocation: ", allocation);

        vm.stopPrank();
    }
```

Test logs:

```js
Logs:
  Shares locked in Aave:  36632762
  USDC in the Vault:  10000000000
  Allocation:  0
```
