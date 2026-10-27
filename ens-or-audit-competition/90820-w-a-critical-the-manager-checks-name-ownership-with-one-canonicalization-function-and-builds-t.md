# 90820 w a critical the manager checks name ownership with one canonicalization function and builds the renewal transaction with another so one invisible character in a url makes a user pay to re&#x20;

**Submitted on Aug 27th 2026 at 12:58:55 UTC by @Valisthea for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90820
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## 1. Summary

The two applications disagree about what an ENS name is.

The Manager canonicalizes with `String.prototype.toLowerCase()`. The Explorer canonicalizes with `ens_normalize()` (ENSIP-15). Because the registrar hashes the label verbatim, those two functions can name two different on-chain names.

The Manager then uses one of them for the **safety half** of the renewal flow and the other for the **transaction half**. Ownership, expiry, protocol version and the third-party renewal warning are all evaluated against the ENSIP-15 normalized name. The label placed in the `renew()` calldata is the lowercased raw name.

A URL containing a single `U+00AD` SOFT HYPHEN, which renders as nothing at all, therefore produces a page that resolves the victim's real name for every check it performs, tells the victim the name is theirs, shows their real expiry, and then signs a transaction that renews a different name belonging to the attacker.

## 2. Impact

This maps to the following bullet of the program's Critical impact list, word for word:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: \[...] showing the wrong recipient or amount on a transaction the user then approves

On a renewal, the name being extended is the recipient of the payment.

Concrete loss to the victim, both halves at once:

1. Their funds extend an asset they do not own and never will.
2. Their own name is not renewed. It lapses at its original expiry while the interface reports the extension succeeded, so they have no reason to retry until the name is gone.

The second half additionally touches the Critical bullet "leaving a name permanently inaccessible after the flow", because a name the user believes they renewed can expire and be re-registered by anyone.

The control that exists specifically to prevent this outcome is disabled by the defect itself. The Manager renders a "That's generous of you" third-party renewal warning when the connected wallet does not own the name. Its only input is `isOwner`, and `isOwner` is computed on the other name.

I am filing this as High rather than Critical because there is no seizure of an existing asset and no key or session compromise. The victim loses the renewal fee and their name is left to lapse.

## 3. Root cause

### 3.1 The two halves of one route

**Authorization and eligibility half. Normalized.**

`apps/manager/src/features/profile/service/profileName.ts:18`

```ts
    normalized = normalize(name)
```

That is viem's ENSIP-15 `normalize`. It is reached from both of the queries the renewal route relies on:

`apps/manager/src/features/profile/service/profileOwner.ts:25`

```ts
  const ethName = normalizeEthName(params.name)
```

`apps/manager/src/features/profile/service/profileExpiry.ts:77`

```ts
  const ethName = normalizeEth2LdName(name)
```

**Transaction half. Lowercased only.**

`apps/manager/src/routes/renew/$name.tsx:61`

```ts
      label: parsedName.value.label,
```

That label is produced by `apps/manager/src/features/register-v2/utils/name-parser.ts:46`

```ts
  const normalized = name.trim().toLowerCase()
```

which is the only canonicalization `parseName` performs. It is then carried unchanged into the transaction:

`apps/manager/src/features/renew/state/renewalUi.machine.ts:349`

```ts
            label: submission.label,
```

`packages/transaction-manager/src/machines/registration/registration.actors.ts:1025` and `:1029`

```ts
  const cleanLabel = label.replace('.eth', '')
    args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS],
```

### 3.2 The suppressed warning

`apps/manager/src/features/profile/components/view/ProfileView.tsx:100` and `:119`

```tsx
    ...profileOwnerQuery(name),
  const isOwner = useIsOwner({ owner })
```

`isOwner` is therefore derived from the normalized name, which resolves the victim's genuine name and returns the victim as its owner. `isOwner` is `true`.

`apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:3`

```ts
) => isOwner === false
```

`apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66` and `:83`

```tsx
  if (shouldShowThirdPartyRenewalWarning(isOwner)) {
      params={{ name }}
```

