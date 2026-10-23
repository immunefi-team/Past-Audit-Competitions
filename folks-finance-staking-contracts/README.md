# Folks Finance: Staking Contracts

## Reports by Severity

<details>

<summary>Low</summary>

* \#69382 \[SC-Low] Irrevocable Migration Permit: Users Cannot Revoke Permit After Migrator Role Revocation
* \#69031 \[SC-Low] User cannot revoke permission from migrator if it does not have \`MIGRATOR\_ROLE\`
* \#68994 \[SC-Low] Users Cannot Revoke Migration Permits After Migrator Role Is Removed
* \#69476 \[SC-Low] Users cannot revoke stale migration approvals after a migrator is offboarded, so old permits can silently reactivate
* \#69263 \[SC-Low] Stale Migration Permit Reactivation in Folks Finance Staking Contract
* \#69493 \[SC-Low] Users cannot revoke permit for a role revoked migrator leading to residual permit risk if such migrator's role is ever reinstated
* \#69423 \[SC-Low] Audit: Multiple authorization and migration bugs in Folks staking lead to direct theft, fund freezing, and operational failure
* \#69794 \[SC-Low] User cannot revoke migration approval if migrator loses MIGRATOR\_ROLE
* \#69330 \[SC-Low] Revoked migrators leave non-revocable stale permits that reactivate on role re-grant
* \#69756 \[SC-Low] Staking.setMigrationPermit - Unnecessary hasRole Check on Revocation Blocks Users From Managing Own Permits
* \#69605 \[SC-Low] Users Cannot Revoke Migration Authorization After Role Revocation, Contrary to Documented Behavior
* \#69777 \[SC-Low] setMigrationPermit does not deliver on specified functionalities
* \#69218 \[SC-Low] Access control defect in \`setMigrationPermit\` leads to irrevocable stale migration permits
* \#69463 \[SC-Low] Stale migration permits can be reactivated by re-granting MIGRATOR\_ROLE to a previously approved migrator
* \#69570 \[SC-Low] Users cannot revoke migration approvals for removed migrators, Contrary to what the docs says
* \#69898 \[SC-Low] Stale migration approvals allow a re-authorized migrator to move user positions without renewed consent
* \#69008 \[SC-Low] Denial of Service on Migration Permit Revocation
* \#69650 \[SC-Low] setMigrationPermit blocks revocation after role revoke, enabling stale consent reuse
* \#68903 \[SC-Low] Users cannot revoke a migration permit after the migrator loses MIGRATOR\_ROLE, allowing stale approval to reactivate if the same address is re-granted the role
* \#68955 \[SC-Low] Unconditional hasRole check in setMigrationPermit() — Authorization Entrapment
* \#69146 \[SC-Low] README states migration permission can be revoked at any time, but revocation becomes impossible after MIGRATOR\_ROLE is removed
* \#69663 \[SC-Low] Users Cannot Revoke Previously Granted Migration Permit After Migrator Role is Revoked
* \#69136 \[SC-Low] Missing revocation condition in setMigrationPermit prevents users from revoking stale migration permissions, violating documented protocol guarantee
* \#69097 \[SC-Low] Broken migration permit revocation allows a re-authorized migrator to transfer user principal and rewards without fresh consent
* \#69836 \[SC-Low] setMigrationPermit blocks users from revoking permits after role removal -- stale permits auto-reactivate on re-grant and drain user funds
* \#69673 \[SC-Low] Users Cannot Revoke a Migration Permit After Role Removal
* \#69966 \[SC-Low] Cannot Revoke Migration Permit After Role Revocation (Stale Permits Re-Activate on Re-Grant)
* \#69717 \[SC-Low] Users are unable to revoke migration permits for deprecated or demoted migrators
* \#69390 \[SC-Low] Users Cannot Revoke Migration Permit At Any Time, Breaking Documented Guarantee
* \#69396 \[SC-Low] Users unable to remove migration permission from migrator who had role revoked
* \#69964 \[SC-Low] Users Cannot Revoke Migration Permission After Migrator Role Revocation
* \#69678 \[SC-Low] Lack of conditional role check in setMigrationPermit prevents users from revoking permits, leading to unauthorized migration and theft of unclaimed yield
* \#69376 \[SC-Low] Incorrect guard in \`setMigrationPermit\` prevents revocation after role removal, breaking documented user control
* \#69738 \[SC-Low] \`setMigrationPermit\` Prevents Users from Revoking Stale Permits After Migrator Role Is Revoked
* \#69275 \[SC-Low] Protocol's Explicit "Revoke at Any Time" Promise Broken — Users Cannot Revoke Migration Consent During Incident Window
* \#69747 \[SC-Low] Broken migration permit revocation allows stale user consent to reactivate after MIGRATOR\_ROLE is re-granted
* \#69505 \[SC-Low] User Cannot Revoke Migration Permit After Migrator Role Is Revoked
* \#69410 \[SC-Low] Migration Permit Cannot Be Revoked After Migrator Role Removal
* \#69527 \[SC-Low] Users Cannot Revoke Migration Authorization After Migrator Role Removal
* \#69524 \[SC-Low] Role Validation on Revocation Can Lock Migration Permits
* \#69769 \[SC-Low] \`setMigrationPermit\` Prevents Users From Revoking Migration Consent After Migrator Role Is Revoked
* \#69860 \[SC-Low] Users are permanently prevented from revoking migration permits if the Migrator's role is temporarily or permanently revoked
* \#69345 \[SC-Low] Migration permits cannot be revoked after \`MIGRATOR\_ROLE\` is revoked, despite README claiming revocation is possible "at any time"
* \#69908 \[SC-Low] Stale migration approvals cannot be revoked after role revocation and automatically reactivate on role re-grant
* \#69929 \[SC-Low] Inability to revoke migrationPermits for revoked migrators leads to permanent state persistence of user approvals
* \#69926 \[SC-Low] Users cannot revoke migration permits after \`MIGRATOR\_ROLE\` is removed, enabling fund migration without re-consent
* \#69814 \[SC-Low] Stale migration permits cannot be revoked after Migrator Role removal
* \#69936 \[SC-Low] Users cannot revoke migration permits once the migrator’s role has been revoked
* \#69956 \[SC-Low] Users Cannot Revoke Migration Permits After MIGRATOR\_ROLE Is Revoked - Stale Permits Enable Unconsented Future Migrations
* \#69100 \[SC-Low] Permit Irrevocability After MIGRATOR\_ROLE Revocation
* \#69962 \[SC-Low] Users cannot revoke migration permission during MIGRATOR\_ROLE rotation window
* \#69141 \[SC-Low] setMigrationPermit revocation silently blocked for de-listed migrators, contradicting documented guarantee
* \#69188 \[SC-Low] \`setMigrationPermit\` revoke blocked after Migrator role revocation
* \#69278 \[SC-Low] Migration permission can not be removed from the \`migrator\` if its \`MIGRATOR\_ROLE\` is revoked in advance
* \#69890 \[SC-Low] Users won't be able to revoke migration permits from revoked migrators

