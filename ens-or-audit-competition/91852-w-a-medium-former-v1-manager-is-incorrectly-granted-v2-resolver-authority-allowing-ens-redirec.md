# 91852 w a medium former v1 manager is incorrectly granted v2 resolver authority allowing ens redirection and nft theft

**Submitted on Sep 4th 2026 at 18:52:09 UTC by @Pig46940 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91852
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

ENS Manager can grant fresh V2 resolver authority to a former v1 manager after that address has relinquished its v1 Registry manager authority over the name. The migration remains executable by the current registrar owner, but Manager does not reread the current v1 Registry manager before building `grantRoles(resource, ROLE_SET_RESOLVER, formerManager)`.

The former manager can use the newly created role to make a legitimate third-party ENS identity resolve to the former manager's own address. A separate victim who enters that ENS identity as the recipient of a name transfer in Portal then receives the attacker-controlled resolution. The resulting Portal-generated ERC-1155 transfer can be approved and mined to the attacker. The PoC proves that the attacker becomes the name owner and that tested recovery attempts by the victim and a formerly approved operator revert.

## Vulnerability Details

The attack uses three primary actors:

* **Recipient Name Owner** owns `revoked-manager.eth` and authorizes its migration.
* **Former Manager / Attacker** is initially the legitimate v1 Registry manager of `revoked-manager.eth`.
* **Transfer Victim** owns the separate name `resolver-role-restoration-transferable.eth`.

The vulnerable sequence starts from a legitimate state: the Recipient Name Owner owns `revoked-manager.eth`, while the independently controlled Former Manager / Attacker is its valid v1 Registry manager. Manager fetches that state through `useV1Names` and retains it in a query with a five-minute `staleTime`.

The Former Manager / Attacker then relinquishes the v1 manager slot to the Recipient Name Owner. Direct contract reads show the Recipient Name Owner as both registrar owner and Registry manager. A v1 resolver write signed by the Former Manager / Attacker reverts, proving that the former manager no longer has v1 Registry manager authority over the name.

If the Recipient Name Owner continues the original mounted migration view before it refetches, the retained domain object still identifies the former manager as current manager:

```
retained domain.owner.id = formerManager
→ classifyName(...).managerAddress = formerManager
→ the current registrar owner remains authorized to migrate
→ current v1 Registry manager is not reread
→ buildRoleGrantCall(...)
→ grantRoles(resourceFor(recipientEnsName), ROLE_SET_RESOLVER, formerManager)
→ migrate and grant are included in the Recipient Name Owner's migration batch
```

Manager's manager-restoration confirmation describes the operation as:

> Restore the existing managers for your names.

The generated migration step descriptors do not identify the Former Manager / Attacker address receiving the restored role. At authorization time, that address is no longer an existing manager.

After the Manager-generated migration batch is signed by the Recipient Name Owner and mined, V2 state shows `getOwner(resourceFor("revoked-manager"))` remains the Recipient Name Owner while the Former Manager / Attacker has received `ROLE_SET_RESOLVER` through the stale restoration.

The attacker then uses Portal's resolver-change path to replace `revoked-manager.eth`'s resolver with an attacker-controlled resolver. The generated `setResolver` calldata matches the mined transaction input, the sender is the Former Manager / Attacker, and fresh Universal Resolver lookup of `revoked-manager.eth` returns the Former Manager / Attacker address.

A different user—the Transfer Victim—then uses Portal's normal transfer page to transfer `resolver-role-restoration-transferable.eth`. The victim enters `revoked-manager.eth`, a legitimate ENS identity still owned by the independent Recipient Name Owner, as the recipient. Portal performs a fresh resolution and returns the Former Manager / Attacker address. Portal displays both `revoked-manager.eth` and the resolved attacker address before the victim opens the wallet.

The resulting production path is:

```
SendNameForm
→ useAddressResolution(recipientEnsName)
→ fresh Universal Resolver result formerManagerAttacker
→ useTransferName
→ transferToken
→ transaction manager
→ EOA wallet request
→ safeTransferFrom(transferVictim, formerManagerAttacker, tokenId(victimName), 1, 0x)
```

The Portal transaction request and mined transaction input are byte-identical. The PoC decodes the mined call as:

```
safeTransferFrom(
  0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB,  // Transfer Victim
  0x6Bdfb8f07B6645990a32b0D76e678F1fCDAb1994,  // Former Manager / Attacker
  67230724246266163570754963139603361716680294319705997524303058030138576863232,
  1,
  0x
)
```

After mining, separate authoritative registry reads show that the Former Manager / Attacker owns `resolver-role-restoration-transferable.eth`. The Transfer Victim's balance is zero and the Former Manager / Attacker's balance is one.

