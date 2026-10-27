# 92908 w a medium historical resolver provenance check reattaches attacker upgraded code during profile editing

**Submitted on Sep 13th 2026 at 06:17:11 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92908
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

ENS Manager discovers a user's existing PermissionedResolver from its historical factory deployment event but never verifies the proxy's current implementation. A delegate who was legitimately granted `ROLE_UPGRADE` can install storage-compatible attacker code before being revoked. Even after the victim revokes that role and detaches the resolver, a later ordinary profile edit silently selects and reattaches the compromised proxy. The victim approves a transaction presented as replacing old profile data with their saved changes, but the attached resolver can ignore those writes and return an attacker address.

The affected commit is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

## Vulnerability Details

`findExistingPermRes()` searches old `ProxyDeployed` events using the expected HCA deployer, owner-derived salt, and implementation recorded at deployment:

```ts
const logs = await publicClient.getLogs({
  address: V2_CONTRACTS.VerifiableFactory,
  event: proxyDeployedEvent,
  args: { sender: deployer ? [eoa, deployer] : eoa },
  fromBlock: V2_DEPLOY_BLOCK,
  toBlock: 'latest',
})
for (const log of [...logs].reverse()) {
  if (log.args.implementation?.toLowerCase() !== impl) continue
  if (log.args.salt !== expectedSalt) continue
  return log.args.proxyAddress as Address
}
```

Affected source: `audit-comp-ens/apps/manager/src/features/migration/service/ensureOwnedPermRes.ts:50-70`.

The event proves what implementation the proxy used when it was deployed, not what it executes now. The official factory's `verifyContract(proxy)` returns the current implementation, but this helper never calls it.

`setupControlledResolver()` trusts the historical result. When a matching proxy exists, it omits fresh deployment and builds a transaction that points the name at that proxy before writing the victim's records to it:

```ts
const existing = await findExistingPermRes({
  eoa: ownerAddress,
  deployer: smartAccount,
  publicClient,
})
const resolver = existing ?? await simulateOwnedPermResAddress(...)

const calls: Call[] = [
  ...(existing ? [] : [buildDeployOwnedPermResCall(ownerAddress)]),
  buildSetResolverCall({ name, newResolver: resolver }),
  ...recordCalls,
]
```

Affected source: `audit-comp-ens/apps/manager/src/features/profile/service/setupControlledResolver.ts:71-112`.

The ordinary trigger is profile editing for an HCA-owned V2 `.eth` name whose resolver is absent. Manager recognizes the connected smart account as an owner in `audit-comp-ens/apps/manager/src/features/profile/components/view/ProfileView.tsx:47-57`, exposes the edit action in `audit-comp-ens/apps/manager/src/features/profile/components/view/ProfileEditAction.tsx:28-50`, and routes a missing resolver through `setupControlledResolver()` in `audit-comp-ens/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts:215-262`.

The confirmation shown to the victim makes a concrete security promise. `ResolverSetupConfirmDialog.tsx` says: "This name still has profile details from a previous owner. Continuing clears them and keeps only the changes you save," and labels the action "Replace & Save." The actual transaction reattaches code already installed by the former upgrader. That code can accept the generated `multicall` while ignoring both proposed record changes.

Affected source: `audit-comp-ens/apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx:38-46`.

Manager's migration path demonstrates the missing invariant. It calls the same factory's `verifyContract(resolver)` and rejects a current implementation that differs from `PermissionedResolverImpl`:

```ts
const resolverImpl = await client.readContract({
  address: V2_CONTRACTS.VerifiableFactory,
  abi: VERIFIABLE_FACTORY_ABI,
  functionName: 'verifyContract',
  args: [resolver],
})
if (!isAddressEqual(resolverImpl, V2_CONTRACTS.PermissionedResolverImpl)) {
  return err(new MigrationInvariantError(...))
}
```

Sibling source: `audit-comp-ens/apps/manager/src/features/migration/service/migrationInvariants.ts:297-325`.

The attacker does not retain upgrade permission after revocation. Revoking `ROLE_UPGRADE` correctly makes a fresh upgrade attempt revert. The flaw is that the already-installed implementation remains in the proxy and Manager later reattaches it based only on historical identity.

## Impact Details

Selected impact: "Malicious interactions with an already-connected wallet," specifically substituting a contract address or transaction parameter in a transaction the user approves.

Manager causes the connected victim to approve `setResolver(victimName, compromisedProxy)` while presenting the operation as replacement with a controlled resolver and only the victim's saved profile changes. The resolver's current implementation is attacker controlled, can ignore the appended record writes, and becomes authoritative for the victim-owned name. The PoC returns the attacker's address while proving that the victim HCA remains the name owner.