Line 66 is the gate that never fires. Line 83 hands the **raw** name onward to the renew route, where the loader gates on the normalized name a second time and returns the lowercased label.

### 3.3 Nothing on the route rejects the character

`apps/manager/src/routes/$name/route.tsx:6` is the only shape check on the parameter:

```ts
      if (name.length < 3 || !name.includes('.')) {
```

A soft hyphen passes it. It also passes the punctuation set at `apps/manager/src/features/register-v2/utils/name-parser.ts:11`, which does not contain `U+00AD`, and it is not whitespace, so the `/\s/` test does not catch it either.

## 4. Why the two labels really are two different names

This is settled from the pinned dependency, not assumed. `@ensdomains/ensjs` is pinned by `pnpm-workspace.yaml` to the `pkg.pr.new` build at commit `e96662c`. At that commit, `packages/ensjs/src/utils/v2/registry/labelToCanonicalId.ts:3-7`:

```ts
export const labelToCanonicalId = (label: string) => {
  const id = hexToBigInt(labelhash(label))
  const low32 = id & 0xffffffffn
  return id ^ low32
}
```

`labelhash` is keccak over the raw string. There is no normalization step anywhere in the on-chain identity derivation, so `ali<U+00AD>ce` and `alice` are two separate names with two separate ids and two separate owners.

The library also takes whatever label it is handed. `packages/ensjs/src/actions/wallet/renewName.ts:92` and `:104`:

```ts
  const [label] = name.split('.')
    args: [label, duration, paymentToken, referrer] as const,
```

## Proof of Concept

## 5. Proof of concept

Steps 1 and 2 need no blockchain access at all and take under a minute. They establish the entire mechanism. Step 3 is the application demonstration.

{% stepper %}
{% step %}

## Show the two canonicalizers disagree

`apps/portal/package.json` pins `@adraffy/ens-normalize` at `^1.11.1`. Run, on any machine:

```bash
npm pack @adraffy/ens-normalize@1.11.1
tar -xzf adraffy-ens-normalize-1.11.1.tgz
node --input-type=module -e "
import { ens_normalize } from './package/dist/index.mjs'
const cp = s => [...s].map(c=>'U+'+c.codePointAt(0).toString(16).toUpperCase().padStart(4,'0')).join(' ')
for (const s of ['ali­ce.eth','ＡＢＣ.eth','❤️.eth','™️.eth']) {
  console.log(JSON.stringify(s))
  console.log('  Manager  toLowerCase   :', cp(s.toLowerCase().split('.')[0]))
  console.log('  Explorer ens_normalize :', cp(ens_normalize(s).split('.')[0]))
}"
```

Output obtained on Node v24.13.0:

```
"ali­ce.eth"
  Manager  toLowerCase   : U+0061 U+006C U+0069 U+00AD U+0063 U+0065
  Explorer ens_normalize : U+0061 U+006C U+0069 U+0063 U+0065
"ＡＢＣ.eth"
  Manager  toLowerCase   : U+FF41 U+FF4C U+FF43
  Explorer ens_normalize : U+0061 U+0062 U+0063
"❤️.eth"
  Manager  toLowerCase   : U+2764 U+FE0F
  Explorer ens_normalize : U+2764
"™️.eth"
  Manager  toLowerCase   : U+2122 U+FE0F
  Explorer ens_normalize : U+0074 U+006D
```

The soft hyphen case is the sharpest because `U+00AD` has no visible glyph: `ali<U+00AD>ce.eth` renders as `alice.eth` in every heading, table cell and confirmation line in the product.
{% endstep %}

{% step %}

## Show the two labels hash to different names

```bash
cast keccak "$(printf 'ali\xc2\xadce')"   # the label the Manager registers and renews
cast keccak "alice"                        # the label the gates resolved
```

Two different labelhashes, therefore two different `.eth` names with two different owners.
{% endstep %}

{% step %}

## Demonstrate in the application

Preconditions: a Sepolia wallet with test funds, and a second wallet.

