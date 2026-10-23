# 51201 sc low contracts without payable entry points cannot withdraw nor claim rewards

**Submitted on Jul 31st 2025 at 22:03:36 UTC by @jovi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51201
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/PlumeStakingRewardTreasury.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Theft of unclaimed yield

## Description

### Brief / Intro

A missing “is-payable” check on stake admission means any non-payable contract can become a staker. Later, every native-coin transfer to that address reverts, freezing both yield and unstake withdraw claims.

### Vulnerability Details

Both reward and principal payouts are executed with raw native transfers:

```solidity
(bool success, ) = user.call{value: amountToWithdraw}("");
if (!success) revert NativeTransferFailed();   // StakingFacet.sol#L416-419
```

```solidity
(bool success, ) = recipient.call{value: amount}("");
if (!success) revert PlumeTransferFailed(recipient, amount); // PlumeStakingRewardTreasury.sol#L179-182
```

If the staker address is a smart contract without a payable `receive()` or `fallback()`, these calls always revert. The protocol nevertheless lets such contracts open stakes, so:

* Rewards accumulate but can never be claimed.
* Unstaked principal moves to the parked/withdrawable pool but `withdraw()` reverts.

The user’s rewards and entire stake end up locked permanently.

## Impact

| Asset / Function           | Outcome                                            |
| -------------------------- | -------------------------------------------------- |
| **Staked PLUME principal** | **Permanent freeze** — `withdraw()` always reverts |
| **Accrued native rewards** | Unclaimable                                        |

## Recommendation

{% stepper %}
{% step %}

### Admission guard

Introduce an admission guard when someone opts to stake. Make sure to also have reentrancy protection so that the caller cannot exploit this mechanism. The 1 wei call ensures the staker is able to receive native amounts.

```solidity
require(msg.value > 0, "Msg.value must be bigger than 1");
require(
    staker.code.length == 0 || address(staker).call{value:1}(""),
    "NON_PAYABLE_STAKER"
);
```

{% endstep %}

{% step %}

### Rescue mechanism

Allow one-time designation of an alternate payable address if a native transfer fails.
{% endstep %}

{% step %}

### Fail-safe escrow

Wrap calls in `try/catch`; credit failed amounts to an escrow users can sweep with an EOA.
{% endstep %}

{% step %}

### Wrap native token

Consider wrapping the native token so that users receive back an ERC20 version of it.
{% endstep %}
{% endstepper %}

## Proof of Concept

{% stepper %}
{% step %}
Deploy a dummy contract without `receive()`/`fallback()`.
{% endstep %}

{% step %}
Stake PLUME to any validator, using the dummy contract as staker.
{% endstep %}

{% step %}
Unstake after some time; wait for cooldown to mature.
{% endstep %}

{% step %}
Call `withdraw()`\
→ tx reverts with `NativeTransferFailed`; principal remains trapped.
{% endstep %}

{% step %}
(Optional) Call `claimRewards()` before unstaking; observe identical revert for rewards.
{% endstep %}
{% endstepper %}
