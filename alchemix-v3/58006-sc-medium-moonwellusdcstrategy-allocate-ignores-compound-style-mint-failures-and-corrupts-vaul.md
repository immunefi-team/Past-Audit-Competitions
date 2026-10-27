# 58006 sc medium moonwellusdcstrategy allocate ignores compound style mint failures and corrupts vault accounting

## #58006 \[SC-Medium] \`MoonwellUSDCStrategy.\_allocate\` ignores Compound-style mint failures and corrupts vault accounting

**Submitted on Oct 29th 2025 at 23:35:12 UTC by @pxng0lin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58006
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

### Description

## Title

`MoonwellUSDCStrategy._allocate` ignores Compound-style mint failures and corrupts vault accounting.

## Bug Description

`MoonwellUSDCStrategy._allocate(uint256 amount)` approves USDC to the Moonwell market and calls `mUSDC.mint(amount)` without checking the return code. Compound-style markets (EIP-20) return `0` on success and a non-zero error on failure. Because `_allocate` blindly returns the requested `amount`, the vault records a successful deployment even when Moonwell rejected the mint, leaving the USDC stranded on the adapter and corrupting accounting.

## Impact

* **False accounting:** The vault believes capital was deployed while assets remain idle on the strategy. Subsequent deallocations assume inflated balances and can revert or underpay.
* **Operational freeze:** If minting continues to fail, every allocation attempt silently reports success, leaving allocators unaware of the issue and preventing rebalancing.

The fork test logs show the vault recording the full 1,000 USDC allocation while `realAssets()` stays zero.

## Risk Breakdown

* **Privileges required:** Allocator/admin calling `_allocate` through the vault.
* **Likelihood:** Medium – Moonwell markets can be paused during incidents (pause-guardian controls inherited from Compound via Comptroller) or return non-zero codes for transient errors.

## Recommendation

Require the mint to succeed before reporting success:

```solidity
uint256 result = mUSDC.mint(amount);
require(result == 0, "Moonwell mint failed");
```

## References

* Vulnerable function: `MoonwellUSDCStrategy._allocate`
* `mUSDC` interface: Compound-style error codes
* Fork PoC in protocol suite: `test_mintFailureKeepsVaultAccountingInflated`
* Minimal harness PoC: `test/MoonwellUSDCStrategyAllocatePoC.t.sol`
* Compound cToken specification confirming `mint` returns 0 on success and non-zero error codes otherwise (includes example `assert(cToken.mint(100) == 0);`): <https://docs.compound.finance/v2/ctokens/#mint>
* ERC-20 specification warning: *“Callers MUST handle false from returns (bool success). Callers MUST NOT assume that false is never returned!”* — <https://eips.ethereum.org/EIPS/eip-20#token>

### Proof of Concept

### Proof of Concept

* Allocation to the strategy with a mock mToken that returns error code `1` from `mint`. `_allocate` reports success even though no mTokens are minted.
* Add the below code to the existing test suite `src/test/strategies/MoonwellUSDCStrategy.t.sol`
* Run the following command in terminal `forge t --mt test_mintFailureKeepsVaultAccountingInflated -vvv`

#### Code

```solidity
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {IERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
```

```solidity
interface IMoonwellMToken {
    function mint(uint256 mintAmount) external returns (uint256);
    function balanceOf(address owner) external view returns (uint256);
}

function test_mintFailureKeepsVaultAccountingInflated() public {
    uint256 amount = 1_000e6;

    bytes memory prevAllocation = abi.encode(uint256(0));
    bytes memory mintCall = abi.encodeWithSelector(IMoonwellMToken.mint.selector, amount);
    vm.expectCall(MOONWELL_USDC_MTOKEN, mintCall);
    vm.mockCall(MOONWELL_USDC_MTOKEN, mintCall, abi.encode(uint256(1)));

    emit log("Simulating Moonwell mint failure with return code 1");
    vm.startPrank(allocator);
    IVaultV2(vault).allocate(strategy, prevAllocation, amount);
    vm.stopPrank();
    vm.clearMockedCalls();

    bytes32 strategyId = IMYTStrategy(strategy).adapterId();
    emit log_named_uint("Vault allocation recorded", IVaultV2(vault).allocation(strategyId));
    emit log_named_uint("Strategy USDC balance", IERC20(USDC).balanceOf(strategy));
    emit log_named_uint("Strategy mUSDC balance", IMoonwellMToken(MOONWELL_USDC_MTOKEN).balanceOf(strategy));
    emit log_named_uint("Strategy realAssets()", IMYTStrategy(strategy).realAssets());

    assertEq(IVaultV2(vault).allocation(strategyId), amount);
    assertEq(IERC20(USDC).balanceOf(strategy), amount);
    assertEq(IMoonwellMToken(MOONWELL_USDC_MTOKEN).balanceOf(strategy), 0);
    assertEq(IMYTStrategy(strategy).realAssets(), 0);
}
```

#### Output

```shell
Simulating Moonwell mint failure with return code 1
Vault allocation recorded: 1000000000
Strategy USDC balance: 1000000000
Strategy mUSDC balance: 0
Strategy realAssets(): 0
```
