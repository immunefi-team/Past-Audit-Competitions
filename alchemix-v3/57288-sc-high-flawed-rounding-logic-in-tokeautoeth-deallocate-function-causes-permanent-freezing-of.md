# 57288 sc high flawed rounding logic in tokeautoeth deallocate function causes permanent freezing of funds

**Submitted on Oct 25th 2025 at 01:13:10 UTC by @terrah for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57288
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Bug Description

The \_deallocate function in TokeAutoEth.sol contains a flawed rounding condition that withdraws all shares from the rewarder when the difference between actual shares held and shares needed is less than or equal to 1 share (1e18). This causes the function to withdraw significantly more shares than requested, approve only the requested amount to the vault, and strand the excess WETH in the strategy contract. The stranded funds become permanently inaccessible because subsequent deallocate calls will underflow and revert.

## Brief/Intro

When deallocating more than approximately 99% of the strategy's position, the TokeAutoEth strategy withdraws all shares from the rewarder instead of only the amount needed. The function returns the requested amount to the vault for accounting purposes but actually withdraws more, leaving excess WETH stranded in the strategy contract. This stranded amount cannot be recovered through any available mechanism in the codebase, resulting in permanent freezing of funds. Additionally, the accounting mismatch causes all subsequent deallocate attempts to revert on underflow.

## Details

The vulnerable code is in TokeAutoEth.sol \_deallocate function:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
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

The problem occurs when shareDiff is less than or equal to 1e18 (one full share). In this case, the function sets sharesNeeded to actualSharesHeld, withdrawing all shares. However, the function only approves the originally requested amount to the vault and returns the requested amount for accounting. This creates a critical discrepancy.

Consider a scenario where the strategy holds 100e18 shares worth approximately 100 WETH. When the vault calls deallocate with 99.5 WETH:

The function calculates sharesNeeded as 99.5e18 shares. The actualSharesHeld is 100e18 shares. The shareDiff becomes 0.5e18, which is less than 1e18. The condition triggers and sharesNeeded is set to 100e18. The function withdraws all 100e18 shares from the rewarder and redeems them for approximately 100 WETH. However, it only approves 99.5 WETH to the vault and returns 99.5. The vault's accounting records a deallocation of 99.5 WETH, believing 0.5 WETH worth of allocation remains.

After this operation, the strategy has zero shares in the rewarder but approximately 0.5 WETH sitting as an ERC20 balance in the contract. The vault's internal accounting believes the strategy still has 0.5 WETH allocated.

When any subsequent deallocate call attempts to withdraw this remaining 0.5 WETH allocation, the function will calculate sharesNeeded as approximately 0.5e18, but actualSharesHeld will be 0. The line shareDiff = actualSharesHeld - sharesNeeded will underflow in Solidity 0.8.28, causing the transaction to revert. This makes the deallocate function permanently unusable.

The stranded WETH cannot be recovered through any mechanism. The MYTStrategy contract has no sweep or rescue function. The removeStrategy function in AlchemistCurator only affects the vault's adapter registry and does not transfer tokens. The owner has no special token recovery powers. The funds remain permanently locked in the strategy contract.

The same bug exists in TokeAutoUSDStrategy.sol with identical logic.

## Impact

This vulnerability causes permanent freezing of funds. Each time a deallocation of more than approximately 99% of the strategy's position occurs, a portion of funds (the remainder) becomes permanently inaccessible. The stranded amount ranges from a fraction of a share up to nearly one full share worth of value.

The funds cannot be recovered by the admin, the vault, or any other party. There is no sweep function, no emergency withdrawal mechanism, and no way to force the strategy to release the stranded tokens. The deallocate function becomes permanently broken after the first trigger, preventing any future rebalancing or withdrawals from that strategy.

Users who have deposited into the vault expecting to be able to withdraw their funds will find that the strategy cannot deallocate to fulfill withdrawal requests. The vault must route all future activity through other strategies, reducing overall capital efficiency and potentially causing liquidity issues during periods of high withdrawal demand.

## Risk Breakdown

Difficulty to Exploit: Low

