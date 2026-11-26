# \#98114 [SC-Critical] LP ownership rotation lets a trader redeem positive VPI revenue and make a queued depositor fund the matching rebate

**Submitted on Nov 5th 2026 at 07:34:51 UTC by @hetroublemakr for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98114
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

Plether immediately recognizes a positive virtual price impact (VPI) charge as revenue owned by the current LP cohort. That revenue raises Junior principal and is included when an existing Junior LP redeems. The same open position nevertheless retains a positive lifetime VPI balance that can produce an equal rebate when the position closes at the same price and economic depth. Plether does not reserve this future rebate in LP entry pricing or the withdrawal firewall.

An attacker can therefore age a Junior LP position, wait for a victim's async Junior deposit to be queued, open a maximum-skew position immediately before that deposit activates, and make the victim enter at NAV that includes the VPI charge. The attacker then redeems all old LP shares, including the charge, and closes the position at the unchanged price after the victim's principal has replaced the withdrawn liquidity. The close rebate is paid from the replacement LP's cash.

The exact release-configuration PoC uses only public Router, vault, and permissionless keeper entry points. With a `1,000,000 USDG` attacker LP position and a `1,000,000 USDG` queued victim deposit, it produces:

```text
Attacker LP deposit:             1,000,000.000000 USDG
Attacker LP redemption:          1,000,799.997578 USDG
Victim value before close:         999,999.999995 USDG
Victim value after close:          999,360.439757 USDG
Victim principal shortfall:            639.560243 USDG
Combined attacker profit:              479.360065 USDG
Open economic depth:              1,000,002.000000 USDG
Close economic depth:             1,000,002.000822 USDG
Open-to-close duration:                   3,601 seconds
```

The close depth is slightly higher than the open depth. The exploit therefore does not use the publicly known lower-depth VPI-rebate amplification. A control with the same open and close but no LP cohort change loses `160.200161 USDG`.


## Impact

An ordinary LP/trader can transfer a matching VPI repayment from itself to a later Junior LP cohort and take part of that cohort's deposited principal as profit. In the exact release-configuration PoC, the victim contributes `1,000,000 USDG`, owns no historical VPI yield, and falls to `999,360.439757 USDG` after an unchanged-price close, while the attacker gains `479.360065 USDG`. This is not theft of unclaimed yield: the victim's post-activation shares are worth essentially exactly its deposited principal before the close and less than principal afterward. Repeating the same ownership rotation against future queued deposits progressively depletes Junior backing.

## Link to Proof of Concept

https://gist.github.com/hetroublehacker/1fd4056425553a827df976a647bd5902

## Proof of Concept

## Steps to Reproduce

1. Clone the frozen competition repository and check out the in-scope commit:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-plether.git
   cd audit-comp-plether
   git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
   ```

2. Save the single gist-ready PoC [`vpi-cohort-rotation-theft.t.sol`](vpi-cohort-rotation-theft.t.sol) at this repository path:

   ```text
   packages/perps/test/perps/accounting/close/VpiLpOwnershipRotationPublicPath.t.sol
   ```

   The PoC SHA-256 is:

   ```text
   926a52d2cde6889c14d9f689cf15fdeac02b30d42d93570df52b099c4bf50cdb
   ```

3. Run the two focused tests from the repository root:

   ```bash
   forge test --root packages/perps \
     --match-path 'test/perps/accounting/close/VpiLpOwnershipRotationPublicPath.t.sol' -vv
   ```

4. Confirm both tests pass and the exploit test prints these exact atom values:

   ```text
   public max-skew position size: 400000000000000000000000
   public opening depth: 1000002000000
   public closing depth: 1000002000822
   public attacker LP payout: 1000799997578
   public victim value before close: 999999999995
   public victim value after close: 999360439757
   public victim principal shortfall: 639560243
   public open protocol fee: 160000000
   public close protocol fee: 0
   public attacker profit: 479360065
   public open-to-close seconds: 3601
   ```

5. Confirm the static-cohort control also passes and prints:

   ```text
   static-cohort attacker loss: 160200161
   ```

6. Verify that the exploit test enforces the facts needed to distinguish the report from the known depth issue:

   ```solidity
   assertGe(run.closeDepth, run.openDepth, "attack must not use the known lower-depth rebate amplification");
   assertApproxEqAbs(run.closeDepth, run.openDepth, 2e6, "the proof must preserve economic depth");
   assertEq(juniorVault.balanceOf(ATTACKER), 0, "attacker must own no LP shares when the rebate is paid");
   assertLt(run.victimValueAfterClose, VICTIM_LP_ASSETS, "unchanged-price close must consume the victim's deposited principal");
   assertGt(run.attackerEndWealth, attackerStartWealth, "combined attacker wealth must increase");
   ```

No RPC endpoint, fork, public testnet, production account, or real user funds are required.
