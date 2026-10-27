# 92951 w a medium missing on chain validation in ens migration lets a third party hijack the resolver of a user s name

## #92951 \[W\&A-Medium] Missing on-chain validation in ENS migration lets a third party hijack the resolver of a user's name

**Submitted on Sep 13th 2026 at 12:47:47 UTC by @Kin for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92951
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

### Description

## Brief/Intro

When a user migrates a .eth name from v1 to v2 in the Manager app, the app builds an owner-signed batch that grants the SET\_RESOLVER role on the new name to whatever address the v1 subgraph reports as the name's registry controller. That address is never checked against on-chain state, never shown to the user before they sign, and the app's own post-migration check reads back the same value it wrote, so it always reports success. If the controller is not the user, which is the normal state right after buying a name and any time the subgraph lags an on-chain reclaim, migration hands a third party permanent resolver control of the user's name. In production that party can repoint the name's ETH address record, so any funds later sent to that name go to them.

## Vulnerability Details

The manager address is read straight from the subgraph and only compared against the registrant, never against chain state.

packages/migration/src/service/classifyNames.ts (around line 162):

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

domain.owner.id is the v1 registry controller as reported by the v1 subgraph (packages/migration/src/service/v1SubgraphClient.ts). The eligibility preflight re-reads the registrant and the wrapped owner on chain, but it never re-reads this controller field, so the manager is the one execution-affecting value in the plan that rests entirely on off-chain data.

That value is then encoded into a role grant that goes into the owner-signed migration batch.

apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:

```ts
const ROLE_SET_RESOLVER = 1n << 24n
// ...
grantRoles(labelToCanonicalId(name.label), ROLE_SET_RESOLVER, name.managerAddress)
```

apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts (around line 608) pushes this manager-role-grant call into the batch and adds a verification expectation carrying the same managerAddress.

Two checks that should catch a wrong recipient do not:

* No disclosure. The signing flow shows only generic copy. apps/manager/src/features/migration/service/migrationService.ts line 334 renders "Allowing your HCA to restore managers", and the select-names screen says records and addresses migrate automatically. No component in the migration confirm or wallet-step dialogs renders managerAddress, so the user signs a grant to an address they are never shown.
* Blind verification. apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts (around line 337) checks hasRoles(resource, ROLE\_SET\_RESOLVER, expectation.account) where expectation.account is the same managerAddress the app just granted to. It confirms the address it wrote, never that the address was correct, so a wrong grant passes as satisfied.

The grant lands in the owner-signed executeByOwner path, which applies no argument policy, so nothing downstream re-checks the recipient either.

A minimal service-layer test against the app code confirms this end to end. For a domain where the registrant is the victim and owner.id is the attacker, classifyName returns managerAddress = attacker while tokenHolder = victim, and buildRoleGrantCall encodes grantRoles(labelToCanonicalId('alice'), 1<<24, attacker). Both assertions pass (see the PoC).

## Impact Details

After migration the granted address holds SET\_RESOLVER on the user's v2 name. It can call setResolver and point the name at a resolver that returns its own address for the ETH address record. Every payment sent to that name then goes to the attacker instead of the owner, for as long as the name exists, with no further interaction from the victim. This is a direct, ongoing theft-of-funds channel and a permanent loss of control over the name.

The setup needs no victim mistake beyond migrating a name they legitimately own:

* Buy then migrate. Transferring the BaseRegistrar NFT does not update the v1 registry controller. A seller who keeps the controller sells the name, the buyer becomes the registrant, and if the buyer migrates before calling reclaim, the app grants the seller permanent SET\_RESOLVER on the buyer's v2 name.
* Subgraph lag. If the user reclaimed control on chain but the subgraph has not indexed it yet, the app grants control to an address with no current on-chain authority at all. This case removes any argument that the grant merely preserves existing v1 power, because the recipient is a party the chain no longer recognizes as controller.

The loss scales with the value routed to the affected names. Any name where the controller and registrant differ at migration time is exposed, and the victim has no signal that it happened because the address is never displayed and the app reports the migration as fully verified.

This maps to the in-scope impact "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer): a migration that assigns the name, its ownership, registry, or resolver to an attacker." The root cause is not in the published known-issues list; the only migration entry there is WEB-390 (subnames missing from the list, a display defect).

## References

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/migration/src/service/classifyNames.ts> (managerAddress taken from domain.owner.id, no on-chain check)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts> (grantRoles SET\_RESOLVER to managerAddress)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts> (manager-role-grant added to the owner-signed batch)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts> (verification reads back the same value)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/migrationService.ts> (generic "restore managers" copy, address never shown)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/migration/src/service/v1SubgraphClient.ts> (domain.owner.id is subgraph data)

### Link to Proof of Concept

<https://gist.github.com/amin00216/0933e7543648d69adb96e21bbefa9036>

### Proof of Concept

## Proof of Concept

