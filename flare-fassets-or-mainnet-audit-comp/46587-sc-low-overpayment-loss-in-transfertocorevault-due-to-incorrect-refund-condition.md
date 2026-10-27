# #46587 \[SC-Low] Overpayment loss in \`transferToCoreVault\` due to incorrect refund condition

**Submitted on Jun 2nd 2025 at 04:58:52 UTC by @nnez for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46587
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

In the `transferToCoreVault` function, callers must attach enough native token (FLR/SGB) via `msg.value` to cover a computed transfer fee (`transferFeeWei`). If they overpay, the contract is intended to refund any excess. The relevant code snippet is:

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol#L81-L88>

```solidity
    // pay the transfer fee and return overpaid transfer fee when the difference is larger than gas use
    // (all transfers are guarded by nonReentrant in the facet)
    if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
        Transfers.transferNAT(state.nativeAddress, transferFeeWei);
        Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
    } else {
        Transfers.transferNAT(state.nativeAddress, msg.value);
    }
```

* **`TRANSFER_GAS_ALLOWANCE`** is a gas‐unit constant (100 000 gas) used to cap the gas limit when issuing the refund via a low‐level `call`. It has no direct relationship with wei or FLR/SGB denominations.
* In the `if` branch (`msg.value > transferFeeWei + TRANSFER_GAS_ALLOWANCE`), the contract correctly sends exactly `transferFeeWei` to the fee recipient and issues a refund of `msg.value – transferFeeWei` (with a limit gas). This covers any overpayment above the required fee.
* In the `else` branch, when `transferFeeWei ≤ msg.value ≤ transferFeeWei + TRANSFER_GAS_ALLOWANCE)`, the contract treats the entire `msg.value` as the fee and does **not** refund the caller’s overpayment. Since gas costs are deducted separately from the sender’s wallet and do not reduce `msg.value`, this branch effectively causes any excess between `transferFeeWei` and `transferFeeWei + TRANSFER_GAS_ALLOWANCE` to be incorrectly sent to fee receiver.

### Example Scenario

Let’s assume:

* `transferFeeWei = 1_000_000 wei` (0.001 FLR)
* `TRANSFER_GAS_ALLOWANCE = 100_000` (gas)

#### Case – Small Overpayment

* Caller sends `msg.value = 1_000_050 wei`
* Since `1_000_050 < 1_000_000 + 100_000`, the contract enters the `else` branch
* **Result**: Entire 1\_000\_050 wei is forwarded as fee; 50 wei overpayment is not refunded

Thus, only when `msg.value` falls in the interval:

```
[transferFeeWei, transferFeeWei + TRANSFER_GAS_ALLOWANCE]
```

the contract incorrectly retains any overpayment (send to fee receiver) instead of refunding it.

## Impact

The refund logic is incorrect, users overpaying by up to `TRANSFER_GAS_ALLOWANCE` wei will not be refunded.

## Recommended Mitigations

```diff
- if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
+ if (msg.value > transferFeeWei) {
      Transfers.transferNAT(state.nativeAddress, transferFeeWei);
      Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
  } else {
      Transfers.transferNAT(state.nativeAddress, msg.value);
  }
```

## Proof of Concept

## Proof-of-Concept

The following test demonstrates that if a fee is paid between `[transferFeeWei, transferFeeWei + TRANSFER_GAS_ALLOWANCE]`, the refund is not triggered due to the misconception on gas consumption (TRANSFER\_GAS\_ALLOWANCE).

### Steps

1. Put the following test in `test/integration/fasset-simulation/14-CoreVault.ts`

```typescript
    it("nnez - incorrect transfer fee refund", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(1000000));
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(100);
        // minter2 also deposits to pool (so some fasset fees will go to them)
        await agent.collateralPool.enter(0, false, { from: minterAddress2, value: toWei(3e8) });
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 10);
        // update time
        await context.updateUnderlyingBlock();
        // agent requests transfer for all backing to core vault
        const info = await agent.getAgentInfo();
        const transferAmount = info.mintedUBA;
        // calculate the transfer fee
        const cbTransferFee = await context.assetManager.transferToCoreVaultFee(transferAmount);
        // paid fee must be large enough, otherwise the request fails
        await expectRevert(context.assetManager.transferToCoreVault(agent.vaultAddress, transferAmount, { from: agent.ownerWorkAddress, value: cbTransferFee.subn(1) }),
            "transfer fee payment too small");
        // Overpay within [transferFeeWei, transferFeeWei + 100_000]
        const payTransferFee = toBN(90_000).add(cbTransferFee);
        console.log("Acutal fee: ", cbTransferFee.toString());
        console.log("We pay: ", payTransferFee.toString());
        console.log("Diff: +", payTransferFee.sub(cbTransferFee).toString());
        const res = await context.assetManager.transferToCoreVault(agent.vaultAddress, transferAmount, { from: agent.ownerWorkAddress, value: payTransferFee });
        console.log("Invoke transferToCoreVault()");
        const actuallyPaidFee = (await calculateReceivedNat(res)).neg();
        console.log("We actually paid: +", actuallyPaidFee.toString());
        
        // Assert that overpaid amount is not refunded
        assertWeb3Equal(actuallyPaidFee, payTransferFee);
    });
```

2. Run `yarn hardhat test "test/integration/fasset-simulation/14-CoreVault.ts" --grep "nnez - incorrect transfer fee refund"`
3. Observe that the test passes (last assert passes)
