# 57930 sc high allocation tracking underflow in strategy deallocation leads to protocol insolvency

**Submitted on Oct 29th 2025 at 13:54:17 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57930
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `MYTStrategy::deallocate()` function contains an arithmetic underflow vulnerability when attempting to deallocate amounts that include accrued rewards. This occurs because the vault's allocation tracking is stale and doesn't account for yield gains, causing the deallocation calculation to underflow when trying to subtract a larger amount (principal + rewards) from a smaller tracked allocation (principal only).

## Vulnerability Details

The root cause lies in the allocation tracking mechanism between `VaultV2` and `MYTStrategy`. When `AlchemistAllocator::deallocate()` is called, it passes the current vault allocation as data to the strategy:

```solidity
// AlchemistAllocator.sol
    function deallocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
            adjusted = adjusted < daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
@>      bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.deallocate(adapter, oldAllocation, amount);
    }
```

However, as documented in [VaultV2.sol line 47](https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L47): "The allocation is not always up to date, because interest and losses are accounted only when (de)allocating in the corresponding markets."

The vulnerability manifests in `MYTStrategy::deallocate()`:

```solidity
    function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));
        }
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountDeallocated = _deallocate(assets);
@>      uint256 newAllocation = oldAllocation - amountDeallocated;
        emit Deallocate(amountDeallocated, address(this));
        return (ids(), int256(newAllocation) - int256(oldAllocation));
    }
```

When a strategy accrues rewards:

1. The strategy's `realAssets()` increases to reflect the new total value (principal + rewards)
2. The vault's `allocation(id)` remains at the original principal amount (stale)
3. When attempting to deallocate the full balance including rewards, `oldAllocation` (1M) - `amountDeallocated` (1.1M with rewards) causes an underflow

With the increase of the value returned from the strategy's `realAssets()` function, the VaultV2 `totalAssets()` increases, which also increase the price of the shares. So the users can withdraw more, as yield accrue.

## Impact Details

Once rewards accrue beyond the tracked allocation, they become permanently locked in the strategy. No deallocation operation can extract the full balance without reverting.

The locked yield is still counted in the vault's `totalAssets()` calculation (via strategy's `realAssets()`), so users believe they can withdraw it. However, when users attempt to exit, the vault cannot retrieve the locked funds, making the protocol effectively insolvent.

## References

VaultV2: <https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L75-L76>

Preview deposit and redeem in VaultV2, which use total assets that include the yield from the strategies: <https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L686-L690> <https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L700-L704>

## Proof of Concept

## Proof of Concept

In order to successfully show the issue, first a problem in the mocks should be addressed - the `MockMTYStrategy::_deallocate()` passes the amount to `MockYieldToken::requestWithdraw()`, who treats it as shares. So apply the following fix to the MockYieldToken:

```solidity
 function requestWithdraw(address recipient, uint256 amount) external returns (uint256) {
        assert(amount > 0);

        // Convert asset amount to shares needed
        uint256 shares = _assetsToShares(amount);

        // Calculate actual value withdrawn (accounting for slippage)
        uint256 value = (amount * (BPS - slippage)) / BPS;

        _burn(msg.sender, shares);
        TokenUtils.safeTransfer(underlyingToken, recipient, value);
        return value;
    }

    function _assetsToShares(uint256 assets) internal view returns (uint256) {
        uint256 supply = mockTokenSupply();
        if (supply == 0) {
            return assets;
        }
        return (assets * supply) / TokenUtils.safeBalanceOf(underlyingToken, address(this));
    }
```

The PoC:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import "../../lib/forge-std/src/Test.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