This proof runs against the app's own code and passes. It shows the Manager app building a grantRoles(SET\_RESOLVER, attacker) call for a name the victim owns, using only the v1 subgraph controller field. That call is then placed into the owner-signed migration batch (code path in Step 4), and once the role is granted on the v2 name the ETHRegistry role semantics give the holder setResolver power, which is the hijack.

## Step 1. Add the test

* Create the file apps/manager/src/features/migration/service/pocManagerRoleGrant.test.ts with this content:

```ts
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { classifyName } from '@ens-apps/migration'
import { decodeFunctionData } from 'viem'
import { describe, expect, it } from 'vitest'
import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { buildRoleGrantCall } from './buildRoleGrantCalls'

const VICTIM = '0x1111111111111111111111111111111111111111' as const
const ATTACKER = '0x2222222222222222222222222222222222222222' as const
const ROLE_SET_RESOLVER = 1n << 24n

// alice.eth: the victim holds the BaseRegistrar NFT (registrant), but the v1
// registry controller (owner.id, from the subgraph) is still the attacker.
// Normal state right after an NFT sale where the buyer has not called reclaim,
// and any time the subgraph lags an on-chain reclaim.
const aliceEth = {
  labelName: 'alice',
  name: 'alice.eth',
  resolver: null,
  owner: { id: ATTACKER },
  registrant: { id: VICTIM },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: String(Math.floor(Date.now() / 1000) + 365 * 24 * 3600) },
  wrappedDomain: null,
} as const

describe('PoC: migration manager-role grant taint', () => {
  it('classifies managerAddress from the subgraph controller, not the registrant', () => {
    const result = classifyName(aliceEth as never, VICTIM)
    expect(result?.type).toBe('classified')
    if (result?.type !== 'classified') throw new Error('not classified')
    expect(result.name.tokenHolder.toLowerCase()).toBe(VICTIM)
    expect(result.name.managerAddress?.toLowerCase()).toBe(ATTACKER)
  })

  it('builds grantRoles(SET_RESOLVER, attacker) into the owner-signed batch', () => {
    const result = classifyName(aliceEth as never, VICTIM)
    if (result?.type !== 'classified') throw new Error('not classified')
    const call = buildRoleGrantCall(result.name)
    const decoded = decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data: call.data })
    expect(decoded.functionName).toBe('grantRoles')
    const [resource, roleBitmap, account] = decoded.args as [bigint, bigint, string]
    expect(resource).toBe(labelToCanonicalId('alice'))
    expect(roleBitmap).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(ATTACKER)
  })
})
```

## Step 2. Run it

```
pnpm install
cd apps/manager
npx vitest run src/features/migration/service/pocManagerRoleGrant.test.ts
```

## Step 3. Result

```
Test Files  1 passed (1)
     Tests  2 passed (2)
```

Test 1 shows the classifier assigns the attacker as manager while the victim holds the token. Test 2 shows the builder encodes a grant of SET\_RESOLVER to the attacker on the victim's name. The address in the signed call is the subgraph controller, with no on-chain reconciliation.

## Step 4. How the grant reaches the signed batch (code path)

* buildAtomicMigrationBatches.ts around line 608: buildNameExecution calls buildRoleGrantCall(classified) and pushes the result as a manager-role-grant inner execution, with a verification expectation carrying the same managerAddress.
* buildAtomicMigrationBatches.ts line 778: buildAtomicMigrationInnerExecutions orders the manager-role-grant phase into the final batch the user signs through executeByOwner.
* verifyAtomicMigrationBatch.ts around line 337: verification reads hasRoles(resource, ROLE\_SET\_RESOLVER, expectation.account) using the same managerAddress, so it confirms the address it granted to and reports success regardless of whether that address was correct.

## Step 5. End-to-end hijack (production)

* Attacker registers alice.eth and is both registrant and controller.
* Attacker transfers the BaseRegistrar NFT to the victim (a normal secondary-market sale). The victim is now the registrant. The v1 controller stays the attacker, because transferring the NFT does not reset it.
* Victim migrates alice.eth to v2 without first calling reclaim. The plan classifies the name with managerAddress = attacker and includes grantRoles(SET\_RESOLVER, attacker) in the owner-signed batch. The confirm screen shows only "restore managers" and never displays the attacker address. The victim signs.
* Migration completes. verifyAtomicMigrationBatch reports success, because it reads back the same attacker address it granted to.
* Attacker calls setResolver on the v2 name and sets the ETH address record to the attacker address. Payments resolving alice.eth now go to the attacker.

## Step 6. Optional on-chain confirmation

* On a Sepolia fork pinned to the manifest addresses, execute the batch from Step 3 through the migration actors, then from the attacker account call setResolver and setAddr on the v2 name and read the resolved address back.
* To show the subgraph-lag variant, set the on-chain controller to the victim after a reclaim while feeding the classifier a subgraph response with owner.id = attacker. The app still emits the attacker grant, which proves the recipient is trusted from off-chain data with no on-chain reconciliation.
