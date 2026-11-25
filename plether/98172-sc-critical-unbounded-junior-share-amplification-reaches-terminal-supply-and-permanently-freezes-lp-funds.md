# \#98172 [SC-Critical] Unbounded Junior share amplification reaches terminal supply and permanently freezes LP funds

**Submitted on Nov 5th 2026 at 18:05:05 UTC by @Ailenia for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98172
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

# 0. In-Scope Asset

- Target asset: `packages/perps/src/TrancheVault.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Asset type: Smart Contract
- Scope link: https://immunefi.com/audit-competition/plether/scope/

# 1. Title

Unbounded Junior share amplification reaches terminal supply and permanently freezes LP funds

# 2. Description

## Brief/Intro

I found that the Junior share-supply amplification can reach a terminal raw-supply state that permanently freezes LP funds. An attacker can co-settle the last amplification deposit with a Senior exit, use a one-atom helper redemption to tune the raw Junior supply, and append 1 USDG to a victim's pending epoch. The epoch finalizes with `totalSupply` between `uint256.max - 999` and `uint256.max`. Every later share-to-asset conversion then overflows at `accruedTotalSupply() + 1000`, so the victim cannot request a redemption. Allocation dust can remove one share atom, but no reachable path can burn the remaining 173 atoms needed to restore conversion headroom.

## Vulnerability Details

Junior conversions add the virtual-share offset without first proving that the raw supply leaves enough headroom:

```solidity
return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
```

The initial coupon wipe needs no oracle move. At the shipped 8% Senior rate, my PoC requests an ordinary 10 USDG Senior redemption and performs a permissionless settlement every 90 days. The matured redemption gives each call real LP epoch work. Each checkpoint proves that Junior principal decreases while the nonzero Junior share supply stays unchanged. The coupon compounds into Senior principal and consumes the 2,500,000 USDG Junior tranche after 12 checkpoints and `93,312,000` seconds, about 2.96 years. This is a supported long-lived pool state: the project's whitepaper section 8.6 records a minimum Junior principal of zero in all three historical realized-cash overlays and states that Junior reaches zero partly because it funds the Senior coupon.

A later trader close across a valid 1% mark move restores positive Junior principal. Each amplification cycle then closes across a one-basis-point mark move to add revenue, coupon settlement consumes that principal without retiring Junior shares, and the next 10 USDG deposit is priced against the full old supply. These are valid oracle observations, not selected prices that solve the terminal equation.

The terminal step does not need a chosen oracle price or an exact market residual. The attacker controls only ordinary share and USDG atom amounts:

1. The attacker requests the final 10 USDG amplification deposit and their maximum Senior redemption for the same epoch.
2. Settlement processes redemptions before deposits. The Senior exit leaves only the locked 1 USDG seed, then the Junior deposit restores about 10 USDG of Junior principal. This creates enough Junior withdrawal capacity for a small tuning redemption.
3. The attacker computes a candidate tuning tuple from public supply and principal. The PoC then settles an unrelated user's 1 USDG Junior deposit, proves the old tuple is stale, rereads state, and computes a different valid tuple. No oracle action is used.
4. The attacker transfers the recalculated share slice to a helper. The helper redeems its full balance, so the minimum redemption-size rule permits the request. The request burns `17311190792269966370455442268904914701723780753370903839925300381296` shares for one USDG atom.
5. A victim requests a `6692.710631` USDG Junior deposit. Before the cutoff, the attacker appends exactly `1.000000` USDG to the same epoch.
6. The PoC asserts the exact aggregate `6693.710631` USDG. The epoch mints successfully and leaves raw Junior supply at:

```text
115792089237316195423570985008687907853269984665640564039457584007913129639109
= uint256.max - 826
```

The victim receives positive Junior shares. After the cooldown, both `estimateRedeemAssets(victimShares)` and `requestRedeem(victimShares, victim, victim)` revert with `Panic(0x11)`. The same request still reverts after one year.

This state is permanent in the deployed contracts. Growing Junior principal does not change raw share supply, changing the Senior rate does not change raw share supply, and maintenance fee changes can only add shares. Claiming the attacker's terminal deposit burns one allocation-dust share without a conversion, moving supply from `uint256.max - 826` to `uint256.max - 827`, but conversions still need 173 more atoms of headroom. The unrelated interloper owns a one-owner epoch, so claiming that entitlement transfers its full share allocation and burns no dust. Every ordinary burn path needs a previously enqueued redemption. A new redemption cannot be enqueued because `_requireMinimumRedeemSize` calls `estimateRedeemAssets` first, which evaluates `accruedTotalSupply() + 1000` and panics. `requestRedeemFromClaimableDeposit` reaches the same minimum-size check. There is no owner burn or rescue function.

The tuning calculation is deterministic for the observed settlement state. For a candidate helper payout `q`, it derives the exact share interval whose canonical redemption quote is `q`, predicts the two seed-coupon checkpoints, and binary-searches only attacker-controlled share atoms inside that interval. It then chooses the smallest aggregate deposit that places final raw supply inside the 1,000-share terminal basin. The unrelated settled deposit in my PoC changes the tuple, and the attacker recomputes successfully from the new public state before committing the helper redemption. If public state changes again before the terminal epoch, the attacker must abort and recompute. No oracle observation participates in the terminal equation.

## Impact Details

Selected impact: **Permanent freezing of funds**.

My PoC proves the protected balance and the permanent failure:

- The victim transfers exactly `6692.710631` USDG into a pending Junior deposit before the attacker appends 1 USDG.
- The epoch finalizes and the victim receives positive Junior shares, so this is not a rejected or refundable deposit.
- Raw supply becomes `uint256.max - 826`.
- The victim's direct quote and production redemption request both revert with exact arithmetic panic `0x11`.
- The victim still owns the same positive shares after the failed request and after one year.
- The attacker claims the terminal allocation, one dust share burns, supply remains at `uint256.max - 827`, and no public or governance path can burn the remaining 173 atoms needed to restore conversion headroom.

The terminal step uses a `6693.710631` USDG aggregate deposit and a one-atom helper payout. It changes the unbounded-supply failure from a retryable pending-deposit freeze into an irreversible raw-supply state.

I also measured the attack as one entity controlling the controller, amplifier, loss trader, revenue cycler, and helper. Excluding the protocol's pre-existing seed floors, the entity starts with `12499997.999989` USDG of recoverable value, injects `523.000000` USDG, and ends with `12500483.457040` USDG that the audit actually claims or withdraws. The ending value consists of `12500301.954367` USDG of claimed Senior redemptions, `181.502672` USDG withdrawn from attacker clearinghouse accounts, and the one-atom helper claim. The irreversible loss is `37.542949` USDG before gas while the victim loses access to `6692.710631` USDG, a measured `178.268x` damage-to-loss ratio. I exclude `11.110451` USDG of terminal Junior accounting NAV because the production redemption request for those shares also panics. This run ties up about 12.5 million USDG of recoverable working capital over the coupon horizon; I do not count working capital as sacrificed capital.

I reproduced the same permanent freeze with one-tenth the initial capital: 1,000,000 Senior and 250,000 Junior USDG. This run used 36 amplification cycles, finalized `2207.692064` USDG from the victim into positive shares, and left raw supply at `uint256.max - 586` before one allocation-dust share burned. My separate full-balance audit actually recovered the attacker's Senior claims and Clearinghouse balances and measured `42.664675` USDG of irreversible loss before gas against the frozen victim balance. The lower-capital run still needs the coupon horizon and has material capital time value.

## Remediation

Reserve virtual-share headroom as an invariant. If a deposit mint would exceed `type(uint256).max - VIRTUAL_SHARES`, mark that epoch rejected and refundable while allowing the rest of settlement to continue. Do not revert the whole settlement. Use an explicit checked error before every conversion.

Also remove the amplification primitive. When Junior principal is fully consumed, retire or reset economically worthless Junior shares before later recapitalization. Add an emergency fail-open redemption or pro-rata migration path for any already terminalized supply.

## References

- `packages/perps/src/TrancheVault.sol:1382-1409`
- `packages/perps/src/TrancheVault.sol:1611-1617`
- `packages/perps/src/TrancheVault.sol:1852-1867`
- `packages/perps/src/HousePool.sol:899-934`
- `packages/perps/src/HousePool.sol:944-1027`
- `packages/perps/WHITEPAPER.md`, section 8.6

## Proof of Concept

Secret PoC gist containing all four Foundry files: https://gist.github.com/niafreu/1edc4591464ce23acf143dd9398bf2af

- Primary terminal PoC: `JuniorVirtualShareTerminalFreezeExperiment.t.sol`
- Primary production-stack base fixture: `JuniorVpiSupplyOverflowFreezePoC.t.sol`
- Lower-capital terminal PoC: `JuniorVirtualShareCapitalScaleExperiment.t.sol`
- Lower-capital base fixture: `JuniorVpiSupplyCapitalScaleExperimentBase.t.sol`

The PoC uses the release contracts and production Router interfaces at the scoped commit. It extends the existing production-stack amplification fixture with the deterministic terminal tuning and exact victim-balance assertions.

The lower-capital PoC changes only the base fixture's initial Senior and Junior deposits and generalizes the terminal-deficit assertions. It uses the same production calls, risk parameters, coupon schedule, oracle observations, and terminal tuning.

Copy the primary terminal PoC and its base fixture into `packages/perps/test/perps/poc/`, then run from the repository root:

```bash
FOUNDRY_PROFILE=quick FOUNDRY_OUT=out-terminal-tune FOUNDRY_CACHE_PATH=cache-terminal-tune forge test --root packages/perps --offline --gas-limit 1200000000 --match-path test/perps/poc/JuniorVirtualShareTerminalFreezeExperiment.t.sol --match-test test_ExactRawSupplyTerminalStatePermanentlyFreezesJuniorRedemption -vv
```

Expected result:

```text
[PASS] test_ExactRawSupplyTerminalStatePermanentlyFreezesJuniorRedemption()
amplification cycles: 30
coupon-only wipe seconds: 93312000
coupon checkpoints: 12
checkpoint Senior claims: 122.144517
burn shares: 17311190792269966370455442268904914701723780753370903839925300381296
terminal aggregate deposit: 6693.710631
terminal Junior supply: 115792089237316195423570985008687907853269984665640564039457584007913129639109
victim deposit permanently frozen: 6692.710631
recoverable Senior exit: 12500149.612623
cycler remainder: 116.999461
```

For the lower-capital control, copy its two files into the same `test/perps/poc/` directory and run:

```bash
FOUNDRY_PROFILE=quick FOUNDRY_OUT=out-capital-scale FOUNDRY_CACHE_PATH=cache-capital-scale forge test --root packages/perps --offline --gas-limit 1200000000 --match-path test/perps/poc/JuniorVirtualShareCapitalScaleExperiment.t.sol --match-test test_ExactRawSupplyTerminalStatePermanentlyFreezesJuniorRedemption -vv
```

Expected lower-capital result: one passing test, 36 amplification cycles, `2207.692064` USDG of victim funds frozen, and raw Junior supply `uint256.max - 586` before the one-share dust burn.
