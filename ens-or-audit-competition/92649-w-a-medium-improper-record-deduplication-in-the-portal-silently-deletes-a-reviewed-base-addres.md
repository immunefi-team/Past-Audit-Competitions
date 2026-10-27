# 92649 w a medium improper record deduplication in the portal silently deletes a reviewed base address and resolves the ens name to an attacker controlled fallback

**Submitted on Sep 11th 2026 at 08:21:17 UTC by @guilhhotina for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92649
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Portal record editor can preserve a deletion of an existing address record while adding a visually valid replacement record for the same coin type. The UI shows only the newly added Base address, but the save transformer serializes both operations in the opposite semantic order: it writes the reviewed Base address and then deletes that same Base slot. The transaction succeeds, the specific Base record ends empty, and the PermissionedResolver falls back to the default EVM address controlled by a third-party delegate. As a result, the ENS name resolves on Base to an attacker-controlled address even though the Portal showed a different address as the final saved record.

## Vulnerability Details

### Affected asset and normal user flow

The root cause is in the in-scope Portal asset:

`apps/portal/`

The vulnerable sequence uses the normal **Edit records** page:

1. A name has an existing Base address record.
2. The user deletes that Base row.
3. The deleted row disappears from the visible table.
4. The user selects `Address`, selects `BASE`, enters the replacement address, and adds it.
5. The table shows one non-deleted Base row containing the replacement address.
6. The user opens the normal save flow and approves the generated transaction.

The interface permits this sequence because deleted records are removed from the visible table, while the address selector still permits adding the same coin type.

### Root cause: UI identities differ from resolver storage identities

Resolver storage identifies an address record by `(node, coinType)`. Both the old and new Base records therefore refer to the same onchain slot.

The Portal does not use that identity for pending records. Existing records use `address-${record.key}`, while every new record receives a random `_uid` that is appended to its UI identity:

```typescript
export const getRecordId = (record: NameRecord | EditableRecord): string => {
  const editableRecord = record as EditableRecord
  const uid = editableRecord._uid ? `-${editableRecord._uid}` : ''

  if (record.type === 'address') {
    return `address-${record.key}${uid}`
  }
}

export function createNewRecord(...) {
  const _uid = generateUid()
  return {
    type: 'address',
    key,
    value,
    id: coinType,
    _uid,
  }
}
```

Deleting the original record adds its canonical UI ID to `deletedIds`. Adding another Base record creates a separate `newRecords` entry with a different `_uid`; it does not cancel the deletion:

```typescript
case 'ADD_RECORD': {
  const newRecord = createNewRecord(...)
  return {
    ...state,
    newRecords: [...state.newRecords, newRecord],
  }
}

case 'DELETE_RECORD': {
  const id = getRecordId(action.record)
  // ...
  return {
    ...state,
    editedValues: nextEditedValues,
    deletedIds: new Set(state.deletedIds).add(id),
  }
}
```

`mergeRecordsWithChanges()` preserves both entries. The original receives `isDeleted: true`, while the replacement receives `isNew: true`. The route filters deleted entries from the table, so the user sees only the new Base address:

```typescript
const visible = records.filter((record) => !record.isDeleted)
```

The pending state therefore contains two UI objects but one onchain storage identity:

```
newRecords:  BASE -> reviewed address, with a new _uid
deletedIds: address-BASE, referring to the original record
onchain key for both: (namehash(name), BASE coin type)
```

### The save transformer submits the deletion last

`transformPendingChangesToSetRecords()` processes all new records before all deleted records and does not coalesce them by coin type:

```typescript
for (const record of newRecords) {
  if (record.type === 'address') {
    coins.push({ coin: record.id, value: record.value })
  }
}

for (const id of deletedIds) {
  const originalRecord = originalRecords.find((r) => getRecordId(r) === id)
  if (originalRecord?.type === 'address') {
    coins.push({ coin: originalRecord.id, value: '' })
  }
}
```

For the visible replacement operation, the exact transformed value is:

```typescript
[
  { coin: BASE_COIN_TYPE, value: REVIEWED_DESTINATION },
  { coin: BASE_COIN_TYPE, value: '' },
]
```

