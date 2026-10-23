# 29744 - \[SC - Insight] Projects can pay rewards at up to below market...

Submitted on Apr 1st 2024 at 20:45:03 UTC by @marchev for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29744

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/RewardTimelock.sol>

Impacts:

* Theft of unclaimed royalties

## Description

## Brief/Intro

Due to a hardcoded setting in the protocol (`PRICE_DEVIATION_TOLERANCE_BPS`) allowing for a 1% price deviation of whitehat awards when paid in non-native tokens and the potential 2% deviation for Chainlink's price feeds for some tokens (e.g. ARB, ENS, FXS, etc.), there's a risk that whitehat hackers could be underpaid by up to 2.97%. This issue arises when the actual market price of an asset falls slightly, such as by 1.99%, which doesn't trigger an oracle update. This results in a temporarily inflated asset price reported by the Chainlink price oracle. This combined with the ability to pay a reward with a deviation of 1% could be exploited by projects to underpay whitehats.

## Vulnerability Details

The vulnerability centers around the `RewardTimelock` smart contract, which handles reward payouts to whitehats during arbitration. Projects specify the dollar value of the award, and after a cooldown period, they can pay in tokens. The smart contract checks if the token amount matches the dollar value, allowing a 1% deviation. However, some tokens have a 2% price deviation in their Chainlink price feeds. If the token's market price drops just below 2%, a project can pay the whitehat less than the full market value, exploiting the gap up to 2.97%:

```
100 - ((100 - 1.99%) * 99%) = 2.97%
```

Furthermore, projects have the capability to exploit the vulnerability even when the price drops more than 2% below the oracle price. They can do this by frontrunning the transaction meant to update the Oracle price feed. This maneuver provides a significant safety margin.

**Example:**

1. A project owes a whitehat $100,000, paid in ARB tokens.
2. After the cooldown, the ARB/USD price by the oracle is $1.60, but the market price is $1.5688.
3. The project pays 61,785 ARB instead of the 63,742.99 ARB due at market rates.

While the price oracle price deviation cannot be avoided, adjusting the hardcoded `PRICE_DEVIATION_TOLERANCE_BPS` to be more flexible and specific to token's price feed could mitigate this issue to a certain extent. This is especially important for feeds that update less frequently as indicated by a 2% deviation and a 86400s heartbeat. It should be noted that all Chainlink price feeds on mainnet with a deviation of 2% have a hearbeat of 86400s which implies they are meant to update less frequently which reduces the need for a favorable price deviation tolerance as high as 1%.

## Impact Details

The hardcoded deviation combined with Chainlink's deviation can lead to significant losses for whitehats. This risk is medium, as it depends on specific price feed deviations (2%) and market conditions. The severity is also considered medium due to the financial impact on whitehats. A more adaptable configuration for the reward value deviation could mitigate the issue.

## Proof of Concept

To illustrate the vulnerability, I provide a coded PoC which demonstrates its impact. Add the following test case to `RewardTimelock.t.sol`:

```sol
    function test_project_can_pay_reward_at_up_to_2_97_percent_below_market_value() public {
        // Token in which the award is paid
        ERC20PresetMinterPauser token = new ERC20PresetMinterPauser("Arbitrum", "ARB");
        token.mint(address(vault), 1_000_000e18);

        uint256 ARB_USD_PRICE_SCALE = 1e8;

        uint256 arbUsdOraclePrice = 1.60e8;   // ARB/USD oracle price = 1.60000000
        uint256 arbUsdMarketPrice = 1.5688e8; // ARB/USD market price = 1.56880000 (1.95% below oracle price)
                                              // NOTE: Deviation for ARB/USD chainlink oracle = [-2%; 2%].
                                              // Thus, -1.95% below oracle price does not trigger an oracle  update.

        uint256 dollarAmount = 100_000;
        uint256 tokenAmountPerOraclePrice = (dollarAmount * 1e18 * ARB_USD_PRICE_SCALE) / arbUsdOraclePrice;
        uint256 tokenAmountPerOraclePriceMinus1Percent = (tokenAmountPerOraclePrice * 99_00) / 100_00;
        uint256 tokenAmountPerMarketPrice = (dollarAmount * 1e18 * ARB_USD_PRICE_SCALE) / arbUsdMarketPrice;

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

        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.queueRewardTransaction, (address(this), dollarAmount)),
            Enum.Operation.Call
        );
        assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);

        vm.warp(block.timestamp + rewardTimelock.txCooldown());
        assertTrue(rewardTimelock.canExecuteTransaction(txHash));

        Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](1);
        erc20Rewards[0] = Rewards.ERC20Reward({ token: address(token), amount: tokenAmountPerOraclePriceMinus1Percent });

        // Mock priceConsumer
        vm.mockCall(
            address(priceConsumer),
            abi.encodeCall(priceConsumer.tryGetSaneUsdPrice18Decimals, (address(token))),
            abi.encode(uint256(arbUsdOraclePrice) * 1e10)
        );

        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 0, 50_000)),
            Enum.Operation.Call
        );
        assertEq(token.balanceOf(address(this)), tokenAmountPerOraclePriceMinus1Percent);

        uint256 whitehatLossAbsolute = tokenAmountPerMarketPrice - tokenAmountPerOraclePriceMinus1Percent;
        uint256 whitehatLossPercentage = whitehatLossAbsolute * 100_00 / tokenAmountPerMarketPrice;

        console.log("--------------------------------------");
        console.log("ARB/USD oracle price = %s (8 decimals)", arbUsdOraclePrice);
        console.log("ARB/USD market price = %s (8 decimals)", arbUsdMarketPrice);
        console.log("--------------------------------------");
        console.log("Dollar amount: $%s", dollarAmount);
        console.log("Expected ARB amount (per market price): %s", tokenAmountPerMarketPrice);
        console.log("Actual ARB amount (per oracle price minus 1 percent): %s", tokenAmountPerOraclePriceMinus1Percent);
        console.log("Loss for whitehat (absolute): %s", whitehatLossAbsolute);
        console.log("Loss for whitehat (%): %s", getPercentageString(whitehatLossPercentage), "%");
        console.log("--------------------------------------");
    }
```

Run the PoC via the following command:

```sh
forge test --mt "test_project_can_pay_reward_at_up_to_2_99_percent_below_market_value" -vvvvv
```
