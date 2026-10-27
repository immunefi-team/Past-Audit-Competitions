# 60027 sc high stuck funds for the later delegators due to an edge case led to double decreasing effective stakes

**Submitted on Nov 17th 2025 at 17:37:44 UTC by @rzizah for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60027
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

{% stepper %}
{% step %}
In the Stargate contract, to exit a delegation while the validator status is active the `requestDelegationExit` is called which invokes `_updatePeriodEffectiveStake` to decrease effective stakes.
{% endstep %}

{% step %}
A validator that has signaled exit remains in an active state until the period ends. The scenario:

* A delegator of an exiting validator calls `requestDelegationExit`, decreasing effective stakes.
* The period ends and the validator reaches EXITED state.
* The delegator can re-delegate by calling `delegate`.
* In `delegate`, the code checks whether the current validator status is EXITED and (in that case) decreases effective stakes again.

Result: the effective stakes are decreased twice for the same delegator, causing later delegators to have stuck funds (can't claim rewards, can't unstake). There is no emergency rescue; stakes and rewards can be permanently stuck.
{% endstep %}
{% endstepper %}

### Vulnerability Details

Context: the protocol staker is mocked in tests. Validators can have multiple statuses (Doesn't exist, Queued, Active, Went offline, SignaledExit, Exited). Focus on SignaledExit:

* In SignaledExit, the validator is treated as active until the period ends. `validatorExitBlock` records the exit request block.
* In Stargate, validators that have signaled exit are treated as active, so a delegator wanting to re-delegate must call `requestDelegationExit`, which removes the delegation from the validator's effective stakes.

Code reference where `requestDelegationExit` decreases effective stake in a future period:

```solidity
// Stargate.sol
523: function requestDelegationExit(
// ...
567:     // decrease the effective stake
568:     _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

Later, after the period ends and the validator becomes EXITED, when the delegator calls `delegate` to a new validator the code again decreases the effective stake of the previous (now-exited) validator:

```solidity
// Stargate.sol
338: function _delegate(StargateStorage storage $, uint256 _tokenId, address _validator) private {
// ...
398:     if (
399:         currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
400:         status == DelegationStatus.PENDING
401:     ) {
402:         // get the completed periods of the previous validator
403:         (, , , uint32 oldCompletedPeriods) = $
404:             .protocolStakerContract
405:             .getValidationPeriodDetails(currentValidator);
406:         // decrease the effective stake of the previous validator
407:         _updatePeriodEffectiveStake(
408:             $,
409:             currentValidator,
410:             _tokenId,
411:             oldCompletedPeriods + 2,
412:             false // decrease
413:         );
414:     }
}
```

This yields a double decrease:

1. When `requestDelegationExit` was called while the validator was still treated as active (decrease for completedPeriods + 2).
2. Again in `_delegate` because the validator is now `EXITED` (decrease for oldCompletedPeriods + 2).

The function `_updatePeriodEffectiveStake` computes and writes the updated effective stake using subtraction if decreasing:

```solidity
// Stargate.sol
1044: function _updatePeriodEffectiveStake(
1045:     StargateStorage storage $,
1046:     address _validator,
1047:     uint256 _tokenId,
1048:     uint32 _period,
1049:     bool _isIncrease
1050: ) private {
1051:     // calculate the effective stake
1052:     uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
1053:
1054:     // get the current effective stake
1055:     uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);
1056:
1057:     // calculate the updated effective stake
1058:     uint256 updatedValue = _isIncrease
1059:         ? currentValue + effectiveStake
1060:         : currentValue - effectiveStake;
1061:
1062:     // push the updated effective stake
1063:     $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
1064: }
```

`delegatorsEffectiveStake` is updated by pushing the new value. The double decrease can cause an underflow or otherwise corrupt expected values used during reward calculations.

The claim logic calls `_claimableRewardsForPeriod` (via `_claimableRewards` -> `_claimRewards`) during `delegate` and `unstake`. If `delegatorsEffectiveStake` gets corrupted (e.g., underflow/overflow), claims can revert, leaving delegations stuck.

References in code where claim is called inside unstake and delegate:

* Claim inside unstake: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L303-L304>
* Claim inside delegate: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L438-L439>

Decrease effective stakes in delegate: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L407-L413>

Decrease effective stakes in requestDelegationExit: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L567-L569>

## Impact Details

* Permanent freezing of delegator funds: delegators cannot redelegate nor unstake; funds stuck.
* Permanent freezing of delegator rewards: rewards cannot be claimed due to arithmetic underflow/overflow in reward calculations, causing reverts.

## Proofs & Tests

The report includes small tests showing protocol staker behavior and a full POC integration test demonstrating the bug.

Notes on the protocol staker behavior:

* Delegation mapping persists after withdrawal (delegation ID still points to original validator data).
* A validator that has signaled exit remains in `ACTIVE` status until period end; after the period ends its status becomes `EXITED`.

Example Go test confirming mapping persistence:

```go
func Test_Delegation_Mapping_Persistence(t *testing.T) {
    // ... (omitted for brevity)
}
```

Sample logs from the Go test show mapping persists after withdrawal:

```
INITIAL: Validator=..., Stake=10000, LastIteration=<nil>
AFTER SIGNAL: Validator=..., Stake=10000, LastIteration=0xc000a0d0c0
WITHDRAWN: 10000 VET
AFTER WITHDRAW: Validator=..., Stake=0, LastIteration=0xc000a0dba0
✅ MAPPING PERSISTS AFTER WITHDRAWAL
```

Example Go test confirming validator exit-state logging:

```go
func Test_Validator_Exit_Status_Logging(t *testing.T) {
    // ... (omitted for brevity)
}
```

Logs:

```
Initial validator status: 2
During exit period: Validator status: 2
After exit period: Validator status: 3
```

## POC (Integration test)

The report provides a Hardhat test to reproduce the issue. Create a `local.ts` file under `packages/config/` with the provided config (kept unchanged).

Create `test/integration/DelegationExitBug.test.ts` with the provided content. The test walks through:

{% stepper %}
{% step %}

* Deploy mocks and contracts (ProtocolStakerMock, StargateNFTMock, VTHO token, etc.)
* Set validators and statuses to ACTIVE.
* Mint NFTs to users representing stake tokens.
  {% endstep %}

{% step %}

* user1 delegates (pending -> active after period).
* user2 delegates to same validator (pending -> active after period).
* user1 calls `requestDelegationExit` while validator still considered active; `_updatePeriodEffectiveStake` decreases future effective stake once.
* Validator signals exit (still active until period ends).
  {% endstep %}

{% step %}

* Advance period so user1's delegation becomes EXITED; validator status becomes EXITED.
* user1 then re-delegates to a different validator. Because the previous validator is now EXITED, `_delegate` decreases the effective stake again for the previous validator — causing a double-decrease for the same delegator.
  {% endstep %}

{% step %}

* user2 tries to unstake. The claim/unstake code runs and encounters an arithmetic overflow/underflow due to the corrupted effective stake values, causing revert and leaving user2 stuck.
  {% endstep %}
  {% endstepper %}

Test code (full test included as provided in the original report — keep exactly as-is when running):

```ts
// test/integration/DelegationExitBug.test.ts
// (Full test code as provided in the report — unchanged)
```

Run the test with:

```
VITE_APP_ENV=local npx hardhat test test/integration/DelegationExitBug.test.ts
```

Example log output from the test run (shows double decrease and revert at unstake):

```
Delegation Exit Bug POC