`prepareSaveRecordsTransaction()` passes this array directly to ENSjs and encodes the resulting resolver multicall. It performs no final coalescing or duplicate-slot validation. Because resolver multicalls execute sequentially, the empty write is final.

### The hidden deletion activates an attacker-controlled fallback

The PermissionedResolver correctly scopes address authorization by `partHash(coinType)`. The PoC grants the third party only the default EVM coin-type role, not the Base-specific role:

```solidity
onlyPartRoles(
    node,
    PermissionedResolverLib.partHash(coinType),
    PermissionedResolverLib.ROLE_SET_ADDR
)
```

The third party stores `0x000000000000000000000000000000000000bEEF` in the default EVM slot. The owner separately stores a legitimate Base-specific address.

After the Portal-generated multicall writes and then deletes the Base-specific slot, the resolver's ENSIP-19 behavior applies:

```solidity
addressBytes = r.addresses[coinType];
if (addressBytes.length == 0 && ENSIP19.chainFromCoinType(coinType) > 0) {
    addressBytes = r.addresses[COIN_TYPE_DEFAULT];
}
```

Consequently:

```
hasAddr(namehash(name), BASE_COIN_TYPE) == false
addr(namehash(name), BASE_COIN_TYPE) == 0x000000000000000000000000000000000000bEEF
```

The third party never receives the Base-specific role and does not need to submit another transaction after the victim saves. The Portal silently removes the Base override that prevented the delegate's default address from controlling Base resolution.

### Complete demonstrated attack sequence

The deterministic PoC performs the following against the real deployed bytecode included in the competition repository:

1. Deploy `VerifiableFactory` and `PermissionedResolverImpl` to an empty local Anvil chain.
2. Deploy and initialize a real PermissionedResolver proxy for the owner.
3. Grant the third party only `authorizeAddrRoles(name, DEFAULT_EVM_COIN_TYPE, attacker, true)`.
4. Have the third party set the default EVM address to `0x...bEEF`.
5. Have the owner set a different legitimate Base-specific address.
6. Invoke the real `editRecordsReducer()` with `DELETE_RECORD` for the original Base row.
7. Invoke the real reducer with `ADD_RECORD` for Base and the reviewed replacement address.
8. Invoke the real `mergeRecordsWithChanges()` and prove the visible state contains exactly one non-deleted Base row with the reviewed address.
9. Invoke the real `transformPendingChangesToSetRecords()` and prove it emits the replacement followed by an empty deletion for the same Base coin type.
10. Invoke the real `prepareSaveRecordsTransaction()` to build the exact EOA transaction used by the Portal.
11. Submit that exact calldata from the controlled owner account.
12. Observe a successful receipt.
13. Read `hasAddr(node, BASE)` and prove the specific Base slot is absent.
14. Read `addr(node, BASE)` and prove the effective result is the third party's `0x...bEEF` address.

## Impact Details

Selected in-scope Critical impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as resolving or rendering an ENS name to an attacker-controlled address.

The PoC demonstrates this impact directly:

* the Portal's final visible record state contains the owner's reviewed Base address;
* the Portal constructs a transaction containing a later hidden deletion for that same Base slot;
* the connected owner submits the exact Portal-generated calldata;
* the transaction succeeds;
* the reviewed Base-specific record does not exist afterward;
* the name's effective Base resolution is the third party's address.

Any wallet, payment application, or user resolving this ENS name for Base can consequently receive the attacker's address instead of the address the owner reviewed and saved. Funds sent using that resolution can be directed to the third party.

I am not claiming that the PoC transfers real funds or NFTs. The demonstrated Critical impact is the separately enumerated category in which the application causes an ENS name to resolve to an attacker-controlled address. The vulnerable state persists until the owner notices the mismatch and restores a Base-specific record or revokes/changes the default address.

### Why this is not a disclosed known issue

This is not a generic permission, transfer, alias, or resolver-contract issue:

* `QA-01` and `WEB-1257` concern transfer combinations and follow-up behavior; this reproduction does not transfer a name.
* The issue does not depend on an alias.
* The third party's authorization remains correctly limited to the default EVM coin type; the Portal removes the Base-specific override and thereby expands the delegate's effective influence.
* The resolver executes the ordered calls and ENSIP-19 fallback as designed. The vulnerable duplicate state and destructive ordering are produced by the Portal.
* The cause is the combination of `_uid`-based UI identity, a surviving canonical tombstone, missing `(record type, coinType)` coalescing, and addition-before-deletion serialization.

## References

Public in-scope branch, currently resolving to commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`:

* Asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* UI record identity and `_uid`: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/records/editRecordUtils.ts#L38-L52>
* New address construction: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/records/editRecordUtils.ts#L135-L160>
* Pending addition and deletion state: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useEditRecordsState.ts#L53-L90>
* Save transaction construction: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L97-L149>
* Addition-before-deletion serialization: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/transformPendingChanges.ts#L48-L158>
* Deleted-row filtering and record creation UI: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/edit-records.tsx#L308-L409>
* Permissioned address authorization and ENSIP-19 fallback: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/references/contracts-v2-8d1c893/contracts/src/resolver/PermissionedResolver.sol#L660-L692>
* Secret Gist: <https://gist.github.com/guilhhotina/7af639deaeb942314f2b3ad7facf7692>

The executable PoC and its environment instructions are supplied in the secret Gist linked in the Proof of Concept field.

## Link to Proof of Concept

<https://gist.github.com/guilhhotina/7af639deaeb942314f2b3ad7facf7692>

## Proof of Concept

## Proof of Concept

### What this PoC proves

This PoC reproduces the complete vulnerable state transition using the Portal's real reducer, visible-record merge, save transformer, transaction builder, and the real `VerifiableFactory` and `PermissionedResolverImpl` bytecode committed to the competition repository.

It proves all of the following in one deterministic execution:

```
Visible Base row: reviewed owner address
Serialized write 1: BASE = reviewed owner address
Serialized write 2: BASE = empty
Transaction receipt: success
Specific Base slot after execution: absent
Effective Base resolution: attacker-controlled default address
```

The third party is authorized only for the resolver's default EVM coin type. The owner controls and initially sets the Base-specific record. The final Base resolution changes because the Portal submits a hidden deletion after the visible replacement, not because the third party can write the Base slot.

### Safety

The PoC uses:

* an empty local Anvil chain at `127.0.0.1:8545`;
* chain ID `11155111` only to match the Portal configuration;
* deterministic local accounts funded through Anvil RPC;
* bytecode artifacts already committed to the audit repository;
* no fork, public RPC, production contract, browser extension, real wallet, real name, or real funds.

### Prerequisites

* a clean checkout of the public `audit-comp-ready` branch;
* project dependencies installed with the repository's documented command;
* Node.js and Foundry/Anvil available locally.

The public branch currently resolves to:

```
1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

### Install the PoC from the secret Gist

From the repository root:

```bash
curl -L "https://gist.githubusercontent.com/guilhhotina/7af639deaeb942314f2b3ad7facf7692/raw/41c4f8d9daf47d4f50d9191de4c174f50e4f611f/record-slot-collision.onchain.poc.test.ts" \
  -o apps/portal/src/features/records/record-slot-collision.onchain.poc.test.ts
```

The Gist is supplementary delivery for the executable file; the full reproduction and expected state transitions are described here.

### Start the isolated chain

In terminal 1:

```bash
anvil --chain-id 11155111 --port 8545 --host 127.0.0.1 --silent
```

No fork URL should be supplied.

### Execute

In terminal 2, from the repository root:

```bash
cd apps/portal
node node_modules/vitest/vitest.mjs run \
  src/features/records/record-slot-collision.onchain.poc.test.ts \
  --reporter=verbose
```

### Exact reproduction sequence

The test performs these steps:

