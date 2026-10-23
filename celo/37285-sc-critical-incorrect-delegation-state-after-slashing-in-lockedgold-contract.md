# #37285 \[SC-Critical] Incorrect Delegation State After Slashing in LockedGold Contract

**Submitted on Dec 1st 2024 at 16:37:37 UTC by @jovi for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37285
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/contracts/governance/LockedGold.sol>
* **Impacts:**
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

The `LockedGold` contract fails to properly update the delegated voting power when an account is slashed. Specifically, the contract neglects to call key functions like `_updateDelegatedAmount` and `revokeFromDelegatedWhenUnlocking` for the slashed account. This leads to an inconsistent state where delegatees retain voting power based on the pre-slashing balance, causing discrepancies in governance integrity.

## Vulnerability Details

### **Context**

The `LockedGold` contract is responsible for managing locked CELO and delegation of governance votes. To ensure the accuracy of voting power, it employs two key functions:

1. **`_updateDelegatedAmount`**:
   * **Purpose**: Recalculates the CELO delegated to each delegatee based on the current total locked balance of the delegator.
   * **Usage**: Invoked after any operation that modifies the total locked gold (e.g., `lock` or `relock`).
   * **Importance**: Ensures that delegated voting power is proportionally updated to reflect the delegator’s current locked gold.
2. **`revokeFromDelegatedWhenUnlocking`**:
   * **Purpose**: Adjusts delegation amounts when gold is being unlocked, ensuring that delegatees do not retain voting power over funds no longer locked.
   * **Usage**: Invoked during the unlocking process before the balance is reduced.
   * **Importance**: Prevents delegatees from holding inflated voting power after funds are unlocked.

These functions are systematically invoked during standard balance-altering operations such as:

* **`lock()`**: Updates delegated amounts after locking additional CELO.
* **`unlock()`**: Revokes delegations for the unlocked amount.
* **`relock()`**: Recalculates delegations after relocking previously unlocked gold.

However, the `slash()` function fails to invoke these critical updates.

### Vulnerability

When an account is slashed using the `slash()` function, the contract reduces the account’s locked balance but does not update the associated delegated amounts. This creates a discrepancy between the account's actual locked balance and the delegatees' voting power.

Key issues:

1. **Delegation Discrepancy**: The slashed account’s delegatees retain voting power based on the pre-slashed balance.
2. **Reporter Delegation State**: The slasher’s account does not properly reflect delegation power from the reward.
3. **Incorrect Temporary State**: While the `updateDelegatedAmount()` function is publicly available for permissionless updates, the temporary inconsistency can be exploited.

The following snippet illustrates the issue:

```solidity
function slash(
    address account,
    uint256 penalty,
    address reporter,
    uint256 reward,
    address[] calldata lessers,
    address[] calldata greaters,
    uint256[] calldata indices
  ) external onlySlasher {
...
_decrementNonvotingAccountBalance(account, maxSlash.sub(difference));
_incrementNonvotingAccountBalance(reporter, reward);
...
}
```

This discrepancy creates an inconsistent state in the delegation system, where the actual locked gold doesn't match the delegated amounts temporarily.

Even though there is the `updateDelegatedAmount` public function that allows permissionless updates of delegated amounts, allowing this explicit transition to a wrong state during slashes may be utilized by exploits to falsely display states that have more voting power than they should.

## Impact Details

1. **Governance Integrity**: Delegatees retain incorrect voting power after slashes, undermining the fairness and accuracy of governance decisions.
2. **Economic Imbalance**:
   * Slashed accounts effectively retain an unfair delegation advantage.
   * Slasher delegatees may not gain immediate delegation power, leading to delayed or inaccurate representation.

## Proof of Concept

## Proof of Concept

The following test demonstrates the vulnerability. It creates a delegator account, locks CELO, delegates voting power, and then slashes the account. The test shows that the delegatee retains voting power based on the pre-slashed balance.

### Test Code

To demonstrate this vulnerability paste the following modified code snippet at the `LockedGold.t.sol` test file:

```solidity
contract LockedGoldTest_slash is LockedGoldTest {
  string slasherName = "DowntimeSlasher";
  uint256 value = 1000;
  address group = actor("group");
  address groupMember = actor("groupMember");
  address reporter = actor("reporter");
  address downtimeSlasher = actor(slasherName);
  address delegatee = actor("delegatee");

  Election electionSlashTest;

  address delegatee1 = actor("delegatee1");
  address delegatee2 = actor("delegatee2");
  address delegatee3 = actor("delegatee3");
  address delegator = actor("delegator");
  address delegator2 = actor("delegator2");

  address delegatorSigner;
  uint256 delegatorSignerPK;
  address delegatorSigner2;
  uint256 delegatorSigner2PK;
  address delegateeSigner1;
  uint256 delegateeSigner1PK;
  address delegateeSigner2;
  uint256 delegateeSigner2PK;

  uint256 percentToDelegate = 30;
  uint256 delegatedAmount = (value * percentToDelegate) / 100;

  uint256 percentToDelegate1 = 30;
  uint256 percentToDelegate2 = 20;
  uint256 percentToDelegate3 = 50;
  uint256 delegatedAmount1 = (value * percentToDelegate1) / 100;
  uint256 delegatedAmount2 = (value * percentToDelegate2) / 100;
  uint256 delegatedAmount3 = (value * percentToDelegate3) / 100;

  function setUp() public {
    super.setUp();
    electionSlashTest = new Election(true);
    registry.setAddressFor("Election", address(electionSlashTest));
    electionSlashTest.initialize(
      address(registry),
      4,
      6,
      3,
      FixidityLib.newFixedFraction(1, 100).unwrap()
    );

    address[] memory members = new address[](1);
    members[0] = groupMember;

    validators.setMembers(group, members);
    registry.setAddressFor("Validators", caller);
    electionSlashTest.markGroupEligible(group, address(0), address(0));
    registry.setAddressFor("Validators", address(validators));
    validators.setNumRegisteredValidators(1);

    lockedGold.lock.value(value)();
    registry.setAddressFor(slasherName, downtimeSlasher);
    lockedGold.addSlasher(slasherName);

    vm.prank(reporter);
    accounts.createAccount();

    
    vm.prank(delegatee1);
    accounts.createAccount();
    vm.prank(delegatee2);
    accounts.createAccount();
    vm.prank(delegatee3);
    accounts.createAccount();
    vm.prank(delegator);
    accounts.createAccount();
    vm.prank(delegator2);
    accounts.createAccount();

    (delegatorSigner, delegatorSignerPK) = actorWithPK("delegatorSigner");
    (delegatorSigner2, delegatorSigner2PK) = actorWithPK("delegatorSigner2");
    (delegateeSigner1, delegateeSigner1PK) = actorWithPK("delegateeSigner1");
    (delegateeSigner2, delegateeSigner2PK) = actorWithPK("delegateeSigner2");

    vm.deal(delegator, 10 ether);
    vm.deal(delegator2, 10 ether);
  }

  function whenVoteSigner_LockedGoldDelegateGovernanceVotes() public {
    helper_WhenVoteSigners(
      WhenVoteSignerStruct(
        delegator,
        delegator2,
        delegatee1,
        delegatee2,
        delegatorSignerPK,
        delegateeSigner1PK,
        delegatorSigner2PK,
        delegateeSigner2PK,
        true
      )
    );
  }

function helper_WhenAccountIsSlashedForAllOfItsLockedGold(
  uint256 penalty,
  uint256 reward
) public {
  address[] memory lessers = new address[](1);
  lessers[0] = address(0);
  address[] memory greaters = new address[](1);
  greaters[0] = address(0);

  uint256[] memory indices = new uint256[](1);
  indices[0] = 0;

  vm.prank(downtimeSlasher);
  lockedGold.slash(delegator, penalty, reporter, reward, lessers, greaters, indices);
}

   function test_POC_ShouldReduceAccountsLockedGoldBalance_WhenAccountIsSlashedForAllOfItsLockedGold()
    public
  {
    uint256 penalty = value;
    uint256 reward = value / 2;

    whenVoteSigner_LockedGoldDelegateGovernanceVotes();

    vm.prank(delegator);
    lockedGold.delegateGovernanceVotes(delegatee1, FixidityLib.newFixedFraction(30, 100).unwrap());
    assertEq(lockedGold.getAccountNonvotingLockedGold(delegator), 1000);
    assertEq(lockedGold.getAccountTotalLockedGold(delegator), 1000);
    assertEq(lockedGold.getAccountTotalGovernanceVotingPower(delegatee1), 300);

    helper_WhenAccountIsSlashedForAllOfItsLockedGold(penalty, reward);
    assertEq(lockedGold.getAccountNonvotingLockedGold(delegator), 0);
    assertEq(lockedGold.getAccountTotalLockedGold(delegator), 0);
    assertEq(lockedGold.getAccountTotalGovernanceVotingPower(delegatee1), 300);

  }
}
```

Run the tests with the following command:

```shell
forge test --match-test test_POC_ShouldReduceAccountsLockedGoldBalance_WhenAccountIsSlashedForAllOfItsLockedGold -vv
```
