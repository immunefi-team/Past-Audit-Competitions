# 69423 sc low audit multiple authorization and migration bugs in folks staking lead to direct theft fund freezing and operational failure

## `#69423` \[SC-Low] Audit: Multiple authorization and migration bugs in Folks staking lead to direct theft, fund freezing, and operational failure

**Submitted on Mar 14th 2026 at 20:53:23 UTC by @enspire for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** `#69423`
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Smart contract unable to operate due to lack of token funds
  * Permanent freezing of unclaimed yield

### Description

## Combined Audit Submission Description

### Brief/Intro

This submission covers the reproduced findings from the Folks staking contracts audit. The reproduced issues span authorization flaws, migration-state/accounting issues, unsafe staking-period configuration, reward reserve handling problems, and lower-severity correctness or observability issues. If left unresolved in production, the report shows that some findings can lead to direct theft of user funds, some can permanently freeze user value or claimability, and others can make the staking system partially or fully unusable.

### Vulnerability Details

The reproduced findings fall into several main categories.

The highest-risk category is the migration authorization lifecycle. In particular, `migratePositionsFrom()` checks `migrationPermits[msg.sender][user]` before moving user positions, but the authorization is not revoked after successful use. This creates a persistent approval model where a migrator can remain authorized long after the user reasonably believes the migration process is complete. A related migration-permit issue exists when role revocation prevents users from removing stale permits, allowing old approvals to silently reactivate if the role is granted again later.

The next category is migration and indexing/accounting edge cases. The audit highlights that repeated migration cycles preserve historical stake entries, which increases storage pressure on the user's stake array. The current public flow in this repository reaches `MAX_STAKES_PER_USER` before the reported `uint8` wraparound path becomes reachable, but the reproduction still confirms the structural concern that migration preserves baggage and can consume future stake capacity. Another migration-related accounting issue is that `capUsed` is not decremented when stakes are migrated out, which can leave stale capacity usage and incorrectly block later deposits.

There is also a class of dangerous manager-controlled configuration issues. The contract accepts extremely large staking durations and unlock durations at configuration time without practical upper bounds. One effect is that a staking period can be accepted on-chain but later become unusable because unlock-time math overflows at stake time. Another effect is that funds can become practically inaccessible because extremely large unlock durations make claimable amounts round down to zero for unrealistic amounts of time.

The reward reserve and operational control logic also contain important weaknesses. In particular, reward tokens prefunded before any users stake are not yet reflected in `activeTotalStaked` or `activeTotalRewards`, so the manager can recover those tokens through `recoverERC20()`. This can leave the contract unable to reserve rewards for future deposits. In addition, pausing the contract blocks staking but does not block withdrawals, meaning the current pause mechanism acts as a soft inflow pause rather than a full emergency freeze.

Finally, several lower-severity findings concern event ordering, silent zero-effect behavior, and missing observability helpers. These include emitting `Staked` before transfer confirmation, allowing zero-claim withdrawals to succeed silently, emitting misleading migration events when nothing is moved, silently falling back from failed permit usage to existing allowance, and lacking a dedicated on-chain solvency view.

Together, these findings demonstrate that the core staking logic is not the only risk area; the migration subsystem, reserve-management assumptions, and operator-controlled configuration surface are especially important from a security and production-readiness perspective.

### Impact Details

The reproduced findings do not all share one single impact type, so the impact should be understood as a combined audit-level set of consequences.

The most severe reproduced impact is direct theft of user funds. The clearest case is the migration-permit persistence issue, where a previously approved migrator can later pull newly created user positions without renewed consent. This affects more than unclaimed yield, because migrated value includes both principal and reserved reward.

The reproduced findings also include permanent or effectively permanent freezing risks. The array-growth and indexing concerns identified in the report are framed as fund-locking risks, while the unlock-duration issue can make withdrawals economically or practically impossible for unrealistic timeframes even though the contract does not formally revert.

Another reproduced impact is service unavailability due to lack of token funds. The reward-reserve recovery issue allows prefunded reward liquidity to be withdrawn before it is tracked as protected balance, which can make future stakes fail because the contract can no longer satisfy reward reservation checks.

Additional findings cause griefing, false operational assumptions, or degraded protocol behavior. For example:

* pause does not provide a full emergency stop
* event emissions can mislead off-chain monitoring
* withdrawals can succeed with zero economic effect
* monitoring systems must reconstruct solvency off-chain instead of reading a dedicated view

At an audit-package level, the report therefore demonstrates three major practical outcomes:

* direct theft risk for user funds
* freezing or practical inaccessibility of user value
* operational failure or degradation of staking functionality

The audit severity distribution reproduced in this package is:

* `2 High`
* `4 Medium`
* `3 Low`
* `4 Informational`

### Proof of Concept

The reproduced audit package is available in my public repository:

* Repository: [ensp1re/folks-finance-audit](https://github.com/ensp1re/folks-finance-audit)
* Audit PDF: [folks-finance-audit-report.pdf](https://github.com/ensp1re/folks-finance-audit/blob/main/folks-finance-audit-report.pdf)
* Video walkthrough: <https://youtu.be/fN1kJHrA5sI>

Main PoC file:

* `test/AuditFindings.t.sol`

The audit-only test suite reproduces the full reported set:

* `test_Audit_H01_MigrationPermitRemainsUsableAfterSuccessfulMigration`
* `test_Audit_H02_RepeatedMigrationCyclesHitMaxUserStakesBeforeUint8Wrap`
* `test_Audit_M01_ZombiePermitReactivatesAfterMigratorRoleIsRegranted`
* `test_Audit_M02_UnboundedStakingDurationBricksPeriod`
* `test_Audit_M03_PauseDoesNotBlockWithdrawals`
* `test_Audit_M04_ManagerCanDrainPreFundedRewardPool`
* `test_Audit_L01_StakedEventIsEmittedBeforeTransferEvent`
* `test_Audit_L02_ZeroClaimWithdrawalSucceedsSilently`
* `test_Audit_L03_ExtremeUnlockDurationMakesWithdrawalsPracticallyZero`
* `test_Audit_I01_SecondMigrationCallEmitsMisleadingEvent`
* `test_Audit_I02_StakeWithPermitFallsBackToExistingAllowance`
* `test_Audit_I03_SolvencyMustBeReconstructedOffChain`
* `test_Audit_I04_MigrationLeavesPeriodCapUsedStale`

Run the full reproduced audit suite:

```shell
forge test --match-path test/AuditFindings.t.sol -vv
```