contract PoC_DeallocationWithRewards is Test {
    VaultV2 vault;
    MockAlchemistAllocator allocator;
    MockMYTStrategy mytStrategy;
    address public operator = address(20);
    address public admin = address(21);
    address public curator = address(22);
    address public usdc;
    address public mockStrategyYieldToken;
    uint256 public defaultStrategyAbsoluteCap = 2_000_000_000e18;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%

    function setUp() external {
        // Create mock USDC with 6 decimals
        TestERC20 mockUSDC = new TestERC20(1_000_000_000e6, 6);
        usdc = address(mockUSDC);

        vm.startPrank(admin);
        mockStrategyYieldToken = address(new MockYieldToken(usdc));
        vault = MYTTestHelper._setupVault(usdc, admin, curator);
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

    function _magicDepositToVault(address vault_, address depositor, uint256 amount) internal returns (uint256) {
        deal(usdc, address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(usdc, vault_, amount);
        uint256 shares = IVaultV2(vault_).deposit(amount, depositor);
        vm.stopPrank();
        return shares;
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }

    function test_PoC_DeallocateWithRewards_CausesUnderflow() external {
        // Setup: Deposit funds to vault
        address user1 = address(0xbeef);
        uint256 depositAmount = 1_000_000e6;
        _magicDepositToVault(address(vault), user1, depositAmount);

        // Allocate all funds to strategy
        vm.startPrank(admin);
        uint256 totalAssets = vault.convertToAssets(vault.totalSupply());
        allocator.allocate(address(mytStrategy), totalAssets);
        vm.stopPrank();

        // Verify allocation - strategy now holds 1M USDC worth of shares
        uint256 strategyBalance = mytStrategy.realAssets();
        assertEq(strategyBalance, depositAmount, "Strategy should have all deposited funds");

        // Check initial state
        uint256 sharesHeld = IERC20(mockStrategyYieldToken).balanceOf(address(mytStrategy));
        console.log("Initial shares held by strategy:", sharesHeld);
        console.log("Initial strategy real assets:", strategyBalance);

        // Simulate rewards accrual: Add 10% more underlying tokens to the yield token
        // This increases the price/value of the shares held by the strategy without changing share count
        uint256 rewardsAmount = 100_000e6; // 10% rewards
        deal(usdc, mockStrategyYieldToken, IERC20(usdc).balanceOf(mockStrategyYieldToken) + rewardsAmount);

        // Check new balance with rewards - realAssets is now 1.1M USDC
        uint256 strategyBalanceAfterRewards = mytStrategy.realAssets();
        uint256 sharesHeldAfterRewards = IERC20(mockStrategyYieldToken).balanceOf(address(mytStrategy));

        console.log("After rewards - shares held by strategy:", sharesHeldAfterRewards);
        console.log("After rewards - strategy real assets:", strategyBalanceAfterRewards);

        assertGt(strategyBalanceAfterRewards, depositAmount, "Strategy should have more than initial deposit");
        assertEq(strategyBalanceAfterRewards, 1_100_000e6, "Strategy should have 1.1M USDC");
        assertEq(sharesHeld, sharesHeldAfterRewards, "Share count should remain the same");

        // The REAL ISSUE: Track the vault's allocation before and after
        bytes32 strategyId = mytStrategy.adapterId();
        uint256 allocationBefore = vault.allocation(strategyId);
        console.log("Vault allocation before deallocate:", allocationBefore);

        // Now deallocate - this should work with corrected requestWithdraw
        // But check what happens to the allocation tracking
        uint256 amountToDeallocate = strategyBalanceAfterRewards;

        vm.startPrank(admin);
        vm.expectRevert(stdError.arithmeticError); // Expected: arithmetic underflow
        allocator.deallocate(address(mytStrategy), amountToDeallocate);
        vm.stopPrank();

        // The PROBLEM demonstrated:
        // allocationBefore = 1M (oldAllocation from vault.allocation())
        // amountToDeallocated = 1.1M (includes 100k gains)
        // In MYTStrategy.deallocate() line 130:
        //   uint256 newAllocation = oldAllocation - amountDeallocated;
        //   uint256 newAllocation = 1M - 1.1M  →  UNDERFLOW!
    }

    function test_PoC_UserCannotWithdrawAfterMaxDeallocation() external {
        // Setup: Set maxRate and deposit 1M USDC
        vm.prank(address(allocator));
        vault.setMaxRate(200e16 / uint256(365 days)); // 200% APR

        address user = address(0xbeef);
        uint256 depositAmount = 1_000_000e6;
        uint256 userShares = _magicDepositToVault(address(vault), user, depositAmount);

        // Allocate all funds to strategy
        vm.startPrank(admin);
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();

        // Simulate 10% yield accrual in the strategy
        uint256 rewardsAmount = 100_000e6;
        deal(usdc, mockStrategyYieldToken, IERC20(usdc).balanceOf(mockStrategyYieldToken) + rewardsAmount);

        // Skip time and accrue interest so vault recognizes gains
        vm.warp(block.timestamp + 19 days);
        vault.accrueInterest();

        // Verify strategy has the yield
        uint256 strategyBalanceAfterRewards = mytStrategy.realAssets();
        assertEq(strategyBalanceAfterRewards, 1_100_000e6, "Strategy should have 1.1M USDC");

        // Allocator deallocates only the principal (1M), not the gains (100k)
        vm.startPrank(admin);
        allocator.deallocate(address(mytStrategy), depositAmount);
        vm.stopPrank();

        // The 100k in gains remain stuck in the strategy
        uint256 remainingInStrategy = mytStrategy.realAssets();
        assertGt(remainingInStrategy, 0, "Yield should remain in strategy");

        uint256 userClaimableBalance = vault.previewRedeem(userShares);
        console.log("User claimable balance according to previewRedeem:", userClaimableBalance);

        // User redeems their shares but only receives the principal
        vm.startPrank(user);
        vm.expectRevert();
        vault.redeem(userShares, user, user);
        vm.stopPrank();

        // User loses the yield - it's permanently stuck in the strategy
    }
}
```

In order to run the two functions use:

```bash
forge test --match-test test_PoC_DeallocateWithRewards_CausesUnderflow -vv

forge test --isolate --match-test test_PoC_UserCannotWithdrawAfterMaxDeallocation -vv
```

\*Note `--isolate` is required for the second one, because VaultV2 uses transient storage.
