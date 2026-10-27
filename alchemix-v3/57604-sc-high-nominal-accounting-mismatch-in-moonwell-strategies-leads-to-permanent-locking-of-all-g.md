# 57604 sc high nominal accounting mismatch in moonwell strategies leads to permanent locking of all generated yield

**Submitted on Oct 27th 2025 at 14:35:40 UTC by @sus\_bandicoot for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57604
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `MoonwellWETHStrategy` and `MoonwellUSDCStrategy` contracts deposit user assets into Moonwell's yield-bearing `mTokens`. However, the Alchemix V3 system (via the Morpho `VaultV2`) tracks these deposits using a fixed, *nominal* asset amount. When deallocating, the strategies are only able to redeem the original nominal amount of assets, not the full value of their appreciated `mToken` shares. This accounting mismatch means all generated yield becomes permanently trapped in the strategy contracts, resulting in a total loss of earnings for the users who deposited funds into the vault.

## Vulnerability Details

The protocol's yield-generation process involves three main components:

1. **`AlchemistAllocator`**: The contract used by operators to allocate funds.
2. **`VaultV2` (Morpho)**: The vault that holds the assets and tracks nominal allocations per strategy.
3. **Strategy Adapters** (e.g., `MoonwellWETHStrategy`): The contracts that interact with the external yield protocols.

**Allocation Flow:**

1. An operator calls `AlchemistAllocator.allocate(...)` to send, for example, `100 WETH` to the `MoonwellWETHStrategy`.
2. The `AlchemistAllocator` calls `VaultV2.allocate(...)`.
3. The `VaultV2` contract transfers `100 WETH` (which users previously deposited into the vault) to the `MoonwellWETHStrategy`. It then calls `MoonwellWETHStrategy.allocate(...)` and internally records the *nominal* `allocation = 100 WETH` for this strategy.

```solidity
// src/VaultV2.sol
// The `allocateInternal(...)` is called right away by the external `allocate(...)` called by `AlchemistAllocator`.
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
    // ...
    // @audit The vault transfers the assets (e.g., 100 WETH) to the adapter
    SafeERC20Lib.safeTransfer(asset, adapter, assets);
    // @audit It then calls the adapter's allocate function
    (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(
        data, assets, msg.sig, msg.sender
    );
    for (uint256 i; i < ids.length; i++) {
        Caps storage _caps = caps[ids[i]];
        // @audit-issue The vault's internal accounting tracks the *nominal* allocation
        _caps.allocation = (int256(_caps.allocation) + change).toUint256();
        // ...
    }
    // ...
}
```

The `VaultV2` then calls the `allocate(...)` function on the `MoonwellWETHStrategy`, which implements the logic to mint `mTokens`:

```solidity
// src/strategies/optimism/MoonwellWETHStrategy.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    // ...
    TokenUtils.safeApprove(address(weth), address(mWETH), amount);
    // @audit Mints mTokens (e.g., 100 WETH -> 100 mWETH; 1:1 ratio assumed for simplicity)
    mWETH.mint(amount);
    return amount; // @audit This is the `change` that is added to `_caps.allocation` in the snippet above
}
```

The strategy now holds `100 mWETH` (assuming a `1:1` minting ratio for simplicity).

**Deallocation Flow:** Over time, the `100 mWETH` held by the strategy appreciate in value due to accruing yield. Let's say the exchange rate doubles (a `1:2` ratio), and those `100 mWETH` are now worth `200 WETH`.

