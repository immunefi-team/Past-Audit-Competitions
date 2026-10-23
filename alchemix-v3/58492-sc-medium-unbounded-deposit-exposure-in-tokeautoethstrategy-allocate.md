# 58492 sc medium unbounded deposit exposure in tokeautoethstrategy allocate&#x20;

**Submitted on Nov 2nd 2025 at 17:47:28 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58492
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The \_allocate() function in the TokeAutoEthStrategy contract deposits funds into the AutopilotRouter using the following logic:

```sol
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(weth), address(router), amount);
    uint256 shares = router.depositMax(autoEth, address(this), 0);
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

The function verifies that the contract’s WETH balance is at least amount, but then calls router.depositMax(autoEth, address(this), 0). The depositMax() function deposits all available WETH held by the contract, not just the amount parameter.

If additional tokens are transferred to the strategy contract (e.g., by an attacker), this call will attempt to deposit more than intended. In particular, an attacker can frontrun the allocation transaction by sending a small amount (e.g., 1 wei) of WETH to the strategy, which causes depositMax() to revert internally.

## Vulnerability Details

The strategy assumes that \_allocate(amount) only deposits amount worth of assets.

However, router.depositMax() ignores this limit and deposits the full contract balance.

Because depositMax() likely computes depositable assets as IERC20(weth).balanceOf(address(this)), if an attacker front-runs the transaction and sends even 1 wei of WETH, it changes the expected amount of tokens to deposit.

This will lead to the transaction to revert and when attack can be repeated when amount pass into allocate is increased.

## Impact Details

The strategy may revert indefinitely on subsequent \_allocate() calls, blocking allocations.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L56>

## Proof of Concept

Copy the test into src/test/strategies/TokeAutoETHStrategy.t.sol Run forge test --mt test\_strategy2 -vvvv

```sol
function test_strategy2() public {
        uint256 amountToAllocate = 50e18;

        // Attacker adds 1 wei to the strategy
        deal(testConfig.vaultAsset, strategy, 50e18 + 1);

        console.log(IERC20(testConfig.vaultAsset).balanceOf(address(strategy)));

        // Vault allocates normally
        vm.startPrank(vault);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        vm.stopPrank();
    }
```