1. Connect to the empty local Anvil chain and fund two deterministic controlled accounts: `OWNER` and `ATTACKER`.
2. Load the repository's `VerifiableFactory.json` and `PermissionedResolverImpl.json` deployment artifacts.
3. Deploy the real factory and resolver implementation bytecode.
4. Deploy a resolver proxy and initialize `OWNER` with all root roles.
5. Grant `ATTACKER` only the address role for `DEFAULT_EVM_COIN_TYPE` through the real `authorizeAddrRoles` function.
6. Have `ATTACKER` write `0x000000000000000000000000000000000000bEEF` to the default EVM slot.
7. Have `OWNER` write `0x0000000000000000000000000000000000000003` to the specific Base coin type.
8. Construct the original Portal `NameRecord` for that Base address.
9. Run the real `editRecordsReducer()` with `DELETE_RECORD` for the original Base row.
10. Run the reducer with `ADD_RECORD` for Base and the reviewed replacement address `0x0000000000000000000000000000000000000004`.
11. Run the real `mergeRecordsWithChanges()` and filter deleted rows exactly as the route does.
12. Require the visible result to contain exactly one Base row with the reviewed replacement address.
13. Run the real `transformPendingChangesToSetRecords()`.
14. Require the exact collision and ordering:

```typescript
expect(transformed.coins).toEqual([
  { coin: BASE_COIN_TYPE, value: REVIEWED_DESTINATION },
  { coin: BASE_COIN_TYPE, value: '' },
])
```

15. Run the real `prepareSaveRecordsTransaction()` to obtain the EOA request the Portal would submit.
16. Submit that exact `to` and `data` from the controlled owner account.
17. Require the transaction receipt to equal `success`.
18. Read the real resolver and require that the specific Base slot is absent:

```typescript
expect(
  await publicClient.readContract({
    address: resolverAddress,
    abi: resolverAbi,
    functionName: 'hasAddr',
    args: [namehash(NAME), BigInt(BASE_COIN_TYPE)],
  }),
).toBe(false)
```

19. Read the effective Base address through the real resolver.
20. Require it to equal the third party's default EVM address:

```typescript
const effectiveAddress = await publicClient.readContract({
  address: resolverAddress,
  abi: resolverAbi,
  functionName: 'addr',
  args: [namehash(NAME), BigInt(BASE_COIN_TYPE)],
})

expect(
  isAddressEqual(getAddress(effectiveAddress), ATTACKER_DESTINATION),
).toBe(true)
```

### Expected output

```
✓ record-slot-collision.onchain.poc.test.ts
  ✓ shows a replacement Base address but deletes its storage slot and resolves an attacker-controlled fallback

Test Files  1 passed (1)
Tests       1 passed (1)
```

The definitive local execution produced `1 passed`, a successful receipt, an absent Base-specific slot, and effective Base resolution equal to `0x000000000000000000000000000000000000bEEF`.

### Source hashes

```
0c7a0cc43166253f3375c39733e5d6933e8af972c1db02adbb210207f20a8632  record-slot-collision.onchain.poc.test.ts
031de881e9b9906dfaeed023be5ea4afc52ce35c1424bb5698939bc3789eaff0  editRecordUtils.ts
bf1067cc01d2e56a0fa22605117d96cd16084c93d6a0bd6613845823463363ca  useEditRecordsState.ts
bc5b212bfcdd0f37377c126406870005c60260917e22a384e42632dac377d22b  $name/edit-records.tsx
9b9dffab62a46847a46a917849bf5f769801d1bb476a727056a88189d8ce0214  transformPendingChanges.ts
2fb737ab41d4776c1138624d3bda7b3c5035b03e29d9eda0d41e038e0d9c507e  saveRecords.ts
d2bec253d9cf9cf982ad8b27ea690050118fb27e6bd373cc74c15459d7982167  PermissionedResolver.sol
6ddfbec8acf3823fdb62cb09b3325ed61f202edbce8d8773c75a0ec6a3152b71  PermissionedResolverImpl.json
4f00ff1e985f895176ebe97e6468fa184adc85a7af63949f51fb2e9f3e984c24  VerifiableFactory.json
```

### Secret Gist

<https://gist.github.com/guilhhotina/7af639deaeb942314f2b3ad7facf7692>
