# 92893 w a low live ens recipient refetch changes the undisclosed owner of a new subname

**Submitted on Sep 13th 2026 at 04:22:57 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92893
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

I found that Portal does not freeze or redisplay the concrete owner address resolved by the Create Subname form. Victim V selects A's known ENS name and reviews address A. While the confirmation remains open, attacker B can use only name-scoped `ROLE_SET_ADDR` to change the live result to B. V's unchanged `Open wallet` action then creates the child for B and grants B every default child role.

## Vulnerability Details

Affected asset: `audit-comp-ens/apps/portal` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The Create Subname form keeps the user-supplied name in `ownerInput` and derives the transaction owner from the live `ownerResolution.address` result:

```tsx
// audit-comp-ens/apps/portal/src/routes/$name/create-subname.tsx:99-110
const [ownerInput, setOwnerInput] = useState(connectedAddress ?? '')
const ownerResolution = useAddressResolution(ownerInput)
const ownerAddress = ownerResolution.address
```

When the user submits the form, `handleSubmit` saves only the parent resolver address before opening the modal. It does not save the reviewed `ownerAddress`, label, registry, or expiry:

```tsx
// audit-comp-ens/apps/portal/src/routes/$name/create-subname.tsx:173-208
const handleSubmit = async (e: FormEvent<HTMLFormElement>) => {
  // ...
  const resolverResult = await ResultAsync.fromPromise(
    getResolver(clientResult.value, { name }),
    () => new Error('Failed to get resolver'),
  )
  // ...
  setResolverAddress(resolverResult.value)
  openTransactionModal()
}
```

Both the modal's estimate and its eventual `onStart` callback continue reading the live `ownerAddress`. The modal identifies only the child name and does not show the resolved owner address:

```tsx
// audit-comp-ens/apps/portal/src/routes/$name/create-subname.tsx:156-170,316-340
createSubname({
  registryAddress: subregistryAddress,
  label: label.trim(),
  owner: ownerAddress,
  resolverAddress,
  // ...
})

<TransactionModal
  transactions={[{
    transactionName: `Create ${label.trim()}.${name}`,
    intent: {
      prepare: ({ walletClient, chainId }) =>
        prepareCreateSubnameTransaction({
          // ...
          owner: ownerAddress,
        }),
    },
    onStart: handleStartTransaction,
  }]}
/>
```

The production query client considers the result stale after one hour, after which an ordinary window-focus event refetches the still-active query:

```ts
// audit-comp-ens/apps/portal/src/utils/queryClient.ts:3-8
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 60,
      retry: false,
    },
  },
})
```

The final builder forwards the refetched address as the static registry owner and grants it `DEFAULT_ROLE_BITMAP`, whose code comment states that the all-`1` value grants every role:

```ts
// audit-comp-ens/apps/portal/src/features/registry/utils/create-subname.helpers.ts:14-27,58-97
export const DEFAULT_ROLE_BITMAP = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)

const writeParams = createSubnameV2WriteParameters(walletClient, {
  registryAddress,
  label,
  owner,
  subregistryAddress,
  resolverAddress,
  roleBitmap,
  expires,
})
```

This omission is inconsistent with the adjacent name-transfer flow, which resolves a recipient and stores that concrete address in `savedParams` before opening its modal at `audit-comp-ens/apps/portal/src/features/transfer/hooks/useTransferName.ts:68-73,106-120`.

The attacker does not need ownership, role-admin authority, registry authority, or resolver-wide authority. Portal's supported Add User flow can grant B only name-scoped `ROLE_SET_ADDR` bit `1`. This lets B change `recipient.eth`'s ETH address while A retains ownership and all administrative authority. The vulnerable flow then amplifies that narrow record capability into ownership and every role over an unrelated child in V's registry.

## Impact Details

I think this is the exact impact I would choose: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient."

V independently selects A's known `recipient.eth` name, never selects B, and Portal first displays A as the concrete owner. B changes only that name's address record through a narrow delegated role. The modal does not disclose the changed address or ask V to confirm B, but V's exact transaction encodes B as the child owner. My PoC shows the official ENSv2 contracts giving B the child token and every default child role.

## Existing Safeguards and Counterarguments

* **"The user entered an ENS name, so its newest address is intended."** Portal displays concrete address A before confirmation, the modal does not show B after the change, and the registry stores one static owner rather than a dynamic reference to the ENS name. The adjacent transfer flow freezes the resolved address.
* **"The attacker cannot see the modal."** Correct. The one-hour wait, focus refetch, and blind timing reduce likelihood and likely concrete severity to Medium, but B controls the A-to-B transition and the resulting transaction deterministically grants B ownership.
* **"The parent can recover."** Recovery requires detection and another unregister/re-register sequence. My PoC proves both the completed authority grant and this limitation. I do not claim permanent bricking or theft of an existing NFT.

## Known Issue Boundary

This is distinct from the published Known Issues. `WEB-1249` concerns replacing an existing child registry and orphaning nested names; my report changes the disclosed recipient of a new child. `QA-01` concerns transferred-name role combinations causing functional errors and preserves separately demonstrated unintended authority. My PoC executes that authority grant through a different confirmation-binding root.