As recovery controls, a transfer attempt by the former owner and one by an operator previously approved by that owner both revert. Restoring `revoked-manager.eth`'s legitimate resolver and revoking the Former Manager / Attacker's resolver role do not reverse the completed transfer. A transfer signed by the Former Manager / Attacker succeeds on a reverted snapshot, confirming that the attacker controls the transferred name.

The security-sensitive role grant is therefore built from a retained manager identity without revalidating the current authoritative v1 manager at authorization time. Manager should grant migrated V2 resolver authority only to an address that is still the current v1 Registry manager when the migration request is prepared and authorized.

Exploitation requires the migration view to retain the former manager identity after that manager relinquishes authority, and the migration must be authorized before that state is refreshed. The attacker must therefore previously have been a legitimate v1 manager.

## Impact Details

The selected published Critical impact is:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves

The first example applies directly. The in-scope Manager flow creates fresh resolver authority for the Former Manager / Attacker after that address's current manager authority is gone. The attacker uses that authority to make `revoked-manager.eth`, which remains owned by the independent Recipient Name Owner, resolve to the attacker.

The in-scope Portal transfer flow then consumes that attacker-controlled resolution and sends the Transfer Victim's ENS ERC-1155 name to the Former Manager / Attacker.

The terminal consequence is authoritative NFT loss to an independent attacker. The Former Manager / Attacker becomes the owner, while the victim's tested recovery and the formerly approved operator's tested recovery both revert. Restoring the recipient identity's resolver or revoking the attacker's resolver role prevents further abuse but does not reverse the completed transfer.

Portal displays the attacker-controlled resolved address before approval; this report relies on the published attacker-controlled ENS-resolution example, not on a display/calldata mismatch.

## References

