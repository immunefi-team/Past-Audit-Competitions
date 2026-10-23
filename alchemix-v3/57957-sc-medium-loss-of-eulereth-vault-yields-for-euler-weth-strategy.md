# 57957 sc medium loss of eulereth vault yields for euler weth strategy

**Submitted on Oct 29th 2025 at 15:29:47 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57957
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/EulerWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

When we allocate assets to third party protocols such as the EulerWETH vault on Ethereum mainnet, we get minted shares by Euler in return and during redemption of shares, we will receive more assets than we supplied into Euler. However, in the current implementation of the EulerWETHStrategy contract, these yields will not be claimed and lost.

## Vulnerability Details

The problem arises because when we interact with the Euler WETH vault, we mainly work with `asset` and not shares and thus are constrained to the `caps[id].allocation` of the strategy leading us to lose the yield gained and only being able to withdraw back the initial supplied assets.

```solidity
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets);

            // @note this function will ultimately deposit x amount of assets into Euler WETH through EulerWETHStrategy.allocate()
@>        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            _caps.allocation = (int256(_caps.allocation) + change).toUint256();

            require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());

            require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());

            require(
                _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
                ErrorsLib.RelativeCapExceeded()
            );
        }
        emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
    }
```

```solidity
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        ...
        uint256 oldAllocation = abi.decode(data, (uint256));
@>        uint256 amountAllocated = _allocate(assets);
        uint256 newAllocation = oldAllocation + amountAllocated;
        emit Allocate(amountAllocated, address(this));
        return (ids(), int256(newAllocation) - int256(oldAllocation));
    }
```

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(vault), amount); //
        vault.deposit(amount, address(this));
        return amount;
    }
```

1. The VaultV2 contract holds 100 WETH which Bob deposited.
2. Later, admin allocates 100 WETH to EulerWETHStrategy. This will call the Euler WETH vault `deposit` function on ETH mainnet. The `_caps.allocation = (int256(_caps.allocation) + change).toUint256()` will then be 100 WETH because `_caps.allocation` of 0 plus `change` of 100 equals 100 WETH allocation
3. A year goes by, users request withdraw or we just want to deallocate from the EulerWETHStrategy. We will only be able to withdraw 100 WETH whereas our asset balance inside Euler vault on Mainnet is now approximately 100.9 WETH because we have earned 0.9 WETH yield for the year. The function sequence will be as below:

```solidity
function deallocateInternal(address adapter, bytes memory data, uint256 assets)
        internal
        returns (bytes32[] memory)
    {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

            // @note call to deallocate maximum which will be 100 WETH, will return -100 WETH as change
@>        (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]]; // e.g 500
            require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
            _caps.allocation = (int256(_caps.allocation) + change).toUint256(); // @note `_caps.allocation` now becomes 0 (aka int256(100) + -100 = 0) since we received -100 WETH from a total of 100.9 WETH (0.9 WETH is still in the Euler vault on mainnet. These 0.9 WETH yield can now not be withdrawn since we have depleted `_cap.allocation`)
        }

        SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), assets);
        emit EventsLib.Deallocate(msg.sender, adapter, assets, ids, change);
        return ids;
    }
```

```solidity
function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
       ...
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256)); // 100 WETH
        uint256 amountDeallocated = _deallocate(assets); // 100 WETH
        uint256 newAllocation = oldAllocation - amountDeallocated; // 100 - 100 = 0
        emit Deallocate(amountDeallocated, address(this));

        return (ids(), int256(newAllocation) - int256(oldAllocation)); // -100
    }

    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        vault.withdraw(amount, address(this), address(this));
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");

        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        return amount; // @note it withdraws 100 WETH from Euler whereas our total balance in asset is 100.9 WETH since yield has accrued in the past year
    }
```

4. As we can already see above from the numbers and my POC attached with this report, only 100 WETH will be withdrawn from Euler whereas the 0.9 WETH yield will be lost in this case.

## Impact Details

Yield gains earned from Euler WETH vault will not be withdrawable and only our initial assets supplied can be withdrawn. Also, even if we try to somehow withdraw those yield in any other way, the calls will revert since the allocation subtraction will underflow. This would result in the yield gains ultimately not being withdrawable.

For the EulerWETHStrategy vault, we should work with shares rather than assets when withdrawing from Euler. Or rather, we should expose a claim function inside the EulerWETHStrategy contract that overrides the `_claimRewards()` function declared in MYTStrategy.sol as this will allow us claim all rewards based on our available shares balance.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/EulerWETHStrategy.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {EulerWETHStrategy} from "../../strategies/mainnet/EulerWETHStrategy.sol";
import {IAllocator} from "../../interfaces/IAllocator.sol";


contract MockEulerWETHStrategy is EulerWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _weth, address _eulerVault, address _permit2Address)
        EulerWETHStrategy(_myt, _params, _weth, _eulerVault, _permit2Address)
    {}
}

interface EulerVault {
    function balanceOf(address user) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
}

contract EulerWETHStrategyTest is BaseStrategyTest {
    address public constant EULER_WETH_VAULT = 0xD8b27CF359b7D15710a5BE299AF6e7Bf904984C2;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "EulerWETH",
            protocol: "EulerWETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockEulerWETHStrategy(vault, params, WETH, EULER_WETH_VAULT, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_pOCLostYield() public {
        uint256 amountToAllocate = 100e18;
        uint256 amountToDeallocate = amountToAllocate;
        bytes32 ID = IMYTStrategy(strategy).adapterId();

        console.log("CURRENT BLOCK TIME AND NUMBER DURING ALLOC AND DEPOSIT INTO EULER");
        console.log("Block timestamp: ", block.timestamp);
        console.log("Block number: ", block.number);

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, address(vault), amountToAllocate);
        vm.stopPrank();

        vm.startPrank(admin);
        IAllocator(allocator).allocate(address(strategy), amountToAllocate);

        uint256 allocationAmount0 = IVaultV2(vault).allocation(ID);
        console.log("Allocation amount to Euler after allocate: ", allocationAmount0);

        uint256 eulerETHStrategyBalanceAfterAlloc = EulerVault(EULER_WETH_VAULT).balanceOf(address(strategy));
        console.log("Balance in Euler Vault after alloc: ", eulerETHStrategyBalanceAfterAlloc);

        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        console.log("Real assets after allocate: ", initialRealAssets);
        require(initialRealAssets > 0, "Initial real assets is 0");

        vm.warp(1774024463);
        vm.roll(24717302);

        console.log("BLOCK TIME AND NUMBER AFTER 1 YEAR YIELD EARNED");
        console.log("Block timestamp: ", block.timestamp);
        console.log("Block number: ", block.number);

        IAllocator(allocator).deallocate(address(strategy), amountToDeallocate);

        uint256 eulerETHStrategyBalance = EulerVault(EULER_WETH_VAULT).balanceOf(address(strategy));
        console.log("Shares left: ", eulerETHStrategyBalance);
        uint256 sharesConvertedToAssets = EulerVault(EULER_WETH_VAULT).convertToAssets(eulerETHStrategyBalance);
        console.log("Stuck yield gains inside 3rd party euler vault: ", sharesConvertedToAssets);

        uint256 allocationAmount = IVaultV2(vault).allocation(ID);
        console.log("Allocation amount to Euler after deallocate: ", allocationAmount);

        vm.stopPrank();
    }
}
```