The vulnerability triggers automatically during normal protocol operations. No special privileges, external dependencies, or complex transaction sequences are required. Any call to deallocate that requests more than approximately 99% of the strategy's current position will trigger the bug. Vault operators, allocators, and admins performing routine rebalancing operations will inadvertently cause this issue.

The bug is deterministic and does not depend on market conditions, oracle prices, or external protocol states. It will occur every time the specific condition is met.

## Recommendation

Remove the flawed rounding logic entirely. The condition checking if shareDiff is less than or equal to 1e18 serves no valid purpose and causes severe accounting issues. Replace the vulnerable section with proper bounds checking:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    
    if (sharesNeeded > actualSharesHeld) {
        sharesNeeded = actualSharesHeld;
    }
    
    rewarder.withdraw(address(this), sharesNeeded, true);
    // ... rest of function
}
```

This ensures the function only withdraws available shares and maintains correct accounting. If dust avoidance is truly necessary, use a much smaller threshold like 1e12 (representing 0.000001 shares or approximately $0.003 worth of value), and ensure the return value matches the actual amount withdrawn.

Apply the same fix to TokeAutoUSDStrategy.sol which contains identical vulnerable logic.

## References

Vulnerable file: TokeAutoEth.sol, function \_deallocate Duplicate vulnerable file: TokeAutoUSDStrategy.sol, function \_deallocate

For comparison, other strategy contracts like EulerARBWETHStrategy.sol, AaveV3ARBWETHStrategy.sol, and MorphoYearnOGWETHStrategy.sol do not implement this flawed logic and function correctly.

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {TokeAutoEthStrategy} from "../../src/strategies/mainnet/TokeAutoEth.sol";
import {IMYTStrategy} from "../../src/interfaces/IMYTStrategy.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MockAlchemistAllocator} from "../../src/test/mocks/MockAlchemistAllocator.sol";
import {MYTTestHelper} from "../../src/test/libraries/MYTTestHelper.sol";
import {TokenUtils} from "../../src/libraries/TokenUtils.sol";
import {IMainRewarder} from "../../src/strategies/interfaces/ITokemac.sol";

interface IERC4626Like {
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
}

interface IWETH {
    function balanceOf(address) external view returns (uint256);
}

contract PocDeallocateRoundingBugTest is Test {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    address public strategy;
    address public vault;
    address public allocator;
    address public admin = address(1);
    address public curator = address(2);
    address public operator = address(3);
    address public vaultDepositor = address(4);
    uint256 private _forkId;

    function setUp() public {
        string memory rpc = vm.envString("MAINNET_RPC_URL");
        _forkId = vm.createFork(rpc, 22_089_302);
        vm.selectFork(_forkId);

        vm.startPrank(admin);
        vault = address(MYTTestHelper._setupVault(WETH, admin, curator));

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: admin,
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strategy = address(new TokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
        vm.stopPrank();

        _setUpVaultAndStrategy(vault, strategy, 10_000e18, 1e18);
        _magicDepositToVault(vault, vaultDepositor, 1000e18);

        vm.makePersistent(strategy);
    }

    function _setUpVaultAndStrategy(address _vault, address _strategy, uint256 absoluteCap, uint256 relativeCap) internal {
        vm.startPrank(admin);
        allocator = address(new MockAlchemistAllocator(_vault, admin, operator));
        vm.stopPrank();

        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (allocator, true)));
        IVaultV2(_vault).setIsAllocator(allocator, true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, _strategy));
        IVaultV2(_vault).addAdapter(_strategy);

        bytes memory idData = IMYTStrategy(_strategy).getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, absoluteCap)));
        IVaultV2(_vault).increaseAbsoluteCap(idData, absoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, relativeCap)));
        IVaultV2(_vault).increaseRelativeCap(idData, relativeCap);
        vm.stopPrank();
    }

    function _magicDepositToVault(address _vault, address depositor, uint256 amount) internal {
        deal(WETH, depositor, amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(WETH, _vault, amount);
        IVaultV2(_vault).deposit(amount, depositor);
        vm.stopPrank();
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        IVaultV2(vault).submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + IVaultV2(vault).timelock(selector));
    }

    function testDeallocateRoundingBugFreezesFunds() external {
        uint256 allocateAmount = 100e18;
        uint256 deallocateAmount = 99.5e18;

        // Step 1-2: Allocate 100 WETH to the strategy via vault
        vm.startPrank(allocator);
        bytes memory prevAllocationAmount = abi.encode(0);
        IVaultV2(vault).allocate(strategy, prevAllocationAmount, allocateAmount);
        vm.stopPrank();

        // Step 3: Verify the strategy holds 100e18 shares in the rewarder
        uint256 sharesAfterAllocation = IMainRewarder(REWARDER).balanceOf(strategy);
        uint256 expectedShares = IERC4626Like(TOKE_AUTO_ETH_VAULT).convertToShares(allocateAmount);
        console.log("Shares after allocation:", sharesAfterAllocation);
        assertApproxEqAbs(sharesAfterAllocation, expectedShares, 1e18, "Initial shares mismatch");

        // Capture vault allocation before deallocate
        bytes32 strategyId = IMYTStrategy(strategy).adapterId();
        uint256 vaultAllocationBefore = IVaultV2(vault).allocation(strategyId);

        // Step 4: Call deallocate with 99.5 WETH
        console.log("\n=== DEALLOCATING 99.5 WETH ===");
        vm.startPrank(allocator);
        bytes memory prevAllocationAmount2 = abi.encode(vaultAllocationBefore);
        IVaultV2(vault).deallocate(strategy, prevAllocationAmount2, deallocateAmount);
        vm.stopPrank();

        // Step 5: Check strategy's share balance in rewarder - should be 0 instead of ~0.5e18
        uint256 sharesAfterDeallocate = IMainRewarder(REWARDER).balanceOf(strategy);
        uint256 expectedRemainingShares = IERC4626Like(TOKE_AUTO_ETH_VAULT).convertToShares(allocateAmount - deallocateAmount);

        console.log("Shares after deallocate:", sharesAfterDeallocate);
        console.log("Expected remaining shares:", expectedRemainingShares);

        // PROOF 1: All shares withdrawn (0) instead of leaving ~0.5e18
        assertEq(sharesAfterDeallocate, 0, "Shares should be 0 due to bug");
        assert(expectedRemainingShares > 0);

        // Step 6: Check strategy's WETH balance - should show stranded WETH
        uint256 strandedWeth = IWETH(WETH).balanceOf(strategy);

        console.log("Stranded WETH in strategy:", strandedWeth);

        // PROOF 2: WETH is stranded in strategy contract
        assertGt(strandedWeth, 0, "No WETH stranded");
        assertApproxEqAbs(strandedWeth, allocateAmount - deallocateAmount, 0.1e18, "Stranded amount mismatch");

        // Step 7: Check vault's allocation - should still show ~0.5 WETH allocated
        uint256 vaultAllocationAfter = IVaultV2(vault).allocation(strategyId);

        console.log("Vault allocation after deallocate:", vaultAllocationAfter);

        // PROOF 3: Vault still thinks there's allocation but rewarder has 0 shares
        assertGt(vaultAllocationAfter, 0, "Vault allocation should be > 0");
        assertApproxEqAbs(vaultAllocationAfter, allocateAmount - deallocateAmount, 0.1e18, "Vault allocation mismatch");

        // Step 8: Attempt to deallocate the remaining amount - should revert with underflow
        console.log("\n=== ATTEMPTING SECOND DEALLOCATE ===");
        uint256 remainingAllocation = vaultAllocationAfter;

        vm.startPrank(allocator);
        bytes memory prevAllocationAmount3 = abi.encode(remainingAllocation);
        vm.expectRevert();
        IVaultV2(vault).deallocate(strategy, prevAllocationAmount3, remainingAllocation);
        vm.stopPrank();

        console.log("Second deallocate reverted as expected - funds are permanently frozen");

        // PROOF 4: Subsequent deallocations are blocked - funds permanently frozen
        // The revert happens because actualSharesHeld (0) - sharesNeeded (0.5e18) underflows
    }
}
```
