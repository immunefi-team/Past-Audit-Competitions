# #45604 \[SC-Low] User Overpayment in \`transferToCoreVault\` Fee Handling

**Submitted on May 17th 2025 at 19:31:37 UTC by @lufP for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45604
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Affected Component(s):**

* Contract: `CoreVault.sol`
* Library: `Transfers.sol` (specifically the `TRANSFER_GAS_ALLOWANCE` constant)
* Function: `transferToCoreVault(Agent.State storage _agent, uint64 _amountAMG)`

\*\* Summary:\*\* The `transferToCoreVault` function in `CoreVault.sol` has a specific condition in its fee payment and refund logic that can lead to a user overpaying by exactly `Transfers.TRANSFER_GAS_ALLOWANCE` (100,000 Wei). This overpaid amount is sent to the fee collector (`state.nativeAddress`) instead of being refunded to the user.

\*\* Detailed Description:\*\* The fee handling logic in `transferToCoreVault` is as follows:

```solidity
// contracts/assetManager/library/CoreVault.sol
// ...
uint256 transferFeeWei = getTransferFee(transferredAMG);
require(msg.value >= transferFeeWei, "transfer fee payment too small"); // User must send at least the fee
// ...
// pay the transfer fee and return overpaid transfer fee when the difference is larger than gas use
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
    // Scenario 1: Significant overpayment
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);                             // Fee collector gets the exact calculated fee
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei); // User gets the remainder refunded
} else {
    // Scenario 2: msg.value is between transferFeeWei (inclusive)
    // and transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE (inclusive)
    Transfers.transferNAT(state.nativeAddress, msg.value); // Fee collector gets the entire msg.value sent by the user
}
// ...
```

The constant `Transfers.TRANSFER_GAS_ALLOWANCE` is defined in `contracts/utils/lib/Transfers.sol`:

```solidity
// contracts/utils/lib/Transfers.sol
uint256 internal constant TRANSFER_GAS_ALLOWANCE = 100_000; // Value is 100,000 (Wei)
```

The issue arises when a user sends `msg.value` such that: `transferFeeWei < msg.value <= transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE`

Specifically, if `msg.value == transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE` (i.e., `transferFeeWei + 100_000` Wei):

1. The condition `msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE` is **false**.
2. The `else` block is executed: `Transfers.transferNAT(state.nativeAddress, msg.value)`.
3. In this case, `state.nativeAddress` (the fee collector) receives the entire `msg.value`, which is `transferFeeWei + 100_000` Wei.
4. The user does not receive any refund. Effectively, they pay the intended `transferFeeWei` plus an additional `100_000` Wei, which is unnecessarily captured by the fee collector.

\*\* Impact:\*\*

* **Financial:** Minor financial loss for users who happen to send `msg.value` within the affected range. The maximum loss per transaction due to this specific issue is `Transfers.TRANSFER_GAS_ALLOWANCE` (100,000 Wei).
* **User Experience:** Users might be confused about the exact fee deduction if they notice the discrepancy.
* **Systemic Risk:** None. This does not compromise the overall security or fund safety of the asset management system beyond the minor overpayment.

\*\* Code Snippet with Issue:\*\*

```solidity
// contracts/assetManager/library/CoreVault.sol: Lines ~79-85
// ...
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) { // TRANSFER_GAS_ALLOWANCE = 100_000
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
} else {
    // BUG: If msg.value is, for example, transferFeeWei + 100_000,
    // this branch is taken, and nativeAddress receives transferFeeWei + 100_000.
    // The user overpays by 100_000 Wei.
    Transfers.transferNAT(state.nativeAddress, msg.value);
}
// ...
```

\*\* Suggested Fix(es):\*\* To ensure the user only pays the `transferFeeWei` and any amount above it (that is not a very large overpayment handled by the `if` branch) is refunded, the `else` block could be modified.

**Option A (Consistent Refund for Minor Overpayments):** Modify the `else` branch to also refund amounts greater than `transferFeeWei`.

```solidity
// ...
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
} else {
    // This branch now handles msg.value >= transferFeeWei && msg.value <= transferFeeWei + TRANSFER_GAS_ALLOWANCE
    Transfers.transferNAT(state.nativeAddress, transferFeeWei); // Collector gets the exact fee
    if (msg.value > transferFeeWei) {
        Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei); // Refund the small overpayment
    }
    // If msg.value == transferFeeWei, no refund is needed.
}
// ...
```

This ensures that if `msg.value` is, for instance, `transferFeeWei + 50_000`, the user gets `50_000` Wei back.

**Option B (Simplified Logic - User always pays at least `transferFeeWei`, excess always refunded if `TRANSFER_GAS_ALLOWANCE` is removed from condition):** If `TRANSFER_GAS_ALLOWANCE` is not meant to be a threshold for a different fee tier but rather a gas stipend for the refund transaction itself (which is how `Transfers.transferNATAllowFailure` uses it internally for its own gas), the logic could be simplified: The `Transfers.TRANSFER_GAS_ALLOWANCE` in the conditional `msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE` is somewhat misleading if its only purpose is to decide whether a refund *occurs at all* versus being a small additional fixed fee. If the goal is: "User pays `transferFeeWei`. If they send more, refund the difference, provided the difference is enough to be worth refunding (e.g., covers refund gas)", then the existing `if` captures this for *large* differences. The `else` currently captures *small* differences as an extra fee.

