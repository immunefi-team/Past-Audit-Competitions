# 93049 w a critical missing resolver contract type validation turns record role delegation into registry wide name theft

**Submitted on Sep 14th 2026 at 03:58:03 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93049
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

I found that the ENS Portal applies `PermissionedResolver` role names and role encoding to any address stored as a name's resolver without verifying the target contract's type. `PermissionedResolver` and `UserRegistry` expose the same `grantRootRoles(uint256,address)` function, but the same bitmap positions have different meanings. A victim who follows the Portal's normal role workflow and selects the displayed resolver permissions `Set Address` and `Set Pubkey` can unknowingly grant an attacker registry-wide `Registrar` and `Unregister` authority. My local PoC uses the audited Portal code and official factory-deployed ENS v2 contracts to burn an independently owned name and re-register it to the attacker.

## Vulnerability Details

The Portal accepts an arbitrary contract address as a name's resolver. `NameResolverRolesOverviewTable.tsx` exposes a `View` link to `/resolver/$address/roles` for every nonzero resolver pointer without verifying that the target is a `PermissionedResolver`.

The destination route in `routes/resolver/$address/roles/index.tsx` also treats the raw route address as a resolver. It does not check ERC-165, factory provenance, or the verified implementation before performing EAC reads or opening the resolver-role form:

```typescript
const { data: hasRootRole } = useQuery({
  ...getHasRolesQueryOptions({
    resolverAddress: address as Address,
    roles: ['ROLE_SET_ADDR'],
    account: accountAddress as Address,
  }),
  enabled: !!accountAddress,
})

const canManageRoles = Boolean(hasRootRole)
const roles = resolver?.roles ?? []
const nodes = resolver?.nodes ?? []
```

Correct indexer behavior does not block the path. If `resolver(id:)` returns `null` for a registry, the route converts it to empty `roles` and `nodes`. The independent `hasRootRoles` check uses resolver role bit zero against the supplied address. A normally initialized `UserRegistry` owner has registry `ROLE_REGISTRAR` at bit zero, so the read succeeds and the route exposes `Add user`.

`ResolverAddUserSheet.tsx` then presents the registry as a resolver. It defaults to `Root (all nodes)`, displays resolver permission names, and labels the transaction `Grant resolver roles`. The mounted component in my PoC confirms that it shows `Set Address` and `Set Pubkey` while never showing `Registrar` or `Unregister`.

When the victim confirms, `prepareGrantResolverRolesTransaction()` passes the unverified route address to ENSJS. Root scope produces this request:

```typescript
args: [encodeResolverRoleBitmap(params.roles), params.targetAccount]
functionName: 'grantRootRoles'
```

Both official contract families implement `grantRootRoles(uint256,address)` with selector `0x072d5d77`, but their role vocabularies collide:

| Portal label |   Raw bit | `UserRegistry` meaning | Consequence                                   |
| ------------ | --------: | ---------------------- | --------------------------------------------- |
| Set Address  |  `1 << 0` | `ROLE_REGISTRAR`       | Register available labels across the registry |
| Set Pubkey   | `1 << 12` | `ROLE_UNREGISTER`      | Unregister existing names across the registry |

For attacker address `0x00000000000000000000000000000000000A11cE`, production Portal code prepares:

```
from:  connected victim
to:    victim's official UserRegistry proxy
value: 0
chain: 11155111
data:  0x072d5d77000000000000000000000000000000000000000000000000000000000000100100000000000000000000000000000000000000000000000000000000000a11ce
```

The call succeeds because the victim holds the corresponding registry admin bits. The attacker can then call `unregister(tokenId)` for a name owned by an independent account and re-register the same label to themselves using `ROLE_REGISTRAR`. The original ERC-1155 is burned and the replacement token belongs to the attacker.

This does not require a bespoke or malicious contract. The PoC creates the victim registry through the audited `prepareDeploySubregistryTransaction()` helper and official `VerifiableFactory`, and verifies it with `verifyContract()`. It also deploys the official root and `.eth` `PermissionedRegistry` topology, makes attacker-owned `bait.eth` point to the victim's official registry, and transfers that name to the victim. The Portal then exposes the role workflow for that resolver pointer.

My negative control sends the same `0x1001` bitmap to an official factory-deployed `PermissionedResolver`. It grants only `Set Address` and `Set Pubkey`, and an `unregister` call still fails. The official interfaces also distinguish the two targets: the registry supports `IPermissionedRegistry` and rejects `IPermissionedResolver`, while the genuine resolver does the opposite. The Portal simply does not enforce that distinction.

This finding is separate from the resolver-sidebar bug where an unmappable name-scoped resource becomes the empty-string root sentinel. My path starts with an explicitly selected root scope and requires no existing role row, unresolved resource, grouping, or fallback from name scope. The wrong value here is the contract family and role vocabulary. Preserving unresolved resources does not fix this issue; the Portal must verify the mutation target's type.

