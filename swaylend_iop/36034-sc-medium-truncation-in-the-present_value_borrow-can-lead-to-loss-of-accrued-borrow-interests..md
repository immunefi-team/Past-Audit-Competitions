# #36034 \[SC-Medium] truncation in the \`present\_value\_borrow()\` can lead to loss of accrued borrow int

**Submitted on Oct 16th 2024 at 12:33:35 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36034
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw?utm\\_source=immunefi>
* **Impacts:**

## Description

## Brief/Intro

The Market calculates the present borrow balance of an account via the

* \`present\_value\_borrow()\` It multiplies the principal by the \`borrow\_index\` and then divides by the base scale.
* \` principal \* base\_borrow\_index / BASE\_INDEX\_SCALE\_15\` \`\`\`rust pub fn present\_value\_borrow(base\_borrow\_index: u256, principal: u256) -> u256 { principal \* base\_borrow\_index / BASE\_INDEX\_SCALE\_15 } \`\`\`

The present value is the principal scaled by the ratio of the borrow indices (which track the accumulative effect of per-block interest changes):

The issue is that the calculation doesn't round in favor of the protocol but in favor of the users unlike the \`principal\_value\_borrow()\`, this rounding permits that when the principal and ratio of borrow indices are both small, the resulting value can be equal to the principal due to the automatic truncation of division within sway.

This means that a loan could accrue no actual interest for some time when it is supposed to, allowing the protocol to lose yield for the period before it becomes bigger to account for the interest.

## Impact Details

An attacker can take small amounts(\`min\_borrow\`) of borrows and take the cumulated amounts and supply in the market to later gain interest from this rounding issue.

## Mitigation

Round up the present value to account for the interests lost

## POC

was deemed as optional for this report

## Proof of Concept

## Proof of Concept
