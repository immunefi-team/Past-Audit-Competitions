# 58313 sc medium incorrect allocation accounting and dust handling in stargateethpoolstrategy causes systematic loss cap mis accounting and deallocation reverts

**Submitted on Nov 1st 2025 at 07:25:28 UTC by @IronsideSec for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58313
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`StargateEthPoolStrategy._allocate` returns the requested amount instead of the actually deposited amount (floored to 1e12), creating an accounting mismatch with the vault. This over-reports allocation, can prematurely trip caps, and leaves ETH “dust” stranded on the strategy that is not reflected in `realAssets()`. Over time, this leads to systematic realized loss when interest/loss is accrued, and can also make deallocation revert due to insufficient wrapped WETH. Sub-1e12 allocations deposit 0 by design, causing a 100% loss of those amounts.

## Vulnerability Details

* Root cause: returning `amount` instead of `amountToDeposit` from `_allocate` after rounding down to the nearest 1e12.

```solidity
47:58:src/strategies/optimism/StargateEthPoolStrategy.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "not enough WETH");
    // unwrap to native ETH for Pool Native
    weth.withdraw(amount);
    uint256 amountToDeposit = (amount / 1e12) * 1e12;
    uint256 dust = amount - amountToDeposit;
    if (dust > 0) {
        emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);
    }
    pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
    return amount;
}
```

* Why this is harmful:
  * The vault uses the strategy’s returned “change” to update the allocation and enforce caps; returning the wrong value overstates allocation.

```solidity
574:587:lib/vault-v2/src/VaultV2.sol
SafeERC20Lib.safeTransfer(asset, adapter, assets);
(bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);

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
```

* The wrong value propagates from `_allocate` through the adapter interface:

```solidity
108:121:src/MYTStrategy.sol
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
    if (killSwitch) {
        return (ids(), int256(0));
    }
    require(assets > 0, "Zero amount");
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 amountAllocated = _allocate(assets);
    uint256 newAllocation = oldAllocation + amountAllocated;
    emit Allocate(amountAllocated, address(this));
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

* Dust is stranded as native ETH on the strategy and not counted in `realAssets()` (which uses Stargate pool `redeemable()`).

```solidity
89:92:src/strategies/optimism/StargateEthPoolStrategy.sol
function realAssets() external view override returns (uint256) {
    // Best available helper: “how much underlying can we redeem right now”
    return pool.redeemable(address(this));
}
```

* On accrual, the vault recomputes total assets using adapters’ `realAssets()`; dust is excluded and becomes a realized loss.

```solidity
653:659:lib/vault-v2/src/VaultV2.sol
function accrueInterestView() public view returns (uint256, uint256, uint256) {
    if (firstTotalAssets != 0) return (_totalAssets, 0, 0);
    uint256 elapsed = block.timestamp - lastUpdate;
    uint256 realAssets = IERC20(asset).balanceOf(address(this));
    for (uint256 i = 0; i < adapters.length; i++) {
        realAssets += IAdapter(adapters[i]).realAssets();
    }
```

* Deallocation reverts are amplified because `_deallocate` requires WETH balance ≥ `amount`, while dust remains as ETH unless explicitly wrapped.

```solidity
84:86:src/strategies/optimism/StargateEthPoolStrategy.sol
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
TokenUtils.safeApprove(address(weth), msg.sender, amount);
return amount;
```

* Proven by PoC tests:
  * Over-reported allocation and floored `realAssets()`:

```solidity
73:96:src/test/strategies/StargateEthPoolStrategy.t.sol
function test_allocate_overreports_and_realizes_dust_loss() public {
    // amount has dust that will be rounded down by 1e12
    uint256 amount = 1e18 + 123; // dust = 123 wei
    uint256 dust = amount % 1e12;
    uint256 expectedDeposited = amount - dust;

    vm.startPrank(allocator);
    bytes32 id = IMYTStrategy(strategy).adapterId();
    bytes memory prevAlloc = abi.encode(0);

    // Allocate via vault (so allocation accounting/caps are exercised)
    IVaultV2(vault).allocate(strategy, prevAlloc, amount);

    // Allocation is overstated by returning `amount` instead of `amountToDeposit`
    uint256 allocation = IVaultV2(vault).allocation(id);
    assertEq(allocation, amount, "allocation should equal requested amount (over-reported)");

    // The actual redeemable underlying tracks only the floored amount
    uint256 real = IMYTStrategy(strategy).realAssets();
    assertApproxEqAbs(real, expectedDeposited, 1, "realAssets should equal deposited (floored) amount");

    vm.stopPrank();
}
```

* Sub-1e12 allocations deposit 0 (100% loss of the requested amount; all stranded as ETH dust):

```solidity
98:113:src/test/strategies/StargateEthPoolStrategy.t.sol
function test_allocate_less_than_1e12_realizes_full_loss() public {
    // Entire amount becomes stranded ETH dust; pool receives 0
    uint256 amount = 1e12 - 1;

    vm.startPrank(allocator);
    bytes32 id = IMYTStrategy(strategy).adapterId();
    bytes memory prevAlloc = abi.encode(0);

    IVaultV2(vault).allocate(strategy, prevAlloc, amount);

    // Allocation is booked, but pool position is 0
    assertEq(IVaultV2(vault).allocation(id), amount, "over-reported allocation");
    assertEq(IMYTStrategy(strategy).realAssets(), 0, "no assets actually deposited in pool");

    vm.stopPrank();
}
```

* Caps can be tripped due to over-reported “change”:

```solidity
115:135:src/test/strategies/StargateEthPoolStrategy.t.sol
function test_allocate_cap_exceeded_due_to_overreported_change() public {
    // Set absolute cap to a multiple of 1e12, then allocate that + 1 wei.
    // With correct behavior, this should pass (since only the 1e12 multiple is deposited).
    // With the bug, allocation 'change' = amount and exceeds cap by 1 -> revert.
    uint256 cap = 2e12; // multiple of 1e12
    uint256 amount = cap + 1; // 1 wei dust

    // Reduce absolute cap to 'cap'
    vm.startPrank(curator);
    bytes memory idData = IMYTStrategy(strategy).getIdData();
    _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.decreaseAbsoluteCap, (idData, cap)));
    IVaultV2(vault).decreaseAbsoluteCap(idData, cap);
    vm.stopPrank();

    // Try to allocate cap + 1 (1 wei dust)
    vm.startPrank(allocator);
    bytes memory prevAlloc = abi.encode(0);
    vm.expectRevert(); // AbsoluteCapExceeded due to over-reported change
    IVaultV2(vault).allocate(strategy, prevAlloc, amount);
    vm.stopPrank();
}
```

## Impact Details

* Systematic realized loss from dust:
  * Each allocation can leave up to < 1e12 wei (\~1e-6 ETH) stranded. At \~$4,000/ETH, that is \~$0.004 per allocation.
  * 250 allocations ≈ $1 of loss; over many operations, losses accumulate.
* Sub-1e12 allocations:
  * Entire requested amount becomes stranded as ETH dust (0 deposited to the pool), causing a 100% loss of that request in accounting terms.
* Cap/Accounting integrity:
  * Allocation is overstated by the dust amount, potentially tripping absolute/relative caps spuriously and misrepresenting exposure.
* Deallocation failures (DoS of allocator actions):
  * `_deallocate` may revert because the strategy checks WETH balance ≥ requested amount while dust is left as ETH; this can block deallocation in certain edge cases.

Vulnerability classification: Accounting error leading to loss of funds (rounding/dust), cap mis-accounting, and potential DoS of deallocation.

## References

* Affected code:
  * Wrong return in `_allocate`:

```solidity
47:58:src/strategies/optimism/StargateEthPoolStrategy.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    ...
    return amount;
}
```

* `realAssets()` excludes ETH dust:

```solidity
89:92:src/strategies/optimism/StargateEthPoolStrategy.sol
return pool.redeemable(address(this));
```

* Deallocation require on WETH:

```solidity
84:86:src/strategies/optimism/StargateEthPoolStrategy.sol
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