## Impact Details

I selected Critical impact: malicious interactions with an already-connected wallet, including modifying transaction arguments or parameters and substituting contract addresses.

The victim is shown a normal Portal flow for granting two resolver record permissions, but the signed request targets a registry and grants registry-wide issuance and deletion powers. The victim's wallet correctly executes the request the Portal built; the security failure is that the application binds resolver semantics to the wrong contract family. The attacker needs no existing role, token approval, leaked credential, race, privileged project account, malicious contract, or public-network interaction.

My PoC also demonstrates the concrete asset consequence: after the misleading Portal grant, the attacker burns a live ENS v2 name owned by a separate account and re-registers its label to themselves. This is direct theft of an ENS name/NFT under accepted impact 44. The report's root cause remains entirely in the in-scope Portal; the official contracts execute their documented interfaces and are only local impact oracles.

## Remediation

Before rendering or preparing resolver-role mutations, verify that the target supports `IPermissionedResolver`, does not support `IPermissionedRegistry`, and is a proxy from the approved `VerifiableFactory` whose current implementation is an approved `PermissionedResolver` implementation for the active chain. Bind the verified address and family to the transaction intent and revalidate immediately before submission. Do not use a successful shared EAC read such as `hasRootRoles` as a type check. Add a regression test where a name's resolver pointer is an official `UserRegistry`: the Portal may display the untrusted pointer, but the resolver role route must fail closed and never expose `Add user`.

## References

* `audit-comp-ens/apps/portal/src/features/resolver/components/NameResolverRolesOverviewTable.tsx`
* `audit-comp-ens/apps/portal/src/routes/resolver/$address/roles/index.tsx`
* `audit-comp-ens/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx`
* `audit-comp-ens/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`
* `audit-comp-ens/apps/portal/src/features/registry/helpers/deploySubregistry.ts`
* `audit-comp-ens/apps/portal/src/features/registry/hooks/useHasRoles.ts`
* `audit-comp-ens/apps/portal/src/lib/roles/resolverRoles.ts`

## Link to Proof of Concept

<https://gist.github.com/niafreu/e80dfafd2e273f619928fc6bad11dae2>

## Proof of Concept

## Proof of Concept

My attached PoC is at: <https://gist.github.com/niafreu/e80dfafd2e273f619928fc6bad11dae2>

### Environment

```
Application: immunefi-team/audit-comp-ens at 1c9b47f18fcddd2e864dfe385c4171061c9811ae
Official contracts: ensdomains/contracts-v2 at 48b3e2d39513b9dd32ef1850877a29009bc807b9
Node.js: 20 or later
Package manager: pnpm 10.27.0
Foundry/Anvil: Solidity 0.8.25, Cancun EVM
Network: fresh loopback Anvil plus local Foundry VM only
```

### Steps to Reproduce

1. Clone the two pinned repositories and install the audited application's locked dependencies as shown in `README.md`.
2. Save the six PoC files in the documented sibling-directory layout.
3. Run the single Vitest command from the README. The UI test mounts the audited `ResolverAddUserSheet`. The end-to-end test starts Anvil, compiles and deploys the official contracts, and performs the full exploit.
4. Observe Step 1 create the victim `UserRegistry` through the audited Portal deployment helper and official `VerifiableFactory`.
5. Observe Step 2 point an ordinary attacker-owned name at that official registry and transfer the name to the victim.
6. Observe Step 3 create an independently owned name in the victim registry and prove the attacker cannot unregister it.
7. Observe Step 4 build `grantRootRoles(0x1001, attacker)` with the audited Portal helper and broadcast it through the audited production EOA transport.
8. Observe Step 5 show that the attacker now unregisters the independent name and re-registers it to themselves.
9. Observe Step 6 send the same bitmap to a factory-deployed genuine `PermissionedResolver`, where it cannot unregister a name.
10. Run `forge test -vv` for an independent official-contract proof of the collision, interface gate, pre-exploit denial, and final theft.

### Expected output

```
Test Files  2 passed (2)
Tests       2 passed (2)

[PASS] test_portalResolverRootGrantStealsExistingRegistryName()
[PASS] test_samePortalGrantOnFactoryDeployedResolverIsOnlyRecordAuthority()
[PASS] test_selectableRoleCollisionTable()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

The end-to-end test also prints a final JSON object containing the factory-verified registry, the two resolver permissions shown to the victim, the signed function and bitmap, the actual registry permissions, the original owner, the attacker's final ownership, and the genuine-resolver control result.

### Expected secure behavior

The Portal must reject a wrong-family EAC target before displaying resolver permissions or preparing a wallet request. No selection of resolver role names should ever grant registry roles.
