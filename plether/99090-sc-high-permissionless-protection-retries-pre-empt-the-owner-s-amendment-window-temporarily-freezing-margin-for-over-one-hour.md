# \#99090 [SC-High] Permissionless protection retries pre-empt the owner's amendment window, temporarily freezing margin for over one hour

**Submitted on Nov 11th 2026 at 20:16:41 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99090
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

The MED-9 fix introduced per-protection close limits so that an owner can bound the execution of a triggered stop-loss. When those limits stop passing, for example after a price gap through a stop-limit, the protection relatches and the owner's sole documented remedy is `amendLatchedPositionProtectionCloseLimits` (`PositionProtectionBook.sol:315-328`). That function requires the record to be `Latched` and the account to have zero pending Router orders (`:324`).

`retryPositionProtectionClose` (`PositionProtectionBook.sol:382-406`) is permissionless: it has no caller check and requires exactly the same two conditions, `Latched` plus zero pending orders. A retry commits a fresh close attempt that copies the stored, still failing close limits (`router/OrderLiquidationBatchLogic.sol:657-680`). Because `executeOrder` is also permissionless, any caller can, in one transaction, execute the current failing attempt, which relatches the record, and immediately retry, which commits a fresh attempt with the same stale limits.

At every transaction boundary after that, the record is `Triggered` and the account has one pending order, so the owner's paths all revert: `amendLatchedPositionProtectionCloseLimits` reverts `ProtectionNotLatched` because the status is no longer `Latched`, `cancelPositionProtection` reverts `ProtectionNotArmed` because only `PendingOpen` and `Armed` can be cancelled (`:408-430`), `cancelPositionProtectionAndCommitClose` reverts for the same reason, and an ordinary close reverts `ProtectionActive` (`router/OrderLiquidationBatchLogic.sol:313-315`). The owner therefore cannot close, reduce, amend or cancel, while the attacker repeats one zero-capital transaction per execution window, 30 seconds in the PoC and 60 seconds at the release configuration.

Impact, measured end to end in the repository suite. A LONG of 10,000 USDG notional with 2,000 USDG of margin arms a stop-loss whose notional ceiling stops passing after a gap, the record latches, and the attacker then performs 126 transactions over 3,906 seconds. At every round boundary the owner amendment, cancellation and ordinary close all revert, the account has one pending order, the protection is `Triggered`, and the position with 1,985.500999 USDG of pledged margin is still open. The attacker spends no USDG, only gas.

A second test measures the damage rather than the lock alone. World A repeats the attack once per window for 39 rounds while the price moves 0.002 per round against the position; the victim attempts all three exits in every round, 117 attempts, and every one reverts. At 1.198 the account is liquidatable, and the production liquidation path, which bypasses the order queue, closes it and forfeits the protection. World B starts from the same snapshot without the attacker: the owner amends, retries and closes at 1.122, then holds a flat account through the same price path. Measured in USDG with six decimals:

- World A, with the attacker: victim loses 1,990.000233, composed of 1,980.000 of realized price loss and a 10.000 liquidation charge; 5.500766 remains.
- World B, without the attacker: victim loses 1,223.988999, composed of 1,220.000 of price and 3.988999 of fees; 771.512000 remains.
- The difference caused by the lock is 766.011234 USDG, which is 38.30 percent of the margin and 7.66 percent of the notional. Of that, 760.000000 is the forced price displacement from 1.122 to 1.198 and 6.011234 is the liquidation path costing more than a voluntary close.
- The attacker spends 40 transactions, 79 calls and 40,988,539 execution gas, posts no capital and earns nothing; the 5.000 bounty goes to the liquidator.
- Conservation holds exactly: world A is -1,990.000233 for the victim, +1,985.000233 for the pool, +0 for the treasury and +5.000000 for the keeper; world B is -1,223.988999, +1,220.000000 and +3.988999.

Known issue separation. SECURITY.md acknowledges that failed protection attempt cleanup is deliberately unpaid, that each retry creates a fresh zero-bounty close record, and that liveness depends on keepers processing the terminal attempt and separately appending a retry. That describes the intended retry lifecycle and keeper incentives. It does not describe, acknowledge or mitigate the collision between the permissionless retry and the owner's amendment window: the retry keeps the pending count above zero and thereby blocks the only owner remedy. This is a consequence of the MED-9 limits interacting with the pre-existing retry mechanism, not a documented design trade-off. The AI scan notes that a Triggered or Latched protection cannot be cancelled to regain an ordinary close, and SECURITY.md documents the one way door after a genuine trigger; both describe the state after a trigger, not an unbounded extension of it by a third party.

Owner escape path, disclosed. On Arbitrum there is no public mempool, so the owner can submit `executeOrder` and `amendLatchedPositionProtectionCloseLimits` as two sequential transactions; if no attacker transaction interposes, the amendment lands while the pending count is zero. However this requires the owner to know that the two calls must land together, which the documented entry points do not describe, and it presents amendment as a standalone action. A smart account or EIP-7702 batching can do both atomically, but an ordinary externally owned wallet using a standard frontend cannot guarantee it, and the attacker can retry at every window indefinitely at the cost of gas.

