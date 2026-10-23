# 59809 sc high user balances are permanently frozen in specific delegation scenarios

**Submitted on Nov 16th 2025 at 02:59:50 UTC by @hrmneffdii for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59809
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

This report details a critical vulnerability discovered in the `Stargate` contract's delegation logic. A flaw in the `_delegate` function can cause a validator's `delegatorsEffectiveStake` to be improperly reduced twice for the same user.

This "double-decrease" bug desynchronizes the validator's internal stake accounting, leading to an arithmetic underflow when another user attempts to interact with their own stake. As a result, other delegators on the same validator will have their funds permanently frozen, as all management functions (e.g., `delegate`, `requestDelegationExit`, `unstake`) will fail.

### Vulnerability details

{% stepper %}
{% step %}

### Path: requestDelegationExit()

When a user (e.g., "Bob" from the PoC) calls `requestDelegationExit` (Step 5 in PoC), the function correctly calls `_updatePeriodEffectiveStake(..., isIncrease: false)`. This reduces the validator's `EffectiveStake` to reflect the user's exit. This is the expected behavior.
{% endstep %}

{% step %}

### Path: \_delegate() (during re-delegation)

Later, the validator exits. If the user then calls `_delegate` to move their stake to a new validator, the logic in `_delegate` can also reduce the original validator's `EffectiveStake` again:

* `_getDelegationStatus` returns `DelegationStatus.EXITED` for the token.
* This triggers the conditional `if (status == DelegationStatus.EXITED || status == DelegationStatus.PENDING)` which withdraws the user's VET via `withdrawDelegation`.
* Later, a second condition `if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || status == DelegationStatus.PENDING)` is true and the code calls `_updatePeriodEffectiveStake(..., isIncrease: false)` a second time.

The code does not account for the stake already having been subtracted during `requestDelegationExit`, causing a double-decrease of the same user's stake from the validator's `EffectiveStake`.
{% endstep %}
{% endstepper %}

### Destructive scenario (PoC summary)

{% stepper %}
{% step %}
A validator has EffectiveStake = 3e18 (Alice 1.5e18 + Bob 1.5e18).
{% endstep %}

{% step %}
Bob calls `requestDelegationExit`. Validator EffectiveStake becomes 1.5e18 (3e18 - 1.5e18). (Correct)
{% endstep %}

{% step %}
Validator exits. Bob calls `delegate` to a new validator. Due to the bug, the code reduces the validator's EffectiveStake again, resulting in EffectiveStake = 0 (1.5e18 - 1.5e18). (Incorrect)
{% endstep %}

{% step %}
Alice (still delegated to the exited validator with 1.5e18) attempts to call `delegate` to move her funds.
{% endstep %}

{% step %}
The `_delegate` logic for Alice attempts `_updatePeriodEffectiveStake(..., isIncrease: false)`:

* currentValue = 0 (incorrect)
* effectiveStake (Alice) = 1.5e18
* updatedValue = currentValue - effectiveStake -> 0 - 1.5e18 This causes an arithmetic underflow (Solidity 0.8.x+), and Alice's transaction reverts.
  {% endstep %}

{% step %}
The same underflow occurs for Alice if she calls `requestDelegationExit` or other management operations; her funds become effectively frozen with no in-contract fix available.
{% endstep %}
{% endstepper %}

{% hint style="danger" %}
Impact: Critical — permanent and irrecoverable freeze of user funds. If one user on a validator executes the sequence (requestDelegationExit, then after validator exit re-delegate), remaining delegators on that validator can be permanently prevented from moving or exiting their stakes due to underflow reverts.
{% endhint %}

## Proof of Concept

### How to run

Run the following test: yarn contracts:test:unit:verbose -- -- --grep "User balances are permanently frozen in specific delegation scenarios"

### Test Case (excerpt)

