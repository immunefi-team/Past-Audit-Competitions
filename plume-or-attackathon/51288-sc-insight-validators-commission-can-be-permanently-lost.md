# 51288 sc insight validators commission can be permanently lost

**Submitted on Aug 1st 2025 at 13:07:43 UTC by @Outliers for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51288
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

Validator commission claims become permanently locked if the validator's `l2WithdrawAddress` gets blacklisted by reward tokens like `USDC` or `USDT`. When a commission claim is requested, the system locks the claim to the original withdraw address and prevents new claims from being initiated, creating an irrecoverable loss scenario.

It should be noted that blacklisting is not a rare event — for instance, by the time this report was written, USDC had 371 blacklisted addresses, while USDT had 2,378 blacklisted addresses (<https://dune.com/phabc/usdt---banned-addresses>), and reasons for blacklisting vary; legitimate addresses can also be blacklisted (<https://bglaw.eu/articles/tether-and-circle-blocking-usdt-and-usdc-addresses/>).

> Stable-coins like USDT and USDC are frequently used in fraud or scams. If an exchange or platform detects fraudulent activity linked to a specific wallet address, they may freeze that address to prevent further abuse or loss. Unfortunately however, alongside with the criminal transactions, many legitimate addresses are also blocked.

## Vulnerability Details

The issue stems from the interaction between three key functions in the commission claiming process.

addValidator - Sets the `l2WithdrawAddress` that will receive commission claims

{% code title="addValidator (snippet)" %}

```solidity
  PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];
        validator.validatorId = validatorId;
        validator.commission = commission;
        validator.delegatedAmount = 0;
        validator.l2AdminAddress = l2AdminAddress;
        validator.l2WithdrawAddress = l2WithdrawAddress; <<@audit
        validator.l1ValidatorAddress = l1ValidatorAddress;
        validator.l1AccountAddress = l1AccountAddress;
        validator.l1AccountEvmAddress = l1AccountEvmAddress;
        validator.active = true;
        validator.slashed = false;
        validator.maxCapacity = maxCapacity;
```

{% endcode %}

requestCommissionClaim : Validator requests commission claim for `USDC`/`USDT`

{% code title="requestCommissionClaim (snippet)" %}

```solidity
 function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId) 
        nonReentrant
        _validateValidatorExists(validatorId)
        _validateIsToken(token)
    {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

        if (!validator.active || validator.slashed) {
            revert ValidatorInactive(validatorId);
        }

        // Settle commission up to now to ensure accurate amount
        PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);
        uint256 amount = $.validatorAccruedCommission[validatorId][token];
        if (amount == 0) {
            revert InvalidAmount(0);
        }
        if ($.pendingCommissionClaims[validatorId][token].amount > 0) { 
            revert PendingClaimExists(validatorId, token); //<@audit
        }
        address recipient = validator.l2WithdrawAddress; <@audit
        uint256 nowTs = block.timestamp;
        $.pendingCommissionClaims[validatorId][token] = PlumeStakingStorage.PendingCommissionClaim({
            amount: amount,
            requestTimestamp: nowTs,
            token: token,
            recipient: recipient
        });
        // Zero out accrued commission immediately
        $.validatorAccruedCommission[validatorId][token] = 0;

        emit CommissionClaimRequested(validatorId, token, recipient, amount, nowTs);
    }
```

{% endcode %}

finalizeCommissionClaim - Attempts to transfer tokens to the blacklisted `l2WithdrawAddress` via `distributeReward`

{% code title="finalizeCommissionClaim (snippet)" %}

```solidity
 function finalizeCommissionClaim(
        uint16 validatorId,
        address token
    ) external onlyValidatorAdmin(validatorId) nonReentrant returns (uint256) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

        PlumeStakingStorage.PendingCommissionClaim storage claim = $.pendingCommissionClaims[validatorId][token];

        if (claim.amount == 0) {
            revert NoPendingClaim(validatorId, token);
        }

        uint256 readyTimestamp = claim.requestTimestamp + PlumeStakingStorage.COMMISSION_CLAIM_TIMELOCK;

        // First, check if the timelock has passed from the perspective of the current block.
        if (block.timestamp < readyTimestamp) {
            revert ClaimNotReady(validatorId, token, readyTimestamp);
        }

        // --- REVISED SLASHING CHECK ---
        // If the validator is slashed, the claim is only considered valid if its timelock was
        // fully completed BEFORE the slash occurred. This invalidates any pending claims.
        if (validator.slashed && readyTimestamp >= validator.slashedAtTimestamp) {
            revert ValidatorInactive(validatorId);
        }

        // For a non-slashed validator, simply require it to be active to finalize a claim.
        if (!validator.slashed && !validator.active) {
            revert ValidatorInactive(validatorId);
        }

        uint256 amount = claim.amount;
        address recipient = claim.recipient; //<@audit
        // Clear pending claim
        delete $.pendingCommissionClaims[validatorId][token];
        // Transfer from treasury
        address treasury = RewardsFacet(address(this)).getTreasury();
        if (treasury == address(0)) {
            revert TreasuryNotSet();
        }

        IPlumeStakingRewardTreasury(treasury).distributeReward(token, amount, recipient);
        emit CommissionClaimFinalized(validatorId, token, recipient, amount, block.timestamp);
        return amount;
    }
```

{% endcode %}

When `finalizeCommissionClaim` is called, it attempts to transfer tokens via `distributeReward` which internally calls: soliditySafeERC20.safeTransfer(IERC20(token), recipient, amount);

If the recipient is blacklisted by the token contract this transfer will revert. Because the code deletes the pending claim only after preparing the transfer (but before calling distributeReward) — actually in this snippet the claim is deleted before calling distributeReward; however the transfer fails and the claim ends up effectively blocking further claims (the revert prevents state changes from persisting in the call that attempted the transfer; the design still relies on a single recipient persisted by request). The net effect is that the existing claim logic prevents new claims from being initiated until the pending claim is removed, and there is no mechanism to:

* update the withdraw recipient for a pending claim,
* cancel a pending claim,
* recover or redirect the locked funds if the recipient is blacklisted.

Therefore a blacklisted withdraw address causes claim finalization to fail and leaves commission unclaimable.

## Impact Details

* Permanent loss of validator commissions for affected tokens
* Ongoing accumulation of unclaimable rewards as commission continues to accrue

## References

Add any relevant links to documentation or code

## Proof of Concept

{% stepper %}
{% step %}

### Step

Validator added with withdraw address 0xABC...
{% endstep %}

{% step %}

### Step

Validator earns commission and requests claim:

```
requestCommissionClaim(1, USDC_ADDRESS);
// Claim locked to recipient = 0xABC...
```

{% endstep %}

{% step %}

### Step

Address 0xABC... gets blacklisted by USDC:

```
// USDC.transfer(0xABC..., amount) will revert
```

{% endstep %}

{% step %}

### Step

Attempt to finalize claim fails:

```
finalizeCommissionClaim(1, USDC_ADDRESS);
// Reverts due to blacklisted address in SafeERC20.safeTransfer
```

{% endstep %}

{% step %}

### Step

Attempt to request new claim fails:

```
requestCommissionClaim(1, USDC_ADDRESS);
// Reverts with PendingClaimExists(1, USDC_ADDRESS)
```

{% endstep %}

{% step %}

### Step

Commission continues to accrue but remains permanently unclaimable:

```
// No function exists to:
// - Update the request withdraw address
// - Cancel the pending claim
// - Recover the locked funds
```

{% endstep %}
{% endstepper %}
