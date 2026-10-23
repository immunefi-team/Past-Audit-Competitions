# 57622 sc low lack of claimed reward handling in myt strategies will keep all external token rewards stuck forever

**Submitted on Oct 27th 2025 at 17:04:15 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57622
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

MYT is a MorphoV2 Vault that seeks to generate yield from a variety of strategies. The base contract `MYTStrategy`, defines a blueprint for protocol-specific strategies to follow by inheriting from it. It is intended that these underlying strategies are whitelisted as adapters to the Morpho Vault. `MYTStrategy` defines abstract functions for the strategies to claim external token rewards as indicated by the `_claimRewards` virtual function. There is however no way to actually swap, transfer or autocompound the external rewards in any underlying strategy meaning even if claimed, all rewards will be stuck in the adapter contract forever.

## Vulnerability Details

`MYTStrategy` allows for arbitrary claiming of token rewards from any underlying strategy.

```
    /// @notice call this function to claim all available rewards from the respective
    /// protocol of this strategy
    function claimRewards() public virtual returns (uint256) {
        require(!killSwitch, "emergency");
        _claimRewards();
    }
```

`_claimRewards();` in turn is an abstract function that is intended to be overriden by the underlying strategy.

However, this is intended to only claim the rewards which will be sent to the adapter contract itself. `MYTStrategy` and all underlying strategies fail to provide a way for the vault to actually swap external reward tokens for more underlying strategy token (ie USDC/ETH) to realize the yield boost from rewards.

At the same time they also fail to implement a function where a privileged actor can withdraw those arbitrary tokens as fees or for manual auto-compounding. As a result, the rewards are not handled even after being claimed, causing them to remain stuck in the non-upgradeable contract — effectively locked forever.

Looking at all the strategies that actually implement `_claimRewards` we can spot `TokeAutoEthStrategy` actually claims the rewards in any token from a rewarder contract but provides no way to swap them for more ETH. The main reward token is TOKE, a regular ERC20 (<https://etherscan.io/address/0x60882D6f70857606Cdd37729ccCe882015d1755E#readContract>)

```
    // rewarder here 0x60882D6f70857606Cdd37729ccCe882015d1755E
    function _claimRewards() internal override returns (uint256 rewardsClaimed) {
        rewardsClaimed = rewarder.earned(address(this));
        rewarder.getReward(address(this), address(MYT), false);
    }
```

Looking at `MYTStrategy` we can tell that there is no way to handle rewards <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol> The Morpho Vault also does not have a function to sweep arbitrary erc20s from underlying strategies <https://github.com/morpho-org/vault-v2/blob/main/src/VaultV2.sol>

## Impact Details

All reward tokens, even if claimed become stuck in the strategy contracts, making depositors miss out on additional yield and/or the protocol to miss out on fees if they choose to keep incentives to the treasury. The only instance where rewards are handled correctly is if the reward token is the same as the underlying MYT token which is rerely the case in DeFi.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* <https://github.com/morpho-org/vault-v2/blob/main/src/VaultV2.sol>
* <https://etherscan.io/address/0x60882D6f70857606Cdd37729ccCe882015d1755E#readContract>

## Proof of Concept

## Proof of Concept

1. Add this test to the end of `TokeAutoETH.t.sol` (make sure to `import "forge-std/console.sol";` first

```

    function test_POC_claimed_rewards_permanently_stuck() public {
        // reward incentives are denominated in TOKE
        address TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;
        uint256 allocateAmount = 10e18; // 10 WETH
        uint256 rewardAmount = 100e18; // 100 TOKE

        vm.startPrank(vault);
        deal(WETH, strategy, allocateAmount);

        // Step1: Allocate the WETH
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, allocateAmount, "", address(vault));

        // Step2: Mock the reward claim. In practice, some time will pass before the rewards are claimable due to staking.
        console.log("Mocking reward claim...");
        mockRewardClaim(REWARDER, strategy, vault, rewardAmount, TOKE_TOKEN);

        // Step3: Claim the rewards
        console.log("Claiming rewards...");
        uint256 claimedRewards = IMYTStrategy(strategy).claimRewards();

        // Step4: Check the TOKE balance in the strategy.
        // TOKE tokens remain in the contract but there is no way to actually transfer or handle them.
        // The only way for current reward handling to succeed is if the rewards are in WETH.
        uint256 tokeBalance = IERC20(TOKE_TOKEN).balanceOf(strategy);
        console.log("TOKE balance in strategy after claim: ", tokeBalance);
        console.log("Contract holds tokens (TOKE) but there is no way to actually transfer or handle them.");

        assertEq(tokeBalance, rewardAmount);

        vm.clearMockedCalls();
        vm.stopPrank();
    }

    function mockRewardClaim(address rewarder, address strategy, address vault, uint256 rewardAmount, address rewardToken) public {
        vm.mockCall(
            rewarder,
            abi.encodeWithSignature("earned(address)", strategy),
            abi.encode(rewardAmount)
        );
        vm.mockCall(
            rewarder,
            abi.encodeWithSignature("getReward(address,address,bool)", strategy, vault, false),
            abi.encode()
        );
        deal(rewardToken, strategy, rewardAmount);
    }
```

2. Run the test

```
forge test --match-test test_POC_claimed_rewards_permanently_stuck -vv
```

3. Observe the logs. The strategy holds the rewards with no way to handle them

```
Ran 1 test for src/test/strategies/TokeAutoETHStrategy.t.sol:TokeAutoETHStrategyTest
[PASS] test_POC_claimed_rewards_permanently_stuck() (gas: 792008)
Logs:
  Mocking reward claim...
  Claiming rewards...
  TOKE balance in strategy after claim:  100000000000000000000
  Contract holds tokens (TOKE) but there is no way to actually transfer or handle them.
```
