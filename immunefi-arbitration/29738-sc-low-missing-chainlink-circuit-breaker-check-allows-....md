# 29738 - \[SC - Low] Missing Chainlink circuit breaker check allows ...

Submitted on Apr 1st 2024 at 06:09:20 UTC by @marchev for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29738

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/immunefi-team/vaults/blob/main/src/RewardTimelock.sol>

Impacts:

* Theft of unclaimed royalties

## Description

## Brief/Intro

Chainlink aggregators feature a built-in circuit breaker that activates during significant price fluctuations, keeping the asset price within a pre-defined range. In scenarios like the LUNA crash, the oracle reports a pre-set minimum value (`minAnswer`) rather than the real market price. This mechanism could enable malicious projects to payout rewards that are below market value by leveraging the inflated price from the Chainlink oracle.

Such a vulnerability has resulted in a $11M exploit on Venus Protocol during the LUNA crash: <https://therecord.media/collapse-of-luna-cryptocurrency-leads-to-11-million-exploit-on-venus-protocol>

## Vulnerability Details

The `RewardTimelock` contract depends on Chainlink price oracles to align reward payouts with the actual market value. It employs `_checkRewardDollarValue`, a function that fetches the asset's current market price with `PriceConsumer#tryGetSaneUsdPrice18Decimals()`, implementing various checks to ensure the price's accuracy and relevance.

Nonetheless, this function lacks verification for the activation of Chainlink's circuit breaker. In extreme price movements, when the asset price falls below `minAnswer` or rises above `maxAnswer`, the Chainlink feed still reports these thresholds instead of the actual market price of the asset. Missing this critical check means a project could exploit the situation and pay whitehats a reduced award under these circumstances.

Let's take the following example:

Example scenario:

1. A significant price drop triggers the Chainlink circuit breaker for a reward token, causing the feed to report `minAnswer`.
2. A project initiates a whitehat reward payout via `RewardTimelock`.
3. After the cooldown period, the project executes the payout at the reported, inflated price.

## Impact Details

This vulnerability means a whitehat could receive a reward less valuable than the market-equivalent dollar amount expected. While such an exploit relies on rare, dramatic market events (akin to the LUNA crash), the potential impact is significant, allowing projects to issue substantially undervalued rewards. Thus, this vulnerability is classified with Low severity to reflect its low likeliehood.

## Solution

The proposed fix involves integrating checks for the activation of Chainlink's circuit breaker. The following code modifications illustrate the necessary adjustments:

```diff
diff --git a/src/oracles/IFeedRegistryMinimal.sol b/src/oracles/IFeedRegistryMinimal.sol
index 1a569d3..79be2f7 100644
--- a/src/oracles/IFeedRegistryMinimal.sol
+++ b/src/oracles/IFeedRegistryMinimal.sol
@@ -12,4 +12,6 @@ interface IFeedRegistryMinimal {
         external
         view
         returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound);
+
+    function getFeed(address base, address quote) external view returns (address aggregator);
 }
diff --git a/src/oracles/FeedRegistryL2.sol b/src/oracles/FeedRegistryL2.sol
index 076f806..4697d90 100644
--- a/src/oracles/FeedRegistryL2.sol
+++ b/src/oracles/FeedRegistryL2.sol
@@ -106,4 +106,8 @@ contract FeedRegistryL2 is IFeedRegistryMinimal, Ownable2Step {
         uint256 timeSinceUp = block.timestamp - startedAt;
         require(timeSinceUp > GRACE_PERIOD_TIME, "FeedRegistryL2: Grace period not over");
     }
+
+    function getFeed(address base, address quote) external view returns (address aggregator) {
+        //TODO: Add actual implementation
+    }
 }
diff --git a/src/oracles/PriceConsumer.sol b/src/oracles/PriceConsumer.sol
index a024bd3..2025e4a 100644
--- a/src/oracles/PriceConsumer.sol
+++ b/src/oracles/PriceConsumer.sol
@@ -4,6 +4,7 @@ pragma solidity 0.8.18;
 
 import { Ownable2Step } from "openzeppelin-contracts/access/Ownable2Step.sol";
 import { IFeedRegistryMinimal } from "./IFeedRegistryMinimal.sol";
+import { IOffchainAggregatorMinimal } from "./IOffchainAggregatorMinimal.sol";
 import { Denominations } from "./chainlink/Denominations.sol";
 import { IPriceFeed } from "./IPriceFeed.sol";
 import { IPriceConsumerEvents } from "./IPriceConsumerEvents.sol";
@@ -53,6 +54,12 @@ contract PriceConsumer is Ownable2Step, IPriceConsumerEvents {
         require(response.updatedAt <= block.timestamp, "PriceConsumer: Feed updatedAt is in the future");
         require(block.timestamp - response.updatedAt <= _getFeedTimeout(base), "PriceConsumer: Feed is stale");
 
+        IOffchainAggregatorMinimal baseAggregator = IOffchainAggregatorMinimal(registry.getFeed(base, Denominations.USD));
+        int256 minAnswer18Decimals = _convertTo18Decimals(baseAggregator.minAnswer(), registry.decimals(base, Denominations.USD));
+        int256 maxAnswer18Decimals = _convertTo18Decimals(baseAggregator.maxAnswer(), registry.decimals(base, Denominations.USD));
+        require(response.answer > minAnswer18Decimals, "PriceConsumer: Min feed price circuit breaker");
+        require(response.answer < maxAnswer18Decimals, "PriceConsumer: Max feed price circuit breaker");
+
         return uint256(response.answer);
     }
```

