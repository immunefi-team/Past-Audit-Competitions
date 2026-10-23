# 31494 - \[SC - High] Alchemix The first epochs ALCX emissions of vo...

Submitted on May 20th 2024 at 13:23:00 UTC by @Norah for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31494

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

* During each epoch, the minter contract distributes emissions to various components of the Alchemix system, one of them being the voter contract.
* Anyone can call the `distribute()` function on the voter contract. If the minter period/epoch is over, the rewards are distributed to the various gauges, and `minter.updatePeriod()` is called, which triggers the minter contract to distribute the emissions for the previous epoch.

## Vulnerability Details

* The problem arises if the `_distribute()` function is called first and then `minter.updatePeriod()`. I - In this scenario, during the first epoch, since there are no emissions at this point, no rewards will be distributed. - Afterward, `minter.updatePeriod()` will be triggered, sending the rewards to the voter contract.
* However, the rewards distribution routine has already been called, so no reward will be transferred to the contract or updated in the claimable\[\_gauge] mapping for the gauges to claim later on.
  * Essentially, there is emissions are now stuck in the `voter` contract forever.
* Ideally, minter.updatePeriod() should be called first, to allow the minter contract to send the rewards, and then the \_distribute() routine should be called to distribute and update the rewards in claimable or send them directly to the gauge
* For more detail check POC.

## Impact Details

* As a result, the rewards for the first epoch will be forever stuck in the voter contract, and gauges will be unable to claim these rewards.
* This will result in the permanent freezing of yield (emissions) for the gauges.

## References

Add any relevant links to documentation or code

## Recommendation

* Update the code of the `distribute()` routine so that `IMinter(minter).updatePeriod()` is called before `_distribute().`
* Also, remove the timestamp check from `_distribute()` and place it in `distribute()`.

```
 function distribute() external {
+++     require(  block.timestamp >= IMinter(minter).activePeriod() IMinter(minter).DURATION(), "can only distribute after period end" );

+++      IMinter(minter).updatePeriod();

            uint256 start = 0;
            uint256 finish = pools.length;

            for (uint256 x = start; x < finish; x++) {
                // We don't revert if gauge is not alive since pools.length is not reduced
                if (isAlive[gauges[pools[x]]]) {  _distribute(gauges[pools[x]]); } 
---    IMinter(minter).updatePeriod();

        }
```

```
   function _distribute(address _gauge) internal {
--- require( block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),  "can only distribute after period end");

        uint256 _claimable = claimable[_gauge];
        // Reset claimable amount
        claimable[_gauge] = 0;

        _updateFor(_gauge);

        if (_claimable > 0) {
            IBaseGauge(_gauge).notifyRewardAmount(_claimable);
        }

        IBribe(bribes[_gauge]).resetVoting();

        emit DistributeReward(msg.sender, _gauge, _claimable);
    }

```

## Proof of Concept

I have created a test showcasing how the rewards sent during the first epoch are stuck in the voter contract for 3 epochs.

* For better understanding, re-run the test after implementing the recommended changes.
* Note that there will still be a lag between gauge\_balance and claimable\_amount (i.e., gauge\_balance will be updated in the next epoch each time), but that is due to a different bug.
* Add the test below to the voting.t.sol file of the test suite and run it with the following command:
  * forge test --fork-url <https://eth-mainnet.g.alchemy.com/v2/{Alchemy-api-key}> --match-test "testFirstEpochRewardsStuck" -vvv

