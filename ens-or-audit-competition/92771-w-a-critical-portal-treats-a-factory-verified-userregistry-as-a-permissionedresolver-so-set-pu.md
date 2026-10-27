# 92771 w a critical portal treats a factory verified userregistry as a permissionedresolver so set pubkey grants registry wide unregister

**Submitted on Sep 12th 2026 at 05:29:27 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92771
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

After the disclosed Explorer and wallet confirmations, one checkbox labeled **Set Pubkey — Can set public key records** at the visible **Root (all nodes)** scope makes Explorer submit `grantRootRoles(4096, holder)` from the already-connected parent administrator's wallet to a genuine `UserRegistry`, where `4096` grants registry-wide `UNREGISTER`, not pubkey authority. Explorer therefore authors an unsafe target/bitmap pair: a legitimate child holder can point only its own leaf at the honest parent registry, yet the official **View → Add user** path gives that previously bounded holder persistent authority to unregister a live sibling and burn its versioned ERC-1155 NFT. The root cause is that the resolver-role route accepts any syntactically valid address and applies `PermissionedResolver` parameters without proving the target contract class.

## Vulnerability Details

### Security boundary and decisive result

A resolver-role write must be bound to an allowlisted `PermissionedResolver` implementation and its exact role namespace. A route label, leaf-resolver pointer, shared EAC selectors, or generic factory verification cannot authorize applying resolver bit meanings to a `UserRegistry`. The account and numeric bytes match the visible form selection; the defect is not a hidden post-confirmation mutation. Explorer itself authors a resolver-derived bitmap for a target it never proved was a resolver, so the displayed **Set Pubkey** permission and the target-executed `UNREGISTER` permission diverge at transaction construction.

The decisive path starts from bounded, ordinary authority. The holder of `axel.bajigur.eth` initially has only child-resource `SET_RESOLVER`, a zero parent-root bitmap, and no ability to alter or unregister sibling `agent.bajigur.eth`. The holder uses Explorer's supported custom-resolver flow to make that one leaf point to its honest parent registry. Explorer follows the pointer, labels the target as resolver roles, and gives the genuine parent administrator an official **View** link. The administrator follows Explorer's ordinary in-product path, deliberately keeps or selects the visible **Root (all nodes)** scope, adds the holder, selects only **Set Pubkey — Can set public key records**, and explicitly confirms and approves the transaction. Explorer then builds this request to the registry:

```
grantRootRoles(4096, 0xe5d87b1ff21eea9a774407a4c5e485ad57fee950)
```

On a genuine `PermissionedResolver`, `4096` is `ROLE_SET_PUBKEY`. On the deployed target, the same bit is `ROLE_UNREGISTER`. The pinned fork executes Explorer's exact 68 bytes against deployed code, changes the holder's root bitmap from zero to exactly `4096`, unregisters `agent.bajigur.eth`, and changes the old sibling-token balance from `1` to `0`. Revocation blocks later writes, but manual re-registration creates token version `…00000001`; it does not resurrect the burned token `…00000000`.

The minimum broken composition is visible in four production operations:

* [`routes/resolver/$address.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/$address.tsx#L10-L18) checks only whether the route parameter is an address.
* [`NameResolverRolesOverviewTable.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/NameResolverRolesOverviewTable.tsx#L57-L100) takes the on-chain leaf pointer, queries it as a resolver, labels it `<name> resolver roles`, and creates the official `/resolver/$address/roles` **View** link.
* [`routes/resolver/$address/roles/index.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/$address/roles/index.tsx#L35-L98) performs a shared `hasRootRoles` check and passes the same untyped address to `ResolverAddUserSheet`.
* [`grantResolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L74) encodes resolver roles and makes that address the transaction target.

The decisive statements are small:

```tsx
// routes/resolver/$address.tsx
if (!isAddress(ctx.params.address, { strict: false })) throw redirect({ to: '/' })

// NameResolverRolesOverviewTable.tsx
<Link params={{ address: resolverAddress }} to="/resolver/$address/roles">View</Link>

// grantResolverRoles.ts
return toEoaCustomIntent({
  from: walletClient.account.address,
  to: resolverAddress,
  data,
  chainId,
})
```

[`resolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L5-L89) assigns `ROLE_SET_PUBKEY = 1n << 12n` and displays **Set Pubkey / Can set public key records**. No statement between the route and the write proves that the target uses that namespace.

The primary affected files are:

* `apps/portal/src/routes/resolver/$address.tsx`
* `apps/portal/src/routes/resolver/$address/roles/index.tsx`
* `apps/portal/src/features/resolver/components/NameResolverRolesOverviewTable.tsx`
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx`
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`
* `apps/portal/src/lib/roles/resolverRoles.ts`
* `apps/portal/src/features/registry/hooks/useHasRoles.ts`

The mounted name route and final handoff additionally exercise `apps/portal/src/routes/$name/roles/index.tsx`, `apps/portal/src/features/records/hooks/useNameResolverAddress.ts`, `apps/portal/src/features/resolver/components/ChangeResolverForm.tsx`, `apps/portal/src/features/resolver/helpers/changeResolver.ts`, and `packages/transaction-manager/src/actors/eoa-transport.actor.ts`. The frozen public Git links are listed under **References**, and the direct tree-equality commands under **Proof of Concept** bind the complete scoped Portal tree.

The indexer's registry-as-resolver projection amplifies reachability, but it is not the security boundary or the requested asset. Explorer also follows an ordinary owner-set raw leaf pointer and accepts a direct dynamic address. A complete client fix must therefore fail closed even if a projection is stale, wrong, or compromised.

### Actors, preconditions, and ordinary reachability

The proof uses live capability fixtures from Ethereum Sepolia at block `11,679,987` (`0xb238f3`), block hash `0x10ddad104deb36c1652602bb52cca52f660ed60f044c79eb20f4c95541bfbeab`:

| Item                         | Address or state                             | Attacker control or required authority                                                                                                                                                                                     |
| ---------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bounded holder               | `0xe5d87b1ff21eea9a774407a4c5e485ad57fee950` | Owns live `axel.bajigur.eth`; has bit 24 only on resource `0xaef2307d656cb6145c9d0bcea7eeeab39a4b4163f04abf9261bfed2c00000000`; has no root bit 12 or 24 and a zero root bitmap                                            |
| Connected victim             | `0xe610b819dd190fc8154d0190bb3f3e9758d42baa` | Genuine root administrator of the parent registry; the live projection and on-chain checks show the needed bit-12 admin authority. This is a namespace user's ordinary admin role, not protocol governance or a leaked key |
| Sibling used to measure harm | `agent.bajigur.eth`                          | Live, unexpired child owned by the victim; old token ID `0x314c1dfbbccab41a44cf9fefc21e632eb8e01bc0346d4414114d4c3d00000000`, balance `1`                                                                                  |
| Parent target                | `0x9673702a3c850fa1c41d94c908083fc85f59461c` | Genuine factory-verified `UserRegistry` for `bajigur.eth`; neither attacker-deployed nor malicious                                                                                                                         |
| Genuine resolver control     | `0x9224a7cc7d35899753456e53f78139a26884ed89` | Factory-verified `PermissionedResolver`, used as the matched semantic control                                                                                                                                              |
| UniversalResolver            | `0xeeeeeeee14d718c2b47d9923deab1335e144eeee` | Deployed lookup used by the application path                                                                                                                                                                               |

The addresses are live capability fixtures. This report does not allege that their real-world controllers are malicious. `vm.prank` impersonates them only in a local fork.

#### Scope behavior in the captured zero-node mounted state

A legitimate subname holder asking its namespace administrator for permission to manage a pubkey record provides a plausible reason to open Explorer's **Add user** feature. The captured production-shaped registry response embedded in the mounted test contains `nodeCount: 0` and `nodes: []`. [`ResolverAddUserSheet`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L117-L186) initializes `selectedNode` to the empty-string root resource and derives every non-root choice only from `nodes`; the [scope combobox](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L248-L289) renders **Root (all nodes)** plus those derived choices. Root is therefore the default and sole rendered scope in that captured/mounted state.

This is a bounded reachability fact, not a claimed permanent post-seed production state. The live pinned pre-state still had the child's old resolver; the post-seed leaf pointer is persisted only on the local fork, then composed in the mounted test with the separately captured zero-node response. If later indexing adds the child to `resolver.nodes`, a child scope may appear, but root remains visible and initially selected. The exploit does not rely on indexer lag or the absence of another option: the administrator must deliberately leave or select the visible root scope, complete Explorer's confirmation, and approve the wallet transaction.

There is no race. Exploitation requires the following deliberate sequence:

1. The child holder changes only `axel.bajigur.eth`'s resolver to its parent registry.
2. The already-connected parent administrator opens `/axel.bajigur.eth/roles` and clicks the production **View** link.
3. The administrator clicks **Add user**, enters or selects the child holder, deliberately keeps or selects the visible **Root (all nodes)** scope, checks **Set Pubkey**, and clicks **Save**.
4. The administrator starts the transaction, opens the wallet, and approves it.
5. The holder later sends an `unregister` transaction.

This is not zero-click. The victim intentionally grants what the official app describes as a resolver record permission to an account that already legitimately controls the displayed child. The defect is that the app binds this consent to a different honest contract class and permission vocabulary.

The attack does not require a hand-built raw-address URL, search misclassification, an attacker-deployed contract, malicious fallback code, EAC bypass, signature bypass, stale or reordered rows, removal of a user, unresolved-resource fallback, name normalization, a chain mismatch, leaked keys, or control of the indexer.

The seed action is the supported custom-resolver mode at `/axel.bajigur.eth/change-resolver`. The production builder targets the parent registry and calls `setResolver` for only the holder's child resource. The fork proves that the holder's existing child-level bit 24 is sufficient and does not affect a sibling. The ordinary route then is:

```
/axel.bajigur.eth/roles
  → NameResolverRolesOverviewTable
  → “axel.bajigur.eth resolver roles”
  → production “View” link
  → /resolver/0x9673702a3c850fa1c41d94c908083fc85f59461c/roles
  → Add user
  → root (explicit/default; also the only scope in the captured zero-node projection)
  → “Set Pubkey — Can set public key records”
```

Both public URLs returned HTTP 200 in the fresh capture. The production-shaped GraphQL query returned a non-null `resolver` for the registry with `nodeCount: 0`, `aliasCount: 0`, `roleHolderCount: 3`, and five live EAC rows. Even if later indexing adds the leaf as a node, root remains a selectable/default scope; the proof does not edit a projected row.

### End-to-end application and deployed-state execution

The complete path retains the vulnerable production route, components, builders, and final EOA transport:

1. The holder uses `/$name/change-resolver` and `prepareChangeResolverTransaction` with its own child, resource bit 24, and the parent registry as the custom address. Explorer builds the exact `setResolver(anyId, registry)` request.
2. The local fork executes those 68 bytes. Only the `axel` leaf pointer changes to `0x9673…461c`; pre-grant attempts to unregister the sibling or change its resolver still revert.
3. `useNameResolverAddress` calls ENSjs `getResolver` with DNS bytes `0x046178656c0762616a696775720365746800`. The deployed UniversalResolver returns `0x9673…461c`, a nonzero node, and offset `0`. The mounted hook fixture is bound to that deployed result.
4. On `/$name/roles`, the real `NameResolverRolesOverviewTable` displays `axel.bajigur.eth resolver roles`; the five registry EAC rows receive resolver labels.
5. The victim clicks the real production `<Link to="/resolver/$address/roles">` and reaches the raw-address page without typing an address.
6. The resolver route calls `getHasRolesQueryOptions` with resolver `ROLE_SET_ADDR` bit 0. The shared `hasRootRoles(1, victim)` selector returns true because registry bit 0 is `REGISTRAR`, so **Add user** is enabled.
7. In the real `ResolverAddUserSheet`, the victim selects the holder, node `''` for root, and `ROLE_SET_PUBKEY`. The modal says resolver roles and `Grant resolver roles for (root)`; it never says `UserRegistry` or `UNREGISTER`.
8. `prepareGrantResolverRolesTransaction` receives the untyped target and resolver bit 12, then creates the exact `grantRootRoles(4096, holder)` request to the registry.
9. The real transaction-manager EOA actor receives a matching Sepolia wallet account and passes `to`, `data`, `value`, account, and wallet chain unchanged to viem. The final RPC send is mocked to prevent a live write.
10. The fork executes the same bytes against the deployed registry. The holder's root bitmap becomes exactly `0x1000`, which the registry interprets as `UNREGISTER`.
11. The holder calls `UserRegistry.unregister(agentLabelId)`. The sibling's old-token balance becomes zero and its owner becomes the zero address.
12. The administrator revokes the root role and re-registers the saved owner, resolver, subregistry, roles, and expiry. A later holder `unregister` reverts, while the restored label has a new token version.

The evidence isolates contract class as the cause. Factory verification identifies the target as the honest deployed `UserRegistryImpl`, while the matched control identifies a genuine `PermissionedResolverImpl`. The mounted suite uses the real name route, **View** link, Add User form, labels, modal, builder, and transport handoff. The genuine-resolver control gives bit 12 only pubkey-writing authority. The positive type gate rejects the registry and accepts the genuine resolver. Before the grant, after revocation, and under the type gate, the sibling action fails.

### Exact operations and byte binding

**Holder's leaf-pointer request**

```
from:     0xe5d87b1ff21eea9a774407a4c5e485ad57fee950
to:       0x9673702a3c850fa1c41d94c908083fc85f59461c
chainId:  11155111
value:    0
selector: 0xbc7b6d62 = setResolver(uint256,address)
anyId:    0xaef2307d656cb6145c9d0bcea7eeeab39a4b4163f04abf9261bfed2c00000000
resolver: 0x9673702a3c850fa1c41d94c908083fc85f59461c
```

```
0xbc7b6d62aef2307d656cb6145c9d0bcea7eeeab39a4b4163f04abf9261bfed2c000000000000000000000000000000009673702a3c850fa1c41d94c908083fc85f59461c
```

Its `keccak256(bytes)` is `0x167e048f3722d09f6f7f4d85177039b9fa67cdba9b45b725ccdce4c7e0d943b1`. Its SHA-256 is `f787cc16578267bf2f21695bb481ba8d6f55e16fbebbb1500bc557572bcd582b`.

**Victim's one-checkbox request**

```
from:     0xe610b819dd190fc8154d0190bb3f3e9758d42baa
to:       0x9673702a3c850fa1c41d94c908083fc85f59461c
chainId:  11155111
value:    0
selector: 0x072d5d77 = grantRootRoles(uint256,address)
bitmap:   4096 = 0x1000 = bit 12
account:  0xe5d87b1ff21eea9a774407a4c5e485ad57fee950
```

Exact calldata:

```
0x072d5d770000000000000000000000000000000000000000000000000000000000001000000000000000000000000000e5d87b1ff21eea9a774407a4c5e485ad57fee950
```

```
length:             68 bytes
SHA-256(bytes):     8864b191415ca9ea58cbe958cfe82a8adce0f5a831c7b659b0ab8d59a83a2dda
keccak256(bytes):   0x639671e7cca7c3fb76540bfa5b790a8e930e2ff70cfd018170d3baa05586fc82
```

The bridge is byte-exact, not an inferred equivalence. The mounted test invokes the production builder and compares `request.data` with this literal. The EOA test proves that the real transport gives viem the same literal, target, account, value, and wallet chain. The fork defines the same 68-byte constant, proves that its keccak equals `abi.encodeWithSelector(0x072d5d77, 1 << 12, holder)`, and executes that constant with `REGISTRY.call`. Running the embedded primary fork test with `-vvvv` prints both equal keccak operands as `0x639671…6fc82` before the call.

Both contract families expose the same EAC `grantRootRoles(uint256,address)` signature and selector `0x072d5d77`. The embedded tests prove that the literal request equals the ABI encoding for that signature and that the registry executes it under its own role namespace. This is role-namespace type confusion, not reliance on a different-signature collision or fallback trick.

|      Bit |      Bitmap | Explorer / `PermissionedResolver` meaning | `UserRegistry` meaning                  | Measured consequence                                                                                 |
| -------: | ----------: | ----------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------- |
|        0 |       `0x1` | Set Address / `ROLE_SET_ADDR`             | `ROLE_REGISTRAR`                        | Supporting three-bit test registers or reissues labels without the intended registrar/payment policy |
|       12 |    `0x1000` | **Set Pubkey / `ROLE_SET_PUBKEY`**        | **`ROLE_UNREGISTER`**                   | Primary one-checkbox test burns a live sibling NFT                                                   |
|       24 | `0x1000000` | Set Name / `ROLE_SET_NAME`                | `ROLE_SET_RESOLVER`                     | Supporting test redirects another live child                                                         |
| combined | `0x1001001` | Three resolver permissions                | `REGISTRAR + UNREGISTER + SET_RESOLVER` | Supporting breadth test reissues, registers a fresh label, and redirects a sibling                   |

Admin bits use the same `managerBit + 128` layout, so the real registry administrator can grant the selected manager bit. Explorer's admin checkboxes are disabled; this report does **not** claim an attacker admin-bit lockout.

### Deployed implementation binding and causation controls

At the pinned block, both targets are genuine factory deployments but they belong to opposite contract families:

| Check                                 | Genuine target registry                                           | Genuine resolver control                                                  |
| ------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Proxy or control address              | `0x9673702a3c850fa1c41d94c908083fc85f59461c`                      | `0x9224a7cc7d35899753456e53f78139a26884ed89`                              |
| `VerifiableFactory.verifyContract`    | `0x624a25d67b59d587752ebec8dded8827dae52050` (`UserRegistryImpl`) | `0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e` (`PermissionedResolverImpl`) |
| `IEnhancedAccessControl` `0x8f452d62` | true                                                              | true                                                                      |
| `IExtendedResolver` `0x9061b923`      | false                                                             | true                                                                      |

The embedded fork verifies both implementation identities directly through `VerifiableFactory.verifyContract` and checks the distinguishing interface matrix shown above. Generic factory verification is insufficient because both honest families pass it; the gate must require the expected implementation family.

The primary fork test measures the complete but-for state transition and recovery:

| Fact                                       |                   Before |                                                     Correct implementation |                                                  Vulnerable execution |                                           After corrective recovery |
| ------------------------------------------ | -----------------------: | -------------------------------------------------------------------------: | --------------------------------------------------------------------: | ------------------------------------------------------------------: |
| Holder parent-root bit 12                  | `false`; root bitmap `0` | Registry target rejected before render, estimate, or send; remains `false` |                                    `true`; root bitmap exactly `4096` |                       `false` after `revokeRootRoles(4096, holder)` |
| Holder can unregister `agent`              |                  Reverts |                                                                    Reverts |                                                              Succeeds |                                                       Reverts again |
| Holder can change `agent` resolver         |                  Reverts |                                                                    Reverts | Not needed for the one-bit arm; succeeds in the three-bit breadth arm |                                            Reverts after revocation |
| `agent` old token balance at victim        |                      `1` |                                                                        `1` |                                                  `0` after unregister |                             Still `0`: the old token remains burned |
| `agent` label owner                        |     Victim administrator |                                                       Victim administrator |                                         Zero address after unregister |                   Victim administrator after manual re-registration |
| Recovered token ID and balance             |           No replacement |                                                      No replacement needed |                                      No replacement in one-bit attack | `0x314c…00000001`, balance `1`; distinct from old `0x314c…00000000` |
| Holder's legitimate `axel` resource bit 24 |                     true |                                                                       true |                                                                  true |                         true; only the illicit root role is removed |

Initial `unregister(agent)` and `setResolver(agent, …)` calls from the holder revert. The exact leaf-pointer request then succeeds only for `axel`. The exact mounted grant gives root bit 12, after which `unregister(agent)` succeeds, reduces the old ERC-1155 balance from one to zero, and clears ownership. Because the one-checkbox arm does not grant `REGISTRAR`, the holder still cannot reissue the label. The administrator revokes bit 12 and re-registers the saved tuple; a later holder unregister attempt reverts.

The primary trace records the exact deployed-code events:

```
EACRolesChanged(resource: 0, account: holder, oldRoleBitmap: 0, newRoleBitmap: 4096)
LabelUnregistered(tokenId: 0x314c…00000000, sender: holder)
TransferSingle(operator: holder, from: victim, to: 0x0000000000000000000000000000000000000000,
               id: 0x314c…00000000, value: 1)
```

The matched resolver control grants bit 12 on the genuine resolver, successfully writes a pubkey record, and fails when the registry `unregister` selector is called. The positive type-gate control rejects the `UserRegistry` implementation and accepts the genuine `PermissionedResolver`. These controls isolate contract class as the cause.

The three-checkbox test is supporting breadth, not a prerequisite for the primary result. It measures `REGISTRAR | UNREGISTER | SET_RESOLVER`, same-label attacker reissue, a fresh free label, and another child's resolver redirection, followed by full administrator cleanup.

### Proof boundaries, recovery, and falsification controls

The mounted application test replaces network and signing boundaries so it cannot contact live users or mutate Sepolia. Each load-bearing value is independently bound to the inline public HTTP/GraphQL reproduction or the pinned fork:

| Boundary in the application test                                          | Why it is replaced                                                                | Independent binding                                                                                                                                     |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ENSv2 owner, leaf resolver result, resolver overview, and role-gate query | Keeps the mounted UI deterministic and prevents live network access during Vitest | The inline public GraphQL reproduction and pinned-fork assertions, including the deployed UniversalResolver call, reproduce each value                  |
| Connected wagmi wallet and address-input resolution                       | No live account is contacted                                                      | Pinned on-chain administrator/holder roles bind both addresses; the EOA test enforces the matching account and exact recipient                          |
| Final viem `sendTransaction` result                                       | Prevents a production write                                                       | The real EOA actor runs up to its viem boundary; Foundry then executes identical bytes against deployed registry semantics                              |
| Sheet portal, gas display, headers, and unrelated name-page sections      | Presentation-only dependencies                                                    | The vulnerable production routes, name-to-resolver table, **View** link, Add User form, labels, modal intent, builders, and transport actor remain real |
| `vm.prank` identities                                                     | A local fork has no private keys                                                  | Initial roles, ownership, implementations, interfaces, and code all come from pinned deployed state; no storage is injected                             |

Mechanically, the mounted suite preserves the real `@tanstack/react-query` module but overrides `useQuery` and dispatches captured fixtures by `options.queryKey?.[0]`: `get-ens-owner`, `get-name-resolver-address`, `resolver-overview`, and `hasRoles`. It also disables the `$name` parent `beforeLoad`/loader and the resolver-role prefetch loader while retaining the production parent component and resolver route guard. Those component-path query functions and prefetches do not execute during the mounted render. A separate assertion executes the real `getHasRolesQueryOptions(...).queryFn` against the isolated chain client and verifies the exact `hasRootRoles(1, victim)` call. The inline public HTTP/GraphQL reproduction and pinned-fork assertions, including the deployed UniversalResolver call, independently bind the values supplied for each intercepted tag.

Revocation ends the hidden continuing authority, but it does not roll back completed operations. An authorized administrator must remove attacker-created or reissued labels, restore resolver/subregistry pointers and role tuples, and re-register burned labels with the saved owner, resolver, subregistry, roles, and expiry. Re-registration increments the token version. The burned old token does not return.

The controls also falsify adjacent explanations: a non-EAC arbitrary target does not yield victim-asset authority; a genuine resolver gives only pubkey authority; one bit cannot reissue the burned label; admin checkboxes cannot grant shifted admin bits; a signer mismatch fails before wallet invocation; and no different-signature selector collision exists. The exploit does not use removal, row reordering, an unknown resource, normalization, or a chain mismatch.

No production state-changing transaction was sent. No user was contacted, and no phishing or social engineering was performed. Public endpoints were used only for read-only requests. Every mutation used impersonation only inside a local fork pinned to block `11,679,987`.

### Strongest adverse prior art: contracts-v2 PR #440

[`ensdomains/contracts-v2#440`](https://github.com/ensdomains/contracts-v2/pull/440), **“Fix `findResolver()` not checking `IExtendedResolver`,”** was created on 2026-09-08 and merged on 2026-09-09. Its body explicitly says **“Security issue”**. This is the strongest adverse fact because it publicly states that resolver addresses need consumer-specific validation before use.

Its complete rule is nevertheless different. Contract code walking a DNS name must require ENSIP-10 `IExtendedResolver` only when the resolver was inherited from above the exact leaf:

```solidity
function validateResolver(address resolver, bool foundAtLeaf)
    internal view returns (address)
{
    return foundAtLeaf ||
        ERC165Checker.supportsERC165InterfaceUnchecked(
            resolver,
            type(IExtendedResolver).interfaceId
        ) ? resolver : address(0);
}
```

This report deliberately exercises the allowed branch. After the bounded holder updates its own leaf, the deployed UniversalResolver returns the `UserRegistry` with `offset == 0`, even though it reports `IExtendedResolver == false`. That behavior is appropriate for generic exact-leaf resolution. It is insufficient for an authorization UI that assigns `PermissionedResolver` role meanings.

| Dimension         | PR #440                                                        | This report                                                                                |
| ----------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Consumer          | Contracts that resolve names                                   | Explorer role-administration UI and wallet builder                                         |
| Input             | Resolver found during name-tree traversal                      | Exact leaf address returned by that traversal, then used as an untyped write target        |
| Required identity | `IExtendedResolver` capability only when inherited             | Allowlisted `PermissionedResolver` implementation and role namespace for every role target |
| Failure           | Inherited resolver cannot handle ENSIP-10 full-name resolution | Valid EAC bits grant different powers on another honest class                              |
| Sink              | Resolution/read call                                           | `grantRootRoles(uint256,address)` wallet write                                             |
| Complete fix      | `foundAtLeaf` OR `supportsInterface(IExtendedResolver)`        | Positive class/provenance gate before display, estimation, grant, edit, and revoke         |

The two-way complete-fix test is executed rather than hypothetical. **PR #440 → this report:** the fork receives the registry as the exact leaf from the deployed UniversalResolver, and the Explorer grant still burns the sibling. **This report → PR #440:** adding a Portal `PermissionedResolver` allowlist does not change contract-side inherited-resolver traversal, so PR #440 remains necessary.

PR #440 does not mention Portal, EAC role administration, `UserRegistry`/`PermissionedResolver` bitmap overlap, `grantRootRoles`, wallet requests, `UNREGISTER`, or NFT burning. A broad reviewer may still treat its generic validation theme as prior notice; that adverse interpretation is disclosed rather than hidden.

### Submitted-report duplicate fences

The complete reviewed local submission corpus contains five reports that share abstractions, surfaces, or outcomes but not this trigger and complete fix:

| Prior report or item                                                                                     | Shared surface or outcome                                                                                                                                                                         | Different root                                                                                            | Different trigger or sink                                                                            | Prior complete fix                                                                                                     | Why that fix leaves this bug; reciprocal complete-fix test                                                                                                                                                                                                         |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `#90254` — migration-plan cache key omits V1 authority fields; closed as matching `#89285`               | Both can end with attacker-favored resolver/authority state and use Manager-era authorization data                                                                                                | Stale V1 migration plan/TOCTOU because owner/resolver authority fields are omitted from cache identity    | V1→V2 migration executes stale planned values; no Explorer resolver-role grant                       | Include current V1 authority fields in the cache key and revalidate before executing migration                         | That cache repair never runs on Add User and leaves the exact request unchanged. Conversely, an Explorer resolver type gate never runs in Manager migration and cannot stop a valid but stale migration plan                                                       |
| `#90446` — Worker push fanout partial commit/retry; Medium confirmed                                     | Only the abstract concept of incomplete or retried work                                                                                                                                           | Global/per-user push-channel capacity and retry after a partially committed backend fanout                | Worker/backend retry and notification suppression; no wallet or EAC sink                             | Enforce correct per-user capacity and persist/retry only unsent recipients after partial commits                       | Worker retry repair does not inspect Explorer targets. An Explorer implementation allowlist does not change fanout state. Both triggers remain under the other's fix                                                                                               |
| `#92510` — raw-NFD authorization/display transfers a distinct NFC token; Critical duplicate              | Explorer makes an approved wallet action execute under an identity or meaning different from its label; `#92510` was confirmed under direct NFT theft, whereas this report requests impact `6288` | Raw versus normalized name/token identity is split across authorization, display, and transfer            | Attacker-gifted raw NFD route → `safeTransferFrom` of a different normalized ERC-1155 token          | Normalize once and bind the authorized/displayed name to the exact token transferred                                   | Normalization does not type a resolver target; this report performs no name transformation or token selection. Conversely, a resolver implementation gate does not bind raw/NFC identity in the transfer path                                                      |
| `#92535` — positional row reuse grants another account's `UPGRADE`; Critical duplicate                   | Same Explorer resolver-roles surface; can emit an unintended root `grantRootRoles`                                                                                                                | Positional TanStack/React row identity lets a stale draft cross accounts after remove/refetch/reorder     | Edit successor row after removal; stale account/permission state is sent on a genuine resolver       | Stable account-based row IDs and reset draft whenever row/account identity changes                                     | This report uses Add User, no edit/removal/refetch/reorder/draft, and sends the exact selected account and bit; stable IDs do nothing. Conversely, this report's type-gate control accepts a genuine resolver, so stale-row substitution remains exploitable there |
| `#92552` — “Remove user from all roles” revokes only one resolver scope; reopened and Critical duplicate | Closest family: same Explorer role helpers and EAC surface                                                                                                                                        | Account-only aggregation collapses multiple raw resources; related unknown resources can collapse to root | Remove/edit a projected grouped row; `revokeRoles` or wrong-scope grant/revoke on a genuine resolver | Key by `(account, resource)`, preserve raw resources, issue one revoke per scope, and fail closed on unknown resources | This report never opens a projected row. It uses Add User, one account, explicit root, and a wrong honest class. The full resource fix leaves it unchanged. Conversely, this report's gate accepts genuine resolvers, so aggregation under-revocation remains      |

The highest-risk roles-family comparison is concrete:

| Dimension               | `#92535`                              | `#92552`                                       | This report                                                        |
| ----------------------- | ------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------ |
| UI action               | Edit successor after removal          | Remove grouped account                         | Add User with explicit/default root                                |
| Identity error          | Positional row/draft crosses accounts | Multiple resources collapse into one operation | Route says resolver; target is another honest contract class       |
| Sent account and bitmap | Stale union for wrong successor       | Flattened operation against one scope          | Exact chosen account and bit; wrong role vocabulary                |
| Required state          | Removal/refetch/reorder               | Multiple or unknown resources                  | Genuine factory-verified `UserRegistry` exposed as resolver        |
| Fix                     | Stable row identity; reset draft      | Key/revoke per account-resource; fail unknown  | Positive implementation/role-namespace binding at render and write |

All mounted assertions for this report pass without invoking row edit, removal, reordering, grouped-resource selection, or unknown-resource fallback, so both prior roles fixes are bypassed structurally. In the other direction, the fork's positive type gate accepts the genuine resolver used by those reports, so this report's fix does not address their row/resource defects.

### Other public and internal prior art reviewed

* Draft contracts-v2 PR [`#444`](https://github.com/ensdomains/contracts-v2/pull/444) explores a `TrustedRegistrySet`. Generic trusted provenance is not typed provenance: an allowlist that approves the genuine `UserRegistry` still does not authorize applying resolver roles to it. A chain-specific `PermissionedResolver` implementation set is required.
* Public contracts-v2 issue [`#418`](https://github.com/ensdomains/contracts-v2/issues/418) and PR [`#428`](https://github.com/ensdomains/contracts-v2/pull/428) concern delegated EAC roles surviving ERC-1155 transfer. This proof has no transfer and no prior holder root grant; the request creates fresh root authority from zero. Transfer cleanup does not stop it.
* Bigname `#768` is a closed, no-current-consequence observation about resolver classification/read features coming from different `COALESCE` branches. Aligning those fields does not bind Explorer's wallet target to a role namespace.
* Bigname `#661` concerns display of registry bit 32. It shows the vocabularies differ but does not report cross-class Explorer decoding or a wallet write.
* Successful exact public searches found no paired `ROLE_SET_PUBKEY`/`ROLE_UNREGISTER`, `ROLE_SET_ADDR`/`ROLE_REGISTRAR`, or “Set Pubkey”/“Unregister” issue. Later unauthenticated searches were rate-limited. The bounded conclusion is only that no match was found in the successful searches and reviewed corpus. Public search and the reviewed local corpus cannot rule out an unseen private primary, so this report claims a different invariant and complete fix from the items reviewed rather than an exhaustive novelty search.

### Recommended remediation

Create a chain-specific typed provenance gate. Resolve every role target through `VerifiableFactory` and require its implementation or code identity to be in an explicit allowlist of supported `PermissionedResolver` versions. Do not accept every factory deployment.

Apply the gate at every point where the wrong type can enter or be used:

1. Fail closed in the raw `/resolver/$address` loader and `NameResolverRolesOverviewTable` before querying, rendering, or labeling roles.
2. Independently revalidate the typed attestation in `prepareGrantResolverRolesTransaction`, revoke/edit helpers, and their estimation and send paths. A cached indexer label or UI-only guard is not sufficient.
3. Apply the inverse rule to registry routes and builders: accept only configured `UserRegistry`/registry implementations and decode only the registry vocabulary.
4. Also require the expected EAC/resolver interfaces and reject known registry interfaces. Use this as defense in depth; ERC-165 alone is spoofable and is not provenance.
5. Correct the indexer/API projection so a registry is not returned as a resolver role entity. Keep the client gate because owner-set leaf pointers, stale caches, and direct routes bypass projection correctness.
6. Show the full target, verified contract family/version, resource scope, recipient, and role names decoded from that validated family in the confirmation step. Warn clearly for root-wide authority.
7. Use typed wrappers instead of a bare `Address` between route loaders, views, and builders, so resolver and registry addresses cannot be interchanged silently.

Regression coverage should reject a registry target in the direct resolver route, ordinary exact-leaf name route, stale or malicious resolver projection, grant, edit, revoke, and estimation paths before wallet invocation. It should preserve rendering, pubkey grant, estimation, and send for a genuine resolver. It should also retain stable-row/draft tests for `#92535`, account-resource/fail-unknown tests for `#92552`, the normalize-once test for `#92510`, and inherited-resolver tests for PR #440. The typed gate does not replace any of those fixes.

Warnings alone are not a complete fix. The authority vocabulary must be selected only after positive contract-class binding.

## Impact Details

The exact selected in-scope impact is **Critical impact `6288`**:

> Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

The measured path maps to that impact as follows. The claim relies primarily on the umbrella connected-wallet interaction and submission of an unsafe transaction; it does not allege literal in-transit byte mutation or address swapping.

| Catalog clause                                | Measured mapping                                                                                                                                                                                                                               |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Already-connected wallet                      | The mounted flow uses the connected registry administrator and the real transaction modal/EOA-request path                                                                                                                                     |
| Modifying transaction arguments or parameters | Explorer constructs a target/bitmap pair whose displayed meaning is `SET_PUBKEY` but whose target-executed meaning is `UNREGISTER`. The integer, account, and address match the form; no post-selection or in-transit byte mutation is alleged |
| Substituting contract addresses               | **Not relied upon as a separate catalog clause.** The builder sends to the exact honest leaf address; the defect is that Explorer never proves it is a `PermissionedResolver` before applying resolver-role semantics                          |
| Submitting malicious transactions             | After the disclosed victim approval, the exact request grants persistent destructive authority to the chosen holder; the fork measures the resulting sibling burn                                                                              |
| Wrong-chain example                           | **Not relied upon.** Request metadata and the wallet are consistently Sepolia. The separately known chain-validation issue is not causal                                                                                                       |
| Blanket-token-approval example                | **Not claimed.** No `setApprovalForAll` occurs. The measured result is a root EAC `UNREGISTER` grant, not a token approval                                                                                                                     |

The immediate consequence is not a misleading read. One ordinary Explorer grant changes a bounded child holder from a zero parent-root bitmap to persistent root `UNREGISTER` authority on the genuine parent registry. The holder can then burn a live sibling NFT and clear its label ownership. Revocation prevents later unauthorized calls but does not revert prior unregister, registration, or resolver changes. Manual re-registration restores a label only under an incremented token version, leaving the old ERC-1155 token permanently burned. The supporting three-bit control also proves unauthorized label registration/reissue and sibling resolver redirection, but the single **Set Pubkey** checkbox and sibling burn are sufficient for the selected impact.

The recovery trace proves a period between unregister and manual re-registration in which the label owner is the zero address. After repair, the original token ID still has balance zero while the incremented replacement ID has balance one. Re-registration does not reuse the old ID or automatically update third-party state keyed to it, so any exact-ID integration would require separate reconciliation. This report does not claim a specific third-party integration or additional financial loss from that residue.

| Question                         | Consequence or limitation                                                                                                                                                                                                                                                          |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Victim interaction               | Required: ordinary navigation, Add User, explicit/default root selection, one checkbox, Explorer confirmation, and wallet approval                                                                                                                                                 |
| Wallet and simulation visibility | A wallet may show the full target and decode `grantRootRoles(4096, holder)`. This is a real defense. It may not explain that bit 12 on this factory-verified address is registry-wide `UNREGISTER` after the official app called it pubkey permission. No wallet bypass is claimed |
| Who pays                         | The holder pays for its own leaf-pointer transaction; the victim pays for the grant; the holder pays for the later harmful transaction                                                                                                                                             |
| Duration                         | The hidden root role lasts until an authorized registry administrator revokes it. Completed state changes do not self-revert when the role is revoked                                                                                                                              |
| Detection                        | `EACRolesChanged`, `LabelUnregistered`, ERC-1155 burn/transfer, registration, and resolver-update events expose the writes. Detection is possible but is not automatic prevention, and Explorer may continue to decode the role under the wrong vocabulary                         |
| Repair                           | Revoke hidden root roles; remove attacker-created or reissued labels; restore resolver/subregistry pointers and role tuples; re-register burned labels with saved owner, resolver, subregistry, roles, and expiry                                                                  |
| Automatic repair                 | None. Closing the modal or later revocation does not restore altered namespace state                                                                                                                                                                                               |
| Irreversible residue             | The old versioned NFT stays burned. Re-registration increments the token version and creates a different ERC-1155 ID                                                                                                                                                               |

The claim has these explicit limits:

* This is not an EAC bypass. Both contracts correctly execute shared EAC functions using their own documented role namespaces.
* The registry is an honest, factory-verified deployment. The proof does not use attacker code or an ERC-165 spoof.
* The victim intentionally approves a collaborator-style root grant. The issue is not zero-click and not an authenticated action without user interaction.
* A wallet can decode the method and address. The claim is the official app's wrong contract-class and role-semantic binding, not hidden bytes or a confirmation bypass.
* No native or ERC-20 funds were transferred or shown lost. Direct fund theft is not claimed.
* Recovery is possible for a sufficiently authorized and informed administrator. It is manual, and the burned token version is not resurrected.
* The end-to-end transport test covers an EOA wallet. No separate claim is made for an untested HCA transport variant.
* The mounted test fixtures network reads, then independently binds every load-bearing value to the inline public HTTP/GraphQL reproduction or pinned fork. It does not send a live transaction.
* The subgraph/indexer and contracts are consequence dependencies, not the requested vulnerable asset. Correcting only the projection is incomplete because Explorer follows a raw owner-set leaf pointer and accepts dynamic routes.

## References

* [Public ENS audit competition page](https://immunefi.com/audit-competition/audit-competition-ens/)
* [ENS competition scope, impacts, and Known Issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Scoped Explorer asset at frozen application snapshot `cda79acaad59711b943fc68207ebb3f1d0ff8596`](https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal)
* [`/$name/roles` ordinary entry route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/roles/index.tsx#L12-L68)
* [`useNameResolverAddress` leaf-pointer lookup](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/records/hooks/useNameResolverAddress.ts#L39-L58)
* [`ChangeResolverForm` supported custom-resolver path](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx#L55-L62)
* [`ChangeResolverForm` custom target and wallet intent](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx#L121-L138)
* [`ChangeResolverForm` custom resolver UI](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx#L223-L250)
* [`ChangeResolverForm` transaction builder handoff](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx#L353-L372)
* [`prepareChangeResolverTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/changeResolver.ts#L62-L85)
* [Resolver address route validates only address syntax](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/$address.tsx#L10-L18)
* [`NameResolverRolesOverviewTable` follows and labels the leaf target](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/NameResolverRolesOverviewTable.tsx#L17-L43)
* [`NameResolverRolesOverviewTable` production **View** link](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/NameResolverRolesOverviewTable.tsx#L50-L100)
* [Resolver roles route, shared gate, and Add User handoff](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/$address/roles/index.tsx#L30-L98)
* [`ResolverAddUserSheet` selection and transaction modal](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L118-L337)
* [`prepareGrantResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L18-L75)
* [Explorer's resolver role bitmaps and labels](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L5-L89)
* [Shared `getHasRolesQueryOptions`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/hooks/useHasRoles.ts#L17-L67)
* [Frozen transaction-manager EOA transport](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/eoa-transport.actor.ts#L22-L70)
* [Resolver role constants at contracts-v2 `f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1`](https://github.com/ensdomains/contracts-v2/blob/f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1/contracts/src/resolver/libraries/PermissionedResolverLib.sol#L7-L39)
* [Registry role constants at contracts-v2 `f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1`](https://github.com/ensdomains/contracts-v2/blob/f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1/contracts/src/registry/libraries/RegistryRolesLib.sol#L9-L41)
* [`PermissionedRegistry.unregister` role check](https://github.com/ensdomains/contracts-v2/blob/f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1/contracts/src/registry/PermissionedRegistry.sol#L192-L202)
* [Public ENS staging GraphQL endpoint used by the Portal](https://staging-graphql.ens.dev/)
* [contracts-v2 PR #440](https://github.com/ensdomains/contracts-v2/pull/440) — exact-leaf resolver validation prior art discussed above

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/f46204b98462e7addac2c86b74acbe78>

## Proof of Concept

## Proof of Concept

**Complete executable source and configurations:** [version-pinned Gist](https://gist.github.com/IronsideSec/f46204b98462e7addac2c86b74acbe78/fec10d24c27102505451981f01917f932775ba48)

The PoC has three connected layers:

1. A mounted Explorer suite runs the frozen ordinary name route, real resolver overview and **View** link, Add User form, role labels, transaction modal, production builders, and captures the exact wallet request.
2. A byte-exact EOA suite runs the real transaction-manager actor to its viem boundary, proves that target/account/calldata/value/chain are preserved, and proves that a mismatched signer fails before wallet invocation.
3. A Foundry suite executes the same request bytes on a local Sepolia fork at block `11,679,987`, measures the new role and sibling burn, compares a genuine resolver, performs recovery, and checks a positive implementation-family gate.

No deployed ENS state is mutated. The application suite is offline. The fork reads an immutable historical state and applies every mutation locally. No real private key is used; `vm.prank` represents live capability holders only inside the fork.

The mounted suite substitutes only the network, connected-account, and final-send boundaries described under **Vulnerability Details**. The final RPC call is recorded rather than broadcast. Every load-bearing value is present in the linked source and separately reproduced by the pinned fork or inline public HTTP/GraphQL commands. The fork injects no contract storage and uses deployed code, roles, ownership, implementations, interfaces, and NFT state.

### Clean checkout and self-contained setup

No attachment is required. The version-pinned Gist above contains the complete three executable tests plus both configuration files. Prerequisites are Git, Node.js 22 with Corepack, pnpm 10, Foundry/Forge 1.5.x, `curl`, and `jq`. Start from a clean public checkout:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile

mkdir -p audit_workspace/reports/LEAD-044-evidence/portal
mkdir -p audit_workspace/reports/LEAD-044-evidence/original-evidence
mkdir -p audit_workspace/reports/LEAD-044-evidence/fork-proof/test
mkdir -p audit_workspace/reports/LEAD-044-evidence/fork-proof/lib
```

Verify the frozen source binding directly from Git:

```bash
git rev-parse \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae:apps/portal \
  cda79acaad59711b943fc68207ebb3f1d0ff8596:apps/portal
git show -s --format=%s cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Expected output:

```
33d5046a405f21a383e5515693511f5f8b17529b
33d5046a405f21a383e5515693511f5f8b17529b
fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86
```

Save the five complete code blocks from the version-pinned Gist at their stated paths. Then fetch the only Foundry dependency at its pinned revision:

```bash
git clone https://github.com/foundry-rs/forge-std.git \
  audit_workspace/reports/LEAD-044-evidence/fork-proof/lib/forge-std
git -C audit_workspace/reports/LEAD-044-evidence/fork-proof/lib/forge-std \
  checkout 77041d2ce690e692d6e03cc812b57d1ddaa4d505
```

No application environment variables are needed. The Portal tests make no network request. The Foundry test makes read-only requests to the public Sepolia archive endpoint hardcoded in the linked test; every state change occurs only in the local fork.

### Reproduce the live public classification

The mounted fixture is visible in the linked test source. The same registry is also returned by the public indexer as a resolver entity. Run:

```bash
curl -sS https://staging-graphql.ens.dev/ \
  -H 'content-type: application/json' \
  --data-binary '{"query":"query getResolverOverview($id: String!) { resolver(id: $id) { id address nodeCount aliasCount roleHolderCount nodes { id } roles { account resource roleBitmap name } } }","variables":{"id":"0x9673702a3c850fa1c41d94c908083fc85f59461c"}}' \
  | jq '.data.resolver | {id,address,nodeCount,aliasCount,roleHolderCount,nodeRows:(.nodes|length),roleRows:(.roles|length)}'

curl -sS -o /dev/null -w '%{http_code}\n' \
  https://explorer.ens.dev/resolver/0x9673702a3c850fa1c41d94c908083fc85f59461c/roles
curl -sS -o /dev/null -w '%{http_code}\n' \
  https://explorer.ens.dev/axel.bajigur.eth/roles
```

Captured and freshly rechecked result:

```
{
  "id": "0x9673702a3c850fa1c41d94c908083fc85f59461c",
  "address": "0x9673702a3c850fa1c41d94c908083fc85f59461c",
  "nodeCount": 0,
  "aliasCount": 0,
  "roleHolderCount": 3,
  "nodeRows": 0,
  "roleRows": 5
}
200
200
```

The zero-node result is a current public observation and the exact fixture used by the mounted test. As disclosed above, it is not asserted to be a guaranteed simultaneous post-seed state; root remains visible/default if later indexing adds the child.

### Run the mounted Explorer and exact EOA tests

From the repository root:

```bash
cd apps/portal
pnpm exec vitest run \
  ../../audit_workspace/reports/LEAD-044-evidence/portal/resolver-registry-type-confusion-mounted.test.tsx \
  ../../audit_workspace/reports/LEAD-044-evidence/portal/eoa-transport-exact.test.ts \
  --config ../../audit_workspace/reports/LEAD-044-evidence/portal/vitest.config.ts \
  --reporter verbose
```

Expected and recorded result:

```
Test Files  2 passed (2)
Tests       9 passed (9)
```

The nine passing tests include:

```
an ordinary child roles page navigates into the one-checkbox registry UNREGISTER grant
the legitimate child owner can build only its own leaf pointer to the parent registry
the production route parent validates only address syntax, not the target contract type
the production hasRoles query calls the shared EAC root selector and accepts the registry bit-0 answer
genuine-resolver control: the same one-bit request is intended Set Pubkey only
three-role breadth control encodes registrar, unregister, and set-resolver at once
lets the legitimate child holder build the leaf resolver pointer to its parent registry
hands the production resolver builder request unchanged to viem sendTransaction
matched-account control fails before the wallet when the signer is not the request.from account
```

The expected `EOA transaction from/account mismatch` log belongs to the passing negative control. It is not a suite failure.

The linked Gist files are the canonical clean-checkout versions used for the 9/9 result. Their exact SHA-256 values are listed under **Gist file layout and integrity**, so the copied source can be checked before execution.

Optional native-form corroboration from `apps/portal`:

```bash
pnpm exec vitest run src/features/resolver/components/ChangeResolverForm.test.tsx --reporter verbose
```

Expected and recorded result: **1 file, 5 tests passed**, including valid custom-resolver submission.

### Run the pinned deployed-state fork

Continue from `apps/portal`:

```bash
cd ../../audit_workspace/reports/LEAD-044-evidence/fork-proof
forge test \
  --fork-url https://sepolia.gateway.tenderly.co \
  --fork-block-number 11679987 \
  -vv
```

Expected and recorded result:

```
Ran 4 tests for test/ResolverRegistryRoleConfusionFork.t.sol:ResolverRegistryRoleConfusionForkTest
[PASS] test_childOwnerSeedsInAppRouteThenSetPubkeyGrantBurnsSiblingNFT()
[PASS] test_genuineResolverSameBitOnlyAuthorizesPubkeyRecordWriting()
[PASS] test_registryTargetAcceptsResolverLabeledRequestAndEnablesNamespaceTakeover()
[PASS] test_typeGateControlRejectsRegistryAndAcceptsResolver()
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

For the calldata equality, calls, events, and before/after/recovery trace:

```bash
forge test \
  --match-test test_childOwnerSeedsInAppRouteThenSetPubkeyGrantBurnsSiblingNFT \
  -vvvv
```

The test itself calls `vm.createSelectFork` at block `11,679,987`. It checks block drift and the cross-language calldata hash before any mutation. The `-vvvv` command prints the calldata equality, exact calls, events, before/after state, and recovery trace directly.

### Tool and source versions

| Tool or source                                            | Version or revision                                                                           |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Node.js                                                   | `v22.19.0`                                                                                    |
| pnpm                                                      | `10.27.0`                                                                                     |
| Vitest                                                    | `4.1.5` (`darwin-arm64`, Node `v22.19.0`)                                                     |
| Viem                                                      | `2.52.2`                                                                                      |
| Foundry Forge/Cast                                        | `1.5.1-stable`, commit `b0a9dd9ceda36f63e2326ce530c10e6916f4b8a2`                             |
| forge-std                                                 | `v1.9.7`, commit `77041d2ce690e692d6e03cc812b57d1ddaa4d505`                                   |
| Solidity                                                  | `0.8.25`; optimizer 200; `via_ir = true`; EVM `cancun`                                        |
| Upstream product revision recorded by imported snapshot   | `63772fd872af472ced58b009499355f3430c2a86`                                                    |
| Imported frozen Explorer snapshot                         | `cda79acaad59711b943fc68207ebb3f1d0ff8596`                                                    |
| Competition source anchor used for the clean recorded run | `1c9b47f18fcddd2e864dfe385c4171061c9811ae`                                                    |
| Shared `apps/portal` Git tree                             | `33d5046a405f21a383e5515693511f5f8b17529b`                                                    |
| Deployed implementation artifact source                   | contracts-v2 `f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1`                                       |
| Sepolia state                                             | block `11,679,987`; hash `0x10ddad104deb36c1652602bb52cca52f660ed60f044c79eb20f4c95541bfbeab` |

### Gist file layout and integrity

No attachment or private artifact is required for the PoC. The version-pinned Gist contains the five complete files in one Markdown document. The `original-evidence/` directory created during setup is output-only: the mounted test writes two reproduced request JSON files there and reads no preexisting artifact. Save the five Gist blocks byte-for-byte at these paths, preserving the final newline in each file:

| Path to create                                                                                       | Purpose                                                                                | SHA-256                                                            |
| ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `audit_workspace/reports/LEAD-044-evidence/portal/vitest.config.ts`                                  | Offline mounted-suite aliases and configuration                                        | `6d37a91dfdfc5bc78734fb370a310f10192807ab3eaa8ed440fa85b2727971b0` |
| `audit_workspace/reports/LEAD-044-evidence/portal/resolver-registry-type-confusion-mounted.test.tsx` | Mounted ordinary route, production components, exact one-checkbox intent, and controls | `6c91cf316d2ae291082702a7fe6cac4c22d6c4c1073695d5800fe56c62ecf97e` |
| `audit_workspace/reports/LEAD-044-evidence/portal/eoa-transport-exact.test.ts`                       | Final EOA byte/account/chain handoff                                                   | `b6a55e9b4809133a1017ea067c8ff5234ecece3f42445e9807da3d497d7585c5` |
| `audit_workspace/reports/LEAD-044-evidence/fork-proof/foundry.toml`                                  | Pinned Solidity and Foundry configuration                                              | `50ecd32ee72c18808b1c2ad0f64a2b52d132239ae7d5d58bcdc81fedaa668723` |
| `audit_workspace/reports/LEAD-044-evidence/fork-proof/test/ResolverRegistryRoleConfusionFork.t.sol`  | Pinned deployed post-state, genuine-resolver control, recovery, and positive type gate | `535adcb2e5cf4b28648e35dba445ccb3aa22f9789a20dfa7cbaa122f71e8ffb8` |

After copying the Gist blocks, verify them from the repository root:

```bash
shasum -a 256 \
  audit_workspace/reports/LEAD-044-evidence/portal/vitest.config.ts \
  audit_workspace/reports/LEAD-044-evidence/portal/resolver-registry-type-confusion-mounted.test.tsx \
  audit_workspace/reports/LEAD-044-evidence/portal/eoa-transport-exact.test.ts \
  audit_workspace/reports/LEAD-044-evidence/fork-proof/foundry.toml \
  audit_workspace/reports/LEAD-044-evidence/fork-proof/test/ResolverRegistryRoleConfusionFork.t.sol
```

Expected hashes, in the same order (`sha256sum` is equivalent on GNU systems):

```
6d37a91dfdfc5bc78734fb370a310f10192807ab3eaa8ed440fa85b2727971b0
6c91cf316d2ae291082702a7fe6cac4c22d6c4c1073695d5800fe56c62ecf97e
b6a55e9b4809133a1017ea067c8ff5234ecece3f42445e9807da3d497d7585c5
50ecd32ee72c18808b1c2ad0f64a2b52d132239ae7d5d58bcdc81fedaa668723
535adcb2e5cf4b28648e35dba445ccb3aa22f9789a20dfa7cbaa122f71e8ffb8
```

The Foundry dependency is the public `foundry-rs/forge-std` revision `77041d2ce690e692d6e03cc812b57d1ddaa4d505` (`v1.9.7`), fetched by the setup commands above. The calldata-level SHA-256 and keccak values are independently stated under **Exact operations and byte binding**.
