# 53056 sc low native withdraw to msg sender only non payable contract stakers cannot withdraw permanent funds lock&#x20;

**Submitted on Aug 14th 2025 at 18:24:58 UTC by @tansegv for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53056
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

`withdraw()` hardcodes the payout address to `msg.sender` and requires the native transfer to succeed. Contract wallets without a payable `receive`/`fallback` cannot accept native PLUME, causing `withdraw()` to revert and making matured funds unreachable.

The staking system pays out matured “parked” PLUME by sending native value to `msg.sender`. For contract-based stakers that lack a payable entry point, this transfer reverts, so `withdraw()` reverts as well. As a result, these users cannot withdraw matured funds at all.

### Vulnerability Details

* **Hardcoded payout target:** payout is always `msg.sender` (the caller). No `withdrawTo(...)` is offered.
* **Raw native send with `require`:**

```solidity
// StakingFacet.sol
function withdraw() external {
    ...
    uint256 amountToWithdraw = $.stakeInfo[user].parked;
    ...
    _removeParkedAmounts(user, amountToWithdraw);
    _cleanupValidatorRelationships(user);
    emit Withdrawn(user, amountToWithdraw);
    (bool success,) = user.call{ value: amountToWithdraw }("");
    if (!success) {
        revert NativeTransferFailed();
    }
}
```

* **Why this fails for many contracts:**\
  When a contract **without** a payable `receive()` or payable `fallback()` receives native value, the call **reverts**. Because the function requires success, the entire `withdraw()` reverts, making the funds effectively **inaccessible** to that account.

### Impact Details

{% hint style="danger" %}

* Impact: Permanent freezing of funds for affected users (contract accounts lacking a payable entry point). They can accumulate parked balances but can’t withdraw them.
* Blast radius: Any smart-account / contract wallet that doesn’t implement a payable receiver (or that is upgraded to drop payable receive) will be unable to exit to native.
* Persistence: Indefinite until protocol adds an alternative withdrawal path (e.g., withdraw-to or wrapped token payout). State changes before the failing transfer are reverted, so accounting stays consistent—but users remain blocked from accessing their funds.
  {% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Deploy a non-payable contract staker

Example:

```solidity
contract NonPayableStaker {
    // no `receive()` and no payable fallback
    function tryWithdraw(address staking) external {
        IStaking(staking).withdraw(); // will revert when staking sends native back
    }
}
```

{% endstep %}

{% step %}

### Fund and stake from the contract

Fund the contract (e.g., in a test harness) and call `stake(validatorId)` sending native from that contract.
{% endstep %}

{% step %}

### Unstake and wait for cooldown

Unstake to start the cooldown; wait for cooldown to mature so funds become "parked".
{% endstep %}

{% step %}

### Attempt withdraw from the non-payable contract

From `NonPayableStaker`, call `withdraw()`.

* Internally: `(bool success,) = msg.sender.call{value: amount}("")` → reverts because there is no payable receive.
* The function reverts with `NativeTransferFailed()`. Funds remain parked and cannot be withdrawn by this staker.
  {% endstep %}
  {% endstepper %}

(Note: In Foundry you can `vm.deal(nonPayable, X)` to fund it without a payable receiver, then run the above flow and assert a revert on `withdraw()`.)
