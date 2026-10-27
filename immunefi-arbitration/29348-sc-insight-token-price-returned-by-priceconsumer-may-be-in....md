# 29348 - \[SC - Insight] Token price returned by PriceConsumer may be in...

Submitted on Mar 14th 2024 at 19:19:29 UTC by @greed for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29348

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/RewardTimelock.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Impact

*Considering its likelihood the issue is marked as LOW*

1. The reward distribution function `RewardTimelock::executeRewardTransaction()` may revert while it should not due to the estimated USD price of tokens being out of bounds.
2. The protocol may send more or less rewards in USD than the promised payout to the whitehat. This will either result in a loss for the protocol and a win for the researcher or a win for the protocol and a loss for the researcher.

## Vulnerability details

When the reward transaction is fired through `RewardTimelock::executeRewardTransaction()`, the price of the tokens being sent to the whitehat is calculated in `_checkRewardDollarValue()` and relies on `PriceConsumer::tryGetSaneUsdPrice18Decimals()`.

This function uses different **feed** to calculate the price of an asset depending on if it has been configured or not.

Under certain circumstances, a **feed** can return an outdated price which in period of high volatility (where a pump or dump can happen in a matter of minutes) can be a critical issue.

In order to get a price that is as accurate as possible, `PriceConsumer::tryGetSaneUsdPrice18Decimals()` has the following check

<https://github.com/immunefi-team/vaults/blob/main/src/oracles/PriceConsumer.sol#L54>

```js
require(block.timestamp - response.updatedAt <= _getFeedTimeout(base), "PriceConsumer: Feed is stale");
```

The function `_getFeedTimeout(base)` can return `1 days` if the corresponding `customFeedTimeout[base]` has not been configured.

The check basically accepts a price that is at worse `1 days` old.

## Recommended mitigation steps

Lower the value of `FEED_TIMEOUT` to make the transaction revert in case the price may not be accurate enough

## Proof of Concept

In order to test the **execution reverts when it should not** case, the following script (based on `testQueuesAndExecutesRewardTx()`) can be added in `test/foundry/RewardTimelock.t.sol`

```sh
forge test --match-test testOldPriceRevert -vvvv
```

*The comments describe the events that occur during the lifecycle of the payout*

```js
function testOldPriceRevert() public {
    uint256 value = 5 ether;
    uint256 dollarAmount = 2000;
    vm.deal(address(vault), value);

    // set right permissions on moduleGuard
    vm.startPrank(protocolOwner);
    moduleGuard.setTargetAllowed(address(vaultDelegate), true);
    moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
    moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
    vm.stopPrank();

    uint256 nonce = rewardTimelock.vaultTxNonce(address(vault));
    bytes32 txHash = rewardTimelock.getQueueTransactionHash(address(this), dollarAmount, address(vault), nonce);

    // Mock vaultIsInArbitration
    vm.mockCall(
        address(arbitration),
        abi.encodeCall(arbitration.vaultIsInArbitration, (address(vault))),
        abi.encode(true)
    );

    // Mock priceConsumer
    // At the time of the queue transaction, the price of 1 ETH == 2000$
    vm.mockCall(
        address(priceConsumer),
        abi.encodeCall(priceConsumer.tryGetSaneUsdPrice18Decimals, (Denominations.ETH)),
        abi.encode(uint256(2000) * 10 ** 18)
    );

    _sendTxToVault(
        address(rewardTimelock),
        0,
        abi.encodeCall(rewardTimelock.queueRewardTransaction, (address(this), dollarAmount)),
        Enum.Operation.Call
    );

    assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);

    // 24 hours elapsed (rewardTimelock.txCooldown() == 86400)
    vm.warp(block.timestamp + rewardTimelock.txCooldown());
    assertTrue(rewardTimelock.canExecuteTransaction(txHash));

    Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](0);

    // Mock priceConsumer
    // At the time of the execute transaction, the real price of 1 ETH == 4000$ (x2)
    // but the priceConsumer price is 24 hours old and still returns 1 ETH == 2000$
    vm.mockCall(
        address(priceConsumer),
        abi.encodeCall(priceConsumer.tryGetSaneUsdPrice18Decimals, (Denominations.ETH)),
        abi.encode(uint256(2000) * 10 ** 18)
    );

    // The protocol checks the price of ETH on coinmarketcap for example which says 1 ETH == 4000$
    // The protocol sends 0.5 ETH == 2000$ which should pass BUT the execution reverts with "RewardTimelock: reward dollar value too low"
    _sendTxToVault(
        address(rewardTimelock),
        0,
        abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 0.5 ether, 50_000)),
        Enum.Operation.Call
    );
    assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);
    assertEq(address(vault).balance, 0);
}
```

***

In order to test the **win or loss of rewards** case, the following script (based on `testQueuesAndExecutesRewardTx()`) can be added in `test/foundry/RewardTimelock.t.sol`

```sh
forge test --match-test testOldPricePass -vvvv
```

*The comments describe the events that occur during the lifecycle of the payout*

```js
function testOldPricePass() public {
    uint256 value = 5 ether;
    uint256 dollarAmount = 2000;
    vm.deal(address(vault), value);

    // set right permissions on moduleGuard
    vm.startPrank(protocolOwner);
    moduleGuard.setTargetAllowed(address(vaultDelegate), true);
    moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
    moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
    vm.stopPrank();

    uint256 nonce = rewardTimelock.vaultTxNonce(address(vault));
    bytes32 txHash = rewardTimelock.getQueueTransactionHash(address(this), dollarAmount, address(vault), nonce);

    // Mock vaultIsInArbitration
    vm.mockCall(
        address(arbitration),
        abi.encodeCall(arbitration.vaultIsInArbitration, (address(vault))),
        abi.encode(true)
    );

    // Mock priceConsumer
    // At the time of the queue transaction, the price of 1 ETH == 2000$
    vm.mockCall(
        address(priceConsumer),
        abi.encodeCall(priceConsumer.tryGetSaneUsdPrice18Decimals, (Denominations.ETH)),
        abi.encode(uint256(2000) * 10 ** 18)
    );

    _sendTxToVault(
        address(rewardTimelock),
        0,
        abi.encodeCall(rewardTimelock.queueRewardTransaction, (address(this), dollarAmount)),
        Enum.Operation.Call
    );

    assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);

    // 24 hours elapsed (rewardTimelock.txCooldown() == 86400)
    vm.warp(block.timestamp + rewardTimelock.txCooldown());
    assertTrue(rewardTimelock.canExecuteTransaction(txHash));

    Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](0);

    // Mock priceConsumer
    // At the time of the execute transaction, the price of 1 ETH == 4000$ (x2)
    // but the priceConsumer price is 24 hours old and still returns 1 ETH == 2000$
    vm.mockCall(
        address(priceConsumer),
        abi.encodeCall(priceConsumer.tryGetSaneUsdPrice18Decimals, (Denominations.ETH)),
        abi.encode(uint256(2000) * 10 ** 18)
    );

    // The protocol relies on `` and sends 1 ETH instead of 0.5 because it thinks 1 ETH == 2000$ still
    // while 1 ETH == 4000$
    // The protocol experienced 2000$ of loss
    _sendTxToVault(
        address(rewardTimelock),
        0,
        abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 1 ether, 50_000)),
        Enum.Operation.Call
    );
    assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);
    // The vault has sent 1 ETH to the whitehat
    assertLe(
        address(vault).balance, value - 1 ether
    );
}
```
