# #36136 \[SC-Insight] Fee calculation error in withdraw function of collateralVault contract

**Submitted on Oct 21st 2024 at 14:58:52 UTC by @ProfitableFrog6412 for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36136
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A fee calculation error has been identified in the \`withdraw\` function of the CollateralVault contract. The problem lies in how the withdrawal fee is computed, resulting in users being charged an excessive fee.

## Vulnerability Details

The vulnerability stems from the incorrect calculation of fees in the \`withdraw\` function. The current implementation calculates the fee using: \`\`\`solidity uint256 fee = Pricing.percentageOf(\_amount, uint256(withdrawalFeeBasisPoints)); \`\`\` This approach applies the withdrawal fee to the entire amount, rather than correctly adjusting it using the configured \`withdrawalFeeBasisPoints\`. Consequently, users end up being charged a larger fee than intended. The correct implementation should utilize: \`\`\`solidity uint256 withdrawAmount = Pricing.amountBeforeFee(\_amount, withdrawalFeeBasisPoints); // real withdraw amount uint256 fee = \_amount - withdrawAmount ; // real fee amount \`\`\` This way, the fee aligns with the configured basis points, ensuring users pay an accurate withdrawal fee.

## Impact Details

If exploited or left unchecked, this bug leads to users being charged a higher fee than expected on each withdrawal. Over time, the cumulative overcharging could result in significant financial losses for the contract’s users, especially for high-value withdrawals. Furthermore, this discrepancy could erode user trust and damage the platform’s reputation.

## References

<https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f?utm\\_source=immunefi#code#F1#L425>

## Proof of Concept

## Proof of Concept

\`\`\`solidity // SPDX-License-Identifier: MIT pragma solidity 0.8.26; import "./Pricing.sol";

contract FeeTest { uint16 public withdrawalFeeBasisPoints = 100;

```
function oldWithdrawAmountTest(uint256 amount)
    public
    view
    returns (uint256)
{
    return
        amount -
        Pricing.percentageOf(amount, uint256(withdrawalFeeBasisPoints));
}

function newWithdrawAmountTest(uint256 amount)
    public
    view
    returns (uint256)
{
    return Pricing.amountBeforeFee(amount, withdrawalFeeBasisPoints);
}

function oldFeeTest()
    public
    view
    returns (uint256 expectedFee, uint256 incorrectRealFee)
{
    uint256 amount &#x3D; 100e18;
    uint256 withdrawAmount &#x3D; oldWithdrawAmountTest(amount);
    expectedFee &#x3D; Pricing.percentageOf(
        withdrawAmount,
        uint256(withdrawalFeeBasisPoints)
    );
    incorrectRealFee &#x3D; amount - withdrawAmount;
}

function Test() public view returns (uint256 oldAmount, uint256 newAmount) {
    uint256 amount &#x3D; 100e18;
    return (oldWithdrawAmountTest(amount), newWithdrawAmountTest(amount));
}
```

} \`\`\`
