# 31498 - \[SC - High] Alchemix ALCX rewards are currently subject to...

Submitted on May 20th 2024 at 16:15:19 UTC by @Norah for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31498

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Temporary freezing of funds for 12 hours

## Description

## Brief/Intro

* Upon the conclusion of the epoch/activePeriod, emissions are allocated to various components within the Alchemix system, including the `voter contract.`
* These emissions are subsequently distributed to the respective `gauges` based on the voting outcomes from the preceding epoch.
* The `distribute()` function of the voter contract can be invoked by anyone, triggering the minter to dispatch the emissions to the voter contract and other destinations, while also updating the Index to reflect the reward per voting weight, accounting for the newly injected rewards.

```
function distribute() external {
        uint256 start = 0;
        uint256 finish = pools.length;

        for (uint256 x = start; x < finish; x++) {
            // We don't revert if gauge is not alive since pools.length is not reduced
            if (isAlive[gauges[pools[x]]]) {
                _distribute(gauges[pools[x]]);
            }
        }
        IMinter(minter).updatePeriod();
    }
```

* Following this, `distribute()` invokes the internal `_distribute()` function for each gauge within the voting contracts, updating the `claimable[gauge]` amount in accordance with the newly updated `Index` via internal call `updateFor()` call.
* Lastly, the voter contract calls `notifyRewardAmount()` on the gauge contract to dispatch the rewards based on the `claimable[gauge]` amount.

```solidity

 function _distribute(address _gauge) internal {
        //Distribute once after epoch has ended
        require(
            block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
            "can only distribute after period end"
        );

        uint256 _claimable = claimable[_gauge];

        // Reset claimable amount
        claimable[_gauge] = 0;

       _updateFor(_gauge);

        if (_claimable > 0) {
            //console2.log("funds being tranfer to the gauge :",_claimable);
            IBaseGauge(_gauge).notifyRewardAmount(_claimable);
        }

        IBribe(bribes[_gauge]).resetVoting();

        emit DistributeReward(msg.sender, _gauge, _claimable);
    }

```

## Vulnerability Details

* The vulnerability arises due to the caching of the `claimable[gauge]` value before the execution of the `_updateFor(_gauge)` function.
* Consequently, the `IBaseGauge(_gauge).notifyRewardAmount(_claimable)` call passes a `claimable[gauge]` value that doesn't incorporate emissions from the previous epoch.
* As a result, the latest emissions are not transferred to the respective gauge but are only updated in the `claimable[gauge]` mapping.
* Only way to invoke `IBaseGauge(_gauge).notifyRewardAmount(_claimable)` is via `distribute()` routine, which can only be called once a epoch after epoch period ends.

## Impact Details

* This leads to rewards being inaccessible to any gauge and consequently users for at least two weeks, until the next invocation of these routines after the two-week epoch duration.
* Although these emission rewards for gauges are not entirely lost, as they are updated in the claimable\[gauge] mapping, they become temporarily unattainable for a two-week period due to the delay in the distribute() routine.

## References

Add any relevant links to documentation or code

## Recommendation

* Change the code of internal `_distribute()` function so that `_updateFor()` is called before the caching the `claimable[gauge]` to be used as parameter for the call `IBaseGauge(_gauge).notifyRewardAmount(_claimable)` .

```

    function _distribute(address _gauge) internal {
        //Distribute once after epoch has ended
        require(
            block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
            "can only distribute after period end"
        );

+++  _updateFor(_gauge);


        uint256 _claimable = claimable[_gauge];

        // Reset claimable amount
        claimable[_gauge] = 0;

---   _updateFor(_gauge);

        if (_claimable > 0) {
            //console2.log("funds being tranfer to the gauge :",_claimable);
            IBaseGauge(_gauge).notifyRewardAmount(_claimable);
        }

        IBribe(bribes[_gauge]).resetVoting();

        emit DistributeReward(msg.sender, _gauge, _claimable);
    }

```

## Proof of Concept