1. From wallet A, register the label `ali<U+00AD>ce` through the Manager. The registration route accepts it: `parseName` lowercases and does not normalize, and `validateENSName` (`apps/manager/src/features/shared/registration/nameUtils.ts:90` onward) rejects only spaces, `..`, an internal dot, labels under three characters, and a set of ASCII punctuation.
2. From wallet B, register `alice`, or use any name wallet B already owns and craft the lookalike against that instead. Wallet B is the victim.
3. Connect wallet B to the Manager and open `https://<manager-host>/ali%C2%ADce.eth`.

**Screenshots to attach (PNG):**

* **Screenshot 1.** The profile page header. It reads `alice.eth`. The soft hyphen is invisible.
* **Screenshot 2.** The same page showing the expiry and owner block. These are wallet B's real values, because `profileOwnerQuery` and `profileExpiryQuery` both normalized the name.
* **Screenshot 3.** The Renew action. The "That's generous of you" third-party warning does **not** appear, because `isOwner` is `true`.
* **Screenshot 4.** The wallet prompt for the renewal, with the decoded calldata visible. The `label` argument is `ali<U+00AD>ce`, not `alice`. In a wallet that does not decode, capture the raw calldata and decode it separately with:

```bash
cast 4byte-decode <calldata>
```

4. Optional, to demonstrate the loss without involving a third party: sign it. Then show with `cast call` that wallet A's `ali<U+00AD>ce.eth` gained duration while wallet B's `alice.eth` expiry is unchanged. Both names are yours, so nobody else is affected.
   {% endstep %}
   {% endstepper %}

## 6. Second instance of the same root cause, in the Explorer

Reported here rather than separately because the root cause is identical and a split would be merged.

The Explorer runs the same divergence in the opposite direction. Its renewal **calldata** normalizes while its **display** does not.

`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191` and `:197`

```ts
  // ensjs splits the label without normalizing, so pass a normalized 2LD name.
      name: `${getLabel(params.name)}.eth`,
```

`getLabel` runs `ens_normalize` at `apps/portal/src/utils/token/getLabel.ts:11`.

`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:292`

```ts
          transactionName: `Extend ${step.name}`,
```

`step.name` is the raw indexer name. It reaches the user at `TransactionStateContent.tsx:113` as `{transaction.transactionName}`. Grepping the whole renew feature, `getLabel` appears only in the calldata builder and in the renewability check (`useIsRenewable.ts:40`). No display path normalizes.

The emoji column of Step 1 matters here: emoji keyboards emit the `U+FE0F` presentation selector, which ENSIP-15 folds and `toLowerCase` does not. Of twenty common emoji tested, sixteen diverge. The Manager's own validation copy invites exactly this input class, at `apps/manager/src/features/shared/registration/nameUtils.ts:104`:

```
        "Something in the name isn't supported. Try letters, numbers, hyphens, or emojis with no spaces.",
```

So a user who registers an emoji name in the Manager, as its UI recommends, and later renews it in the Explorer, signs a renewal for a different name. For `™️.eth` that different name is `tm.eth`.

## 7. Why this is not a duplicate

**EXP-INPUT-005** ("ENS name validators accept homoglyphs, bidirectional-control and other control characters \[...] causing homograph-style display confusion in the UI rather than incorrect resolution") is the nearest published entry, and it is a different root cause.

That entry describes a permissive **validator**, and its stated remedy is to reject deceptive characters. That remedy does not fix this. The defect here is that one route runs its authorization half through `normalize()` and its transaction half through `toLowerCase()`, so the two halves can name different on-chain entries **at all**. Tighten the accepted character set as much as you like: any surviving disagreement between the two functions reproduces the bug. Making the two functions identical is the fix, and that is a different change in a different place.

Three further distinctions:

1. The published entry frames its impact as display confusion **rather than incorrect resolution**. This report demonstrates incorrect resolution: a signed transaction acting on a name other than the one every gate approved.
2. The previously unstated consequence is that `isOwner`, the sole input to the third-party renewal warning, is computed on the wrong name. A safety control is inverted, not merely a label.
3. No confusable pair is required. `™️` and `tm` are not lookalikes, and `❤️` versus `❤` differs by an invisible codepoint rather than a similar glyph.