Furthermore, a new interface, `src/oracles/IOffchainAggregatorMinimal.sol`, is required to fetch the `minAnswer` and `maxAnswer` directly from the aggregator:

```sol
// SPDX-License-Identifier: Immuni Software PTE Ltd General Source License
// https://github.com/immunefi-team/vaults/blob/main/LICENSE.md
pragma solidity 0.8.18;
interface IOffchainAggregatorMinimal {
    function minAnswer() external view returns (int192);
    function maxAnswer() external view returns (int192);
}
```

## Proof of Concept

The following coded PoC demonstrates how a payout could be performed even if the Chainlink price oracle's circuit breaker is activated.

Add the following import in `test/foundry/RewardTimelock.t.sol`:

```sol
import { ERC20PresetMinterPauser } from "openzeppelin-contracts/token/ERC20/presets/ERC20PresetMinterPauser.sol";
```

Then add the following test case to it as well:

```sol

    function testNoCircuitBreakerForMinMaxPriceWhenExecutingRewardTx() public {
        // Token in which the award would be paid
        ERC20PresetMinterPauser token = new ERC20PresetMinterPauser("Token", "TOK");
        console.log("TOK/USD (Market price after flash crash) = 1e8");
        console.log("TOK aggregator minAnswer = 10e8");
        token.mint(address(vault), 10_000 ether);

        uint256 dollarAmount = 50_000; // Award dollar amount
        uint256 tokenAmount = 5_000 ether; // Award token amount (5,000 TOK x 10 = 50,000 USD)

        // *** MOCKS SETUP - START ***
        // Mock arbitration.vaultIsInArbitration
        vm.mockCall(
            address(arbitration),
            abi.encodeCall(arbitration.vaultIsInArbitration, (address(vault))),
            abi.encode(true)
        );

        address tokenAggregator = makeAddr("tokenAggregator");
        // Mock feedRegistry.getFeed()
        vm.mockCall(
            address(feedRegistry),
            abi.encodeWithSignature("getFeed(address,address)", token, Denominations.USD),
            abi.encode(tokenAggregator)
        );

        // Mock tokenAggregator.minAnswer() & tokenAggregator.maxAnswer()
        vm.mockCall(
            address(tokenAggregator),
            abi.encodeWithSignature("minAnswer()"),
            abi.encode(int192(10e8))
        );

        vm.mockCall(
            address(tokenAggregator),
            abi.encodeWithSignature("maxAnswer()"),
            abi.encode(int192(1_000_000_000e8))
        );

        // Mock feedRegistr.latestRoundData()
        vm.mockCall(
            address(feedRegistry),
            abi.encodeWithSignature("latestRoundData(address,address)", address(token), Denominations.USD),
            abi.encode(
                31337, // roundId
                int256(10e8), // answer is equal to minAnswer
                0, // startedAt (ignored)
                block.timestamp + rewardTimelock.txCooldown() - 1 hours, // updatedAt
                0 // answeredInRound (ignored)
            )
        );

        // Mock feedRegistry.decimals()
        vm.mockCall(
            address(feedRegistry),
            abi.encodeWithSignature("decimals(address,address)", address(token), Denominations.USD),
            abi.encode(8)
        );
        // *** MOCKS SETUP - END ***

        // Set right permissions on moduleGuard
        vm.startPrank(protocolOwner);
        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
        vm.stopPrank();

        uint256 nonce = rewardTimelock.vaultTxNonce(address(vault));
        bytes32 txHash = rewardTimelock.getQueueTransactionHash(address(this), dollarAmount, address(vault), nonce);

        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.queueRewardTransaction, (address(this), dollarAmount)),
            Enum.Operation.Call
        );
        assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);
        console.log("Transaction reward for %s USD queued", dollarAmount);

        vm.warp(block.timestamp + rewardTimelock.txCooldown());
        assertTrue(rewardTimelock.canExecuteTransaction(txHash));

        Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](1);
        erc20Rewards[0] = Rewards.ERC20Reward({ token: address(token), amount: tokenAmount });

        console.log("Expected behavior: executeRewardTransaction fails since Chainlink price feed circuit breaker is hit (answer == minAnswer)");
        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 0, 50_000)),
            Enum.Operation.Call,
            true
        );
    }
```

Run the PoC via `forge test --mt "testNoCircuitBreakerForMinMaxPriceWhenExecutingRewardTx" -vvvvv`

The expected behavior is that the reward payout fails but the actual behavior is that it gets executed successfully.
