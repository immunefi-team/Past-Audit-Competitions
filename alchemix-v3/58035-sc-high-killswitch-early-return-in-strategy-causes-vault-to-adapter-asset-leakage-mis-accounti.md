# 58035 sc high killswitch early return in strategy causes vault to adapter asset leakage mis accounting and deallocation dos

**Submitted on Oct 30th 2025 at 06:43:50 UTC by @IronsideSec for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58035
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

When `killSwitch` is enabled on strategies inheriting `MYTStrategy`, `allocate()` returns early without refunding tokens. Because the vault transfers assets to the adapter before invoking `allocate()`, assets are stranded on the adapter, `realAssets()` excludes them, allocations remain unchanged, and subsequent deallocations revert (no approvals). This creates an immediate asset mismatch, eventual realized loss on next accrual, and a denial-of-service on exiting via `deallocate`.

### Vulnerability Details

* The vault sends assets to the adapter before calling `allocate()`:

```solidity
572:589:lib/vault-v2/src/VaultV2.sol
        SafeERC20Lib.safeTransfer(asset, adapter, assets);
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
        // ... caps checks using 'change' ...
```

* Strategy `allocate()` returns early under `killSwitch`, but does not refund:

```solidity
113:121:src/MYTStrategy.sol
        if (killSwitch) {
            return (ids(), int256(0));
        }
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountAllocated = _allocate(assets);
        // ...
```

* Strategy `deallocate()` also early-returns under `killSwitch`, never setting approvals required by the vault’s subsequent `transferFrom`:

```solidity
130:145:src/MYTStrategy.sol
        if (killSwitch) {
            return (ids(), int256(0));
            /* ... */
        }
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountDeallocated = _deallocate(assets);
        // ...
```

* The example strategy reports only ERC4626 position as `realAssets()` and ignores idle tokens stranded on the adapter:

```solidity
43:45:src/strategies/mainnet/PeapodsUSDC.sol
function realAssets() external view override returns (uint256) {
    return vault.convertToAssets(vault.balanceOf(address(this)));
}
```

* Proof-of-concept (non-reverting) shows:
  * Vault calls `allocate()` → assets move to adapter.
  * `killSwitch` early-return → allocation unchanged.
  * Adapter holds raw USDC; `realAssets()` remains 0.
  * Vault’s token balance decreases exactly by the allocated amount.

```solidity
69:109:src/test/strategies/PeapodsUSDCStrategy.t.sol
function test_killSwitch_allocate_leaks_assets_and_misaccounts_no_revert() public {
    uint256 amountToAllocate = 1_000 * 10 ** testConfig.decimals;
    vm.prank(admin);
    IMYTStrategy(strategy).setKillSwitch(true);

    bytes32 id = IMYTStrategy(strategy).adapterId();
    uint256 initialVaultTokenBal = TokenUtils.safeBalanceOf(USDC, vault);
    uint256 initialAdapterTokenBal = TokenUtils.safeBalanceOf(USDC, strategy);
    uint256 initialAlloc = IVaultV2(vault).allocation(id);
    uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();

    assertEq(initialAdapterTokenBal, 0, "adapter start bal");
    assertEq(initialRealAssets, 0, "realAssets start");
    assertEq(initialAlloc, 0, "alloc start");

    vm.startPrank(allocator);
    IVaultV2(vault).allocate(strategy, abi.encode(initialAlloc), amountToAllocate);
    vm.stopPrank();

    assertEq(TokenUtils.safeBalanceOf(USDC, strategy), amountToAllocate, "adapter raw assets");
    assertEq(IMYTStrategy(strategy).realAssets(), 0, "realAssets still zero");
    assertEq(IVaultV2(vault).allocation(id), initialAlloc, "alloc unchanged");
    assertEq(
        TokenUtils.safeBalanceOf(USDC, vault),
        initialVaultTokenBal - amountToAllocate,
        "vault token bal decreased"
    );
}
```

Why this is dangerous:

* Funds are effectively stuck on the adapter until owner action, and vault exits via `deallocate()` revert due to missing approvals.
* `realAssets()` excludes the stranded tokens, so the next `accrueInterest()` realizes a loss (share price drop) despite funds not actually being deployed.
* Allocations and caps remain inconsistent (no increase in allocation despite asset movement), undermining vault risk controls.

### Impact Details

* Loss realization: Next accrual sets `totalAssets` to a value excluding stranded tokens → share price decreases, harming depositors.
* Exit DOS: Vault `deallocate()` reverts (adapter didn’t approve), preventing liquidity retrieval during emergency.
* Allocation/cap inconsistency: Allocations unchanged while assets moved, weakening risk caps and accounting invariants.
* Governance/safety friction: Enabling `killSwitch` (intended for emergencies) paradoxically leaks funds to the adapter and blocks emergency exits.

Severity: High.

### Recommended Remediation

* In `allocate()` when `killSwitch == true`, refund immediately to the vault and return zero change:

```solidity
if (killSwitch) {
    address asset_ = IVaultV2(address(MYT)).asset();
    IERC20(asset_).transfer(msg.sender, assets); // msg.sender is the vault
    return (ids(), int256(0));
}
```

* In `deallocate()` when `killSwitch == true`, approve the vault to pull and return the correct negative change (or avoid early-return and perform approvals first):

```solidity
if (killSwitch) {
    address asset_ = IVaultV2(address(MYT)).asset();
    IERC20(asset_).approve(msg.sender, assets);
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 newAllocation = oldAllocation - assets;
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

* Count idle tokens in `realAssets()` to avoid loss realization when funds are temporarily held idle on the adapter (strategy-specific):

```solidity
function realAssets() external view override returns (uint256) {
    return vault.convertToAssets(vault.balanceOf(address(this)))
        + TokenUtils.safeBalanceOf(address(usdc), address(this));
}
```

### References

* Vault transfers then calls `allocate`:

```solidity
572:589:lib/vault-v2/src/VaultV2.sol
        SafeERC20Lib.safeTransfer(asset, adapter, assets);
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
        // ...
```

* Strategy `allocate()` killSwitch early-return:

```solidity
113:121:src/MYTStrategy.sol
        if (killSwitch) {
            return (ids(), int256(0));
        }
        // ...
```

* Strategy `deallocate()` killSwitch early-return:

```solidity
130:145:src/MYTStrategy.sol
        if (killSwitch) {
            return (ids(), int256(0));
            /* ... */
        }
        // ...
```

* Strategy `realAssets()` excludes idle:

```solidity
43:45:src/strategies/mainnet/PeapodsUSDC.sol
function realAssets() external view override returns (uint256) {
    return vault.convertToAssets(vault.balanceOf(address(this)));
}
```

* PoC test (non-reverting, calls `VaultV2.allocate()`):

```solidity
69:109:src/test/strategies/PeapodsUSDCStrategy.t.sol
function test_killSwitch_allocate_leaks_assets_and_misaccounts_no_revert() public {
    // ...
}
```

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/52387980717c94bdd092cfa847883fce>

## Proof of Concept

## Proof of Concept

* create a file src/test/PermissionedProxy.t.sol and paste the POC from gist : <https://gist.github.com/IronsideSec/52387980717c94bdd092cfa847883fce>
* then run forge t --mt test\_killSwitch\_allocate\_leaks\_assets\_and\_misaccounts\_no\_revert -vvvv
