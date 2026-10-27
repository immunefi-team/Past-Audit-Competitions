# 92963 w a critical cross name profile state makes a description only save set an attacker address on another ens name

**Submitted on Sep 13th 2026 at 14:12:33 UTC by @xylz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92963
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Manager profile editor can remain open while the route changes from one ENS name to another. Its XState actor keeps the record snapshot from the name that opened the editor, while the TanStack form and save hook can receive newer records and the current route name. The save path does not bind those values to one name. It can therefore diff records from one name and encode the result for another.

The end-to-end proof gives the attacker control only over a gifted SOURCE name. The attacker has never owned, managed, or held any role on the victim's TARGET name. After an ordinary browser reconnect updates the open SOURCE form, the victim changes only Description, presses Back once, sees TARGET in the editor header, and clicks Save Profile. The address field is never opened or mounted. Manager nevertheless asks the victim wallet to send one transaction containing both the intended Description update and `setAddr(TARGET, 60, attacker)`.

The transaction succeeds against TARGET's real PermissionedResolver. TARGET's ETH resolution changes from the victim to the attacker, Manager displays and copies the attacker as TARGET's “Main receiving address,” and an independent account sends 0.001 ETH to that copied value. The owner later repairs the address, but the completed payment remains in the attacker's balance.

## Vulnerability Details

The proof uses ENS Apps revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`.

### 1. The actor snapshot and form values have different update rules

`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.tsx:13-41` creates one actor from `records` and separately supplies `records` to TanStack Form as `defaultValues`:

```tsx
const dialogActor = useActorRef(editProfileDialogMachine, {
  input: { records },
})

const form = useAppForm({
  defaultValues: records,
})
```

The actor's `savedRecords` changes only when the dialog receives `OPEN` or after a successful save. A prop change while the dialog remains open does not update it. This is visible in `EditProfileDialog.machine.ts:231-289`.

The form behaves differently. The pinned React Form adapter updates its options on every render, and Form Core replaces values when `defaultValues` changes while the form is untouched. That allows a normal query update to change hidden form values without changing the actor's saved snapshot.

### 2. Reconnect supplies changed SOURCE records to the untouched form

`apps/manager/src/features/profile/components/view/ProfileView.tsx:80-96` queries profile records for the live route name. `apps/manager/src/features/profile/service/profileRecords.ts:82-169` uses the indexer to discover record keys, then reads the resolver and record values through the chain client. `apps/manager/src/router.tsx:28-36` configures query `staleTime` as zero, and the pinned TanStack Query version normally refetches stale active queries on reconnect.

In the proof, SOURCE initially points to the same PermissionedResolver used by TARGET and has no ETH record there. The victim opens SOURCE's editor and remains on General. While the browser is offline, the attacker changes only SOURCE's registry pointer back to a separate, preseeded resolver whose ETH record is the attacker. The browser then reconnects. No synthetic DOM event, query invalidation, or direct refetch call is used.

Normal reconnect processing obtains the changed SOURCE resolver and ETH record. Because the victim has not touched the form, the hidden attacker ETH value becomes part of the form state. The actor still holds SOURCE's opening snapshot: the TARGET resolver and no ETH value.

### 3. Back navigation changes the visible name without remounting the editor

`apps/manager/src/routes/$name/index.tsx:165-173` reads the route parameter and renders the same unkeyed `ProfileView` component. The prop chain through `ProfileActions.tsx:107-120` and `ProfileEditAction.tsx:19-50` likewise has no semantic key for the name.

After the victim edits only Description on SOURCE, one browser Back changes the route and editor header to TARGET while preserving the dirty editor. `EditProfileDialogHeader.tsx:60-82` displays the current name next to the Save Profile button, so TARGET is the name the victim sees when saving.

### 4. Save combines SOURCE record state with TARGET identity

`EditProfileDialog.machine.ts:127-194` computes `before` from the actor's retained `savedRecords`, computes `after` from the current form, and takes the name from the live save dependencies:

```ts
const before = transformToServiceFormat(savedRecords)
const after = transformToServiceFormat(currentRecords)

