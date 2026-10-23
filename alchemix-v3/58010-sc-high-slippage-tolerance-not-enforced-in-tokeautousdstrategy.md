# 58010 sc high slippage tolerance not enforced in tokeautousdstrategy

**Submitted on Oct 30th 2025 at 00:11:10 UTC by @SOPROBRO for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58010
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `TokeAutoUSDStrategy` lacks slippage protection when allocating funds, as it calls `router.depositMax` with `minSharesOut = 0`. This allows deposits to proceed even if far fewer shares are received than expected, resulting in MYT users potentially suffering losses due to under-collateralization.

## Vulnerability Details

The DAO manages allocation, and if it decides to allocate into `TokeAutoUSDStrategy`, the flow goes from `MYTStrategy::allocate` -> `TokeAutoUSDStrategy::_allocate` -> `router.depositMax`. The following parameters are used to call `depositMax`:

```solidity
uint256 shares = router.depositMax(autoUSD, address(this), 0);
```

The problem is that it sets the `minSharesOut` as 0. See the code in Etherscan(1), which is enforced in `AutopilotRouterBase`, when depositing into the vault:

```solidity
if ((sharesOut = vault.deposit(amount, to)) < minSharesOut) {
	revert MinSharesError();
}
```

From the docs of Auto Finance, we can see that slippage is expected, and they state(2) :

> Depending on the conditions of the Autopool, the overall market, and the timing of the debt reporting process slippage may be encountered on both entering and exiting the Autopool. **It is very important to always check the shares received on entering, and the assets received on exiting, are greater than an expected amount**.

Therefore, not enforcing a `minSharesOut` will subject MYT users to excess slippage, resulting in loss of funds backing the MYT.

> Note: This is also found in `TokeAutoEth.sol`

## Impact Details

The impact of this is a loss of funds for MYT users, resulting in system-wide liquidation due to the Meta-Yield Token itself losing backing. Therefore, I believe this is high severity, as the vault opens itself up to a slippage, and other slippage-type attacks due to the lack of slippage enforcement.

## References

(1) <https://etherscan.io/address/0x37dd409f5e98ab4f151f4259ea0cc13e97e8ae21#code#F1#L95>

(2) <https://docs.auto.finance/developer-docs/integrating/4626-compliance#slippage>

## Proof of Concept

## Proof of Concept

Add the following imports to `TokeAutoUSDStrategy.t.sol`

```solidity
import "forge-std/Test.sol";
import {IERC4626} from "../../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
```

Then add the following test, and run in the console `forge test --mt test_allocate_shares_slippage -vv`.

```solidity
function test_allocate_shares_slippage() public {
	uint256 amountToAllocate = 1e18;

	vm.startPrank(vault);

	// Fund the strategy
	deal(testConfig.vaultAsset, strategy, amountToAllocate);

	// Reference the real AutoUSD vault
	IERC4626 autoUSD = IERC4626(TOKE_AUTO_USD_VAULT);
	uint256 stakedSharesBefore = IERC20(address(autoUSD)).balanceOf(REWARDER);
	uint256 fakeShares = amountToAllocate / 2; // simulate slippage
	vm.mockCall(
		AUTOPILOT_ROUTER,
		abi.encodeWithSelector(bytes4(keccak256("depositMax(address,address,uint256)")), TOKE_AUTO_USD_VAULT, address(strategy), 0),
		abi.encode(fakeShares)
	);

	// Mint fakeShares to the strategy
	deal(address(autoUSD), address(strategy), fakeShares);
	// --- Allocate via strategy ---
	bytes memory prevAlloc = abi.encode(0);
	IMYTStrategy(strategy).allocate(prevAlloc, amountToAllocate, "", address(vault));

	// --- Check how many shares got staked ---
	uint256 stakedSharesAfter = IERC20(address(autoUSD)).balanceOf(REWARDER);
	console.log("Staked shares (slipped): %e", stakedSharesAfter - stakedSharesBefore);
}
```

A `mockCall` was used to mimic the router returning less shares than expected, which would usually be checked if slippage was enforced. However, we can see that the code continued, and only half of what was allocated was sent to the rewarder.

```md
Logs:
  Staked shares (slipped): 5e17
```
