# 90907 w a critical the migration flow silently grants a third party permanent role set resolver over the user s name and destroys the only revocation path in the same transaction

**Submitted on Aug 28th 2026 at 01:55:31 UTC by @jokerbuena for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90907
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

**Program:** Audit Competition | ENS (`audit-competition-ens`)\
**Asset:** `apps/manager` — <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager\\>
**Repo:** `immunefi-team/audit-comp-ens` @ `audit-comp-ready`, HEAD `1c9b47f`\
**Severity:** High

**Impact selected — verbatim from this program's taxonomy:**

> *Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: — a registration or **migration that assigns the name, its ownership, registry, or resolver to an attacker** — redirecting the recipient of a name transfer — leaving a name permanently inaccessible after the flow*

***

## Summary

Migrating a name whose v1 registry `owner` differs from its registrant — the state every ERC-721 sale leaves behind when the buyer does not call `reclaim()` — makes the Manager app encode a `grantRoles(resource, ROLE_SET_RESOLVER, <previous owner>)` call into the migration batch. The grantee's address is never rendered anywhere in the flow. The same migration sends the v1 token away, which removes `reclaim()` — the one action that let the owner take back control without anyone's cooperation.

**This report is not about the v2 registry retaining delegated roles.** ENS has stated that retention is intended, and this report accepts that (see *Relationship to contracts-v2 #418*). It is about the Manager app **creating** such a grant, to an address it never names, in a transaction that simultaneously removes the user's ability to undo it.

## The defect

```ts
// apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9,17-25
// RegistryRolesLib uses nybble-packed roles in the remediated V2 deployment.
const ROLE_SET_RESOLVER = 1n << 24n

return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
}
```

`buildAtomicMigrationBatches.ts:48` repeats the same constant.

`managerAddress` is, by construction, **an address that is not the owner** — the field is `null` in every other case:

```ts
// packages/migration/src/service/classifyNames.ts:161-167   (cited, not accused — see Scope)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

The seller of a name does nothing at all; the victim triggers the grant by migrating.

## The app already knows this hazard — in a listed asset, in its own words

```ts
// apps/portal/src/features/records/hooks/useCanEditRecords.ts:38-40
 * - Permissioned resolvers: authorized by resolver `ROLE_SET_*` (root or
 *   name-scoped). After a transfer that keeps the resolver, the previous owner
 *   typically retains these roles while the new token owner does not.
```

The codebase documents that **the previous owner retains resolver roles while the new owner does not**, and builds a hook around that hazard. The migration flow then creates exactly that state, without saying so.

## Not an oversight — the app verifies that the grantee received the power

The grant call and a post-migration verification expectation **keyed on the third party's address** are pushed in the same block:

```ts
// apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:613-623
        call: buildRoleGrantCall(classified),
      })
      verificationExpectations.push({
        id: expectationId(name, 'manager-role'),
        type: 'manager-role',
        name,
        label: classified.label,
        registry: V2_CONTRACTS.ETHRegistry,
        resource: labelToCanonicalId(classified.label),
        account: classified.managerAddress,
        roleBitmap: ROLE_SET_RESOLVER,
      })
```

and read back on-chain, where `true` counts as success:

```ts
// apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:25,234
'function hasRoles(uint256 anyId, uint256 roleBitmap, address account) view returns (bool)',
…
args: [expectation.roleBitmap, expectation.account],
```

A post-migration check that fails if the third party did **not** obtain the role.

## The UI *cannot* show the beneficiary — the type it consumes has no field for one

This is not an omission that a template change would fix. The dialog renders `MigrationStepDescriptor`, and no branch of that union can carry an address:

```ts
// apps/manager/src/features/migration/service/buildStepDescriptors.ts:13-30
export type MigrationStepDescriptor =
  | { readonly type: 'deploy-hca' }
  | {
      readonly type: 'approval'
      readonly approvalId: MigrationApprovalId
      readonly count?: number
      readonly name?: string
      readonly tokenId?: bigint
    }
  | {
      readonly type: 'atomic-batch'
      readonly index: number
      readonly total: number
      readonly count: number
    }
  | {
      readonly type: 'cleanup'
      readonly approvalId: MigrationApprovalId
    }
```

`name` is the ENS name, `tokenId` an integer. The word `address` does not occur once in that file — while `type` occurs 14 times, so the search does see it. The grantee is structurally absent from the data the confirmation screen is given.

Consistent with that, `grep -rl managerAddress --include='*.tsx' apps/manager/src/features/migration` → **0 of 24 files** (positive control on the same corpus: `return` 23/24, `className` 22/24). `grantRoles` → 0. `ROLE_` → 0.

### What the victim actually reads before signing — rendered, attached

The real `WalletConfirmationStepsDialog`, fed by the real `buildMigrationPlan` for `bought.eth` (registrant = victim, registry owner = `0x…deadbeef`), renders:

```
Wallet confirmations
Your wallet will show 2 requests in this order.

  1. Create migration account
     Create the secure account that performs the name upgrade.

  2. Upgrade 1 name
     Migrate the selected names and restore their records.