return {
  kind: 'update',
  currentRecords,
  params: {
    name: deps.name,
    before,
    after,
    resolverAddress: savedRecords.resolverAddress,
  },
}
```

`useEditProfileDialogSave.ts:357-433` supplies the current route's `name` when it sends `SAVE_REQUESTED`. `profileRecordTransactions.ts:158-205` diffs the mismatched snapshots, and `profileRecordTransactions.ts:372-456` encodes every difference for that live name and sends it to the saved resolver.

The SOURCE opening resolver deliberately equals TARGET's actual PermissionedResolver. The live write-access check succeeds for TARGET, so this is a direct owner-EOA update with no resolver-setup warning. The exact wallet request captured before authorization contains:

```
setText(namehash(TARGET), "description", "Updated public profile")
setAddr(namehash(TARGET), 60, attacker)
```

The second call comes entirely from the hidden cross-name state. The victim did not type, view, or interact with that address.

### 5. The Manager review surface does not disclose the added address write

The editor remains on General throughout the sequence. Addresses is a separate tab in `EditProfileDialogTabs.tsx:171-197`, and the test confirms that no address field was mounted before or after reconnect. `EditProfileDialogBody.tsx:65-126` uses the diff to decide whether saving is enabled but does not render the individual record changes. The visible header identifies TARGET and offers the Save Profile button.

The proof captures the EIP-1193 `eth_sendTransaction` request before its test wallet authorizes it, decodes the two inner calls, and then mines that exact target and calldata. It does not make a claim about how a particular browser-extension wallet may decode raw calldata.

## Impact Details

### Incorrect payment address and completed transfer

Before the affected save, both ENSjs and the deployed `UniversalResolverV2` return the victim for TARGET. After the exact Manager-generated transaction is mined, both return the attacker while TARGET remains on its original resolver and under the victim's ownership.

Manager's public TARGET profile then displays the attacker under “Main receiving address.” A separate payer account copies that value from Manager and sends 0.001 ETH to it. The receipt succeeds, and the attacker's balance increases by exactly 0.001 ETH. A later victim repair restores future TARGET resolution to the victim but does not reverse the completed payment.

This matches the program's Critical impact for causing the application to display or pre-fill an incorrect address, name, or transaction detail so that a user sends funds or submits a transaction to an unintended recipient. The program gives resolving or rendering an ENS name to an attacker-controlled address as an explicit example.

### Connected wallet receives modified transaction arguments

The victim's only visible record edit is Description. Manager adds an ETH address write for another name to the same wallet request. This separately matches the Critical category for malicious interactions with an already-connected wallet, including modifying transaction arguments or parameters.

### Authorization boundary

The attacker has zero TARGET registry roles before and after the exploit and cannot write TARGET's resolver directly. TARGET was created for the victim and remains owned by the victim. The attacker's only authority is a SOURCE-specific `ROLE_SET_RESOLVER` retained after SOURCE was gifted to the victim without a victim signature. That authority lets the attacker change SOURCE's pointer; it does not authorize any TARGET record.

### Conditions reproduced by the proof

* The victim opens the gifted SOURCE profile editor, and SOURCE initially points to TARGET's PermissionedResolver.
* The attacker changes SOURCE's resolver pointer while the browser is offline, before it reconnects.
* The victim changes Description, uses Back once to return to a previously viewed TARGET, and approves the direct profile-update transaction.
* The independent loss occurs when another user later relies on TARGET's Manager-displayed receiving address and pays it.
* The local proof uses Anvil, the repository's ordinary connected Manager fixture, a headless EIP-1193 wallet, and mocked indexer responses only for private fixture-name discovery. Resolver reads, role checks, encoded calldata, receipts, Universal Resolution, Manager display/copy behavior, payment, and repair are chain-backed.

These conditions affect opportunity. They do not change the transaction or outcome demonstrated: a Description-only edit produces an undisclosed attacker-address write for the TARGET shown in the editor, and Manager then presents that address for payment.

## Recommended Fix

1. Key or remount the profile editor, actor, and form by a semantic identity containing chain ID and normalized name or node.
2. Keep the actor snapshot and form defaults in one name-bound state transition. Do not let a query update replace form values without updating the corresponding saved snapshot.
3. Store the name, node, owner, resolver, and record snapshot together. Reject Save if any part differs from the identity that opened the editor.
4. Immediately before encoding, re-read the active name and resolver, recompute the diff from one internally consistent snapshot, and reject a route change during an open edit.
5. Show the name, resolver, and every text or address-record change in the in-app review before asking the wallet to sign.

## References

ENS Apps frozen revision:

`https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596`

TanStack React Form 1.27.1 option update:

`https://github.com/TanStack/form/blob/%40tanstack%2Freact-form%401.27.1/packages/react-form/src/useForm.tsx#L258-L266`

TanStack Form Core 1.27.1 default-value update:

`https://github.com/TanStack/form/blob/%40tanstack%2Fform-core%401.27.1/packages/form-core/src/FormApi.ts#L1446-L1469`

TanStack Query 5.90.12 reconnect handling:

`https://github.com/TanStack/query/blob/%40tanstack%2Fquery-core%405.90.12/packages/query-core/src/queryClient.ts#L90-L94`

`https://github.com/TanStack/query/blob/%40tanstack%2Fquery-core%405.90.12/packages/query-core/src/queryClient.ts#L603-L607`