Remediation: allow the owner to amend while an attempt is live and invalidate that attempt, re-reading the amended limits into the next attempt rather than requiring zero pending orders; alternatively restrict `retryPositionProtectionClose` to the owner after a failure caused by owner-set limits, since a third-party retry can only block; alternatively give the owner an exit from `Latched` and `Triggered`, mirroring `cancelPositionProtectionAndCommitClose` for `Armed`.

## Link to Proof of Concept

https://gist.github.com/crypt0b0y/606ad4145472dc7de24d461680783ad2

## Proof of Concept

Environment: forge 1.8.1, solc 0.8.35, optimizer 200, via_ir, HEAD a82d4fef. Runs are local, no network transactions, in-scope contracts unmodified; only the Pyth feed and the settlement token are mocked. Release risk parameters are applied through the harness: vpiFactor 0.01e18, maintMarginBps 10, initMarginBps 20, fadMarginBps 300.

END TO END REPRODUCTION (primary evidence): packages/perps/test/poc/G2_EndToEnd.t.sol, tests test_attack_EndToEnd and test_control_EndToEnd, 2 passed. The test starts from a healthy pool and does not begin from a latched snapshot. Run:

forge test --root packages/perps --match-path "test/poc/G2_EndToEnd.t.sol" -vv

Step by step:
1. Healthy pool, totalAssets 1,002,000 USDG. The trader commits and executes a LONG of 10,000e18 with 2,000e6 of margin at 1.00; pledged margin becomes 1,995.500999.
2. The trader arms a stop-loss at 1.10 with maxExecutionNotionalAssets of 11,000e6, a stop-limit whose notional ceiling is computed at the stop price.
3. The basket gaps to 1.12. A keeper triggers the protection; the close attempt executes and fails on its own ceiling: status Latched (8), reason 8, failure selector 0x1b7598b3, constraint ExecutionNotional, actual 11,200 against limit 11,000.
4. The attacker front-runs with a retry, then runs 39 rounds of 60 seconds, each round executing the failed attempt and retrying in one transaction. All 39 attacker attempts fail on the same ceiling, so no attempt ever expires or executes; the record stays Triggered with one pending order.
5. The victim attempts all three exits in every round, 117 attempts in total, and every one reverts: amend returns OrderRouter__ProtectionNotLatched (0x5b54e624), cancel returns OrderRouter__ProtectionNotArmed (0x2dee7d8f), an ordinary close returns OrderRouter__ProtectionActive (0x3a0535a3).
6. The price moves 0.002 per round against the position. At 1.198 the account is liquidatable (notional 11,980, equity 5.500764) and router.executeLiquidation, which bypasses the order queue, closes it and forfeits the protection.
7. World B, from the same healthy start and without the attacker: the owner amends the ceiling, retries and closes at 1.122, then holds a flat account through the same price path.

Measured losses, USDG with six decimals:
- World A, with the attacker: 1,990.000235, being 1,980.000 realized price plus a 10.000 liquidation charge and 0.000235 carry dust; 5.500764 remains.
- World B, without the attacker: 1,223.988999, being 1,220.000 of price plus 3.988999 of fees; 771.512000 remains.
- Difference caused by the lock: 766.011236 USDG, which is 38.30 percent of the 2,000 margin and 7.66 percent of the notional. It decomposes into 760.000000 of forced exposure between 1.122 and 1.198, 6.011001 because liquidation charges 10.000 where a voluntary close pays 3.988999, and 0.000235 of carry dust.
- Attacker cost: 40 transactions, 79 calls, 58,253,944 execution gas plus about 840k intrinsic, no USDG capital, no profit; the 5.000 keeper bounty goes to the liquidator.
- Conservation asserted to zero: world A is -1,990.000235 for the victim, +1,985.000235 for the pool, +0 treasury, +5.000000 keeper. World B is -1,223.988999, +1,220.000000, +3.988999, +0.

SUPPORTING TESTS in the same secret gist: G2_ProtectionRetryGrief.t.sol, 2 passed, shows 126 attacker transactions over 3,906 seconds with the position and 1,985.500999 USDG of pledged margin still locked, and a control without the attacker where the owner amends and exits; G2_LiquidationDuringLock.t.sol, 1 passed, isolates the liquidation step from a latched snapshot and reproduces the same 766 USDG difference.

Controls: without the attacker the owner amends and exits on the first step. The attacker cannot create the precondition, since the ceiling failure comes from the victim's own limit after a price gap. If the owner submits execute and amend together, or atomically through a smart account, the amendment lands before any retry and the lock does not start.

Not tested: the take-profit side, a partially filled position, several protections for one account, and competition with another keeper for the same window.
