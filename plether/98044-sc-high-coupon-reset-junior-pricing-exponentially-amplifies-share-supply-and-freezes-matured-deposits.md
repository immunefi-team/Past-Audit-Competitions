# \#98044 [SC-High] Coupon-reset Junior pricing exponentially amplifies share supply and freezes matured deposits

**Submitted on Nov 4th 2026 at 15:20:08 UTC by @Ailenia for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98044
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

I found that an LP can repeatedly deposit into Junior just after coupon accounting reduces Junior principal to zero. Each small deposit is priced against low principal but the full existing share supply, which increases `totalSupply`. After 31 ordinary one-basis-point trading cycles, a later funded deposit makes the settlement share quote exceed `uint256`. Settlement reverts atomically, while cancellation treats the deposit as active. My PoC freezes exactly 1,000,000 USDG for more than seven days.

## Vulnerability Details

Junior deposit shares are calculated from the current principal and accrued supply:

```solidity
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

Production settlement uses a zero deposit fee, so `denominator` is `pricingAssets + 1`. There is no limit on `pricingSupply` or the resulting share quote.

The Senior coupon can reduce all Junior principal to zero without reducing Junior share supply. A later trader loss restores a small amount of Junior principal. A matured deposit is then priced against that small principal and the full old supply. If `D` is the deposit, `P` is restored Junior principal, and `S` is supply, the new supply is approximately:

```text
S' = S + floor(D * (S + 1000) / (P + 1))
```

When `D > P`, this formula can increase supply sharply. The physical pool revenue is not necessarily the same as the pricing principal after reconciliation, so I do not assume a fixed per-cycle multiplier. My PoC uses a fully closed 1,100 USD LONG at an observed price of 1.0000 and closes it at 1.0001. This one-basis-point adverse move leaves about 0.612 USDG of physical pool revenue and no open position. An adaptive loop continues until the measured supply crosses the overflow threshold; it needs 31 cycles. The coupon consumes the resulting Junior principal before the next cycle.

The PoC first stages an ordinary full Junior loss at the supported v2 release capital ratio. It uses the shipped 10,000,000 USDG Senior and 2,500,000 USDG Junior starting amounts, 1 USDG seeds, 1% VPI factor, 40% maximum skew, and 8% Senior rate. Every wipe order uses the exact production `OrderRouter` bytecode and its native V3 `commitOrder(OrderRequest)` and `executeOrder` interfaces with MockPyth observations accepted by the in-scope Oracle path. The broad price excursion is a market precondition; I do not claim that the attacker controls it. The exploit starts after this supported waterfall state exists and does not require an exact residual or an oracle price that solves a target equation. Starting from the full-wipe state, the PoC executes 31 public amplification cycles. Every exploit cycle uses a funded deposit, production Router order calls, fixed observations separated in time, a fully closed position, and permissionless LP settlement.

After the amplification, the victim funds a 1,000,000 USDG deposit. The attacker closes a separate losing position that produces 10,405.967578 USDG of physical pool revenue. This gives the pending reconciliation ample positive Junior principal, but the victim's quote still exceeds `uint256`. The PoC proves that both the direct production quote and the full settlement revert with exact Solidity panic code `0x11` before the epoch can finalize.

The revert rolls back the entire settlement, including reconciliation. The pending trading revenue stays pending. A retry performs the same reconciliation and reaches the same overflowing quote. The victim also cannot cancel. `cancelPendingDeposit()` previews the pending revenue, sees that Junior is not terminally wiped or impaired, and reverts with `TrancheVault__DepositEpochAlreadyActive` after maturity.

### Attack sequence

1. Normal trading losses fully consume Junior principal. This is a supported waterfall state and does not require an exact residual or selected price equation.
2. The attacker restores a small positive Junior balance through a closed losing trade, then lets the Senior coupon consume it.
3. The attacker repeats a public cycle: request a 10 USDG Junior deposit, wait for maturity and coupon accrual, close a 1,100 USD position after a one-basis-point adverse move, and settle the epoch.
4. Each cycle has no open position at settlement. The adaptive loop measures supply and reaches the overflow threshold in 31 cycles.
5. A victim funds and matures a 1,000,000 USDG Junior deposit.
6. The attacker closes a losing position that leaves positive pending Junior revenue. Every settlement retry overflows the share quote and rolls back.
7. The victim cannot claim or cancel. The exact escrow remains frozen after 24 hours and after seven days.

### Evidence boundary

| Item | What the PoC does |
| --- | --- |
| Actor funding | Test-token minting supplies ordinary six-decimal actor capital. No production attack step calls a mint function or depends on privileged token creation. |
| Market observations | MockPyth supplies observations to the release Oracle integration. The initial broad excursion models the market precondition; the recurrence uses fixed 1.0000 and 1.0001 observations and does not solve for a special price. |
| State transitions | Deposits, orders, execution, epoch settlement, cancellation, and retries use the production public entry points. No storage is edited. |
| Impact | The test directly measures the victim's funded vault escrow, pending and claimable request balances, exact panic bytes, failed cancellation, and seven-day retry. |

The reported 242,492,351 test gas is the aggregate for a scenario spanning 31 hourly cycles and many independent public calls. It is not one production transaction or one block.

### Existing safeguards and why they fail

Settlement retry is not recovery because the revert restores the pending revenue and the next call reaches the same overflowing quote. Cancellation is not recovery because it previews that revenue, classifies the matured epoch as active, and reverts. The victim cannot reduce or split a request after funding it. These ordinary user and permissionless recovery paths remain unavailable after 24 hours and after seven days in the PoC.

### Duplicate distinction

This is not the reported redemption-rounding or zero-payout epoch behavior: the victim has a funded deposit and settlement reverts before it can mint claimable shares. It is not the Senior coupon checkpoint-frequency issue or the documented coupon cap/shortfall limitation: the coupon only leaves worthless Junior shares outstanding, while the root cause is an unbounded deposit quote. The broad SC-Q4 ledger recommendation does not disclose this concrete overflow and non-cancellable escrow path. It is also not a negative-VPI, external-reserve, or infeasible near-`uint256` deposit amplification. Every recurrence uses physical revenue from a fully closed position, 31 deposits of 10 USDG, and a final 10,405.967578 USDG pool sacrifice to freeze an ordinary 1,000,000 USDG deposit.

## Impact Details

Selected impact: **Temporary freezing of funds for at least 24 hours**.

My PoC proves the protected balance movement and persistence:

- The victim transfers exactly 1,000,000 USDG into `TrancheVault` escrow.
- The direct quote and the settlement transaction return the same exact arithmetic panic, `Panic(0x11)`.
- `pendingDepositRequest` remains exactly 1,000,000 USDG and `claimableDepositRequest` remains zero.
- The vault's escrow balance does not move when settlement fails.
- Cancellation reverts after 24 hours.
- A fresh settlement retry returns the same exact `Panic(0x11)`, and another cancellation still fails after seven days.

The setup uses 31 deposits of 10 USDG and about 31 hours of coupon accrual. The final closed trade sacrifices 10,405.967578 USDG to the pool. This freezes 1,000,000 USDG in the demonstrated case, a denial ratio above 90 times before gas and execution fees. The attacker can target a larger victim epoch without increasing the supply-amplification setup cost.

This is not dependent on a freely mintable test token or a near-`uint256` deposit. Every attack and victim amount is an ordinary six-decimal USDG amount, and the PoC uses the latest v2 release limits.

## Remediation

Prevent accrued Junior supply from growing beyond a bound that keeps every accepted deposit quote representable. Check both `totalSupply + shares` and the full-precision quotient before accepting or finalizing an epoch.

Also remove the amplification primitive. When Junior principal is fully consumed, reset or proportionally retire the economically worthless Junior shares before accepting later deposits. Another option is to restart the tranche through a controlled recapitalization path with a bounded initial share ratio.

Finally, make an unrepresentable quote fail open for depositors. Reject the affected epoch and enable its existing refund path instead of reverting the whole settlement.

## References

- `packages/perps/src/TrancheVault.sol:1398-1409`
- `packages/perps/src/TrancheVault.sol:832-860`
- `packages/perps/src/HousePool.sol:914-934`
- `packages/perps/src/HousePool.sol:1030-1050`
- `packages/perps/src/HousePool.sol:1549-1585`

## Proof of Concept

https://gist.github.com/niafreu/509fc4724df92a6b41057d798ebd149f

## Reproduction

1. Check out the official Plether competition repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Save the gist file as `packages/perps/test/perps/poc/JuniorVpiSupplyOverflowFreezePoC.t.sol`.
3. Use Foundry 1.8.5. From the repository root, run:

```bash
FOUNDRY_PROFILE=quick FOUNDRY_VIA_IR=false FOUNDRY_OUT=out-junior-vpi-production-router FOUNDRY_CACHE_PATH=cache-junior-vpi-production-router forge test --root packages/perps --match-path test/perps/poc/JuniorVpiSupplyOverflowFreezePoC.t.sol --match-contract JuniorVpiSupplyOverflowFreezePoCTest --match-test test_PostWipeAmplificationFreezesMaturedDeposit -vv
```

No RPC URL or environment variable is required. The test uses only MockPyth and the fixture USDG token for out-of-scope dependencies. It deploys the exact in-scope production contracts and does not modify them.

Expected checkpoints:

```text
[PASS] test_PostWipeAmplificationFreezesMaturedDeposit()
amplification cycles: 31
victim escrow frozen: 1000000.000000
terminal Junior supply: 15460596072974948328493449578959717077375142424324281884890253070976575779450
```

The test asserts the exact victim escrow, zero claimable balance, `Panic(0x11)` on settlement, failed cancellation after 24 hours, and the same failure after seven days. Its 242,492,351 gas is aggregate across 31 hourly cycles and many calls, not one transaction.