`https://github.com/TanStack/query/blob/%40tanstack%2Fquery-core%405.90.12/packages/query-core/src/queryObserver.ts#L771-L785`

## Link to Proof of Concept

<https://gist.github.com/Xylzlol/93a747ef34d9ef4739daea3de7ec1d96>

## Proof of Concept

## Proof of Concept

The focused test runs the frozen Manager app against local Anvil contracts. It uses the repository's normal `connectedPage` fixture and a real Chrome browser. The only browser-network control is Playwright's `BrowserContext.setOffline`; the test does not dispatch online/offline DOM events, invalidate a query, or call a query's refetch method.

### Step-by-step reproduction

1. Create TARGET directly for the victim with a dedicated PermissionedResolver. Set TARGET's ETH record to the victim. Verify the attacker has zero TARGET registry roles and cannot write TARGET records.
2. Create SOURCE under a controlled gifter account. Deploy a separate SOURCE resolver, seed its ETH record with the attacker, and give the attacker SOURCE-only `ROLE_SET_RESOLVER`.
3. Point SOURCE temporarily to TARGET's PermissionedResolver, where SOURCE has no ETH record, and gift SOURCE directly to the victim. The gift requires no victim signature and preserves the attacker's SOURCE-only role.
4. In Manager, visit TARGET first, then SOURCE. Open SOURCE's Edit Profile dialog and remain on General. Confirm the address field count is zero and Save Profile is disabled.
5. Put the browser offline with `page.context().setOffline(true)`. Confirm the native offline event and `navigator.onLine === false`.
6. While the browser is offline, use the attacker's SOURCE role to restore only SOURCE's registry pointer to the preseeded resolver. Do not change TARGET.
7. Bring the browser online with `page.context().setOffline(false)`. Confirm the native online event. Wait for normal reconnect processing to update the active SOURCE query to the attacker resolver and attacker ETH value. No direct refetch API is called.
8. Confirm Save Profile becomes enabled even though no visible field changed and the address field remains unmounted. Then change only Description to `Updated public profile`.
9. Use browser Back once. Confirm the route and editor header now show TARGET while the same dirty editor remains open.
10. Click Save Profile. Before authorizing the wallet request, capture and decode its exact calldata. Confirm it targets TARGET's PermissionedResolver and contains both `setText(TARGET, description, Updated public profile)` and hidden `setAddr(TARGET, 60, attacker)`.
11. Authorize and mine that exact request. Confirm TARGET still uses its original resolver, but ENSjs and raw Universal Resolution now return the attacker. Open TARGET as an independent viewer and copy the attacker from Manager's “Main receiving address.” Send 0.001 ETH to that copied value and confirm the attacker balance rises by exactly 0.001 ETH.
12. Repair TARGET from the victim account. Confirm future resolution returns to the victim while the earlier payment remains in the attacker's balance.

### Revisions and prerequisites

Repository:

```
https://github.com/immunefi-team/audit-comp-ens
```

Revision:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Start the repository's pinned E2E infrastructure and a local Manager build as described in `e2e/README.md`. The recorded run used:

```
ANVIL_RPC_URL=http://127.0.0.1:8545
MANAGER_APP_URL=http://127.0.0.1:3102
chainId=11155111
Chrome channel
```

From the repository root, run the focused test:

```powershell
$env:ANVIL_RPC_URL = 'http://127.0.0.1:8545'
$env:MANAGER_APP_URL = 'http://127.0.0.1:3102'
pnpm exec playwright test --config=e2e/projects/manager/playwright.profile-route-trigger-audit.config.ts --grep "offline reconnect refetch plus one Back redirects target payment" --trace on
```

Recorded result:

```
1 passed (50.5s)
```

Then run the independent evidence verifier:

```powershell
powershell -ExecutionPolicy Bypass -File .\ENS_PROFILE_ROUTE_RECONNECT_VERIFY.ps1
```

Expected completion token:

```
ENS_PROFILE_ROUTE_RECONNECT_CRITICAL_EVIDENCE_VERIFIED
```

### Recorded identities and receipts

```
SOURCE: reconnect-empty-source-1789305614.eth
TARGET: reconnect-payment-target-1789305605.eth
victim:  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
attacker: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
payer:   0x90F79bf6EB2c4f870365E785982E1f101E93b906

TARGET PermissionedResolver:
0x15B7434c097a4B58fDd126795A140a4176b8d301

Attacker's separate SOURCE resolver:
0x1d63ff1A7C9CafEf21E90118D0c13C570818a2C5

SOURCE gift:
0x265079c451a1a0e3795df1fc9bc774a4be49dfe9d63e47d895138f3d9c48c816

SOURCE pointer restore:
0xed7e7219180ade9f5596be1af9b3349af29d370125a03bb6c030edd4aaaf0d16

Affected TARGET write:
0x2295abc3d4896e249c8dbd9cc45f622fea2261926bc37fa2fd0e914e784cbb1e
block 11694879, status success

Independent payment:
0xf7c5282b0924227b4e5f57affabc85347b9078288d49ad592fa2165e8a7a0c5a
value 1000000000000000 wei, status success

Victim repair:
0xe2dd22ce6b103ecf2e53689e490f7a70e537d7696abdab3af222a732eeaddfc6
status success
```

