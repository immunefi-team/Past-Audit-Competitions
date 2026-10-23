# #46265 \[SC-Medium] Logic flaw in transferToCoreVault allows creation of zero-value redemption request

**Submitted on May 27th 2025 at 14:27:27 UTC by @nnez for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46265
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A logic flaw in the `transferToCoreVault` function allows agents to unintentionally create redemption requests with a **zero amount**, leading to a situation where confirming the redemption is impossible unless the agent voluntarily transfers underlying assets **for free** to the Core Vault.

## Vulnerability Details

In fAsset v1.1, the **Core Vault** mechanism was introduced to improve capital efficiency. It enables agents to reclaim collateral by transferring underlying assets to the Core Vault, freeing up minting capacity. The process consists of three steps:

1. The agent calls `transferToCoreVault` with `_amountAMG`.
2. The agent makes the underlying payment to the Core Vault.
3. The agent calls `confirmRedemptionPayment` to finalize.

Within `transferToCoreVault`, there is an explicit check to forbid zero-value transfers:\
See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol#L57>

```solidity
require(_amountAMG > 0, "zero transfer not allowed");
```

However, the actual amount used to create the redemption request is `transferredAMG`, which is calculated by:

```solidity
(uint64 transferredAMG, ) = Redemptions.closeTickets(_agent, _amountAMG, false, false);  
...
uint64 redemptionRequestId = RedemptionRequests.createRedemptionRequest(
    RedemptionRequests.AgentRedemptionData(_agent.vaultAddress(), transferredAMG),
    state.nativeAddress, underlyingAddress, false, payable(address(0)), 0,
    state.transferTimeExtensionSeconds, true);
```

The function `closeTickets` attempts to close existing agent redemptions up to `_amountAMG`, returning the actual amount matched (`transferredAMG`). **If no tickets are available at the time of the call**, `transferredAMG` could become zero, even if `_amountAMG` was non-zero.

This situation can occur due to a race condition: a user may redeem just before the agent invokes `transferToCoreVault`, consuming all available redemption tickets.

As a result, the following redemption request is created with a **zero amount**, bypassing the intent of the original validation.

## Impact

Redemption requests with zero amount cannot be completed via `confirmRedemptionPayment` (for core vault redemption), since no valid payment can be proven for zero value. However, the system still expects a payment to Core Vault to finalize the request.

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol#L156>

This forces the agent to either:

* Transfer an arbitrary small amount of underlying for free, or
* Leave the process incomplete for two hours (an operational delay) and call `redemptionPaymentDefault` to unstick the request.
* If they're not active, assuming that agent bot doesn't account for this case, then agent also risks losing collateral to a penalty for someone else calling `redemptionPaymentDefault` for them.

## Recommendation

Add an explicit check after `closeTickets` to ensure `transferredAMG > 0` before continuing with the redemption request creation. For example:

```solidity
require(transferredAMG > 0, "no tickets available for transfer");
```

This ensures the agent only proceeds when a meaningful transfer is possible and prevents zero-value redemptions from being created.

## Proof of Concept

## Proof-of-Concept

The following test creates an agent with zero redemption ticket and invoke `transferToCoreVault` with non-zero `_amountAMG` then shows that a redemption request with zero-value is created.

### Steps

1. Add the following test in `test/integration/fasset-simulation/14-CoreVault.ts`

```
    it("nnez - zero amount redemption", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(100);
        await context.updateUnderlyingBlock();
        
        const txHash = await agent.performTopupPayment(toBN(100));
        await agent.confirmTopupPayment(txHash);

        const res = await context.assetManager.transferToCoreVault(agent.vaultAddress, 10, { from: agent.ownerWorkAddress });
        const log: any = res.logs[0].args;
        
        console.log(res);
        
        assertWeb3Equal(0, log.valueUBA);
        assertWeb3Equal(0, log.feeUBA);
    });
```

2. Run `yarn hardhat test "test/integration/fasset-simulation/14-CoreVault.ts" --grep "nnez - zero amount redemption"`
3. Observe that test passes, an indication that a redemption request with zero value is created
