# 92559 w a critical explorer reuses a former owner ens resolution and encodes the wrong recipient in an ensv2 nft transfer

**Submitted on Sep 10th 2026 at 16:38:50 UTC by @Parthx108 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92559
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

**Affected in-scope asset:** [Explorer app files (`apps/portal`)](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal)

**Submitted severity:** Critical

**PoC classification:** Websites and Applications (Web2), with a demonstrated on-chain NFT-ownership consequence

**Selected in-scope impact:**

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves

Explorer's transfer form accepts either an address or an ENS name as the recipient. For an ENS input, `useAddressResolution` obtains an address through the global QueryClient. The form retains the typed name for display, but the transaction path passes only `resolution.address` into `startTransfer`; `SavedParams` has no ENS-name field. The `resolved-address` query inherits the application's one-hour `staleTime`, and no uncached current-resolution check occurs before recipient capture or `safeTransferFrom` encoding.

For a valid ENS name without an address record, Explorer's application-specific fallback uses the name's current owner as the recipient address. After a normal handoff of that name from the attacker to its new owner, a fresh fallback returns the new owner. If the victim is using the ENS input to address that post-handoff owner but leaves the pre-handoff form mounted, Explorer can continue to supply and later encode the attacker's former-owner address.

The vulnerable root is first-party Explorer transaction construction: the application loses the name/address association at the transaction boundary and performs no current-resolution check before encoding an irreversible transfer. TanStack Query and the ENSv2 Registry behave as configured; the Registry is used only to prove the resulting ownership transition.

## Brief/Intro

An ENS name is a human-readable identity such as `recipient.eth`. Its address can change. When it has no address record, Explorer applies its own fallback and uses the current owner as the recipient address.

An ERC-1155 transaction must eventually contain a fixed recipient address; this report does not claim that already-approved calldata should follow later ENS changes. The defect occurs earlier: Explorer creates the transaction intent from a cached name/address association, stops carrying the entered name in the transaction parameters, and neither rechecks the association nor repeats it in Explorer's final confirmation. After an ordinary handoff of the recipient name, Explorer can therefore encode the former owner even though a current lookup returns the new owner.

## Vulnerability Details

**Root cause: the transaction path loses the name/address association and performs no current-resolution check**

Explorer's singleton QueryClient treats queries as fresh for one hour by default:

```typescript
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 60, // 1 hour
      retry: false,
    },
  },
})
```

The security-sensitive `resolved-address` query supplies a key and query function but no `staleTime` override, so it inherits that default:

```typescript
const getResolvedAddressQueryKey = createQueryKey<
  'resolved-address',
  GetResolvedAddressParams
>('resolved-address')

export const getResolvedAddressQueryOptions = (
  params: GetResolvedAddressParams,
) =>
  resultQueryOptions({
    queryKey: getResolvedAddressQueryKey(params),
    queryFn: ({ queryKey: [, queryParams] }) => getResolvedAddress(queryParams),
  })
```

`useAddressResolution` trims and debounces the input, then mounts that query. The exact trimmed/debounced string is part of the query key. For a valid ENS name without an address record, `resolveAddressOrName` falls back to the mutable current ENS owner:

```typescript
const resolved = await getEnsAddress(client, {
  name: nameOrAddress,
  universalResolverAddress,
})

let resolvedAddress = resolved
if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) {
    resolvedAddress = ownerResult.value?.owner ?? null
  }
}
```

The transfer form then severs the relationship between the name and its resolved address. It extracts only `resolution.address` and passes only that address to `startTransfer`:

```typescript
const resolution = useAddressResolution(recipientInput)
const { address: recipient, isResolving } = resolution

const runTransfer = () => {
  if (!recipient || !canStart) return
  startTransfer({ recipient, options: effectiveOptions })
}
```

`useTransferName` stores that address in `savedParams`. Every later transaction step uses the retained address; there is no recipient-name field to re-resolve:

```typescript
type SavedParams = {
  readonly recipient: Address
  readonly tokenId: bigint
  readonly options: TransferOptions
}

onSuccess: (params) => {
  startedStepsRef.current = new Set()
  setSavedParams(params)
  openModal()
},

const { recipient, tokenId, options } = savedParams

.with('transfer-token', () =>
  transferToken({ ...common, registryAddress, tokenId, recipient, id }),
)
```

Finally, `transferToken` encodes the retained address as the ERC-1155 recipient:

```typescript
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

The pre-modal form displays the full resolved hexadecimal address and an irreversible-transfer warning; this is a meaningful mitigation. After the victim clicks the transfer button, however, Explorer's transaction modal shows the target name and generic step titles such as `Transfer name - <target name>`, but neither the typed recipient name nor the recipient address. No application stage re-resolves the name or asks the victim to review a changed name/address pair before submission.

**Exploit sequence**

1. The victim owns a transferable first-class ENSv2 target name. Separately, the attacker owns an ordinary, valid, normalized ENSv2 recipient-identity name whose handoff to a new owner is expected.
2. The recipient-identity name has no exact resolver or address record. Before that handoff, the victim enters the exact name in Explorer's transfer form in preparation for sending the target to the identity's post-handoff owner. Explorer falls back to the current owner, displays the attacker's full address with a warning, and caches it.
3. The attacker transfers the recipient-identity name normally to the new owner. This handoff is independent of the victim's target. The new owner becomes current owner, while the former owner's name-resource Registry role bitmap becomes zero.
4. A current owner-fallback lookup now returns the new owner. The victim keeps the original transfer form mounted and no invalidation, reload, or refetch occurs before the click, so the still-fresh `resolved-address` query supplies the cached former-owner address without another resolution execution.
5. The victim clicks **Transfer name**. `SendNameForm` passes only the cached raw address; `useTransferName` captures it in `savedParams` and opens the transaction flow.
6. A later query refetch can repair the displayed query result, but it cannot rewrite the primitive address already captured in `savedParams`.
7. The victim signs Explorer's prompted transfer steps. The sender, chain, Registry, target token, amount, and function are correct, but `safeTransferFrom` encodes the attacker/former owner as recipient.
8. The transaction succeeds and the attacker becomes owner of the victim's target name. Before this Explorer-created transaction, the attacker's direct attempt to transfer the target is rejected.

**Preconditions and bounded feasibility**

* The victim must use the ENS name to address its post-handoff owner, resolve it before the handoff, keep the same form mounted, avoid an invalidation/refetch/reload before clicking, and sign the resulting transaction.
* The attacker cannot remotely prime the victim's cache, and the proof does not assume that the attacker can observe the victim's form state. The required handoff timing is therefore coordinated or opportunistic, not remotely forceable.
* Explorer visibly displays the full former-owner address and warns the victim to verify it. Independent address verification prevents the loss.
* A reload, new tab, or first lookup after the handoff is safe. The cache is in-memory and is not shared across tabs.
* During the first hour, the cached result is fresh and stale-only refetch triggers do not replace it. At one hour it merely becomes stale; elapsed time alone does not execute a refetch, so the value can remain until a refetch trigger occurs. When unobserved, the query is eligible for TanStack Query's browser-default garbage collection after five minutes.
* Raw `0x` inputs do not use ENS resolution and are not affected.

These prerequisites make the route timing-dependent and user-assisted. They do not reduce it to a theoretical UI issue: the accepted Critical impact expressly covers a wrong recipient on a transaction that the user then approves, and the PoC completes the resulting ownership transition. Explorer constructs a victim-authorized transaction that transfers the victim's NFT to an account that cannot transfer it independently.

**Recommended remediation**

Define and enforce one recipient semantic consistently:

* If an entered ENS name represents a dynamic identity, preserve the normalized name with the reviewed address, perform an uncached lookup when **Transfer name** is clicked, and abort for renewed review if the address changed.
* If Explorer intentionally treats the first resolution as a fixed-address snapshot, label that behavior explicitly and repeat the exact fixed address in Explorer's final confirmation.

In either design, the final confirmation should display the normalized recipient name when applicable, full recipient address, sender, chain, target Registry, target name/token, and the exact recipient encoded in calldata. It must render from the same immutable intent that will be submitted.

## Impact Details

The PoC proves a persistent wrong-recipient transfer of a victim-owned ENSv2 ERC-1155 name. Before the victim's transaction, the victim owns the target and the former owner of the separate recipient-identity name cannot transfer it independently. Explorer's production transfer helper then submits successful calldata decoding to:

```
safeTransferFrom(victim, formerRecipientNameOwner, targetTokenId, 1, 0x)
```

After that transaction, the Registry's `getOwner(targetResource)` returns the attacker. At the same point, an explicitly refreshed control returns the recipient name's new owner and produces different current-owner calldata.

This maps directly to the program's Critical impact **“Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.”** That category expressly includes showing the wrong recipient on a transaction that the user then approves. The persistent ownership transition proves the consequence; this report does not claim wallet/session compromise, an unprompted Registry authorization bypass, a real external-user loss, or any particular market value.

The result is not caused by a wrong chain, wrong sender, substituted Registry, wrong token ID, smart-account divergence, browser defect, or Registry authorization bypass. Those values and checks are correct. Explorer alone supplies the obsolete recipient argument.

No disclosed known issue describes this root:

* QA-01/WEB-1257 concern actions that fail after a transfer; the demonstrated transfer here succeeds to the wrong owner.
* EXP-INPUT-005 concerns deceptive or non-normalized input; this proof uses ordinary valid normalized `.eth` names.
* SEC-TXM-002 concerns a cached Hybrid Custody Account address; no smart account participates here.
* EXP-4337-002 and EXP-4337-003 concern chain and signer validation; both are correct here.

This root is also separate from a stale profile-record display after Explorer changes a resolver. That issue requires resolver-sensitive record invalidation or re-keying after the application's own mutation. This report instead arises from Explorer's failure to reconcile a still-displayed ENS recipient identity with its retained address at transaction preparation after an external ownership change. The shared one-hour default is an amplifier, not the complete root or fix.

The ENS contracts are out of scope and are not alleged to be vulnerable. They are used only as an integration witness showing that Explorer's wrong recipient reaches a persistent, successful ownership transition.

## References

* [ENS competition scope: Explorer is in scope; the wrong-recipient transaction impact is Critical; ENS contracts are out of scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Frozen global QueryClient configuration](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/queryClient.ts#L1-L10)
* [Frozen `resolved-address` query without a freshness override](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/queries/getResolvedAddress.ts#L13-L45)
* [Frozen address-record then owner-fallback policy](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/roles/helpers/addUser.handlers.ts#L11-L40)
* [Frozen mounted address-resolution query](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/hooks/useAddressResolution.ts#L48-L63)
* [Frozen first-class ENSv2 transfer route and authorization gate](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/%24name/ownership/transfer.tsx#L133-L197)
* [Frozen `SendNameForm` raw-recipient handoff and warning](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L68-L135)
* [Frozen full-address recipient preview](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L228-L281)
* [Frozen recipient capture and reuse by `useTransferName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L37-L45)
* [Frozen pending-intent storage and modal opening](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L121)
* [Frozen transfer step consuming the retained recipient](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L124-L180)
* [Frozen final ERC-1155 calldata encoding](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L83)
* [Frozen transaction overview renders generic step titles](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transaction-manager/components/TransactionsOverviewContent.tsx#L51-L117)
* [TanStack Query important defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)
* [Immunefi step-by-step PoC requirements for audit competitions](https://immunefisupport.zendesk.com/hc/en-us/articles/33260632501777-Step-by-step-POCs-for-Audit-Competitions)
* [Immunefi Web2 PoC guidelines](https://immunefisupport.zendesk.com/hc/en-us/articles/18722746161553-Web2-PoC-guidelines)

## Link to Proof of Concept

<https://gist.github.com/savyasachi-bugbounty/fbc15aa9ff3d2104e5676385bee495d8/c400b64af0bbd831d23542cb0c2b868e2d76cb55>

## Proof of Concept

## Proof of Concept

**Code Gist (secret/unlisted, immutable revision):** [Critical-04 runnable PoC](https://gist.github.com/savyasachi-bugbounty/fbc15aa9ff3d2104e5676385bee495d8/c400b64af0bbd831d23542cb0c2b868e2d76cb55)

**Separate setup Gist (secret/unlisted, immutable revision):** [Critical-04 setup guide](https://gist.github.com/savyasachi-bugbounty/893884b62b91035caba29c8c856bec93/feb820f744ad47d48abfd06d4446500d0219cd33)

The code Gist contains exactly four files:

* `stale-owner-fallback-transfer.poc.ts` — private-fork ownership-transfer proof;
* `cache-lifecycle.poc.ts` — browser-mode TanStack lifecycle control;
* `run-primary.sh` — one-command environment, fork, typecheck, and execution gate; and
* `tsconfig.json` — portable TypeScript configuration for the frozen checkout.

The setup Gist contains the exact checkout, dependency, directory-layout, execution, safety, and expected-output instructions.

This is a Websites and Applications report because the vulnerable root is Explorer's treatment of the recipient identity. The executable proof uses a private Ethereum Sepolia fork only to demonstrate the persistent consequence. No transaction is sent to public Sepolia or mainnet.

**Proof fidelity and explicit boundary**

The fork proof imports Explorer's frozen QueryClient, exact `resolved-address` query options, transfer-plan builder, resolver-detach helper, final `transferToken` helper, and transaction manager. It obtains the Registry from the frozen checkout's pinned ENSJS Sepolia contract map and imports Portal's configured universal resolver. It also checks the frozen source at runtime for the owner fallback, `SendNameForm` raw-address handoff, `useTransferName` capture/reuse, route gate, modal labels, and ERC-1155 sink.

Because production `getEnsOwner` closes over the browser's fixed Wagmi client, a fork-only name cannot exercise that owner read through the public client embedded in the browser configuration. The harness therefore injects the private-fork client into an equivalent production-shaped policy: it calls `getEnsAddress`, asserts that the resolverless identity returns exactly `null` rather than throwing, and then calls the exported `resolveEnsOwner` with the same client. The combined path mirrors production's error handling.

The proof does not mount or click the React `SendNameForm`, execute the `useTransferName` hook, render the modal, or automate a wallet UI. It demonstrates the cache semantics and executes the exact final helper and on-chain sink; frozen-source assertions bind the intervening form/hook/modal wiring rather than claiming a browser end-to-end execution. The harness models the mounted field with an active QueryObserver and models `savedParams.recipient` capture with a local immutable `savedRecipient`.

**Step-by-step reproduction**

1. Verify at runtime that the frozen source still contains the one-hour global policy, inherited `resolved-address` query, owner fallback, raw-recipient handoff, captured-recipient reuse, generic modal labels, authorized transfer route, and `safeTransferFrom` sink. Assert that the private fork starts at Sepolia block `11556186` with chain ID `11155111`.
2. Create a victim-owned first-class ENSv2 target and an attacker-owned recipient-identity name. Detach the identity's resolver with Explorer's helper, prove the resolver is zero, and prove `getEnsAddress` returns exactly `null`.
3. Spread the exact production query options into Explorer's QueryClient, replace only the fork-bound query function, mount an active observer, and prove the owner fallback caches the attacker with one resolution-query execution.
4. Transfer the recipient-identity name normally from the attacker to the new owner (`intendedRecipient` in the harness) using Explorer's exact transfer helper. Prove the current owner and a fresh fallback are the new owner, and the former owner's name-resource Registry role bitmap is zero.
5. Fetch the same fresh production query and prove it makes no new resolution-query execution and returns the former owner. Capture that value as the pending recipient. Explicitly refetch as a negative control before any target transaction: the live query repairs to the new owner, but the already-captured value remains the former owner.
6. Prove the victim owns the target and holds the exact transfer role required by the frozen route. Simulate a direct attacker `safeTransferFrom` and require it to revert.
7. Call `buildTransferPlan` with the explicit transfer options matching the observed fixture, execute its resolver-detach step, and call Explorer's exact `transferToken` helper with the captured former-owner address. The proof does not claim to execute the option-derivation hook.
8. Require a successful receipt, decode the submitted calldata, prove its sender, Registry, token ID, amount, and selector are correct but its recipient is the attacker, and prove the attacker is now target owner. Construct an explicitly refreshed control and require different current-owner calldata. The JSON `verdict` must equal `CONFIRMED`, and the runner must end with `Critical-04 primary PoC: PASS`.

The lifecycle control separately verifies the production one-hour setting, the browser's five-minute inactive-query garbage-collection default, absence of an automatic read merely because an active query becomes stale, and persistence of an already-captured value after an explicit refetch. It uses a short equivalent stale interval so the semantic control completes immediately; it does not pretend to wait one hour.

Successful execution exits with status `0`. The complete command and expected output are in the separate setup Gist.

## Run the PoC

Enter the historical-state Sepolia RPC URL without writing it into shell history, then execute the one-command runner:

```bash
set -euo pipefail
printf 'Historical-state Sepolia RPC URL: ' >&2
IFS= read -r -s SEPOLIA_RPC_URL
printf '\n' >&2
export SEPOLIA_RPC_URL
bash AuditFolder/Critical-04-PoC-Folder/run-primary.sh
unset SEPOLIA_RPC_URL
```

The runner uses local port `18546`. If that port is occupied, set `POC_ANVIL_PORT` to a different unused port before running it.

An `IndexedDB unavailable` warning is expected in the headless Node process. The transaction manager falls back to in-memory storage; this does not change transaction construction, receipts, or ownership assertions.

## Expected result

Successful execution exits with status `0`, reports two `CONFIRMED` verdicts, prints all eight primary stages, and ends with:

```
Critical-04 primary PoC: PASS
```

The lifecycle JSON must include values equivalent to:

```
productionStaleTimeMs = 3600000
browserInactiveGcTimeMs = 300000
activeQueryAfterStaleTime.cachedValue = former-owner
activeQueryAfterStaleTime.currentExternalValue = current-owner
activeQueryAfterStaleTime.automaticRefetches = 0
explicitRefetch.previouslyCapturedValue = former-owner
"verdict": "CONFIRMED"
```

The private-fork JSON must establish:

```
initialBlock = 11556186
chainId = 11155111
formerOwnerRolesAfterHandoff = 0
currentOwner = intended recipient
freshFallbackAfterHandoff = intended recipient
cachedAtTransferReview = former owner
resolutionQueryExecutionsBeforeExplicitRefetch = 1
formerOwnerDirectTargetTransferBlocked = true
transferPlan = ["detach-resolver", "transfer-token"]
transactionTarget = configured ENSv2 Registry
decodedFrom = victim
decodedTo = attacker/former identity owner
decodedAmount = 1
receiptStatus = success
ownerAfter = attacker/former identity owner
freshControlCalldataDiffers = true
"verdict": "CONFIRMED"
```

Names, token IDs, block numbers after the initial fork point, and local transaction hashes are created per run and are not expected to match earlier executions.