```solidity

    function testFirstEpochRewardsStuck() public {
        
        uint256 period = minter.activePeriod();

        //first empty the gauge reciever (passthrough gauge) for better reward tracking 
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
        address[] memory gauges = new address[](1);//(10023200000000000000000-5037600000000000000000)/10023200000000000000000
        gauges[0] = address(sushiGauge);
        address gauge_reciever = sushiGauge.receiver();

        hevm.prank(admin);
        voter.vote(tokenId, pools, weights, 0);

        hevm.warp(period + nextEpoch);
        hevm.roll(block.number + 1);
        
        console2.log("1st call ");
        voter.distribute();

        //Ideally as Voter.distribute() is being called after epochs end,
        //therefore the rewards emitted during this epoch should be sent to the gauge
        //Instead of that, due to mentioned voter recieve the alcx tokens but these are 
        //neither transfer to the gauge nor the gauges claimable_amount is updated.
        uint256 claimable_amount = voter.claimable(address(sushiGauge)); // this will be non zero amount
        uint256 Gauge_balance = alcx.balanceOf(address(gauge_reciever));// Actual balance of Gauge is zero.
        uint256 voter_balance = alcx.balanceOf(address(voter));//some amount

        //As result,here claimable amount is zero  
        //first epoch emmision sent to the emissions are forever stuck in voter contract

        // assertEq(claimable_amount,0);
        // assertEq(Gauge_balance,0);
        // assertGt(voter_balance,0);

        console2.log("claimable amount  :",claimable_amount);
        console2.log("Gauge_balance     :",Gauge_balance);
        console2.log("voter_balance     :",voter_balance);
        console2.log("");

        //During second epoch, 
        //Atleast rewards will get updated into the claimable balance for gauge.
        //Due to another bug this are not being sent to the gauge at this instance (which they would have been).
        //nonetheless,first epochs balance is still stuck in the voter contract.
        hevm.warp(block.timestamp + nextEpoch);
        console2.log("2nd call ");
        voter.distribute();


        claimable_amount = voter.claimable(address(sushiGauge)); 
        Gauge_balance = alcx.balanceOf(address(gauge_reciever));
        voter_balance = alcx.balanceOf(address(voter));

        console2.log("claimable amount  :",claimable_amount);
        console2.log("Gauge_balance     :",Gauge_balance);
        console2.log("voter_balance     :",voter_balance);
        
        //first epoch reward stuck = voter_balance = - claimable_amount(to be sent)
        console2.log("Rewards stuck in the voter during (2nd Epoch) :", voter_balance - claimable_amount);
        console2.log("");  

        //Here we can see second epochs balance is now transfer to the gauge
        //3rd epoch balance is updated in claimable amount.
        //First epochs rewards are still stuck here
        hevm.warp(block.timestamp + nextEpoch);//5037599999999989148379 + 4985599999999982279363 - 9919200000000010851621
        console2.log("3rd call ");
        voter.distribute();

        claimable_amount = voter.claimable(address(sushiGauge));
        Gauge_balance = alcx.balanceOf(address(gauge_reciever));
        voter_balance = alcx.balanceOf(address(voter));

        console2.log("claimable amount  :",claimable_amount);
        console2.log("Gauge_balance     :",Gauge_balance);
        console2.log("voter_balance     :",voter_balance);

        //first epoch reward stuck = voter_balance - claimable_amount(to be sent)
        console2.log("Rewards stuck in the voter during (3rd Epoch) :", voter_balance - claimable_amount);
        console2.log(""); 

        //Now change the code as mentioned below for the distribute() routine of the Voter.sol and remove the timestamp check _distribute() routine
        //and place it in distribute() function before calling update.period function.
        //and re-run the test, there will only negible residue will stuck in the voter.sol only due to rounding error.

        //There will be still lag between gauge_balance adn claimable_amount (i.e gauge_balance will be updated in the next epoch each time) due to different bug.
        /*
    
        function distribute() external {
            uint256 start = 0;
            uint256 finish = pools.length;

            require(
                block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
                "can only distribute after period end"
            );

            IMinter(minter).updatePeriod();

            for (uint256 x = start; x < finish; x++) {
                // We don't revert if gauge is not alive since pools.length is not reduced
                if (isAlive[gauges[pools[x]]]) {
                    _distribute(gauges[pools[x]]);
                }
            }
            //IMinter(minter).updatePeriod();
        }


        function _distribute(address _gauge) internal {
            // Distribute once after epoch has ended
            // require(
            //     block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
            //     "can only distribute after period end"
            // );

            uint256 _claimable = claimable[_gauge];

            // Reset claimable amount
            claimable[_gauge] = 0;
            
            _updateFor(_gauge);

            if (_claimable > 0) {
                IBaseGauge(_gauge).notifyRewardAmount(_claimable);
            }

            IBribe(bribes[_gauge]).resetVoting();

            emit DistributeReward(msg.sender, _gauge, _claimable);
        }
        */
    }

```
