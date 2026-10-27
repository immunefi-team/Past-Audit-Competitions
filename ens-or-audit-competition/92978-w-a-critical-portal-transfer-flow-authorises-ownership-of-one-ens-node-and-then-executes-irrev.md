# 92978 w a critical portal transfer flow authorises ownership of one ens node and then executes irreversible writes against a different one

**Submitted on Sep 13th 2026 at 15:58:59 UTC by @spryon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92978
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The portal's transfer flow decides *"you are allowed to do this"* against one ENS node and then performs its irreversible operations against a different one. The ownership gate on `/$name/ownership/transfer` resolves the **raw** URL parameter, while the `detach-resolver`, `detach-registry` and `transfer-token` steps derive their target from `getLabel(name)`, which applies ENSIP-15 normalisation. For a name where those two forms differ, a victim who genuinely owns the name in the URL passes the authorisation check and then signs transactions that detach the resolver and subregistry of and transfer the token for **a different name they also own**. In production this bricks the victim's real ENS name (it stops resolving, and every subname under it stops resolving) from a flow they were correctly told they were authorised to perform, with the token transfer available as an escalation.

## Vulnerability Details

### The shape: check resource A, act on resource B

Every row below is in one user-approved flow, for one URL parameter:

| Concern                                                        | Code                                                                                                                                                                                               | Node           |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| **Authorisation gate** (`isAddressEqual(address, data.owner)`) | [`transfer.tsx:27,104`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L27)                     | **RAW**        |
| `registryAddress` passed to the transaction                    | [`transfer.tsx:106-110`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L106-L110)              | **RAW**        |
| Permission check (`ROLE_CAN_TRANSFER_ADMIN`)                   | [`useCanTransferName.ts:59`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L59)      | **NORMALIZED** |
| Step `set-eth-addr`                                            | [`setEthAddress.ts:37,40,47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/setEthAddress.ts#L37-L47)    | **RAW**        |
| Step `detach-resolver`                                         | [`useTransferName.ts:132,137`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L132-L137) | **NORMALIZED** |
| Step `detach-registry`                                         | [`useTransferName.ts:132,140`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L132-L140) | **NORMALIZED** |
| Step `transfer-token`                                          | [`useTransferName.ts:113,143`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L113-L143) | **NORMALIZED** |

### Every hop

**1 the gate that exists is never reached.** The `/$name` layout at [`routes/$name.tsx:29`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name.tsx#L29) *imports* `isValidEnsName`, but uses it only inside the TLD-redirect condition (`if (isValidEnsName(name) && isTLD(name))`). It never rejects a non-normalised name. The real rejection lives on the **index leaf only** [`routes/$name/index.tsx:442`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/index.tsx#L442) → [`:459`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/index.tsx#L459) `<InvalidNameMessage />` so no sibling route under `/$name/` inherits it. The PoC confirms `isValidEnsName` **would** have rejected the input.

**2 the gate resolves the raw node.** [`transfer.tsx:22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L22) takes the raw param; [`:27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L27) runs `useQuery(getEnsOwnerQueryOptions({ name }))`. That chain [`useEnsOwner.ts:34`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/useEnsOwner.ts#L34) → [`resolveEnsOwner.ts:49-50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/ens/resolveEnsOwner.ts#L49-L50) `getOwnerV2(client, { name })` / `getNameRegistries(client, { name })` normalises at no hop.

**3 authorisation passes.** [`transfer.tsx:104`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L104) `isAddressEqual(address, data.owner)`, and on success renders `<AuthorizedTransfer />` at [`:106-110`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L106-L110). The victim does own the raw node, so this is a truthful "yes".

**4 the steps switch nodes.** In [`useTransferName.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L131-L143) both forms coexist inside one function:

```ts
const common = { name, walletClient, publicClient: pc, signer, chainId }   // :131  RAW
const label = getLabel(name)                                               // :132  NORMALIZED

await match(step)
  .with('set-eth-addr',    () => setEthAddress({ ...common, recipient, id }))          // :135 RAW
  .with('detach-resolver', () => detachNameResolver({ ...common, label, … }))          // :137 NORM
  .with('detach-registry', () => detachNameRegistry({ ...common, label, … }))          // :140 NORM
  .with('transfer-token',  () => transferToken({ ...common, tokenId, … }))             // :143 NORM
  .exhaustive()
```

`getLabel` ([`getLabel.ts:10-20`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/getLabel.ts#L10-L20)) runs `ens_normalize` then `ens_split`. The `tokenId` is likewise derived from the normalised label at [`:113`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L113).

**5 `set-eth-addr` writes to the raw node.** [`setEthAddress.ts:37,40,47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/setEthAddress.ts#L37-L47) passes the raw `name` into `getResolver`, `createSetForwardResolutionRequest` and `setForwardResolution`, reaching `setAddrParameters({ name, … })` at [`setForwardResolution.ts:55-59`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/l2-primary/src/utils/setForwardResolution.ts#L55-L59), i.e. `namehash(RAW)`. So the flow is not even internally consistent: step 1 goes one way and steps 2-4 the other.

### This is a canonicalisation disagreement, not a permissive charset

The known-issue index carries **EXP-INPUT-005** ("name validators accept homoglyphs / bidi / control chars"). This is not that, and the distinction is asserted rather than argued.

The portal *has* the correct validator and even documents this spoofing vector in [`isNormalized.ts:59-65`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNormalized.ts#L59-L65). The defect is that the validator is not applied on this route and that two code paths in one flow canonicalise differently. ENSIP-15 applies NFKC compatibility mappings; the gate path applies none, so the split also fires on ordinary visible characters:

| Input                               | Gate authorises | Steps execute against | Unicode `Cf`? |
| ----------------------------------- | --------------- | --------------------- | ------------- |
| `vitalik­.eth` (U+00AD soft hyphen) | `namehash(raw)` | `vitalik.eth`         | **yes**       |
| `ﬁnance.eth` (U+FB01 ligature)      | `namehash(raw)` | `finance.eth`         | no            |
| `①.eth` (U+2460 circled digit)      | `namehash(raw)` | `1.eth`               | no            |
| `ｖitalik.eth` (U+FF56 fullwidth)    | `namehash(raw)` | `vitalik.eth`         | no            |

Three of the four carry **no format character at all**, asserted in the PoC against Unicode General\_Category `Cf`. Hardening a validator against invisible and bidi characters would leave this fully exploitable.

### Attacker prerequisites and the concrete scenario

1. Victim owns `vitalik.eth`.
2. Attacker registers the lookalike label. The registrars gate on length, not ENSIP-15 stated in the codebase at [`isNameOrAddress.ts:9-10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNameOrAddress.ts#L9-L10): *"the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length)"*. The attacker transfers it to the victim; an unsolicited ERC-721 transfer needs no consent. **The victim now owns both nodes** which is what makes the gate pass.
3. Victim opens `/<lookalike>/ownership/transfer`, from an attacker-supplied link or simply by navigating to dispose of the junk name they were sent. The page renders a string visually identical to their real name.
4. On confirm, `detach-resolver`, `detach-registry` and `transfer-token` execute against the **real** `vitalik.eth`.

The detach steps are documented in-repo as irreversible ([`transfer.tsx:134-138`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L134-L138)), and the form text at [`SendNameForm.tsx:49,57`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx#L49-L57) describes them as detaching "this name's" resolver and registry while "this name" in the transaction is a different name from the one that was authorised.

## Impact Details

### Selected impact: transaction argument modification on an already-connected wallet

This is the mapping I am claiming, and it is the one the PoC demonstrates directly: the arguments of the submitted transactions (`label` for both detach steps, `tokenId` for the transfer) address a different ENS node than the one the application checked authorisation against. Nothing here is inferred from platform behaviour the divergence is computed by the repository's own functions and asserted.

**On the "hijacking or bricking an ENS name" impact:** that entry reads "during a **Manager** flow (registration, v1 to v2 migration, or transfer)". This finding is a *transfer* flow, but it is in the **portal**, not `apps/manager`. If the wording is read as covering ENS management flows generally, this matches it squarely. If it is read as naming the Manager app specifically, the transaction-argument mapping above stands on its own. I am flagging the ambiguity rather than relying on a reading that favours me.

### Losses

**Primary — brick.** Completing the flow detaches the resolver and the subregistry of the victim's real name. The name stops resolving, and every subname beneath it stops resolving with it. This follows from the victim merely finishing a flow they were told they were authorised to perform; the attacker does not need to control the recipient field or influence anything after step 3. The repo labels these operations irreversible.

**Escalation — hijack.** If the victim is induced to enter an attacker-chosen recipient, the `transfer-token` step moves the **real** name's token to that address. This additionally requires social engineering, so I claim it as an escalation rather than the primary impact.

**Secondary — the flow's own safety property is voided.** `set-eth-addr` exists to point the name's ETH address at the recipient "so it can no longer resolve to you". Because it writes to the raw node while the transfer moves the normalised one, the real name keeps resolving to the previous owner after the transfer completes.

### Relationship to my other submission

I have separately reported a canonicalisation split in **`apps/manager`** **`/renew/$name`** (escalated for fixing). The underlying class is related, and I am disclosing that rather than leaving it to be discovered. They are distinct issues: different application, different route, different sink (`ETHRegistrar.renew()` argument versus an ownership gate followed by detach/transfer calls), and different impact (fund misdirection versus authorisation bypass and brick). Fixing the manager's renew loader does not touch any file cited here.

### Recommended fix

1. **Apply the gate that already exists.** Move `isValidEnsName` into the `/$name` **layout** ([`routes/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name.tsx)) so every leaf inherits it, or normalise and `redirect()` to the canonical name in `beforeLoad`. The PoC shows the existing validator would have rejected the input.
2. **Derive one canonical name once** and thread that single value through the gate, the permission check and all four steps — in particular make `setEthAddress` take the same normalised label the other three use, so the flow is at least internally consistent.

## References

All repository links are permalinks pinned to commit [`1c9b47f`](https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae).

**The gate**

* [`apps/portal/src/routes/$name.tsx#L29`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name.tsx#L29) — `isValidEnsName` used only inside the TLD-redirect branch
* [`apps/portal/src/routes/$name/index.tsx#L442`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/index.tsx#L442) and [`#L459`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/index.tsx#L459) — the real rejection, on the index leaf only
* [`apps/portal/src/routes/$name/ownership/transfer.tsx#L22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L22), [`#L27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L27), [`#L104`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L104), [`#L106-L110`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L106-L110)
* [`apps/portal/src/features/profile/hooks/useEnsOwner.ts#L34`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/useEnsOwner.ts#L34) and [`#L68`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/useEnsOwner.ts#L68)
* [`apps/portal/src/utils/ens/resolveEnsOwner.ts#L49-L50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/ens/resolveEnsOwner.ts#L49-L50) — raw name to the chain reads

**The steps**

* [`apps/portal/src/features/transfer/hooks/useTransferName.ts#L113`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L113) — normalised `tokenId`
* [`apps/portal/src/features/transfer/hooks/useTransferName.ts#L131-L143`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L131-L143) — raw `common` and normalised `label` in one function
* [`apps/portal/src/features/transfer/helpers/setEthAddress.ts#L37-L47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/setEthAddress.ts#L37-L47) — raw name into the resolver write
* [`packages/l2-primary/src/utils/setForwardResolution.ts#L55-L59`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/l2-primary/src/utils/setForwardResolution.ts#L55-L59) — `setAddrParameters` ⇒ `namehash(raw)`
* [`apps/portal/src/utils/token/getLabel.ts#L10-L20`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/getLabel.ts#L10-L20) — `ens_normalize` + `ens_split`
* [`apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L59`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L59) — normalised permission check

**Irreversibility and the UI's claims**

* [`apps/portal/src/routes/$name/ownership/transfer.tsx#L134-L138`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/ownership/transfer.tsx#L134-L138) — "the transfer flow runs irreversible detach"
* [`apps/portal/src/features/transfer/components/SendNameForm.tsx#L49-L57`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx#L49-L57) — "Detaches this name's resolver / registry"

**Dedup and prerequisites**

* [`apps/portal/src/utils/token/isNormalized.ts#L39`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNormalized.ts#L39) — `isValidEnsName`, the validator that exists
* [`apps/portal/src/utils/token/isNormalized.ts#L59-L65`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNormalized.ts#L59-L65) — the repo documenting this spoofing vector
* [`apps/portal/src/utils/token/isNameOrAddress.ts#L9-L10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNameOrAddress.ts#L9-L10) — registrars gate on length, not ENSIP-15

**Specifications**

* ENSIP-15 (ENS Name Normalization) — <https://docs.ens.domains/ensip/15>
* UAX #15 (Unicode Normalization Forms, NFKC) — <https://unicode.org/reports/tr15/>
* Unicode General\_Category `Cf` — <https://www.unicode.org/reports/tr44/#General\\_Category\\_Values>

## Proof of Concept

**11/11 assertions pass, \~1.5 s, offline.**

Every node in this PoC is produced by **running the repository's own code** — the gate (`getEnsOwner` / `getEnsOwnerQueryOptions`), the label derivation (`getLabel`), the validator (`isValidEnsName`) and the actual write-request builder (`createSetForwardResolutionRequest`, from which the written node is read out of the encoded arguments). Nothing is re-implemented or transcribed. Only the chain reads are mocked, as an in-memory table that records **which name each read was asked about** — that recording is the evidence for which node the gate authorised.

### Run

```bash
# put `f9-transfer-split.poc.test.ts` in `apps/portal/src/f9-transfer-split.poc.test.ts`
cd apps/portal
npx vitest run src/f9-transfer-split.poc.test.ts --reporter=verbose
```

### What each assertion establishes

| Test                                         | Establishes                                                                                                                                            |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| the portal HAS the right validator           | `isValidEnsName(raw)` is `false` — the control exists and would have caught this; it is simply not applied on this route                               |
| the gate resolves the RAW node               | The real `getEnsOwner` is executed; the chain reads were asked about the **lookalike**, and it answers "you own it", so the authorisation check passes |
| set-eth-addr writes to the RAW node          | The real request builder's `args[0]` equals `namehash(raw)`                                                                                            |
| detach / transfer target the NORMALIZED node | Real `getLabel(raw)` ⇒ `vitalik`, i.e. `namehash(vitalik.eth)` — the victim's real name                                                                |
| **THE SPLIT**                                | Authorised node ≠ executed node, and step 1 ≠ steps 2–4                                                                                                |
| contrast                                     | An already-normalised name keeps every hop on one node — a divergence, not a blanket mismatch                                                          |
| four divergent cases                         | The split reproduces for the soft hyphen **and** for the ligature, circled digit and fullwidth letter                                                  |
| format-character check                       | Three of the four carry no Unicode `Cf` character                                                                                                      |

### Observed output

```
 ✓ F9 — transfer authorises node A, writes to node B > the portal HAS the right validator — it is simply not applied here 158ms
 ✓ F9 — transfer authorises node A, writes to node B > the authorisation gate resolves ownership of the RAW node 2ms
 ✓ F9 — transfer authorises node A, writes to node B > set-eth-addr writes to the RAW node (the authorised one) 2ms
 ✓ F9 — transfer authorises node A, writes to node B > detach-resolver / detach-registry / transfer-token target the NORMALIZED node 1ms
 ✓ F9 — transfer authorises node A, writes to node B > THE SPLIT — one approved flow spans two different nodes 2ms
 ✓ F9 — transfer authorises node A, writes to node B > contrast — an already-normalized name keeps every hop on one node 1ms
 ✓ F9 — the split is not confined to invisible characters > authorised node differs from executed node for U+00AD soft hyphen (format char) 2ms
 ✓ F9 — the split is not confined to invisible characters > authorised node differs from executed node for U+FB01 ligature (ﬁ) 1ms
 ✓ F9 — the split is not confined to invisible characters > authorised node differs from executed node for U+2460 circled digit (①) 1ms
 ✓ F9 — the split is not confined to invisible characters > authorised node differs from executed node for U+FF56 fullwidth letter (ｖ) 1ms
 ✓ F9 — the split is not confined to invisible characters > three of the four carry no Unicode format character 1ms

 Test Files  1 passed (1)
      Tests  11 passed (11)
   Duration  1.54s
```

### Complete PoC source

```ts
/**
 * PoC F9: the portal transfer flow authorises ownership of one ENS node and
 * then executes irreversible writes against a different one.
 *
 * Surface: apps/portal  /$name/ownership/transfer
 *
 * Every value below is produced by RUNNING the repository's own code:
 *   - isValidEnsName        utils/token/isNormalized.ts
 *   - getEnsOwner / …QueryOptions
 *                           features/profile/hooks/useEnsOwner.ts  (the gate)
 *   - getLabel              utils/token/getLabel.ts                (the steps)
 *   - createSetForwardResolutionRequest
 *                           packages/l2-primary/src/utils/setForwardResolution.ts
 *
 * Nothing is re-implemented or transcribed. Only the chain reads are mocked, as
 * an in-memory table that records which name each read was asked about.
 */
import { sepoliaWithEns } from '@ens-apps/indexer/chain'
import { okAsync } from 'neverthrow'
import { namehash } from 'viem'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const VICTIM = 'vitalik.eth'
/** U+00AD soft hyphen renders as nothing, so the URL looks like VICTIM. */
const RAW = 'vitalik­.eth'

const VICTIM_OWNER = '0x1111111111111111111111111111111111111111'
const REGISTRY = '0x3333333333333333333333333333333333333333'
const RESOLVER = '0x4444444444444444444444444444444444444444'
const RECIPIENT = '0x5555555555555555555555555555555555555555'

/** ENSIP-9 coin type for Ethereum, as used by the real set-eth-addr step. */
const MAINNET_COIN_TYPE = 60

/**
 * The victim owns BOTH names: their real one, and the lookalike the attacker
 * airdropped to them (an unsolicited ERC-721 transfer needs no consent). That
 * is what makes the authorisation gate pass.
 */
const CHAIN_OWNER: Record<string, string> = {
  [VICTIM]: VICTIM_OWNER,
  [RAW]: VICTIM_OWNER,
}

/** Names each chain read was asked about — the evidence for which node won. */
const ownerReads: string[] = []
const registryReads: string[] = []

vi.mock('@/lib/wagmi/helpers', () => ({
  // resolveEnsOwner reads `client.chain` for the legacy-registry address before
  // any lookup, so the stub client carries the app's real chain config.
  safeGetClient: () => okAsync({ chain: sepoliaWithEns } as never),
}))
vi.mock('@ensdomains/ensjs/public/v2', () => ({
  getOwner: async (_c: unknown, { name }: { name: string }) => {
    ownerReads.push(name)
    return (
      (CHAIN_OWNER as Record<string, string>)[name] ??
      '0x0000000000000000000000000000000000000000'
    )
  },
  getNameRegistries: async (_c: unknown, { name }: { name: string }) => {
    registryReads.push(name)
    return ['0x0000000000000000000000000000000000000000', REGISTRY]
  },
}))
vi.mock('@ensdomains/ensjs/public/v1', () => ({
  getOwner: async () => null,
}))

// ---- real repository code under test -------------------------------------
import { createSetForwardResolutionRequest } from '@ens-apps/l2-primary/utils/setForwardResolution'
import {
  getEnsOwner,
  getEnsOwnerQueryOptions,
} from '@/features/profile/hooks/useEnsOwner'
import { getLabel } from '@/utils/token/getLabel'
import { isValidEnsName } from '@/utils/token/isNormalized'

/**
 * The node the `set-eth-addr` step actually writes to, obtained by running the
 * real request builder and reading the namehash out of its encoded arguments.
 */
function setEthAddrNode(name: string): string {
  const request = createSetForwardResolutionRequest({
    name,
    coinType: MAINNET_COIN_TYPE,
    resolverAddress: RESOLVER as `0x${string}`,
    targetAddress: RECIPIENT as `0x${string}`,
  }) as { args: readonly unknown[] }
  return String(request.args[0])
}

/** The node the detach / transfer steps target: `getLabel(name)` + `.eth`. */
const stepNode = (name: string) => namehash(`${getLabel(name)}.eth`)

describe('F9 — transfer authorises node A, writes to node B', () => {
  beforeEach(() => {
    ownerReads.length = 0
    registryReads.length = 0
  })

  it('the portal HAS the right validator — it is simply not applied here', () => {
    // routes/$name.tsx:29 imports isValidEnsName but only uses it inside the
    // TLD-redirect branch, so no leaf under /$name/ inherits a rejection.
    expect(isValidEnsName(RAW)).toBe(false) // it WOULD have caught this
    expect(isValidEnsName(VICTIM)).toBe(true)
  })

  it('the authorisation gate resolves ownership of the RAW node', async () => {
    // transfer.tsx:27  useQuery(getEnsOwnerQueryOptions({ name }))  — raw param
    const options = getEnsOwnerQueryOptions({ name: RAW })
    expect(JSON.stringify(options.queryKey)).toContain(RAW)

    const result = await getEnsOwner({ name: RAW })
    expect(result.isOk()).toBe(true)

    // The chain was asked about the lookalike, and it answers "yes, you own it",
    // so transfer.tsx:104 isAddressEqual(address, data.owner) passes.
    expect(ownerReads).toEqual([RAW])
    expect(registryReads).toEqual([RAW])
    expect(result._unsafeUnwrap()?.owner).toBe(VICTIM_OWNER)
  })

  it('set-eth-addr writes to the RAW node (the authorised one)', () => {
    expect(setEthAddrNode(RAW)).toBe(namehash(RAW))
  })

  it('detach-resolver / detach-registry / transfer-token target the NORMALIZED node', () => {
    // useTransferName.ts:132  const label = getLabel(name)
    expect(getLabel(RAW)).toBe('vitalik')
    // …which is the victim's REAL name, not the one that was authorised.
    expect(stepNode(RAW)).toBe(namehash(VICTIM))
    expect(stepNode(RAW)).not.toBe(namehash(RAW))
  })

  it('THE SPLIT — one approved flow spans two different nodes', async () => {
    await getEnsOwner({ name: RAW })

    const authorisedNode = namehash(ownerReads[0]) // what the gate checked
    const executedNode = stepNode(RAW) // what 3 of 4 steps write to

    expect(authorisedNode).toBe(namehash(RAW))
    expect(executedNode).toBe(namehash(VICTIM))
    expect(authorisedNode).not.toBe(executedNode)

    // And the flow is not even internally consistent: step 1 goes one way,
    // steps 2-4 the other.
    expect(setEthAddrNode(RAW)).not.toBe(stepNode(RAW))
  })

  it('contrast — an already-normalized name keeps every hop on one node', async () => {
    const result = await getEnsOwner({ name: VICTIM })
    expect(result.isOk()).toBe(true)

    expect(ownerReads).toEqual([VICTIM])
    expect(namehash(ownerReads[0])).toBe(stepNode(VICTIM))
    expect(setEthAddrNode(VICTIM)).toBe(stepNode(VICTIM))
  })
})

/**
 * The same pre-emption used for the renew finding: this is a canonicalisation
 * disagreement, not the permissive-charset issue recorded as EXP-INPUT-005.
 * ENSIP-15 applies NFKC compatibility mappings; the gate path applies none.
 */
describe('F9 — the split is not confined to invisible characters', () => {
  const CASES = [
    { desc: 'U+00AD soft hyphen (format char)', raw: RAW, canonical: VICTIM, formatChar: true },
    { desc: 'U+FB01 ligature (ﬁ)', raw: 'ﬁnance.eth', canonical: 'finance.eth', formatChar: false },
    { desc: 'U+2460 circled digit (①)', raw: '①.eth', canonical: '1.eth', formatChar: false },
    { desc: 'U+FF56 fullwidth letter (ｖ)', raw: 'ｖitalik.eth', canonical: 'vitalik.eth', formatChar: false },
  ] as const

  it.each(CASES.map((c) => [c.desc, c] as const))(
    'authorised node differs from executed node for %s',
    (_desc, c) => {
      // The gate never normalises, so it authorises namehash(raw) …
      expect(namehash(c.raw)).not.toBe(namehash(c.canonical))
      // … while the steps run getLabel(), which does.
      expect(stepNode(c.raw)).toBe(namehash(c.canonical))
      expect(setEthAddrNode(c.raw)).toBe(namehash(c.raw))
      expect(setEthAddrNode(c.raw)).not.toBe(stepNode(c.raw))
    },
  )

  it('three of the four carry no Unicode format character', () => {
    for (const c of CASES) expect(/\p{Cf}/u.test(c.raw)).toBe(c.formatChar)
    expect(CASES.filter((c) => !c.formatChar)).toHaveLength(3)
  })
})
```