1. The `VaultV2`'s recorded `allocation` for the strategy is still the *nominal* `100 WETH` (set during the allocation flow).
2. An operator wishing to withdraw must call `AlchemistAllocator.deallocate(...)`. The problem is that they can only deallocate up to the *nominal* amount, `100 WETH`. Attempting to deallocate anything above the *nominal* `100 WETH`, e.g. the full `200 WETH` (the strategy's real value) would fail, as the system's accounting logic would underflow. The following steps demonstrate this process.
3. The operator calls `AlchemistAllocator.deallocate(adapter, 100 WETH)`. This function first fetches the *current* nominal allocation from the vault and packs it as `bytes memory oldAllocation`.

```solidity
// src/AlchemistAllocator.sol
function deallocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    // ...
    // @audit The current nominal allocation (100 WETH) is fetched from the vault
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    // @audit The 'amount' (100 WETH) and 'oldAllocation' (100 WETH)
    // are passed to the vault
    vault.deallocate(adapter, oldAllocation, amount);
}
```

4. The `AlchemistAllocator` then calls `vault.deallocate(adapter, oldAllocation, 100 WETH)`, where `oldAllocation` is the encoded `100 WETH` and `amount` (the `assets` parameter) is also `100 WETH`.

The `VaultV2` receives this call, calls the adapter's `deallocate` function, and prepares to update its internal nominal `allocation` based on the `change` returned.

```solidity
// src/VaultV2.sol
// The `deallocateInternal(...)` is called right away by the external `deallocate(...)` called by `AlchemistAllocator`.
function deallocateInternal(address adapter, bytes memory data, uint256 assets)
    internal
    returns (bytes32[] memory)
{
    require(isAdapter[adapter], ErrorsLib.NotAdapter());

    // @audit Calls the adapter's (MYTStrategy's) deallocate function
    // - 'data' contains the encoded 'oldAllocation' (100 WETH)
    // - 'assets' is the 'amount' to deallocate (100 WETH)
    (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(
        data, assets, msg.sig, msg.sender
    );

    for (uint256 i; i < ids.length; i++) {
        Caps storage _caps = caps[ids[i]];
        require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
        // @audit Updates the nominal allocation. Note that in case of deallocation, the `change` CAN be negative (its `int256`).
        _caps.allocation = (int256(_caps.allocation) + change).toUint256();
    }

    // @audit Pulls the redeemed assets back from the adapter
    SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), assets);
    // ...
    return ids;
}
```

The vault's call is received by the `MYTStrategy.deallocate(...)` function, which decodes the `oldAllocation` (from `data`) and calls the internal `_deallocate(...)` function.

This is the place that would cause the transaction to revert if the operator attempted to withdraw more than the initial *nominal* allocation of assets. The underflow in the subtraction blocks the deallocation of the strategy earnings.

```solidity
// src/MYTStrategy.sol
function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
    // ...
    // @audit 'assets' is 100 WETH
    require(assets > 0, "Zero amount");
    // @audit The old nominal allocation (100 WETH) is decoded from 'data'
    uint256 oldAllocation = abi.decode(data, (uint256));

    // @audit Calls the strategy-specific _deallocate implementation
    // @audit 'assets' (100 WETH) is passed
    uint256 amountDeallocated = _deallocate(assets); // returns 100 WETH

    // @audit-issue This calculation is the root of the problem.
    // 'amountDeallocated' is equal to the 'assets' input, which is controlled by the operator's call to AlchemistAllocator.
    // If 'assets' were > 'oldAllocation' (e.g., 200 WETH), this would underflow.
    uint256 newAllocation = oldAllocation - amountDeallocated; // @audit 100 - 100 = 0
    emit Deallocate(amountDeallocated, address(this));
    // @audit Returns change = 0 - 100 = -100
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

Finally, the `MoonwellWETHStrategy._deallocate(...)` function executes the redemption logic. As can be seen, the return value of a call to `_deallocate(...)` in the snippet above, is simply the input argument to the function (shown below).

```solidity
// src/strategies/optimism/MoonwellWETHStrategy.sol
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 ethBalanceBefore = address(this).balance;

    // @audit-issue This redeems a fixed *underlying amount* (100 WETH).
    // It does *not* redeem all shares. If 100 mWETH are worth 200 WETH (1:2 ratio), 
    // this call redeems 100 WETH by burning only 50 mWETH.
    mWETH.redeemUnderlying(amount);

    // ... logic to wrap ETH back to WETH ...
    uint256 ethBalanceAfter = address(this).balance;
    // ...
    uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
    if (ethRedeemed + ethBalanceBefore >= amount) {
        weth.deposit{value: ethRedeemed}();
    }
    // ...
    // @audit Approves the 100 WETH back to the VaultV2 (msg.sender)
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount; // returns 100 WETH
}
```

The call to `mWETH.redeemUnderlying(100 WETH)` successfully redeems `100 WETH` by burning only a *portion* of the strategy's `mWETH` (e.g., `50 mWETH`, given our `1:2` ratio). The `100 WETH` are returned to the `VaultV2`.

The `VaultV2`'s `allocation` for this strategy is now updated to `0` (`100` + `(-100)`).

The remaining `50 mWETH` (worth the other `100 WETH`), which represent the entire yield, are now permanently locked in the `MoonwellWETHStrategy` contract. Since the `VaultV2`'s allocation is `0`, no further `deallocate` calls can be made to retrieve them. There is no other mechanism to withdraw these remaining shares.

## Impact Details

The primary and direct impact is the **permanent and total loss of all yield** generated by the `MoonwellWETHStrategy` and `MoonwellUSDCStrategy`.

While users can recover their principal (the initial nominal deposit)\*, the entire purpose of these strategies is defeated, as all profits are irrecoverably locked. This renders these two strategies unprofitable by design, causing a direct loss of all earned yield for the vault's depositors. This guaranteed outcome aligns with a **High-severity** rating.

\*A secondary impact stems from the `realAssets()` view function, which will report a misleadingly high value.

```solidity
// src/strategies/optimism/MoonwellWETHStrategy.sol
function realAssets() external view override returns (uint256) {
    // @audit This balance includes the trapped yield.
    uint256 mTokenBalance = mWETH.balanceOf(address(this));
    if (mTokenBalance == 0) return 0;
    uint256 exchangeRate = mWETH.exchangeRateStored();
    // @audit This returns the full value (e.g., 200 WETH),
    // even when only 100 WETH is recoverable.
    return (mTokenBalance * exchangeRate) / 1e18;
}
```

This function calculates the value of *all* `mTokens` held, including the trapped, irrecoverable yield. This inflates the perceived value of the strategy and the vault's overall share price, misleading operators and users about the system's true performance.

Furthermore, this inflated share price creates the *potential* for a more severe, Critical-severity scenario. The first users to redeem their shares would be paid out at an artificially high price, drawing from the vault's *actual* recoverable principal. This comes at the direct expense of the last depositors, who would be left holding worthless shares backed only by the trapped yield.

This report focuses on the guaranteed **High-severity** impact (the permanent loss of yield), as the Critical-severity bank run scenario is conditional on a specific mass-exit event and is of lower likelihood.

Since `MoonwellWETHStrategy` and `MoonwellUSDCStrategy` are identical except for the underlying asset, the same issue is present in both contracts.

## References

* `AlchemistAllocator`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* `MYTStrategy`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* `MoonwellUSDCStrategy`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* `MoonwellWETHStrategy`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* `VaultV2` (Morpho): <https://github.com/morpho-org/vault-v2/blob/main/src/VaultV2.sol>
* `mWETH`: <https://optimistic.etherscan.io/address/0xb4104C02BBf4E9be85AAa41a62974E4e28D59A33#code>

## Proof of Concept

## Proof of Concept

The test output demonstrates the bug. The logs show that after allocating 1000 WETH and letting yield accrue (20 days), the strategy's underlying value increased to 1000.32 WETH.

After the operator deallocates the original 1000 WETH, the vault's internal `allocation` for the strategy drops to `0`. The remaining `0.32 WETH` (the yield) is still in the strategy, as shown by the "Remaining trapped yield" log.

The final step proves these funds are locked. The attempt to withdraw this remaining yield reverts, as the `VaultV2` contract (and `MYTStrategy`) no longer permits deallocation from a strategy it considers to have a zero balance.

**PoC Instructions**:

1. The test contract can be found below. Copy it and paste it into: `src/test/strategies/MoonwellWETHStrategy.t.sol`
2. Run the test with the following command:

```bash
forge test --mt test_PoC_moonwellRewardsAreLocked --mc MoonwellWETHStrategyTest -vvv
```

**Logs:**

```shell
Ran 1 test for src/test/strategies/MoonwellWETHStrategy.t.sol:MoonwellWETHStrategyTest
[PASS] test_PoC_moonwellRewardsAreLocked() (gas: 806786)
Logs:
  --- 1. Allocating nominal assets ---
  Nominal allocation in Vault  : 1000.000000000000000000
  Actual underlying value      : 999.999999999948732595

