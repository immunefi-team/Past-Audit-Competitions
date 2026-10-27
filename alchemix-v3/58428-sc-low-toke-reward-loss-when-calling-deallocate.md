# 58428 sc low toke reward loss when calling deallocate

**Submitted on Nov 2nd 2025 at 09:21:02 UTC by @SOPROBRO for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58428
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

When you `deallocate` funds, `TOKE` rewards are sent to the strategy contract instead of the MORPHO vault (`MYT`). Since the strategy has no mechanism to withdraw or forward these tokens, the rewards become permanently locked.

## Vulnerability Details

During deallocation in `TokeAutoUSDStrategy` and `TokeAutoEth`:

```solidity
rewarder.withdraw(address(this), sharesNeeded, true);
```

This triggers the following in `rewarder::_withdraw` (1):

```solidity
if (claim) {     
	_processRewards(account, account, true); 
}
```

Since `account` is `address(this)` (the strategy), the rewarder sends rewards to the strategy contract itself (2):

```
_processRewards(address account, address recipient, bool claimExtras)
```

The strategy does not expose any method to recover these rewards, effectively locking all accrued `TOKE` tokens.

## Impact Details

The impact of this is locked `TOKE` rewards, particularly in `TokeAutoUSDStrategy`, which lacks a `_claimRewards` implementation altogether — meaning the only reward path is through `deallocate`, which locks the rewards.

## References

(1) <https://etherscan.io/address/0x60882D6f70857606Cdd37729ccCe882015d1755E#code#F3#L84>

(2) <https://etherscan.io/address/0x60882D6f70857606Cdd37729ccCe882015d1755E#code#F3#L141>

## Proof of Concept

Add the following import to the top of `TokeAutoETHStrategy`

```solidity
import "forge-std/Test.sol";
```

Then add the following test, and run in the console `forge test --mt test_strat_loses_rewards -vv`

```solidity
function test_strat_loses_rewards() public {
	uint256 amountToAllocate = 1e18;
	uint256 amountToDeallocate = amountToAllocate / 2;
	vm.startPrank(vault);
	deal(testConfig.vaultAsset, strategy, amountToAllocate);
	bytes memory prevAllocationAmount = abi.encode(0);
	IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

	uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
	require(initialRealAssets > 0, "Initial real assets is 0");
	bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
	vm.roll(block.number + 1);
	// IMYTStrategy(strategy).claimRewards();
	IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
	console.log("Balance of TOKE in MYT Vault %e", IERC20(0x2e9d63788249371f1DFC918a52f8d799F4a38C94).balanceOf(vault));
	console.log("Balance of TOKE in Strategy %e", IERC20(0x2e9d63788249371f1DFC918a52f8d799F4a38C94).balanceOf(strategy));
}
```

From the logs, we can clearly see the `TOKE` is sent to the Strategy instead of the MYT Vault during deallocation, you can uncomment the `// IMYTStrategy(strategy).claimRewards();` to see the `claimRewards` will send the TOKE to the MYT Vault.

```md
Logs:
  Balance of TOKE in MYT Vault 0e0
  Balance of TOKE in Strategy 7.1596749037491e13
```