Checked and cleared against the rest of the published list: **QA-03** is a price display mismatch in the registration and extension modals, explicitly display-only. **QA-05** is renewal being blocked immediately after registration. **WEB-1303** and **WEB-1255** concern grace-period helpers and the `isV2` flag. **EXP-4337-002**, **EXP-4337-003** and **SEC-TXM-002** are transport-level chain, sender and address issues in the transaction manager, all downstream of where this label is chosen. None of them touches label canonicalization.

## 8. Recommended fix

**The durable fix is one function.** Export a single ENSIP-15 canonicalizer from `packages/utils` and use it in the Manager's `parseName` and `normalizeQuery`, and in the Explorer's `getLabel`. Two applications in one product must not disagree about what a name is.

**The immediate fix**, on both renewal routes, is to refuse or disclose whenever the raw name and its canonical form differ:

```ts
const canonical = ens_normalize(rawName)
if (canonical !== rawName) {
  // Either refuse the flow, or render the canonical name as the thing that will
  // actually be renewed, so the user approves what will happen.
}
```

**Defence in depth:** compute the label once, at the boundary where the name enters the feature, and pass that single value to both the gates and the calldata builder. The bug is only possible because two independent call sites derive the label separately.

**Regression tests to pin:**

* A route parameter containing `U+00AD` must not render as, or resolve the owner of, its soft-hyphen-free twin.
* A name whose raw label differs from its ENSIP-15 normalization must either be refused for renewal or must display the label the transaction will actually hash.
* `™️.eth` and `❤️.eth` are the two emoji cases worth adding to the suite.

## 9. Full citation index

Every line below was verified byte for byte against the frozen tree.

| Claim                                  | Location                                                                                      |
| -------------------------------------- | --------------------------------------------------------------------------------------------- |
| Ownership resolves the normalized name | `apps/manager/src/features/profile/service/profileName.ts:18`                                 |
| Owner query normalizes                 | `apps/manager/src/features/profile/service/profileOwner.ts:25`                                |
| Expiry query normalizes                | `apps/manager/src/features/profile/service/profileExpiry.ts:77`                               |
| `isOwner` fed from that query          | `apps/manager/src/features/profile/components/view/ProfileView.tsx:100`, `:119`               |
| Warning predicate                      | `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:3`                         |
| Warning gate                           | `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66`                 |
| Raw name handed onward                 | `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:83`                 |
| Renew route re-gates on normalized     | `apps/manager/src/routes/renew/$name.tsx:27`                                                  |
| Renew route returns lowercased label   | `apps/manager/src/routes/renew/$name.tsx:61`                                                  |
| Only canonicalization in the parser    | `apps/manager/src/features/register-v2/utils/name-parser.ts:46`                               |
| Punctuation set, no `U+00AD`           | `apps/manager/src/features/register-v2/utils/name-parser.ts:11`                               |
| Only route-parameter shape check       | `apps/manager/src/routes/$name/route.tsx:6`                                                   |
| Label into the machine                 | `apps/manager/src/features/renew/state/renewalUi.machine.ts:349`                              |
| Label into the calldata                | `packages/transaction-manager/src/machines/registration/registration.actors.ts:1025`, `:1029` |
| Validator copy inviting emoji          | `apps/manager/src/features/shared/registration/nameUtils.ts:104`                              |
| Explorer signs the normalized label    | `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191`, `:197`                  |
| Explorer displays the raw name         | `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:292`                          |
| `getLabel` normalizes                  | `apps/portal/src/utils/token/getLabel.ts:11`                                                  |
| On-chain id is keccak of the raw label | ensjs `e96662c`, `packages/ensjs/src/utils/v2/registry/labelToCanonicalId.ts:3-7`             |
| Library takes the label verbatim       | ensjs `e96662c`, `packages/ensjs/src/actions/wallet/renewName.ts:92`, `:104`                  |