</details>

<details>

<summary>Insight</summary>

* \#69245 \[SC-Insight] No View Function to Compute Current Claimable Amounts
* \#69420 \[SC-Insight] Avoid the use of floating pragma to ensure same compiler version used for testing is also used for deployment
* \#69587 \[SC-Insight] recovered event missing recipient makes fund attribution impossible with multiple managers
* \#69772 \[SC-Insight] After a revert, stakeWithPermit might be prevented
* \#68906 \[SC-Insight] Missing Reentrancy Guard on function \`recoverERC20\`
* \#68880 \[SC-Insight] Missing "reward" Parameter in "Staked" Event Breaks Off-Chain Accounting
* \#68995 \[SC-Insight] Event Parameter Typo, \`referer\` in Staked Event vs \`referrer\` in StakeParams Struct
* \#68970 \[SC-Insight] Insufficient Event Emission in migratePositionsFrom Leads to Loss of Migration Accounting Visibility
* \#68879 \[SC-Insight] Essential Function Declarations Missing from \`IStakingV1\`
* \#68870 \[SC-Insight] Reward Calculation Intermediate Multiplication Overflow
* \#68849 \[SC-Insight] \`elapsed\` Computed Twice in \`\_withdraw\` (Code Optimization)
* \#69540 \[SC-Insight] Missing Return Value on withdraw and Missing View Function for Withdrawable Amount
* \#68872 \[SC-Insight] Copy-Paste Typo in Error Parameter Names
* \#68983 \[SC-Insight] \`stakeTime\` field in \`UserStake\` struct is stored but never used on-chain, wasting storage on every stake
* \#69870 \[SC-Insight] Events emitted after external calls in recoverERC20 and migratePositionsFrom violate CEI pattern

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#69382 \[SC-Low] Irrevocable Migration Permit: Users Cannot Revoke Permit After Migrator Role Revocation
* \#69031 \[SC-Low] User cannot revoke permission from migrator if it does not have \`MIGRATOR\_ROLE\`
* \#68994 \[SC-Low] Users Cannot Revoke Migration Permits After Migrator Role Is Removed
* \#69245 \[SC-Insight] No View Function to Compute Current Claimable Amounts
* \#69476 \[SC-Low] Users cannot revoke stale migration approvals after a migrator is offboarded, so old permits can silently reactivate
* \#69263 \[SC-Low] Stale Migration Permit Reactivation in Folks Finance Staking Contract
* \#69493 \[SC-Low] Users cannot revoke permit for a role revoked migrator leading to residual permit risk if such migrator's role is ever reinstated
* \#69423 \[SC-Low] Audit: Multiple authorization and migration bugs in Folks staking lead to direct theft, fund freezing, and operational failure
* \#69794 \[SC-Low] User cannot revoke migration approval if migrator loses MIGRATOR\_ROLE
* \#69330 \[SC-Low] Revoked migrators leave non-revocable stale permits that reactivate on role re-grant
* \#69756 \[SC-Low] Staking.setMigrationPermit - Unnecessary hasRole Check on Revocation Blocks Users From Managing Own Permits
* \#69420 \[SC-Insight] Avoid the use of floating pragma to ensure same compiler version used for testing is also used for deployment
* \#69587 \[SC-Insight] recovered event missing recipient makes fund attribution impossible with multiple managers
* \#69605 \[SC-Low] Users Cannot Revoke Migration Authorization After Role Revocation, Contrary to Documented Behavior
* \#69772 \[SC-Insight] After a revert, stakeWithPermit might be prevented
* \#69777 \[SC-Low] setMigrationPermit does not deliver on specified functionalities
* \#68906 \[SC-Insight] Missing Reentrancy Guard on function \`recoverERC20\`
* \#69218 \[SC-Low] Access control defect in \`setMigrationPermit\` leads to irrevocable stale migration permits
* \#69463 \[SC-Low] Stale migration permits can be reactivated by re-granting MIGRATOR\_ROLE to a previously approved migrator
* \#69570 \[SC-Low] Users cannot revoke migration approvals for removed migrators, Contrary to what the docs says
* \#68880 \[SC-Insight] Missing "reward" Parameter in "Staked" Event Breaks Off-Chain Accounting
* \#68995 \[SC-Insight] Event Parameter Typo, \`referer\` in Staked Event vs \`referrer\` in StakeParams Struct
* \#68970 \[SC-Insight] Insufficient Event Emission in migratePositionsFrom Leads to Loss of Migration Accounting Visibility
* \#69898 \[SC-Low] Stale migration approvals allow a re-authorized migrator to move user positions without renewed consent
* \#69008 \[SC-Low] Denial of Service on Migration Permit Revocation
* \#69650 \[SC-Low] setMigrationPermit blocks revocation after role revoke, enabling stale consent reuse
* \#68903 \[SC-Low] Users cannot revoke a migration permit after the migrator loses MIGRATOR\_ROLE, allowing stale approval to reactivate if the same address is re-granted the role
* \#68955 \[SC-Low] Unconditional hasRole check in setMigrationPermit() — Authorization Entrapment
* \#69146 \[SC-Low] README states migration permission can be revoked at any time, but revocation becomes impossible after MIGRATOR\_ROLE is removed
* \#69663 \[SC-Low] Users Cannot Revoke Previously Granted Migration Permit After Migrator Role is Revoked
* \#69136 \[SC-Low] Missing revocation condition in setMigrationPermit prevents users from revoking stale migration permissions, violating documented protocol guarantee
* \#69097 \[SC-Low] Broken migration permit revocation allows a re-authorized migrator to transfer user principal and rewards without fresh consent
* \#69836 \[SC-Low] setMigrationPermit blocks users from revoking permits after role removal -- stale permits auto-reactivate on re-grant and drain user funds
* \#69673 \[SC-Low] Users Cannot Revoke a Migration Permit After Role Removal
* \#69966 \[SC-Low] Cannot Revoke Migration Permit After Role Revocation (Stale Permits Re-Activate on Re-Grant)
* \#69717 \[SC-Low] Users are unable to revoke migration permits for deprecated or demoted migrators
* \#68879 \[SC-Insight] Essential Function Declarations Missing from \`IStakingV1\`
* \#69390 \[SC-Low] Users Cannot Revoke Migration Permit At Any Time, Breaking Documented Guarantee
* \#69396 \[SC-Low] Users unable to remove migration permission from migrator who had role revoked
* \#69964 \[SC-Low] Users Cannot Revoke Migration Permission After Migrator Role Revocation
* \#68870 \[SC-Insight] Reward Calculation Intermediate Multiplication Overflow
* \#68849 \[SC-Insight] \`elapsed\` Computed Twice in \`\_withdraw\` (Code Optimization)
* \#69678 \[SC-Low] Lack of conditional role check in setMigrationPermit prevents users from revoking permits, leading to unauthorized migration and theft of unclaimed yield
* \#69376 \[SC-Low] Incorrect guard in \`setMigrationPermit\` prevents revocation after role removal, breaking documented user control
* \#69540 \[SC-Insight] Missing Return Value on withdraw and Missing View Function for Withdrawable Amount
* \#69738 \[SC-Low] \`setMigrationPermit\` Prevents Users from Revoking Stale Permits After Migrator Role Is Revoked
* \#69275 \[SC-Low] Protocol's Explicit "Revoke at Any Time" Promise Broken — Users Cannot Revoke Migration Consent During Incident Window
* \#69747 \[SC-Low] Broken migration permit revocation allows stale user consent to reactivate after MIGRATOR\_ROLE is re-granted
* \#69505 \[SC-Low] User Cannot Revoke Migration Permit After Migrator Role Is Revoked
* \#69410 \[SC-Low] Migration Permit Cannot Be Revoked After Migrator Role Removal
* \#69527 \[SC-Low] Users Cannot Revoke Migration Authorization After Migrator Role Removal
* \#69524 \[SC-Low] Role Validation on Revocation Can Lock Migration Permits
* \#69769 \[SC-Low] \`setMigrationPermit\` Prevents Users From Revoking Migration Consent After Migrator Role Is Revoked
* \#69860 \[SC-Low] Users are permanently prevented from revoking migration permits if the Migrator's role is temporarily or permanently revoked
* \#69345 \[SC-Low] Migration permits cannot be revoked after \`MIGRATOR\_ROLE\` is revoked, despite README claiming revocation is possible "at any time"
* \#69908 \[SC-Low] Stale migration approvals cannot be revoked after role revocation and automatically reactivate on role re-grant
* \#69929 \[SC-Low] Inability to revoke migrationPermits for revoked migrators leads to permanent state persistence of user approvals
* \#69926 \[SC-Low] Users cannot revoke migration permits after \`MIGRATOR\_ROLE\` is removed, enabling fund migration without re-consent
* \#69814 \[SC-Low] Stale migration permits cannot be revoked after Migrator Role removal
* \#69936 \[SC-Low] Users cannot revoke migration permits once the migrator’s role has been revoked
* \#68872 \[SC-Insight] Copy-Paste Typo in Error Parameter Names
* \#68983 \[SC-Insight] \`stakeTime\` field in \`UserStake\` struct is stored but never used on-chain, wasting storage on every stake
* \#69956 \[SC-Low] Users Cannot Revoke Migration Permits After MIGRATOR\_ROLE Is Revoked - Stale Permits Enable Unconsented Future Migrations
* \#69100 \[SC-Low] Permit Irrevocability After MIGRATOR\_ROLE Revocation
* \#69870 \[SC-Insight] Events emitted after external calls in recoverERC20 and migratePositionsFrom violate CEI pattern
* \#69962 \[SC-Low] Users cannot revoke migration permission during MIGRATOR\_ROLE rotation window
* \#69141 \[SC-Low] setMigrationPermit revocation silently blocked for de-listed migrators, contradicting documented guarantee
* \#69188 \[SC-Low] \`setMigrationPermit\` revoke blocked after Migrator role revocation
* \#69278 \[SC-Low] Migration permission can not be removed from the \`migrator\` if its \`MIGRATOR\_ROLE\` is revoked in advance
* \#69890 \[SC-Low] Users won't be able to revoke migration permits from revoked migrators

</details>