If the intention is that any amount `> transferFeeWei` should be refunded, then:

```solidity
// ...
// Minimum payment is already ensured by: require(msg.value >= transferFeeWei, "transfer fee payment too small");

Transfers.transferNAT(state.nativeAddress, transferFeeWei); // Collector always gets the exact fee
if (msg.value > transferFeeWei) {
    // Refund any amount paid over the exact transferFeeWei
    // The Transfers.transferNATAllowFailure function already has a gas stipend (TRANSFER_GAS_ALLOWANCE) for the .call
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
}
// ...
```

This simplifies the logic: the collector gets `transferFeeWei`, and any overpayment is refunded. The `Transfers.TRANSFER_GAS_ALLOWANCE` constant's role would then primarily be for the internal gas forwarding of the `transferNAT` functions, not for the conditional fee logic in `CoreVault`.

\*\* Test Case Suggestion:\*\* A new test case should be added to specifically verify the behavior when `msg.value == transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE`.

```typescript
// In test/integration/fasset-simulation/14-CoreVault.ts
it("should correctly handle fee when msg.value is exactly transferFeeWei + TRANSFER_GAS_ALLOWANCE", async () => {
    // ... (setup agent, minter, calculate transferAmount and cbTransferFee) ...
    const transferGasAllowance = toBN(100000); // Assuming Transfers.TRANSFER_GAS_ALLOWANCE is 100_000 Wei
    const criticalMsgValue = cbTransferFee.add(transferGasAllowance);

    const balanceBefore = await context.wNat.balanceOf(agent.ownerWorkAddress); // Or web3.eth.getBalance if it's plain NAT

    const res = await context.assetManager.transferToCoreVault(agent.vaultAddress, transferAmount, { 
        from: agent.ownerWorkAddress, 
        value: criticalMsgValue 
    });
    
    const balanceAfter = await context.wNat.balanceOf(agent.ownerWorkAddress);
    const txCost = await getTransactionCost(res); // Helper to get gas cost of the transaction itself
    const expectedBalanceAfter = balanceBefore.sub(cbTransferFee).sub(txCost); // If bug fixed (Option A/B)
    // Current (buggy) behavior check:
    // const expectedBalanceAfterBuggy = balanceBefore.sub(criticalMsgValue).sub(txCost);
    // assertWeb3Equal(balanceAfter, expectedBalanceAfterBuggy, "User's balance should decrease by criticalMsgValue + gas");

    // To check if fixed (e.g., with Option A or B from suggestions):
    // The sender should have effectively paid only cbTransferFee + actual transaction gas.
    // Their balance should decrease by cbTransferFee + txCost. The 100,000 Wei should be refunded.
    assertWeb3Equal(balanceAfter, expectedBalanceAfter, "User should be refunded the TRANSFER_GAS_ALLOWANCE amount");
    
    // Additionally, check the fee collector's balance.
    // It should only increase by cbTransferFee.
});
```

## Proof of Concept

## Proof of Concept

\*\* Steps to Reproduce:\*\*

1. Identify an agent and an amount (`_amountAMG`) for a `transferToCoreVault` operation.
2. Calculate the required `transferFeeWei` using `getTransferFee(_amountAMG)` (or `assetManager.transferToCoreVaultFee(amountUBA)` from client side).
3. The `Transfers.TRANSFER_GAS_ALLOWANCE` constant is `100_000` Wei.
4. Call `transferToCoreVault` with `msg.value` set to exactly `transferFeeWei + 100_000` Wei.
5. Observe:
   * The transaction succeeds.
   * The sender's native token balance will decrease by `transferFeeWei + 100_000` Wei.
   * The `state.nativeAddress` balance will increase by `transferFeeWei + 100_000` Wei.
   * The sender will not receive a refund for the `100_000` Wei.

**4. Expected Behavior:** Ideally, the user should only pay the calculated `transferFeeWei`. Any amount sent above this, if not intended for another purpose (like covering a *variable* gas cost returned to the user, which `TRANSFER_GAS_ALLOWANCE` seems to hint at but isn't implemented as such a refund mechanism here), should be refunded. Alternatively, if `TRANSFER_GAS_ALLOWANCE` is an intentional, small, fixed additional fee for transactions that don't significantly overpay, this should be clearly documented and the logic possibly refactored for clarity (e.g., `effectiveFee = transferFeeWei + SMALL_FIXED_MARGIN_FEE`).

Given the naming (`TRANSFER_GAS_ALLOWANCE`) and the `if` condition checking for "difference is larger than gas use", the current behavior where this amount is silently added to the collected fee in the `else` branch is unexpected and leads to overpayment. A more intuitive behavior for the `else` branch might be to transfer `transferFeeWei` to the collector and refund `msg.value - transferFeeWei` to the user if `msg.value > transferFeeWei`.

\*\* Actual Behavior:\*\* If `msg.value` is between `transferFeeWei + 1` Wei and `transferFeeWei + 100,000` Wei (inclusive), the user pays the entire `msg.value` to the fee collector. The maximum overpayment in this specific path is `100,000` Wei.
