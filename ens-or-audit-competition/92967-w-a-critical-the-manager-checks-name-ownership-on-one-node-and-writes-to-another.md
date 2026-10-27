# 92967 w a critical the manager checks name ownership on one node and writes to another

**Submitted on Sep 13th 2026 at 14:35:56 UTC by @nomorecaffeine for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92967
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Bug Description

The profile route takes a name from the URL and never normalizes it:

[route.tsx#L3-L13](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/$name/route.tsx#L3-L13)

```ts
export const Route = createFileRoute('/$name')({
  params: {
    parse: ({ name }) => {
      if (name.length < 3 || !name.includes('.')) {
        throw new Error('Invalid ENS name')
      }

      return {
        name,
      }
    },
```

The page then resolves that name twice, under two different normalizations.

The ownership half normalizes first:

[profileOwner.ts#L24-L55](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileOwner.ts#L24-L55)

```ts
  const ethName = normalizeEthName(params.name)
  ...
  const v2Owner = yield* fromPromise(
    ensjsv2_getOwner(client, { name: ethName.name }),
```

The records half does not. [profileRecords.ts#L49](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileRecords.ts#L49) queries the indexer with `{ id: name }`, and the on-chain read takes the same raw string into `getRecords(client, { name, ... })`. ensjs performs no normalization of its own: `getRecords` passes `name` straight into `getTextParameters`, `getAbiParameters` and `getContentHashParameters`.

An ENS label is an arbitrary string on chain. Normalization is a client-side convention, not a registrar constraint, so the two halves address two different registrations, with two different owners, whenever the URL carries a label that normalization would change:

* Alice owns `café.eth`, whose label is NFC (`é` is U+00E9).
* Mallory registers the same label in NFD (`e` followed by U+0301). It is a different string, a different namehash and a different registration. Both render as `café.eth`.
* Mallory points the NFD name's `addr(60)` record at her own address.
* Mallory sends Bob a link to `https://<manager>/café.eth`, NFD form. Bob's URL bar and the page header show `café.eth`.
* The Owner field is resolved from the normalized name, so it shows **Alice's** real address.
* The ETH address record is resolved from the raw name, so it shows **Mallory's** address.

The split also decides whether the user may act. `isOwner` is computed from the ownership half:

[ProfileView.tsx#L115-L119](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/view/ProfileView.tsx#L115-L119)

```ts
  const owner = ownerData?.owner as Address | undefined
  ...
  const isOwner = useIsOwner({ owner })
```

`isOwner` gates the Edit Profile action ([ProfileActions.tsx#L107](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/view/ProfileActions.tsx#L107)), and the save that action performs is addressed to the resolver the **records** half found:

[profileRecordTransactions.ts#L455](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileRecordTransactions.ts#L455)

```ts
    calls: [{ to: resolverAddress, data, value: 0n }],
```

`resolverAddress` reaches it from `savedRecords.resolverAddress` ([useEditProfileDialogSave.ts#L216](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts#L216)), which is the raw name's resolver.

Measured on live Sepolia through the app's own functions, for a name already indexed in NFD:

```
==== ébf-f51f02a5.eth ====
codepoints raw     = U+0065 U+0301 U+0062 U+0066 U+002D U+0066 U+0035 U+0031 U+0066 U+0030 U+0032 U+0061 U+0035 U+002E U+0065 U+0074 U+0068
normalizeEthName   = ébf-f51f02a5.eth
renders identically= true
namehash(raw)      = 0x606b2ef1050de96f19a4d4ed6f15028d3d1689b1a28dd933bd7dedcddef9e9af
namehash(normal)   = 0xc22d8d028e8f3410a24715c68d8f24c408a493a620a4126a0f593cf14aef39d6
OWNER half (normalized) -> 0x9aDaD7ECDB9a481f66abcCa385d6E0B5E5E3FC3E
RECORDS half (RAW)      -> addr60 0x13D7d7267154D7712aF147F964226BC8D6B3D951 resolver 0xe25379bFf197F110aCE1E686fc1584fb342d2ABC
RECORDS control (norm)  -> addr60 0x9aDaD7ECDB9a481f66abcCa385d6E0B5E5E3FC3E resolver 0xF4609d9DA07A6Af80b94aFfE9e29E9eB74a5F58F
```

The first two lines are not a repetition: the input is `e` + U+0301, the normalized form is U+00E9, and they hash to the two nodes shown beneath.

The split is in the keying, not in any character filter. It needs no deceptive character at all: the PoC reproduces the same divergence on `CRYPTO.eth`, plain ASCII in the wrong case, and no validator runs on this route either way.

## Impact

The app decides a connected user owns the name from one node and then builds the record write against the other. The `to:` of the transaction they sign is therefore a contract chosen by whoever owns the twin, and the ownership check that unlocked the action never looked at the node being written.

The page gives the user little to catch it with, because Owner, Registered and Expires are resolved from the normalized name ([profileRegistration.ts#L66](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileRegistration.ts#L66), and `profileExpiry.ts:77` does the same) and therefore describe the genuine registration, wrapping a node the user does not own in the real name's ownership and dates.

The substituted target receives a `setText` or `setAddr` call with `value: 0`, so nothing is drained by the call itself; what the user loses is the write, which lands on a contract the attacker controls while the interface reports success. The victim must arrive by a link, because no in-app navigation produces an unnormalized URL.

## Attacker prerequisites

Register the twin of a name the victim owns, an ordinary registration at ordinary cost, and point it at a resolver that permits anyone to write. The permissive resolver matters: the edit dialog probes write access first ([useEditProfileDialogSave.ts#L219-L226](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts#L219-L226)) and routes to resolver setup when the probe fails. Then get one link opened. No role, no approval, no prior relationship, and no interaction by the name's real owner.

A twin with no resolver at all is not enough: in the same run, `Crypto.eth` makes the records query reject, and the route renders its error component instead.

## Recommendation

Derive one canonical key at the route boundary, so every consumer downstream receives the same string:

[route.tsx#L5-L13](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/$name/route.tsx#L5-L13)

```diff
     parse: ({ name }) => {
       if (name.length < 3 || !name.includes('.')) {
         throw new Error('Invalid ENS name')
       }
 
       return {
-        name,
+        name: normalize(name),
       }
     },
```

To preserve the typed spelling in the UI, keep the raw string for display only and pass the normalized one to every query and every write, which is the split `profileOwner.ts` already makes and `profileRecords.ts` does not.

## Link to Proof of Concept

<https://gist.github.com/nguyen-xuan-quoc/d316c21536bc6b1e625c336dfc038d97>

## Proof of Concept

## Proof of Concept

Three tests against live Sepolia, 3 of 3 passing. Each reads both halves for one URL: the ownership half through the app's own `normalizeEthName` then ensjs `getOwner`, the records half through ensjs `getRecords` on the raw parameter. The app's function is imported rather than reimplemented, so what is measured is its own normalization.

* **`a URL that renders identically to the real name reads a different node`** asserts the input and its normalized form are different strings with the same glyphs, that they namehash to different nodes, that `addr(60)` differs between the halves, and that the normalized half's `addr(60)` equals the owner. That last assertion shows the canonical name is internally coherent, so only the raw half describes something else.
* **`the same split occurs with nothing but a capital letter`** repeats it on `CRYPTO.eth`, plain ASCII.
* **`control: a name already in normalized form reads one node on both halves`** runs `crypto.eth` and asserts both halves agree on node, address and resolver.

`resolverAddress` is the assertion that carries the claimed impact, because that value becomes the transaction's `to:`.

<https://gist.github.com/nguyen-xuan-quoc/d316c21536bc6b1e625c336dfc038d97>

The gist also contains the output of the run, so the results can be compared without executing anything. To run it:

1. Clone the audited revision and install:

```sh
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens && git checkout audit-comp-ready
pnpm install --frozen-lockfile
```

2. Copy the test into `apps/manager/src/poc/`.
3. Run it:

```sh
cd apps/manager
npx vitest run src/poc/01-poc-manager-name-key-divergence.test.ts --reporter=verbose --disable-console-intercept
```

**One reproduction note.** The indistinguishable input is written in the test as `'\u0065\u0301bf-f51f02a5.eth'`, using escapes. Typing it as a literal accented character instead silently composes it to NFC, which makes the input equal to its own normalized form, and the first test then fails on its own assertion. The escapes are deliberate.

The control is what makes the run evidence rather than illustration: on an already-normalized name the two halves collapse onto one node and one resolver, so the divergence tracks the normalization difference and nothing else.

```
==== control crypto.eth ====
namehash(raw) == namehash(normal) = true
resolver raw / normalized        = 0x75C7495CfD896FB78C4F6eF9f49Aa291232Cd116 / 0x75C7495CfD896FB78C4F6eF9f49Aa291232Cd116
```

No transaction is sent to any network and no shipped file is modified. The hops from `isOwner` to the Edit Profile gate to `savedRecords.resolverAddress` to the `to:` field are traced from the files linked above rather than driven in a browser.
