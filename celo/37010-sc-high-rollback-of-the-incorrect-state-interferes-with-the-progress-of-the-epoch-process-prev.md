# #37010 \[SC-High] Rollback of the incorrect state interferes with the progress of the epoch process,

**Submitted on Nov 22nd 2024 at 05:10:33 UTC by @innertia for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37010
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/contracts-0.8/common/EpochManager.sol>
* **Impacts:**
  * Temporary freezing of funds
  * Smart contract unable to operate due to lack of token funds
  * Griefing
  * Issuance of an unauthorized amount of StableToken (not to the attacker)
  * Blocking of execution of functions of other contracts

## Description

## Brief/Intro

`EpochProcess` is managed by `EpochProcessStatus`, which has three statuses: `NotStarted`, `Started`, and `IndivudualGroupsProcessing`. They should change and cycle in this order, and each status has a different function that can be started. However, after being changed from `Started` to `IndivudualGroupsProcessing`, there is a function launch route that reverts back to `Started`. This causes various problems such as prevents the user from receiving rewards, blocks the launch of the associated contract function, etc.

## Vulnerability Details

In `startNextEpochProcess`, the status is changed to `Started`, assuming that the epoch has not already started.

```
    require(!isOnEpochProcess(), "Epoch process is already started");
    epochProcessing.status = EpochProcessStatus.Started;
```

<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L194-L196>

Then, in the next `setToProcessGroups`, the status is changed to `IndivudualGroupsProcessing`, assuming the epoch has started.

```
    require(isOnEpochProcess(), "Epoch process is not started");
    EpochProcessState storage _epochProcessing = epochProcessing;
    _epochProcessing.status = EpochProcessStatus.IndivudualGroupsProcessing;
```

<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L224-L227>

Let's check the `isOnEpochProcess` here.

```
function isOnEpochProcess() public view returns (bool) {
    return epochProcessing.status == EpochProcessStatus.Started;
  }
```

<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L612-L614>

It only checks whether the status is `Started`. In other words, it is possible to start `startNextEpochProcess` at this point, and this can be looped as many times as you like. This will have various effects. These are shown below.

## Impact Details

* Stopping of functions due to interference with state transitions In order to end an epoch, it is necessary to invoke either `processGroup` or `finishNextEpochProcess`. However, it is possible to switch the status to `Started` or `IndividualGroupsProcessing` using front-running, etc., and to prevent the invocation of these functions. As a result, it is not possible to end the epoch while the attack is being carried out. This is a fundamental flaw in the protocol. Furthermore, the `isBlocked` function can also prevent the activation of functions in other contracts that rely on it. In addition, the state transition will not progress, and it will not be possible to receive the reward for the epoch.

```
  function isBlocked() external view returns (bool) {
    return isOnEpochProcess();
  }
```

<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L454-L456>

* Unplanned minting and movement of tokens. The `startNextEpochProcess` internally starts the `allocateValidatorsRewards`. This has a function to mint the cUSD required for rewards, and by repeating the cycle described above, cUSD that is not scheduled will be minted one after another. This will destroy the protocol's issuance plan. In some cases, it may be impossible to issue cUSD due to reaching the upper limit. There is also a function that sends CeloTokens from the `CeloUnreleasedTreasury` to the `RESERVE_REGISTRY`. If this is looped, an illegal amount of tokens will be sent to the `RESERVE`. There is also a possibility that the tokens in the `CeloUnreleasedTreasury` will run out and this function will no longer be able to be activated.

  // Mint all cUSD required for payment and the corresponding CELO validators.mintStableToEpochManager(totalRewards); <https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L662C17-L663>

  getCeloUnreleasedTreasury().release( registry.getAddressForOrDie(RESERVE\_REGISTRY\_ID), CELOequivalent ); <https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L670-L673>

## References

<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L194-L196.\\>
<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L224-L227.\\>
<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L612-L614.\\>
<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L454-L456.\\>
<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L662C17-L663.\\>
<https://github.com/celo-org/celo-monorepo/blob/7b68b9816632e7458a975cfc2d6f36ce057b02a9/packages/protocol/contracts-0.8/common/EpochManager.sol#L670-L673>.

## Proof of Concept

## Proof of Concept

Add a test to `EpochManagerTest_setToProcessGroup` contract in the following file. <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/test-sol/unit/common/EpochManager.t.sol>

```
function test_IncorrectRollbackOfState() public {

(address[] memory groups,,) = getGroupsWithLessersAndGreaters();

  

//Start startNextEpochProcess

epochManager.startNextEpochProcess();

(uint256 status_first,,,,) = epochManager.getEpochProcessingState();

//status_first is 1 (Started).

assertEq(status_first, 1);

  

//Start setToProcessGroups

epochManager.setToProcessGroups();

(uint256 status_second,,,,) = epochManager.getEpochProcessingState();

//status_second is 2 (IndivudualGroupsProcessing)

assertEq(status_second, 2);

  

//Start the startNextEpochProcess again

epochManager.startNextEpochProcess();

(uint256 status_third,,,,) = epochManager.getEpochProcessingState();

//status_third reverts to started. You can repeat this.

assertEq(status_third, 1);

}
```
