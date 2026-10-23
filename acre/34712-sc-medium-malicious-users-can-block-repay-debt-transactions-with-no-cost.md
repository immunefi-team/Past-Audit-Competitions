# #34712 \[SC-Medium] Malicious users can block repay debt transactions with no cost

**Submitted on Aug 21st 2024 at 18:18:25 UTC by @brivan for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34712
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Block stuffing
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief

The \`repayDebt\` function can be exploited by malicious users to block debt repayments by manipulating the conversion rate with minimal cost. This allows attackers to indefinitely prevent legitimate debtors from repaying their debts.

## Vulnerability Details

Authorized debtors can call the \`mintDebt\` function to mint shares, which they are required to repay later. This function tracks the amount of assets in debt as follows: \`\`\`solidity function mintDebt( uint256 shares, address receiver ) public whenNotPaused returns (uint256 assets) { assets = convertToAssets(shares);

```
// Increase the debt of the debtor.
```

\>>> currentDebt\[msg.sender] += assets;

```
// ...
```

} \`\`\` Debtors can subsequently call the \`repayDebt\` function to repay their debt. The amount of debt to be repaid is calculated based on the current conversion rate: \`\`\`solidity function repayDebt( uint256 shares ) public whenNotPaused returns (uint256 assets) { >>> assets = convertToAssets(shares);

```
// Check the current debt of the debtor.
```

\>>> if (currentDebt\[msg.sender] < assets) { revert ExcessiveDebtRepayment( msg.sender, currentDebt\[msg.sender], assets ); } // Decrease the debt of the debtor. currentDebt\[msg.sender] -= assets;

```
// ...
```

} \`\`\` The check \`if (currentDebt\[msg.sender] < assets)\` in the \`repayDebt\` function can be exploited by malicious users to front-run the transaction. By transferring a small amount of \`tBTC\` (e.g., 2 wei) directly to the \`stBTC\` contract, the conversion rate can be manipulated, causing the \`assets\` value returned by \`convertToAssets(shares)\` to exceed the current debt. This results in the \`repayDebt\` transaction reverting due to the \`ExcessiveDebtRepayment\` error. **Note that a minimal transfer of \`tBTC\` (equivalent to 2 wei) is sufficient to block the debt repayment**.

Consider the following scenario, demonstrated by a runnable Proof of Concept (PoC) included with this submission:

1. Bob is authorized to incur a debt of \`100\`. He calls \`mintDebt\` to mint 100 shares, and the debt is recorded based on the current conversion rate.
2. Later, Bob decides to repay his debt and initiates a transaction to call \`repayDebt\`.

   2.1 Alice, a malicious actor, front-runs Bob’s transaction by transferring \`2 wei\` of \`tBTC\` to the \`stBTC\` contract, thereby increasing the conversion rate

   2.2 When Bob’s transaction begins executing, the \`assets\` value calculated by \`convertToAssets(shares)\` is now higher than his recorded debt, causing the transaction to revert: \`\`\`solidity assets = convertToAssets(shares); if (currentDebt\[msg.sender] < assets) {revert ExcessiveDebtRepayment( msg.sender, currentDebt\[msg.sender], assets );} \`\`\`

This attack incurs no significant cost to the malicious user, as Alice only needs to slightly increase the \`stBTC\` assets (by 2 wei) just before the \`repayDebt\` function is executed.

## Impact Details

Malicious users can indefinitely block \`repayDebt\` transactions at no significant cost.

## References

<https://github.com/thesis/acre/blob/dc156f5a7f02142c1f80627267d14a26e5c99b30/solidity/contracts/stBTC.sol#L350-L353>

## Proof of Concept

Copy and paste the following test case into \`test/stBTC.test.ts\`: \`\`\`ts describe("repayDebt", () => { beforeAfterSnapshotWrapper()

```
  it.only(&quot;pocc&quot;, async () &#x3D;&gt; {
    const debtAllowance &#x3D; to1e18(100)
    await stbtc
      .connect(governance)
      .updateDebtAllowance(externalMinter.address, debtAllowance)
    await stbtc
      .connect(externalMinter)
      .mintDebt(debtAllowance, externalMinter.address)

    await tbtc.connect(depositor1).transfer(stbtc, 2) // transfers 2wei equivalent

    await expect(
      stbtc.connect(externalMinter).repayDebt(debtAllowance),
    ).to.be.revertedWithCustomError(stbtc, &quot;ExcessiveDebtRepayment&quot;)
  })
```

// ... } \`\`\`