Nothing is signed automatically. Review every request in your wallet.
```

and `plan.stepDescriptors` is, verbatim:

```json
[
  { "type": "deploy-hca" },
  { "type": "atomic-batch", "index": 0, "total": 1, "count": 1 }
]
```

Request 2 is the batch containing `grantRoles(resource, ROLE_SET_RESOLVER, 0x…deadbeef)`. The screen calls it *"restore their records"*, and closes with *"Review every request in your wallet"* — a review that cannot surface the grantee, because no layer between the plan and the dialog carries the address.

Screenshots attached, in order: **(1)** an earlier run of the fork suites, before tests F and G existed — 5 Sepolia + 4 mainnet; **(2)** this dialog, rendered from the real plan; **(3)** the current run of both suites — 7 Sepolia + 4 mainnet, timestamped `2026-08-27T20:58:31Z` at HEAD `1c9b47f`. Figures quoted in this report are the ones in (3).

*(Reproduced by adding a view-only route, `src/routes/zz-poc-c6.tsx`, which changes no application logic: it calls `buildMigrationPlan` with the PoC's fixture and passes `plan.stepDescriptors` to the unmodified component.)*

The only text shown elsewhere in the flow:

```tsx
// apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:84,87
<Trans>Approve manager restoration</Trans>
<Trans>Restore the existing managers for your names.</Trans>
```

`your` qualifies *names*, not *managers*. And the feature enables on exactly one condition:

```ts
// apps/manager/src/features/migration/service/migrationService.ts:346-348
requiresManagerRestoration: plan.classified.some(
  (name) => name.managerAddress !== null,
),
```

Combined with the `classifyNames.ts` definition: **this feature activates only when the manager is not the owner.** When the user is their own manager, `managerAddress` is `null` and nothing is restored, because there is nothing to restore. There is no case in which it returns to a user an authority that was already theirs — yet the screen says `your`.

## Relationship to `ensdomains/contracts-v2` #418

Issue #418 (*"Delegated EAC roles survive ERC1155 ownership transfer"*) reports that `_update` transfers only the sender's roles, so a grant made to a third party before a sale survives it. An ENS maintainer answered:

> *"this is expected behavior \[…] we decided that transfer retains existing grants \[…] Nick wanted restricted-permissioned tokens to be transferable."*

**This report takes that ruling as given and claims something else.** #418 is about the registry **retaining** a grant its owner deliberately made, in `contracts-v2` — a repository outside this competition's asset table. This report is about `apps/manager` **originating** such a grant:

|                              | #418                                   | this report                             |
| ---------------------------- | -------------------------------------- | --------------------------------------- |
| layer                        | `contracts-v2` registry semantics      | `apps/manager` migration flow           |
| in this competition's scope  | no                                     | yes                                     |
| who creates the grant        | the owner, deliberately, pre-sale      | **the app, on the victim's behalf**     |
| does the beneficiary act     | yes — grants, then calls `setResolver` | no — the seller does nothing            |
| is the beneficiary disclosed | the owner chose them                   | **never rendered**                      |
| revocation                   | `revokeRoles` remains available        | `reclaim()` destroyed in the same batch |

If retention is intended, then the design places the burden on the party that creates the grant to make it visible. Here that party is the application, and it names no one — while its own code (`useCanEditRecords.ts:38-40`) documents that this is precisely the state in which a previous owner keeps authority the new owner lacks.

### "Intended" would mean a choice was designed — measured, there is none

Two measurements replace the argument.

**1. The target's own tests never assert who receives the role.**\
`buildAtomicMigrationBatches.test.ts:223-242` locks the *sequence* of verification expectations, `'manager-role'` among them, and asserts nothing about the grantee. Nowhere in this codebase's test suite is a third party holding `ROLE_SET_RESOLVER` stated as the expected outcome — only that an expectation of that type exists.

**2. The grant cannot be declined.**

```ts
// apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-614
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    …
    call: buildRoleGrantCall(classified),
  })