The attacker must previously have received the exceptionally powerful resolver-wide `ROLE_UPGRADE` and must install the implementation while authorized. The attacker cannot force the victim's later profile edit, and the victim can recover after detection by assigning a fresh resolver. The vulnerability remains security-relevant because explicit role revocation and resolver detachment do not prevent Manager from independently reactivating the installed code.

## Known Issue and Duplicate Boundaries

This is not the closed permissioned-resolver attestation report: that report used a self-asserted EIP-1967 slot only to select fixed UI text, while this report uses a genuine historical factory event to build and execute a state-changing attachment after the implementation changes. It is also distinct from the stale V1 custom-resolver report, which carries a stale V1 pointer through migration, and from the dormant-record reports, where the resolver still executes the official implementation. A live `verifyContract()` check fixes this report but does not fix either stale pointer or uncleared record state.

## References

* `audit-comp-ens/apps/manager/src/features/migration/service/ensureOwnedPermRes.ts:50-70`
* `audit-comp-ens/apps/manager/src/features/profile/service/setupControlledResolver.ts:71-136`
* `audit-comp-ens/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts:215-262`
* `audit-comp-ens/apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx:38-46`
* `audit-comp-ens/apps/manager/src/features/migration/service/migrationInvariants.ts:297-325`

## Remediation

Before returning a proxy from `findExistingPermRes()`, call `VerifiableFactory.verifyContract(proxy)` and require the current implementation to equal `V2_CONTRACTS.PermissionedResolverImpl`. Also apply the owner and root-role invariants already used by the migration path. If any current-state check fails, do not reattach that proxy; fail closed or deploy a fresh resolver. Add a regression test where the historical event records the official implementation but `verifyContract()` returns a different current implementation.

## Link to Proof of Concept

<https://gist.github.com/niafreu/8b6de42cceecb50103ac8dad16091c2a>

## Proof of Concept

## Proof of Concept

My PoC is available at <https://gist.github.com/niafreu/8b6de42cceecb50103ac8dad16091c2a>. It has one runner, uses the scoped Manager source at the assessed commit, and uses official `ensdomains/contracts-v2` source at `48b3e2d39513b9dd32ef1850877a29009bc807b9`.

### Steps

1. Check out `audit-comp-ens` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and install its dependencies.
2. Check out `ensdomains/contracts-v2` at `48b3e2d39513b9dd32ef1850877a29009bc807b9` with submodules.
3. Run:

```bash
chmod +x pocs/stale-owned-resolver-implementation/run-poc.sh
./pocs/stale-owned-resolver-implementation/run-poc.sh \
  /path/to/audit-comp-ens \
  /path/to/contracts-v2/contracts
```

### What the production-code stage proves

The Vitest stage imports the real `findExistingPermRes()`, `setupControlledResolver()`, `buildSetResolverCall()`, and record-call builder. It supplies the exact historical factory identity created by the contract stage: sender is the HCA, salt is `computeOwnedResolverSalt(owner, 0)`, and the deployment-time implementation is official.

The helper returns the compromised proxy without a live implementation read. The captured request is from the HCA, contains no factory deployment, and has exactly two calls:

```
PermissionedRegistry.setResolver(labelhash("victim"), proxy)
proxy.multicall([setText(...), setAddr(...)])
```

The exact calldata hashes are:

```
setResolver  0xcbd4a212ed2f79e678c508af8271f9faacb3c72be767472e1f9399fc846e7205
multicall    0x1cc1898c33ad43914b724896ed05d7676829ca36d1781c187d95e88b63a144c7
```

### What the official-contract stage proves

The Foundry stage deploys the official VerifiableFactory, PermissionedResolver, and PermissionedRegistry. It performs this sequence:

1. An owner-controlled HCA deploys the resolver proxy with the exact production salt and initializer. The real `ProxyDeployed` event identifies the HCA sender, salt, official implementation, and proxy.
2. The HCA owns `victim.eth` and has real `ROLE_SET_RESOLVER` authority.
3. The victim grants the delegate only root `ROLE_UPGRADE`; the delegate upgrades the proxy to storage-compatible attacker code.
4. The victim revokes that role. The test proves the delegate no longer holds it and that a fresh upgrade attempt reverts.
5. The victim detaches the resolver through the HCA.
6. The test asserts the two calldata hashes above and executes those exact calls through the HCA.
7. The HCA remains the name owner, the registry points at the compromised proxy, and address resolution returns the attacker address.
8. As a negative control, `factory.verifyContract(proxy)` returns the attacker implementation rather than the official one.

Expected output:

```
Test Files  1 passed (1)
Tests       1 passed (1)

[PASS] testProductionCallsReactivateRevokedDelegateCode()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```
