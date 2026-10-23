# 50433 sc high validator list griefing unrestricted stakeonbehalf allows user asset freeze permanently

**Submitted on Jul 24th 2025 at 15:24:20 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50433
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief/Intro

The `StakingFacet::stakeOnBehalf` allows any user to stake on behalf of another user without requiring any form of authorization or consent. A malicious actor can grief victims by associating their accounts with a large number of validators, potentially causing the victim's funds to be locked permanently.

### Vulnerability Details

The `stakeOnBehalf` function in the StakingFacet contract does not check if the staker has authorized the caller to stake on their behalf by any signature verification or delegation verification.

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);
...
}
```

As a result, any malicious address can use this function for any user, causing the user's `userValidators` array to be populated with many validators:

```solidity
mapping(address => uint16[]) userValidators; // user => list of validators they've staked with
```

Functions like `withdraw`, `restakeRewards`, and `restake` later iterate over this array within `_processMaturedCooldowns` and `_cleanupValidatorRelationships` multiple times. When the victim address attempts to restake or withdraw tokens, the transaction can revert due to a denial-of-service (DoS) caused by exceeding the gas limit.

## Impact Details

A malicious attacker can lock a victim’s staked funds and rewards by preventing them from calling `withdraw`, `restakeRewards`, or `restake`, as these functions will revert due to gas/iteration limits when iterating over an inflated `userValidators` array. This denial-of-service attack can result in permanent loss of access to funds, effectively freezing the victim’s assets within the staking contract.

## Proof of Concept

{% stepper %}
{% step %}

### Step

The attacker selects any user address ("victim") who has staked significant funds in the contract.
{% endstep %}

{% step %}

### Step

The attacker calls the `stakeOnBehalf` function (which does not require any signature, approval, or explicit permission from the victim), passing the victim's address as the `staker` parameter and choosing a validator ID.

Relevant code:

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, 
    ....
}

function _performStakeSetup(
  ...
    PlumeValidatorLogic.addStakerToValidator($, user, validatorId);
}
```

{% endstep %}

{% step %}

### Step

Repeat the call many times, each time with a different validator ID. Each call appends that validator to the victim's `userValidators` mapping array:

```solidity
mapping(address => uint16[]) userValidators; // user => list of validators they've staked with
```

{% endstep %}

{% step %}

### Step

The victim’s `userValidators` array becomes bloated with many entries. When the victim later calls `withdraw`, `restakeRewards`, or `restake`, the contract iterates over the entire `userValidators` array.

Example iteration in `_processMaturedCooldowns`:

```solidity
// StakingFacet::_processMaturedCooldowns
uint16[] memory userAssociatedValidators = $.userValidators[user];

for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
    uint16 validatorId = userAssociatedValidators[i];
    PlumeStakingStorage.CooldownEntry memory cooldownEntry = $.userValidatorCooldowns[user][validatorId];
    ...
}
```

And again in `_cleanupValidatorRelationships -> removeStakerFromAllValidators`:

```solidity
// PlumeValidatorLogic::removeStakerFromAllValidators

uint16[] memory userAssociatedValidators = $.userValidators[staker];

for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
    uint16 validatorId = userAssociatedValidators[i];
    if ($.userValidatorStakes[staker][validatorId].staked == 0) {
        removeStakerFromValidator($, staker, validatorId);
    }
}
```

{% endstep %}

{% step %}

### Step

With enough entries, these iterations hit gas limits and cause transactions to revert. As a result, the victim is unable to withdraw or restake their funds.
{% endstep %}

{% step %}

### Step

The victim cannot recover or restake their funds, resulting in a permanent denial-of-service unless the contract logic is fixed.
{% endstep %}
{% endstepper %}