=== Step 1: user1 delegates to active validator ===
user1 delegation status: 1 (PENDING)
Validator A effective stake after user1 delegation increased: 1500000000000000000

=== Step 2: user2 delegates to same validator ===
user2 delegation status: 1 (PENDING)
Validator A effective stake after user2 delegation increased: 3000000000000000000
user1 delegation status: 2 (ACTIVE)
user2 delegation status: 2 (ACTIVE)

=== Step 3: user1 requests delegation exit, validator signals exit ===
user1 requested delegation exit
validator A signaled exit
user1 hasRequestedExit: true
user1 delegation status: 2 (ACTIVE)
Validator A effective stake after user1 delegation exit request decreases: 1500000000000000000

=== Step 4: Period 3 ends - user1's delegation exits, then re-delegates to different validator ===
user1 delegation status after period end: 3 (EXITED)
Validator A validation details: Result(6) [...]
Validator A status check - current: 3 expected: 3
Validator A status confirmed as EXITED before re-delegation
user1 new delegation status: 1 (PENDING)
Validator A effective stake after user1 re-delegation decreases again as validator is in exit state: 0
Validator B effective stake after user1 re-delegation increase: 1500000000000000000

=== Step 5: user2 attempts to unstake ===
user2 delegation status before unstake: 3
user2 delegation isLocked: false
user2 delegation endPeriod: 4294967295
❌ user2 unstake failed: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
```

Test result: failing unstake due to arithmetic overflow caused by double decrease.

## References

* Stargate.sol (requestDelegationExit decrease): <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L567-L569>
* Stargate.sol (delegate decrease on exited validator): <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L396-L414>
* \_updatePeriodEffectiveStake implementation: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L1044-L1064>
* Claim inside unstake: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L303-L304>
* Claim inside delegate: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L438-L439>

***

If you want, I can:

* Propose a minimal code patch to prevent the double-decrease (e.g., avoid decreasing in `_delegate` when the delegation already performed a scheduled decrease, or track whether a decrease was applied for the period), or
* Convert the provided POC test into a smaller unit test focused on the exact logic path (isolated reproduction). Which would you prefer?