* [`useV1Names` retains the indexed v1 manager for five minutes](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29)
* [`managerAddress` is derived from the retained domain owner](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L148-L186)
* [Role-grant construction uses `managerAddress`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26)
* [The manager-role grant is added to the migration batch](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
* [Manager confirmation describes restoration without identifying the address](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L108)
* [Portal resolves the current recipient input](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/hooks/useAddressResolution.ts#L38-L89)
* [Portal's recipient query uses ENS resolution](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/queries/getResolvedAddress.ts#L13-L45)
* [Transfer form passes the resolved address into the transfer preparation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L63-L109)
* [Portal displays the complete resolved recipient address](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L228-L281)
* [Saved recipient reaches the transfer step](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L145)
* [`safeTransferFrom` is encoded with the resolved recipient](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L92)

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/48f73e6a7cae79f16b5ed0fd2723a765>

## Proof of Concept

## Proof of Concept

The attached PoC is a single TypeScript witness. Actor and resource identifiers use the same names as this report (`recipientNameOwner`, `formerManagerAttacker`, `transferVictim`, `legitimateRecipient`, `formerApprovedOperator`, and `recipientIdentityName`). It imports the audited Manager classification, preflight, request-building, Portal resolver, transfer UI, transfer helper, and transaction-manager paths. Small non-decisive dependency shims are embedded as virtual modules in the Vitest configuration instead of separate TypeScript files. All authoritative execution is confined to a fresh loopback Anvil process.

It uses distinct deterministic local private-key accounts for the Former Manager / Attacker, the Recipient Name Owner, the Transfer Victim, the Legitimate Recipient, and the Former Approved Operator. The Former Approved Operator is used only as a recovery control and has no role in creating, redirecting, or receiving the transfer. The Former Manager / Attacker's same address is both the malicious resolution result and the final NFT owner. These are public test fixtures and must never be funded.

The PoC performs the following:

1. establishes truthful initial v1 state with the Recipient Name Owner as registrar owner and the Former Manager / Attacker as manager;
2. mounts the production `useV1Names` observer;
3. has the Former Manager / Attacker relinquish manager authority to the Recipient Name Owner;
4. proves the Recipient Name Owner now controls both authoritative v1 ownership dimensions and proves the former manager's old v1 write reverts;
5. passes the retained former-manager identity through production classification, preflight, and migration construction;
6. mines the exact request signed by the Recipient Name Owner and proves the Former Manager / Attacker receives `ROLE_SET_RESOLVER`;
7. runs a fresh-query negative control that does not grant the former manager that role;
8. has the attacker replace `revoked-manager.eth`'s resolver and proves fresh Universal Resolver lookup returns the Former Manager / Attacker address;
9. creates a new Portal QueryClient with no recipient cache;
10. mounts the production transfer form and proves `revoked-manager.eth` and the complete Former Manager / Attacker address remain displayed through **Open wallet**;
11. captures, submits, and mines the exact Portal wallet request;
12. proves the Former Manager / Attacker owns `resolver-role-restoration-transferable.eth` and that the victim and former-operator recovery attempts revert; and
13. proves a return signed by the Former Manager / Attacker succeeds on a reverted snapshot control.

Required tools are Node.js 22, pnpm, and Anvil. Check out the audited application revision and the referenced contracts-v2 revision, then run from the PoC directory:

The four-file executable PoC bundle is available in this Gist:

<https://gist.github.com/HiroyukiNaito/48f73e6a7cae79f16b5ed0fd2723a765>

From an empty working directory, reproduce it as follows. The contracts repository is used only for the pinned local deployment artifacts; no public chain or deployed ENS service is contacted.

```bash
# Confirm the required local tools are available.
node --version   # Node.js 22
pnpm --version
anvil --version

# Check out the executable public snapshot of the audited application revision.
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
git -C audit-comp-ens checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
corepack enable
pnpm --dir audit-comp-ens install --frozen-lockfile

# Check out the exact contracts-v2 revision used by the local fixture.
git clone https://github.com/ensdomains/contracts-v2.git contracts-v2
git -C contracts-v2 checkout 8d1c89350729f87b3968cf41ef3d683951444a44
git -C contracts-v2 submodule update --init --recursive

# Download the four-file PoC bundle from the Gist.
git clone https://gist.github.com/HiroyukiNaito/48f73e6a7cae79f16b5ed0fd2723a765.git ens-stale-manager-poc
chmod +x ens-stale-manager-poc/run-nft-loss-poc.sh

# Run the PoC against a fresh loopback-only Anvil instance.
cd ens-stale-manager-poc
ENS_THEFT_POC_ANVIL="$(command -v anvil)" \
  ./run-nft-loss-poc.sh ../audit-comp-ens ../contracts-v2
```

In the supplied audit workspace, both repositories are auto-detected and the command is simply `./run-nft-loss-poc.sh`. The runner checks both exact Git revisions, installed Portal dependencies, and Anvil before it sets the local fixture values and starts the test. No Sepolia or other public RPC is used.

Expected decisive output:

```
Test Files  1 passed (1)
Tests       1 passed (1)
classification = CONFIRMED ATTACKER-CONTROLLED ENS RESOLUTION CAUSES NFT TRANSFER TO ATTACKER
actors.allSecurityPrincipalsUseDistinctPrivateKeys = true
gate2.migrationRequest.byteEqual = true
gate3.recipientIdentityPostMigration.owner = 0x2C64c9406ca613ea68e6F17D4DF219529Fd1d4C8
gate3.recipientIdentityPostMigration.formerManagerIsLegitimateOwner = false
gate3.attackerResolverChange.byteEqual = true
gate4.freshQueryClientHadNoRecipientEntry = true
gate4.connectedAccount = 0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
gate4.walletClientAccount = 0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
gate4.displayedInput = revoked-manager.eth
gate4.displayedResolvedAddress = 0x6Bdfb8f07B6645990a32b0D76e678F1fCDAb1994
gate4.byteEqual = true
gate5.authoritativeAfterTransfer.latestOwner = 0x6Bdfb8f07B6645990a32b0D76e678F1fCDAb1994
gate5.authoritativeAfterTransfer.balances.transferVictim = 0
gate5.authoritativeAfterTransfer.balances.formerManagerAttacker = 1
gate5.authoritativeAfterTransfer.roleResource = gate5.authoritativeAfterTransfer.expectedRoleResource
gate5.authoritativeAfterTransfer.transferVictimRoles = 0
gate5.victimRecovery.status = reverted
gate5.formerOperatorRecovery.status = reverted
gate5.attackerReturnOnSnapshot.status = success
gate6.freshStateNegativeControl.formerManagerRoleGranted = false
gate6.freshStateNegativeControl.resolverChangeStatus = reverted
gate6.freshStateNegativeControl.resolutionBecameFormerManagerAddress = false
externalAccounting.forwardedExternalRequests = 0
externalAccounting.publicChainReads = 0
externalAccounting.publicChainWrites = 0
```

The exact sanitized source was run twice in separate processes against the public snapshot of the current audited revision. Both runs passed with identical application requests, decoded transfer, role/resolution state, ownership state, recovery controls, and network verdict. Transaction and block hashes and receipt-poll counts varied without changing semantics.

The runtime bundle needs only the TypeScript witness, the Vitest configuration, the Anvil state fixture, and the runner.

The GraphQL boundary is intercepted in-process and populated from direct local authoritative reads. The PoC contacts no subgraph, public RPC, wallet extension, deployed ENS service, or public chain.