```

The sole condition is a derived state value. There is no user flag, no preference, no parameter. Searching `apps/manager/src` for `skipManager|declineManager|optOut|withoutManager|skipRestoration|includeManagerRole` returns **0** — with a positive control that matters: `skipApprovalPhase` exists in **8** files. **The application knows how to make a migration phase optional. It does so for approvals. It does not do so here.**

A design decision is a decision offered. The user who reaches this screen cannot migrate without granting a stranger permanent authority over their name, and is never told the stranger's address.

## Feasibility

* **The required state already exists**, with no action by any attacker — it is what an ERC-721 sale without `reclaim()` leaves behind.
* **The attacker provokes nothing**: no link, no signature outside the normal flow, no race. The seller need only do nothing. The victim triggers it by migrating — the action the app invites.
* **Cost of attack: zero.** The position is a by-product of a sale already collected.
* **Nothing expires.** The role is permanent until explicitly revoked, and the screen that requests it does not name its beneficiary — so the user has neither reason to revoke nor the information to know whom.

The only non-trivial condition is that the victim migrates. This competition is about the migration application.

## Not a known issue

Compared against the 61 published known issues (10 QA, 6 R2, 8 R3, 4 SEC-MGR, 1 SEC-TXM, 7 EXP-*, 25 WEB-*). Nearest neighbours:

* **QA-01** — role combinations that **break** actions after transfer: a loss of capability for the victim. This is the exact inverse — a gain for a third party.
* **WEB-1257** — *"After a parent name is transferred, its registry is not detached, and the previous owner **can no longer** set a subname they still hold as their primary name."* Closest by vocabulary, opposite in meaning: there the previous owner loses something on their own subname.
* **WEB-716** — a UI omission about a legitimately-held **subregistry** role; not the assignment of a **resolver** role to a third party during migration.
* **WEB-806 / WEB-390 / WEB-1249** — subnames and records after migration; unrelated to `managerAddress`.
* **WEB-1303 / WEB-1255 / WEB-1304** — profile editing, grace window, NameWrapper expiry.
* **R2-03 / R2-06** — HCA session keys. **SEC-TXM-002** — hidden smart-account address.

## Scope

Accused files, all in `apps/manager` (listed asset): `buildRoleGrantCalls.ts`, `buildAtomicMigrationBatches.ts`, `migrationService.ts`, `verifyAtomicMigrationBatch.ts`, `components/WalletConfirmationStepsDialog.tsx`. Supporting citation in `apps/portal` (listed asset): `features/records/hooks/useCanEditRecords.ts`.

`packages/migration/src/service/classifyNames.ts` is cited for the *definition* of `managerAddress` and is not accused; removing it leaves the claim standing, since `buildRoleGrantCalls.ts` grants the role to `name.managerAddress` regardless of its origin.

## Suggested fix

{% stepper %}
{% step %}

## Name the beneficiary

Render `classified.managerAddress` in `WalletConfirmationStepsDialog` and in the step descriptors, and require an explicit per-name confirmation before including the grant. A user can then recognise an address that is not theirs.
{% endstep %}

{% step %}

## Default to not granting

Offer restoration as an opt-in for users who genuinely delegate management, rather than deriving it from a v1 state the user may not know exists.
{% endstep %}
{% endstepper %}

### The non-disclosure is an outlier in this app, not a policy

The same application names an address when it asks the user to authorise something:

```ts
// apps/manager/src/features/wallet/components/EnableSessionModal.tsx:42,89-95
const formatAddress = (address?: string) => { … }
…
{smartAccountAddress && ( … {formatAddress(smartAccountAddress)} )}
```

And it ships a dedicated atom for the purpose, `components/atoms/CopyableAddress/CopyableAddress.tsx`.

Across `apps/manager/src`, **16 of 363** `.tsx` files render an address. Inside `features/migration`, that count is **0 of 24**.

So an authorisation modal in this app names the smart account it concerns, while the authorisation modal that grants a third party permanent `ROLE_SET_RESOLVER` over the user's name names no one. The convention exists, the component exists, and this one flow is the exception.

## Link to Proof of Concept

<https://gist.github.com/JokerBUENA/6e4d164cbb8f123a472670b30e096369>

## Proof of Concept

## Measured on-chain

Produced live on **2026-08-27T20:13:24Z**, at repo HEAD `1c9b47f`. Values that track chain state drift between runs, which is how you can tell they are read rather than hardcoded.

**No transaction was ever broadcast.** Everything below runs in a local Foundry fork: state is read from the public RPC at the current block, and every write (`grantRoles`, `setResolver`, `reclaim`, `transferFrom`) executes only in the local EVM, against a snapshot, using `vm.prank`. Nothing is signed, nothing is sent, no account is touched on Sepolia or mainnet. `forge test` never opens a wallet.

### The precondition exists in production

```
Registry       0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e   owner(bytes32)
BaseRegistrar  0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85   ownerOf(uint256)