* I've created a test to demonstrate that rewards for epoch `X` are only transferred during epoch `X+1`, meaning after two weeks.
* Also, run this test after implementing the recommendations for better understanding as mentioned in POC.
* Note: The rewards for the first epochs are zero due to a different bug (refer to report #31494).
* I have attached output of the test in both the scenarios.
* Add the following test into the voting.t.sol file of the test suite and execute it using the following command:
  * forge test --fork-url <https://eth-mainnet.g.alchemy.com/v2/{Alchemy-api-key}> --match-test "testEmissionRewardsDistributionDelay" -vvv

```solidity
    function testEmissionRewardsDistributionDelay() public {
        
        uint256 period = minter.activePeriod();

        //first lets empty the gauge reciever (passthrough gauge) for better reward tracking 
        vm.startPrank(address(sushiGauge.receiver()));
        alcx.transfer(address(0xdead),alcx.balanceOf(address(sushiGauge.receiver())));
        vm.stopPrank();

        //we create a voter who will vote during the first epoch.
        //which will trigger the emission of the reward to the voter contract as the 
        //voting weight will be more than zero.

        uint256 tokenId = createVeAlcx(admin, 100*TOKEN_1, MAXTIME, false);
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        address[] memory gauges = new address[](1);
        gauges[0] = address(sushiGauge);
        address gauge_reciever = sushiGauge.receiver();

        hevm.prank(admin);
        voter.vote(tokenId, pools, weights, 0);

        hevm.warp(period + nextEpoch);
        hevm.roll(block.number + 1);
        
        console2.log("1st call ");
        voter.distribute();

        //Ideally as Voter.distribute() is being called after epochs end,
        //therefore the rewards emitted during this epoch should be updated in claimable amount and sent to the respective gauge.
        //But due to diffferent vulnerability (see report #31494) both the claimable_amount and Gauge_balance are zero.
        uint256 claimable_amoun_first = voter.claimable(address(sushiGauge)); 
        uint256 Gauge_balance_first = alcx.balanceOf(address(gauge_reciever));

        console2.log("First Epoch Readings : ");
        console2.log("claimable amount     : ",claimable_amoun_first);
        console2.log("Gauge_balance        : ",Gauge_balance_first);
        console2.log("");

        //During second epoch, 
        //Now, Ideally rewards collected during the last epochs emission should be sent to the particular gauges.
        //As we are calling after the epoch end period, these rewards be sent to the voter contract and SupplyIndex also will be updated.
        //But due to mentioned vulnerability instead of these being sent to respective gauges, these are merely updated into the Gauge_balance mapping.
        //As a result these will be sent to the gauge contract when the `notifyRewardAmount()` is called, which will happen in the next epoch only by minter contract after minter period.
        hevm.warp(block.timestamp + nextEpoch);
        console2.log("2nd call ");
        voter.distribute();


        uint claimable_amount_second = voter.claimable(address(sushiGauge)); 
        uint Gauge_balance_second = alcx.balanceOf(address(gauge_reciever));

        console2.log("Second Epoch Readings : ");
        console2.log("claimable amount      : ",claimable_amount_second);
        console2.log("Gauge_balance         : ",Gauge_balance_second);  
        console2.log("");

        //Now when voter.distribute() is called in after the end of third epoch
        //`claimable_amount_second` - which was first epochs emission reward, will now finally be transferred to the  after delay of two weeks.
        hevm.warp(block.timestamp + nextEpoch);
        console2.log("3rd call ");
        voter.distribute();

        uint claimable_amount_third = voter.claimable(address(sushiGauge)); 
        uint Gauge_balance_third = alcx.balanceOf(address(gauge_reciever));

        //Here we can see that 2nd epochs claimable balance is equal to the gauge balance being transffered in after the third epoch,

        assertEq(claimable_amount_second,Gauge_balance_third);

        console2.log("Third Epoch readings : ");
        console2.log("claimable amount      : ",claimable_amount_third);
        console2.log("Gauge_balance         : ",Gauge_balance_third);  
        console2.log("");
        
        //Now we as we go into the 5th epoch or end of 4th epoch
        //same things repeats, we get the last to last epoch (3rd epoch) reward transferred to the gauges.
        //While fourth epochs rewards are merely being updated in the claimable mapping.
        hevm.warp(block.timestamp + nextEpoch);
        console2.log("4th call ");
        voter.distribute();

        uint claimable_amount_fourth = voter.claimable(address(sushiGauge)); 
        uint Gauge_balance_fourth = alcx.balanceOf(address(gauge_reciever));

        //Here we can see that 3rd epochs claimable balance is equal to the gauge balance being transffered in after the fourth epoch,

        assertEq(claimable_amount_third,Gauge_balance_fourth - Gauge_balance_third);

        console2.log("Fourth Epoch readings : ");
        console2.log("claimable amount      : ",claimable_amount_fourth);
        console2.log("Gauge_balance recived : ",Gauge_balance_fourth - Gauge_balance_third);  
        console2.log("");

        /*

        - Now i would recommend running the poc again with the commmenting out the above two assertions :
           - assertEq(claimable_amount_third,Gauge_balance_fourth - Gauge_balance_third);
           - assertEq(claimable_amount_second,Gauge_balance_third);

        - Implementing the following the change in the `_distribute()` function tine of voter.sol and then running the test
        and reading the logs.

        - It should have now zero claimable_amount  for all four epochs, as the rewards are now immedietly being sent to the 
         gauges at the end of the epochs.

        function _distribute(address _gauge) internal {
            //Distribute once after epoch has ended

            require(
                block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
                "can only distribute after period end"
            );

+++         _updateFor(_gauge);

            uint256 _claimable = claimable[_gauge];

            // Reset claimable amount
            claimable[_gauge] = 0;

----        _updateFor(_gauge);

            if (_claimable > 0) {
                //console2.log("funds being tranfer to the gauge :",_claimable);
                IBaseGauge(_gauge).notifyRewardAmount(_claimable);
            }

            IBribe(bribes[_gauge]).resetVoting();

            emit DistributeReward(msg.sender, _gauge, _claimable);
        }

        */

    }

```