`QA-03` and `EXP-INPUT-005` are display-only inconsistencies, while my report decodes and executes B as the child owner. `SEC-TXM-002` concerns smart-account address derivation.

## Duplicate Check

No Known Issue covers a live recipient refetch after Create Subname confirmation. The closest transfer flow freezes the resolved recipient. `WEB-1249` changes an existing child's registry rather than substituting the owner of a new child. My producer, transaction, authority transition, and fix are distinct.

## Remediation

I recommend freezing an immutable transaction plan at the first form submission. It should contain the normalized label, concrete owner address, registry, resolver, and expiry. Render that address in the confirmation and use only the frozen plan for estimation and submission. If Portal re-resolves the name, abort and require fresh confirmation whenever the address changes. Add an A-to-B regression test for an open modal.

## References

* `audit-comp-ens/apps/portal/src/routes/$name/create-subname.tsx:99-110,156-208,316-340`
* `audit-comp-ens/apps/portal/src/features/registry/utils/create-subname.helpers.ts:14-27,58-97`
* `audit-comp-ens/apps/portal/src/features/transfer/hooks/useTransferName.ts:68-73,106-120`
* `audit-comp-ens/apps/portal/src/utils/queryClient.ts:3-8`

## Link to Proof of Concept

<https://gist.github.com/niafreu/2a254e048ee80ca68a47119e2d04c509>

## Proof of Concept

## Proof of Concept

My six-file local PoC is available at <https://gist.github.com/niafreu/2a254e048ee80ca68a47119e2d04c509>. It uses the pinned Portal route and transaction builder plus official ENSv2 `PermissionedResolver` and `UserRegistry` contracts at commit `48b3e2d39513b9dd32ef1850877a29009bc807b9`. It starts loopback Anvil and blocks every non-loopback fetch.

### Preconditions

1. Install the repository dependencies for the pinned commit.
2. Install Foundry and ensure `anvil` and `forge` are on `PATH`. 3. Place the six PoC files together. Set `ENS_REPO_DIR` to the audit repository and `CONTRACTS_V2_DIR` to the official checkout's `contracts` directory.

### Run

From any directory:

```bash
export ENS_REPO_DIR=/absolute/path/to/audit-comp-ens
export CONTRACTS_V2_DIR=/absolute/path/to/contracts-v2/contracts

PATH=/absolute/path/to/node-22/bin:$PATH \
  "$ENS_REPO_DIR/apps/portal/node_modules/.bin/vitest" run \
  --disableConsoleIntercept \
  --reporter=verbose \
  --config /absolute/path/to/this-poc/vitest.config.ts
```

### What the PoC executes

1. Deploy official `PermissionedResolver` and `UserRegistry` instances on fresh local Anvil through the official `VerifiableFactory`.
2. Make A own `recipient.eth` and resolve it to A.
3. Use the production Portal role builder to grant distinct B only name-scoped `ROLE_SET_ADDR`; decode and assert bitmap `1`, B, and no admin/root role.
4. Render the actual Create Subname route. Enter `child` and `recipient.eth`, and observe that Portal displays A.
5. Open the real transaction modal. It displays `Create child.parent.eth` but no owner address.
6. Have B exercise only the delegated record permission to change the official resolver's ETH address A-to-B.
7. Advance beyond the production one-hour stale interval and trigger an ordinary focus refetch. The underlying form resolves B while the same modal stays open and still discloses no address.
8. Click the real `Open wallet` callback. The production `useCreateSubname -> createSubname -> transactionManager.startTransaction -> EOA signer` path submits the request to local Anvil. Capture the exact intent at `startTransaction` and decode it as `register("child", B, ..., DEFAULT_ROLE_BITMAP, ...)` without rebuilding it.
9. Assert that the transaction executed by the production signer makes B owner and gives B the full default role bitmap.
10. Have V exercise retained parent authority to unregister the B-owned child, then pass frozen A through the production builder to re-register it. Assert the new generation is owned by A and gives B no child role.

### Observed output

```
reviewedRecipient: 0x70997970C51812dc3A010c7d01b50e0d17dc79C8
attackerRole: recipient.eth / ROLE_SET_ADDR only
attackerNameRoleBitmapBeforeExploit: 1
attackerRootRoleBitmapBeforeExploit: 0
attackerMutation: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
modalDisclosureAfterRefetch: Create child.parent.eth (no address)
decodedChildOwner: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
executedChildOwner: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
fixedSnapshotControlOwner: 0x70997970C51812dc3A010c7d01b50e0d17dc79C8
recoveredChildOwner: 0x70997970C51812dc3A010c7d01b50e0d17dc79C8

Test Files  1 passed (1)
Tests       1 passed (1)
```

My positive path proves the UI-to-calldata-to-official-contract transition. The production hook, helper, transaction manager, EOA signer, and local wallet execute the same captured intent. I do not reconstruct a second exploit request.

I keep three non-semantic harness seams explicit. The local address-resolution adapter reads `addr(recipientNode)` directly from the deployed resolver, the modal transaction-list subscription is empty because I observe the singleton transaction manager directly, and post-success indexer polling is disabled. I do not stage any resolver value, calldata field, role, owner, or post-state.

My unregister/re-register control proves V's recovery. It also proves that freezing the reviewed concrete recipient prevents B from retaining the child.