### Exact wallet request

The test captures one `eth_sendTransaction` request before wallet authorization. Its outer destination is TARGET's PermissionedResolver. The mined transaction has the same sender, destination, and calldata as the captured request.

Decoded inner calls:

```
setText(
  0x1323d1552d9db5fcbcba300f336e300dd88565b2d9ef14537d45e453e7efc382,
  "description",
  "Updated public profile"
)

setAddr(
  0x1323d1552d9db5fcbcba300f336e300dd88565b2d9ef14537d45e453e7efc382,
  60,
  0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc
)
```

The node in both calls is TARGET. Coin type 60 is ETH. The visible edit accounts for the first call. The second call is the hidden attacker-address write.

### Before, exploit, and repair controls

The evidence records both ENSjs and raw `UniversalResolverV2` results:

```
before save:        TARGET -> victim
after affected save: TARGET -> attacker
after repair:       TARGET -> victim
```

Across all three phases, the resolver used for TARGET remains:

```
0x15B7434c097a4B58fDd126795A140a4176b8d301
```

The attacker has no TARGET authority:

```
registry roles before: 0
registry roles after:  0
direct TARGET resolver write before: rejected
direct TARGET resolver write after:  rejected
```

The independent payer reads and copies the attacker address from Manager before paying. The recorded balance check is exact:

```
payment value:             1000000000000000 wei
attacker balance increase: 1000000000000000 wei
balance after repair:      unchanged from post-payment balance
```

### Reconnect chronology and clock reconciliation

The repository's E2E time fixture installs the browser clock from the current Anvil block time. Browser event timestamps and Node chronology timestamps are therefore different clock domains. The final evidence samples browser `Date.now()` between Node timestamps and records the offset.

```
Node before sample:        1789305631389
Browser sample:            1789314653589
Node after sample:         1789305631393
Node midpoint:             1789305631391
Round trip:                4 ms
Browser-to-Node offset:    -9022198 ms

normalized offline request: 1789305631402
normalized offline event:   1789305631403
normalized online request:  1789305632484
normalized online event:    1789305632489
```

The verifier recomputes the midpoint and offset, checks both event normalizations, enforces request-before-event ordering, and requires each event to follow its request within five seconds.

### Evidence boundaries

* The contracts and transactions run on local Anvil. No mainnet or public-testnet transaction is sent.
* The E2E uses the repository's ordinary connected Manager fixture, not the profile feature-override fixture.
* Private fixture-name discovery is mocked at the indexer boundary. Profile values, resolver addresses, permissions, transaction execution, Universal Resolution, display/copy behavior, payment, and repair are verified against the local chain.
* `BrowserContext.setOffline` changes browser network state. The test does not dispatch a synthetic online event and does not call `refetch`, `refetchQueries`, or `invalidateQueries`.
* A headless EIP-1193 wallet captures and authorizes the request. No claim is made about a particular browser extension's transaction-decoding UI.
* The payer and repair accounts are test accounts. The payment proves the consequence once another user relies on the displayed address; it does not claim a measured production loss rate.

### Evidence files and SHA-256

```
ENS_PROFILE_ROUTE_RECONNECT_E2E.json
AD9EE0A903AC0B31901FFDBF3A11311D1FCC94739E5EBFCBB98D27E0FBF4367D

profile-route-trigger-audit.spec.ts
81876622B59E7A0048F412EC09B8DDA0142578D52EBC47D353C2E47C32C33FE1

ENS_PROFILE_ROUTE_RECONNECT_PASS_TRACE.zip
2081CEC16BEC8E7F415CC2C6772031B3AA3D2CEA81315B95FF763FEEBC8BF80F

ENS_PROFILE_ROUTE_RECONNECT_VERIFY.ps1
0D3E43C648681F2547F8F6D134480A56A16288A67E00CE5F01B3BA9148B9D7C3

playwright.profile-route-trigger-audit.config.ts
35CF4CF4EF542AE0462AD6EFF5D0739AC63B38F5ACE2FC40E410CE391CD096C1
```

The secret Gist should contain the focused test, machine-readable evidence, verifier, configuration, source-provenance excerpts, and a checksum manifest. The 17 MB Playwright trace is kept locally and identified by its hash because Gist is text-only.