* Change propagation to caps:

```solidity574:587:lib/vault-v2/src/vaultv2.sol

_caps.allocation = (int256(_caps.allocation) + change).toUint256();
```

* Accrual based on adapters’ `realAssets()`:

```solidity
653:659:lib/vault-v2/src/VaultV2.sol
realAssets += IAdapter(adapters[i]).realAssets();
```

* Adapter return used as “change”:

```solidity
108:121:src/MYTStrategy.sol
uint256 amountAllocated = _allocate(assets);
```

* Proof-of-concept tests:
  * Over-reported allocation and floored `realAssets()`:

```solidity
73:96:src/test/strategies/StargateEthPoolStrategy.t.sol
function test_allocate_overreports_and_realizes_dust_loss() public { ... }
```

* Sub-1e12 allocation deposits 0:

```solidity
98:113:src/test/strategies/StargateEthPoolStrategy.t.sol
function test_allocate_less_than_1e12_realizes_full_loss() public { ... }
```

* Cap exceeded due to over-reported change:

```solidity
115:135:src/test/strategies/StargateEthPoolStrategy.t.sol
function test_allocate_cap_exceeded_due_to_overreported_change() public { ... }
```

* Recommendation (concise):
  * Return `amountToDeposit` from `_allocate`.
  * Immediately re-wrap dust ETH back to WETH to keep balances consistent and avoid realized loss.
  * In `_deallocate`, wrap existing ETH dust before computing remaining redemption, then wrap all redeemed ETH to meet the requested WETH amount.

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/e0e6e14f6f6bd6e8700a654450a309c0>

## Proof of Concept

## Proof of Concept

1. replace `src/test/strategies/StargateEthPoolStrategy.t.sol` with whole file in `https://gist.github.com/IronsideSec/e0e6e14f6f6bd6e8700a654450a309c0`
2. run `forge t --mt test_allocate -vvvv`