vitalik.eth    registrant  0x220866B1A2219f40e72f5c628B65D54268cA3A9D
               controller  0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045    -> DIFFERENT
```

1 of 14 sampled names diverge. The sample is famous names — names that are not resold — so it is a floor, not a rate.

### The calldata grants the role, and the role confers the power

Sepolia fork against deployed `ETHRegistry` `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2` (14,730 bytes of code; counter-witness `0x…dEaD` → 0 bytes). Resource id via `labelToCanonicalId` (labelhash with the low 32 bits cleared), witness name `hello`, owner `0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8` — 9 of 10 probed labels return the zero address.

```
hasRoles before : false      bitmap before : 0
hasRoles after  : true       bitmap after  : 16777216      ( = 1 << 24 )

setResolver without the role : false      reverted
setResolver with the role    : true       succeeded
```

The two `setResolver` calls are identical except for the grant. This is what establishes that bit 24 **is** the set-resolver authority on the deployed registry: `ROLE_SET_RESOLVER()` reverts as a getter, being a `RegistryRolesLib` constant, so the correspondence is proved by effect.

**And the registry then points the name at the attacker's resolver.** The program states its primary concern as *"any bug that could lead to a loss of user funds. In particular \[…] direct users to transfer funds to an address other than the intended one"* — the resolver is what decides which address a name resolves to, so the chain does not stop at "the call succeeds":

```
resolver before : 0x0000000000000000000000000000000000000000
resolver after  : 0x587b6432f3c2E94408C78dcA567096B313D6b862    (attacker's)

without the grant : setResolver reverts, and getResolver("hello") is unchanged
```

Read through `getResolver(string label)` — selector `0xe4ae7d77`, the same signature the application's own ABI declares. What is measured is that the registry now returns an attacker-chosen resolver for the victim's name; the payment that follows depends on the resolver contract, which this test does not deploy.

```
[PASS] test_A_temoin_hello_existe_dans_le_registre_v2
[PASS] test_B_contre_temoin_un_nom_absent_rend_zero
[PASS] test_C_le_calldata_de_migration_accorde_vraiment_le_role
[PASS] test_D_controle_negatif_un_tiers_sans_grant_ne_detient_rien
[PASS] test_E_le_role_donne_vraiment_le_pouvoir_de_changer_le_resolver
[PASS] test_F_le_registre_pointe_le_nom_vers_le_resolveur_de_l_attaquant
[PASS] test_G_controle_negatif_sans_grant_le_resolveur_ne_bouge_pas
Suite result: ok. 7 passed; 0 failed; 0 skipped
```

### The escalation: the v1 revocation path is destroyed

Mainnet fork, on `vitalik.eth` — a name actually in the divergent state:

```
reclaim() by the REGISTRANT, token in hand
    controller before  0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    controller after   0x220866B1A2219f40e72f5c628B65D54268cA3A9D

reclaim() by an INTRUDER                -> false, state unchanged
reclaim() by the SAME owner, token gone -> false
```

Two pairs carry it. **B vs C**: the same call succeeds for the registrant and fails for a third party — the capability belongs to the owner. **B vs D**: the same call by the same address succeeds with the token and fails without it — the irreversibility is a measured difference.

Migration is what sends the token away:

```ts
// apps/manager/src/features/migration/service/buildMigrationHelperCall.ts:79,83
to: V2_CONTRACTS.MigrationHelper,
functionName: 'migrate',
```

and the service observes it: *"Refusing to retry … because the source token is no longer owned by the wallet"* (`migrationService.ts:748`). Test D reproduces that departure by transferring the token, then measures what remains to the owner.

### Application-level PoC

`POC_C6_manager_grant.test.ts` — 3/3 on the real encoded `executeByOwner` calldata, `buildAtomicMigrationBatches` not mocked. Mutation-tested: forcing `managerAddress` to `null` gives 2 failed / 1 passed, `git checkout` restores 3/3. Asserts `grantArgs[1] === 1n << 24n`, `grantArgs[2] === ATTACKER`, and that `JSON.stringify(plan.stepDescriptors)` does **not** contain the grantee address. Negative control with a clean name (`registry owner === registrant`) encodes no `grantRoles` at all.

```bash
cp POC_C6_manager_grant.test.ts apps/manager/src/features/migration/service/zzPocManagerGrant.test.ts
cd apps/manager && pnpm exec vitest run src/features/migration/service/zzPocManagerGrant.test.ts

cd pocs/fork-c6 && SEPOLIA_RPC_URL=https://ethereum-sepolia-rpc.publicnode.com forge test -vv
cd pocs/fork-c3 && ETH_RPC_URL=https://eth-mainnet.public.blastapi.io forge test -vv --match-contract C6Reclaim
```