```typescript
// audit-comp-vechain-stargate-hayabusa/packages/contracts/test/unit/Stargate/Delegation.test.ts

    it("User balances are permanently frozen in specific delegation scenarios", async () => {
        // Defining the actor
        let Alice = user;
        let Bob = otherUser;
        let completePeriod;

        // 1. Alice and Bob stake to Stargate
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(Alice).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenIdAlice = await stargateNFTMock.getCurrentTokenId();

        tx = await stargateContract.connect(Bob).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenIdBob = await stargateNFTMock.getCurrentTokenId();

        console.log("1. Alice tokenId", tokenIdAlice.toString(), " Bob tokenId", tokenIdBob.toString());

        // 2. Alice and Bob delegate their tokenId
        tx = await stargateContract.connect(Alice).delegate(tokenIdAlice, validator.address);
        await tx.wait();
        let delegationStatusAlice = await stargateContract.getDelegationStatus(tokenIdAlice);
        expect(delegationStatusAlice).to.equal(DELEGATION_STATUS_PENDING);

        tx = await stargateContract.connect(Bob).delegate(tokenIdBob, validator.address);
        await tx.wait();
        let delegationStatusBob = await stargateContract.getDelegationStatus(tokenIdBob);
        expect(delegationStatusBob).to.equal(DELEGATION_STATUS_PENDING);

        console.log("2. Alice and Bob successfully delegate their tokenId");

        const delegationIdAlice = await stargateContract.getDelegationIdOfToken(tokenIdAlice);
        const delegationIdBob = await stargateContract.getDelegationIdOfToken(tokenIdBob);

        // 3. Set validator completed periods to make delegateId active
        completePeriod = 1;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, completePeriod);
        await tx.wait();
        expect(await stargateContract.getDelegationStatus(tokenIdAlice)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );
        expect(await stargateContract.getDelegationStatus(tokenIdBob)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );
        console.log("3. Set validator completed periods to make their delegation active");
        console.log("   Validator effective stake", await stargateContract.getDelegatorsEffectiveStake(validator.address, completePeriod + 2));

        // 4. Advanced completed periods and validator will exit
        completePeriod = 100;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, completePeriod);
        await tx.wait();
        console.log("4. Advanced completed period and validator will exit");

        // 5. Bob request to exit
        tx = await stargateContract.connect(Bob).requestDelegationExit(tokenIdBob);
        await tx.wait();
        console.log("5. Bob request delegation exit successfully");
        console.log("   Validator effective stake", await stargateContract.getDelegatorsEffectiveStake(validator.address, completePeriod + 2));

        // 6. Validator exits after advanced completed periods
        tx = await protocolStakerMock.helper__setValidatorStatus(validator.address, 3);
        await tx.wait();
        console.log("6. Validator has been exited");

        // 7. Bob is going to delegate to another validator
        tx = await stargateContract.connect(Bob).delegate(tokenIdBob, otherValidator.address);
        await tx.wait();
        delegationStatusBob = await stargateContract.getDelegationStatus(tokenIdBob);
        expect(delegationStatusBob).to.equal(DELEGATION_STATUS_PENDING);
        console.log("7. Bob successfully delegates his token Id to another validator");
        console.log("   Validator effective stake", await stargateContract.getDelegatorsEffectiveStake(validator.address, completePeriod + 2));

        // 8. Alice is going to delegate to another validator as well, but reverts due to effective stake underflow
        await expect(
            stargateContract.connect(Alice).delegate(tokenIdAlice, otherValidator.address)
        ).to.be.reverted;
        console.log("8. Alice want to delegate her token Id but fails");

        // 9. Alice is going to request delegation exit, but reverts due to effective stake underflow
        await expect(
            stargateContract.connect(Alice).requestDelegationExit(tokenIdAlice)
        ).to.be.reverted;
        console.log("9. Alice want to request delegate exit but fails");
    })
```

<details>

<summary>Logs</summary>

```bash
  shard-u2: Stargate: Delegation
1. Alice tokenId 10001  Bob tokenId 10002
2. Alice and Bob successfully delegate their tokenId
3. Set validator completed periods to make their delegation active
   Validator effective stake 3000000000000000000
4. Advanced completed period and validator will exit
5. Bob request delegation exit successfully
   Validator effective stake 1500000000000000000
6. Validator has been exited
7. Bob successfully delegates his token Id to another validator
   Validator effective stake 0
8. Alice want to delegate her token Id but fails
9. Alice want to request delegate exit but fails
```

</details>
