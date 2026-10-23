# #46210 \[SC-Insight] Incorrect timestamp comparison in function "beforeCollateralWithdrawal" allows agent to withdraw at last second without being challenged

**Submitted on May 26th 2025 at 17:08:55 UTC by @dldLambda for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46210
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/AgentsExternal.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A logic flaw in the withdrawal mechanism allows an agent to finalize a withdrawal without accounting for valid challenges submitted in the final second before the allowed withdrawal timestamp. In particular, due to the use of a non-strict inequality (>=) in the line "require(block.timestamp >= withdrawal.allowedAt, "withdrawal: not allowed yet");" , the last second is unintentionally included in the range allowed for output, which is incorrect.

## Vulnerability Details

The bug is contained in the following lines (function "beforeCollateralWithdrawal"):

```
require(block.timestamp >= withdrawal.allowedAt, "withdrawal: not allowed yet");
```

In function "announceWithdrawal" variable "withdrawal.allowedAt" is created as follows:

```
withdrawal.allowedAt = (block.timestamp + settings.withdrawalWaitMinSeconds).toUint64();
```

This is what is meant by "settings.withdrawalWaitMinSeconds" - this is the period of time during which the agent must wait before he can withdraw funds, this is necessary so that other participants have time to challenge his actions if this agent is malicious:

```
        // Agent has to announce any collateral withdrawal ar vault destroy and then wait for at least
        // withdrawalWaitMinSeconds. This prevents challenged agent to remove all collateral before
        // challenge can be proved.
        // rate-limited
        uint64 withdrawalWaitMinSeconds;
```

The (>=) sign is incorrect - because it includes the last second, when there is a possibility of a challenge, in the allowed range for the agent - a potentially malicious agent can steal a lot of funds if someone tries to stop it at the last second.

Why is this a bug?

Because logically we should wait for the period to expire completely before we can say that the period has expired and some actions are allowed.

This means that we should not do require(block.timestamp >= withdrawal.allowedAt, "withdrawal: not allowed yet"), instead we should do require(block.timestamp > withdrawal.allowedAt, "withdrawal: not allowed yet");.

One might say that this is the intended behavior, however in another function ("announceDestroy") the identical check is used correctly:

```
uint256 destroyAllowedAt = block.timestamp + settings.withdrawalWaitMinSeconds;
agent.destroyAllowedAt = destroyAllowedAt.toUint64();
```

and later (in function "destroyAgent")

```
require(block.timestamp > agent.destroyAllowedAt, "destroy: not allowed yet");
```

Simple example:

Let the action be allowed to be performed within 1 minute from 14.00.00 to 14.01.00. This means that the first moment when the term expires will be 14.01.01, that is, the next second after that. But according to the logic of the current function, this is not so.

## Impact Details

Direct evasion of penalties: Agents can withdraw before valid challenges are finalized.

Bypassing slashing logic: Slashing or fraud proofs become ineffective in last second.

Economic loss: Collateral that should be slashed or penalized is instead withdrawn.

Security model broken: Disputes become unreliable when submitted near the end of the timeout.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsExternal.sol#L58> - announceWithdrawal function

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsExternal.sol#L98> - beforeCollateralWithdrawal function

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsCreateDestroy.sol#L123> - announceDestroy function

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsCreateDestroy.sol#L145C14-L145C26> - destroyAgent function

## Proof of Concept

## Proof of Concept

My test will show the possibility of withdrawal of funds at the last second of the "deadline".

To check, add and run the following test:

```
        it("collateral withdrawal dldLambda check", async () => {
            const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
            const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000));

            const fullAgentCollateral = toWei(3e8);
            await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);

            await context.updateUnderlyingBlock();

            const lots = 3;
            const crt = await minter.reserveCollateral(agent.vaultAddress, lots);
            const txHash = await minter.performMintingPayment(crt);
            const minted = await minter.executeMinting(crt, txHash);
            assertWeb3Equal(minted.mintedAmountUBA, context.convertLotsToUBA(lots));
            const agentInfo = await agent.checkAgentInfo({ totalVaultCollateralWei: fullAgentCollateral, freeUnderlyingBalanceUBA: minted.agentFeeUBA, mintedUBA: minted.mintedAmountUBA.add(minted.poolFeeUBA) });


            const minVaultCollateralRatio = agentInfo.mintingVaultCollateralRatioBIPS;
            const vaultCollateralPrice = await context.getCollateralPrice(agent.vaultCollateral());
            const lockedCollateral = vaultCollateralPrice.convertUBAToTokenWei(agentInfo.mintedUBA).mul(toBN(minVaultCollateralRatio)).divn(MAX_BIPS);
            const withdrawalAmount = fullAgentCollateral.sub(lockedCollateral);

            await agent.announceVaultCollateralWithdrawal(withdrawalAmount);

            await agent.checkAgentInfo({ reservedUBA: 0, redeemingUBA: 0, announcedVaultCollateralWithdrawalWei: withdrawalAmount });

            await deterministicTimeIncrease(context.settings.withdrawalWaitMinSeconds); //<----- this will set the time exactly to the last moment

            await agent.withdrawVaultCollateral(withdrawalAmount);

        });
```

And you will see that this test is successful:

```
    simple scenarios - agent manipulating collateral and underlying address
      ✔ collateral withdrawal dldLambda check (91ms)


  1 passing (2s)

✨  Done in 5.51s.
```

(command - yarn testHH --grep "collateral withdrawal dldLambda check")