--- 2. Warping time to accrue yield ---
  Accrued yield over 20 days   : 0.320503066606509794
  Total underlying value now   : 1000.320503066555242389

--- 3. Deallocating nominal assets ---
  Remaining trapped yield      : 0.320503066690078559
  Misleading realAssets() val  : 0.320503066690078559
  Vault's nominal allocation   : 0.000000000000000000

--- 4. Proving funds are locked ---
  Attempting to withdraw the trapped yield... This MUST revert with an arithmetic underflow.
  The call reverted in MYTStrategy.deallocate(...) with underflow as expected.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 265.84ms (2.40ms CPU time)
```

**Test file**: `src/test/strategies/MoonwellWETHStrategy.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MoonwellWETHStrategy} from "../../strategies/optimism/MoonwellWETHStrategy.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";

contract MockMoonwellWETHStrategy is MoonwellWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _mWETH, address _weth, address _permit2Address)
        MoonwellWETHStrategy(_myt, _params, _mWETH, _weth, _permit2Address)
    {}
}

contract MoonwellWETHStrategyTest is BaseStrategyTest {
    address public constant MOONWELL_WETH_MTOKEN = 0xb4104C02BBf4E9be85AAa41a62974E4e28D59A33;
    address public constant WETH = 0x4200000000000000000000000000000000000006;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MoonwellWETH",
            protocol: "MoonwellWETH",
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
        return address(new MockMoonwellWETHStrategy(vault, params, MOONWELL_WETH_MTOKEN, WETH, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698; // Sep-28-2025
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

    function test_PoC_moonwellRewardsAreLocked() public {
        IMoonwellWETH mToken = IMoonwellWETH(MOONWELL_WETH_MTOKEN);
        // --- Setup Interfaces ---
        // Cast strategy and vault to their interfaces to check their state
        IMYTStrategy mytStrategy = IMYTStrategy(strategy);
        IVaultv2 morphoVault = IVaultv2(vault);
        bytes32 strategyId = mytStrategy.adapterId();
        
        uint256 allocateAmount = 1000e18; // 1000 WETH

        assertEq(mToken.balanceOf(strategy), 0, "Strategy should have 0 mWETH tokens before allocation.");

        // --- 1. ALLOCATION ---
        emit log("--- 1. Allocating nominal assets ---");
        vm.prank(operator);
        IAlchemistAllocator(allocator).allocate(address(strategy), allocateAmount);

        uint256 shareBalanceBeforeWaiting = mToken.balanceOf(strategy);
        uint256 underlyingBalanceBeforeWaiting = mToken.balanceOfUnderlying(strategy);
        uint256 nominalAllocation = morphoVault.allocation(strategyId);

        emit log_named_decimal_uint("Nominal allocation in Vault  ", nominalAllocation, 18);
        emit log_named_decimal_uint("Actual underlying value      ", underlyingBalanceBeforeWaiting, 18);
        
        assertGt(shareBalanceBeforeWaiting, 0, "Strategy should have received mWETH tokens after allocation.");
        assertEq(nominalAllocation, allocateAmount, "Vault's allocation should match the nominal amount allocated.");
        assertApproxEqAbs(underlyingBalanceBeforeWaiting, allocateAmount, 1e8, "Underlying balance should be close to nominal.");


        // --- 2. ACCRUE YIELD ---
        emit log("\n--- 2. Warping time to accrue yield ---");
        vm.warp(block.timestamp + 20 days);

        uint256 underlyingBalanceAfterWaiting = mToken.balanceOfUnderlying(strategy);
        uint256 assetIncrease = underlyingBalanceAfterWaiting - underlyingBalanceBeforeWaiting;
        emit log_named_decimal_uint("Accrued yield over 20 days   ", assetIncrease, 18);
        emit log_named_decimal_uint("Total underlying value now   ", underlyingBalanceAfterWaiting, 18);

        assertGt(underlyingBalanceAfterWaiting, underlyingBalanceBeforeWaiting, "mWETH shares should increase in value over time.");


        // --- 3. DEALLOCATION ---
        emit log("\n--- 3. Deallocating nominal assets ---");
        vm.prank(operator);
        IAlchemistAllocator(allocator).deallocate(address(strategy), allocateAmount);

        uint256 shareBalanceAfterDeallocation = mToken.balanceOf(strategy);
        uint256 underlyingBalanceAfterDeallocation = mToken.balanceOfUnderlying(strategy);
        uint256 nominalAllocationAfter = morphoVault.allocation(strategyId);
        uint256 realAssetsAfter = mytStrategy.realAssets();

        emit log_named_decimal_uint("Remaining trapped yield      ", underlyingBalanceAfterDeallocation, 18);
        emit log_named_decimal_uint("Misleading realAssets() val  ", realAssetsAfter, 18);
        emit log_named_decimal_uint("Vault's nominal allocation   ", nominalAllocationAfter, 18);

        // --- 4. VERIFY LOCKED STATE ---
        assertGt(shareBalanceAfterDeallocation, 0, "Bug: Strategy should have remaining shares (yield) which cannot be withdrawn.");
        assertEq(nominalAllocationAfter, 0, "Bug: Vault's nominal allocation is now 0, preventing further withdrawals.");
        assertApproxEqAbs(underlyingBalanceAfterDeallocation, assetIncrease, 1e8, "Assets locked in contract should equal the yield.");
        assertApproxEqAbs(realAssetsAfter, assetIncrease, 1e8, "realAssets() should misleadingly report the trapped yield.");

        // --- 5. PROVE PERMANENT LOCK ---
        // Any attempt to withdraw the remaining yield will fail with an arithmetic underflow
        // in MYTStrategy.deallocate() because oldAllocation (fetched from vault.allocation(id)) is 0.
        // uint256 newAllocation = oldAllocation - amountDeallocated; // REVERTS: 0 - assetIncrease
        emit log("\n--- 4. Proving funds are locked ---");
        emit log("Attempting to withdraw the trapped yield... This MUST revert with an arithmetic underflow.");
        
        vm.expectRevert(stdError.arithmeticError); // Underflow 0x11 error in MYTStrategy.deallocate(...)
        vm.prank(operator);
        IAlchemistAllocator(allocator).deallocate(address(strategy), underlyingBalanceAfterDeallocation);

        emit log("The call reverted in MYTStrategy.deallocate(...) with underflow as expected.");
    }   
}

interface IAlchemistAllocator {
    function allocate(address, uint256) external;
    function deallocate(address, uint256) external;
}

interface IMoonwellWETH {
    function balanceOf(address) external returns (uint256);
    function balanceOfUnderlying(address) external returns (uint256);
}

// Minimal interface for Morpho's VaultV2
interface IVaultv2 {
    function allocation(bytes32 id) external view returns (uint256);
}
```
