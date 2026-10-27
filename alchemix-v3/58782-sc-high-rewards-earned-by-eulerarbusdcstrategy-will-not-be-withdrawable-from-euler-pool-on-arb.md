# 58782 sc high rewards earned by eulerarbusdcstrategy will not be withdrawable from euler pool on arbitrum

**Submitted on Nov 4th 2025 at 13:22:00 UTC by @kenzo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58782
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/EulerARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

Rewards earned by EulerARBUSDCStrategy will not be withdrawable from Euler pool on Arbitrum.

## Vulnerability Details

In the Alchemix strategy contracts, a strategy contract is deployed for every MYT contract. MYT contract is the VaultV2 contract which users deposit assets into. These assets can be deposited in strategies through allocations and then earn yield.

The problem is that in the `EulerARBUSDCStrategy` contract, only the initial full allocation of assets to Euler USDC on Arbitrum pool can be withdrawn because once allocation becomes zero, all deallocation call from VaultV2 will revert and `EulerARBUSDCStrategy` contract does not expose a function to claim earned yields from Euler. So, this yield will be locked and will not be sent to MYT contract for the users to claim

```solidity
contract EulerARBUSDCStrategy is MYTStrategy {
    IERC20 public immutable usdc;
    IERC4626 public immutable vault;

    constructor(address _myt, StrategyParams memory _params, address _usdc, address _eulerVault, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _usdc)
    {
        usdc = IERC20(_usdc);
        vault = IERC4626(_eulerVault);
    }

    function _allocate(uint256 amount) internal override returns (uint256 depositReturn) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        depositReturn = amount;
        TokenUtils.safeApprove(address(usdc), address(vault), amount);
        vault.deposit(amount, address(this));
    }

    function _deallocate(uint256 amount) internal override returns (uint256 withdrawReturn) {
        uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
        vault.withdraw(amount, address(this), address(this));
        withdrawReturn = amount;
        uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
        uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
        if (usdcRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    }

    ...
}
```

This is the `EulerARBUSDCStrategy` and it inherits MYTStrategy. In MYT base contract there is claim rewards function but that is not enough because when we allocate assets to Euler, we use `asset` not `share` amounts. Since if we deposit 10k USDC into Euler and earn 36 USDC after 1 month, when we withdraw 10k USDC from Euler, allocation will become 0 and not all shares will be burned in Euler and we still have 36 USDC yield assets in Euler and shares corresponding to this amount.

Also, in the VaultV2 contract, we cannot force deallocation once the current `caps[id].allocation` is 0 since it will revert from there.

Issue steps is:

1. User deposits 10k USDC into VaultV2 contract
2. After sometime e.g 2 hours, operator or the admin calls the allocator contract to allocate 10k USDC from VaultV2 to the `EulerARBUSDCStrategy`
3. Euler mints shares to the `EulerARBUSDCStrategy` and the 10k USDC is deposited into Euler
4. After sometime e.g 1 month, user wants to withdraw back 10k USDC, the contract deallocates the 10k allocated to Euler
5. Allocation to Euler becomes 0, 10k USDC is sent to user.
6. 36 USDC is then locked in the Euler pool. This is the yield amount gained from the allocation amount of 10k USDC for 1 month. Only way the `EulerARBUSDCStrategy` contract withdraws from Euler is, if we trigger `_deallocate` which we cannot since allocation is 0.

The forked POC demonstrates the issue, please check the logs to see the stuck USDC in Euler.

Alchemix team needs to add a function to `EulerARBUSDCStrategy` contract to claim rewards/withdraw yields otherwise these yields will be stuck in Euler.

Note that I have also logged this issue in another one of my reports and this time it affects another asset in scope which is the `EulerARBUSDCStrategy` contract.

## Impact Details

Stuck yields in Euler that cannot be claimed since allocation is already depleted and since we work with assets in allocation instead of shares.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/EulerARBUSDCStrategy.sol>

## Proof of Concept

## Proof of Concept

Paste the below codes into the EulerARBUSDCStrategy.t.sol file:

```solidity
import {IAllocator} from "../../interfaces/IAllocator.sol";

interface IVault {
    function balanceOf(address user) external view returns (uint256);
}

function getForkBlockNumber() internal pure override returns (uint256) {
        return 387_030_683;
    }

function test_eulerARBUSDCYieldLocked() public {
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

    uint256 sharesLef = IVault(EULER_USDC_VAULT).balanceOf(address(strategy));
    console.log("Shares locked in Euler: ", sharesLef);

    uint256 usdcInVault = IVault(USDC).balanceOf(address(vault));
    console.log("USDC in the Vault: ", usdcInVault);

    uint256 allocation = IVaultV2(vault).allocation(allocationID);
    console.log("Allocation: ", allocation);

    vm.expectRevert();
    IAllocator(allocator).deallocate(address(strategy), amountToDeallocate);

    vm.stopPrank();
}
```

In the logs below for this test, you will see that 36 USDC (earned for 1 month on Euler) is locked in Euler and further deallocation calls will revert:

```js
Logs:
  Shares locked in Euler:  36731136
  USDC in the Vault:  10000000000
  Allocation:  0
```
