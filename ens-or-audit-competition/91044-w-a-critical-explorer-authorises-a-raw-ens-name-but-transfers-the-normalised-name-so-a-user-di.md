# 91044 w a critical explorer authorises a raw ens name but transfers the normalised name so a user disposing of a decoy burns their genuine name

**Submitted on Aug 29th 2026 at 04:12:10 UTC by @archcypher for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91044
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

### Brief/Intro

The Explorer's transfer flow authorises and displays the **raw** route parameter but derives every write from the **ENSIP-15-normalised** label, so the name it acts on is not the name it shows. An attacker registers a decoy 2LD whose raw label normalises to a victim's real label (`POCTESTENS` → `poctestens`) and pushes it to the victim as an ERC-1155, which an EOA cannot refuse. When the victim opens the decoy's transfer page to dispose of it, every label the Explorer presents names the decoy while the calldata carries the genuine name's token id — so the victim signs their **genuine** `poctestens.eth` away to the address they picked for the junk one, and the junk one stays in their wallet. In production this irrecoverably removes a live `.eth` name from its owner whenever the destination they chose is one-way — a conventional burn address being one such destination.

### Vulnerability Details

The route parameter is split into two different names inside one flow.

**Authorise + display — the RAW parameter**

```ts
// apps/portal/src/routes/$name/ownership/transfer.tsx:27
const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
//   -> utils/ens/resolveEnsOwner.ts:45  getOwnerV2(client, { name })   // no normalisation
// apps/portal/src/routes/$name/ownership/transfer.tsx:104
... isAddressEqual(address, data.owner)      // owner of the DECOY -> AuthorizedTransfer renders
```

**Every write — the NORMALISED label**

```ts
// apps/portal/src/utils/token/getLabel.ts:10
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)     // "POCTESTENS.eth" -> "poctestens"
  const labels = ens_split(normalized)
  ...
}

// apps/portal/src/features/transfer/hooks/useTransferName.ts:113
getEnsTokenId({ label: getLabel(name), registryAddress })   // token id of the GENUINE name
// apps/portal/src/features/transfer/hooks/useTransferName.ts:132
const label = getLabel(name)   // -> detachNameResolver / detachNameRegistry
```

`labelhash("POCTESTENS") = 0xc1e3293b…` and `labelhash("poctestens") = 0xf779601d…` are different names with different owners; the ENSv2 registrar imposes no character constraints on labels, so the decoy is registrable.

**The two guards the team built for this flow are on the wrong side of the split, so both pass.**

* `useCanTransferName.ts:57` calls `getLabel(name)` and then `hasRoles(ROLE_CAN_TRANSFER_ADMIN)`. Its own docstring says it exists so the UI can *"refuse to start such a transfer instead of walking the user into that trap"* — the trap being that *"the transfer flow runs its (irreversible) detach steps before the token `safeTransferFrom`"*. Because it checks the **genuine** name, which the victim does control, it returns true.
* `useTransferDetachTargets.ts:60,63-65` reads roles from the **normalised** label but resolver/registry targets from the **raw** name. `SendNameForm.tsx:92-96` zeroes any hidden option, so this decides the plan.

**Nothing rejects the route parameter.** `routes/$name.tsx:29` uses `isValidEnsName` only to decide the TLD redirect. `routes/$name/ownership/transfer.tsx:16-19` declares `component` and `notFoundComponent` and no validation at all. The "Invalid name" screen at `routes/$name/index.tsx:442,458` is a component-level check inside the **index page**, not a route guard — and `routes/$name.tsx:60-72` renders `ProfileSidebar` unconditionally around the `<Outlet />`, so from that very screen the sidebar's Ownership link reaches the unguarded transfer route in one click. The direct URL works too.

**Ordering.** `buildTransferPlan` emits the config steps before the token move by design — its header states *"Config steps run before the token moves, since the sender loses the roles to make them once the ERC-1155 token transfers."* `SendNameForm.tsx:69-72` defaults all three options to `true`. The final `transfer-token` step is appended unconditionally, so the wrong-name transfer happens whether or not either detach option is offered.

This is not a defect in `@ensdomains/ensjs`; each ensjs call hashes exactly the name it is handed. The defect is that `apps/portal` hands one route parameter to the authorisation, the guards and the writes under two different canonicalisations.

### Impact Details

**Selected impact — Critical:&#x20;*****"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters."*** The Explorer builds a transaction whose `tokenId` argument identifies a different on-chain name than the one it authorised, displayed and labelled. This holds for **every** recipient and requires no assumption about permanence. Concretely, in the run below:

```
tokenId placed in safeTransferFrom  = 111935725446…831040   (poctestens.eth — the genuine name)
tokenId of the name shown on screen =  87697738635…202752   (POCTESTENS.eth — the decoy)
```

The Explorer is the only surface that names the transaction, and it names the wrong one: `useTransferName.ts:177` builds each step's title as `` `${STEP_LABELS[step]} - ${name}` `` from the **raw** route name and `TransactionStateContent.tsx:113` renders it. Nothing downstream maps a `uint256` back to a label, so a user has no way to reconcile the two.

**What the substituted argument costs the user.** The genuine name leaves their wallet to an address they selected for the **decoy** and never authorised for their genuine name. In the demonstrated case that address is `0x000…dEaD`, a conventional burn address, so the name is unrecoverable — a live `.eth` name permanently lost, along with everything resolving through it.

**Loss.** One `.eth` second-level name per victim, irrecoverable where the destination is one-way, plus the resolver and subregistry configuration of that name. Attacker cost is a single registration and gas.

**Stated plainly, so severity is not overstated:**

* **The detach steps are&#x20;*****not*****&#x20;inherently irreversible.** The ERC-1155 transfer carries the name's roles, so whoever holds the token afterwards inherits `ROLE_CAN_TRANSFER_ADMIN`, `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY` and can call `setResolver` / `setSubregistry` again. This is demonstrated in `reversibility-probe.mjs` / `.out`, supplied in the linked gist, where a live recipient restores both. Permanence comes from the one-way destination of the final transfer, not from the detaches.
* **The attacker gains nothing and cannot choose the recipient.** `recipientInput` starts empty (`SendNameForm.tsx:68`), the route takes no search params, and the recipient field is properly validated — `isNameOrAddress.ts:23-25` → `isValidEnsName` rejects any non-normalised label. The irony is the finding: the app applies exactly the right validator to the recipient field and never applies it to the route parameter that selects which name it writes to. **Direct theft of user NFTs / funds is not claimed.**
* The attacker's step is an on-chain ERC-1155 push, not a message to the victim: no leaked credentials and no privileged role. The victim's step is a normal app workflow — disposing of an unwanted airdropped name is what the transfer page exists for.

**Relationship to the published Known Issues.** Nothing in the known-issues document touches `getLabel`, label derivation, or the transfer flow's authorise-vs-write split. **EXP-INPUT-005** is the nearest: it describes permissive name *validators* and bounds its practical impact to *"display confusion in our UI rather than incorrect resolution"*. Here three transactions execute against a different on-chain name than the one authorised and displayed — the materially higher consequence the document expressly keeps eligible (*"new consequences of a listed root cause that materially change its severity"*). **QA-01**'s own text carves in: *"A demonstrated security consequence … would be a new finding."* WEB-1249 / WEB-1257 / WEB-1305 are defects on the correct name; R3-01…R3-08 are transaction-manager liveness.

**Recommended fix.** Reject or redirect a non-normalised route parameter at the `/$name` **layout** route so no child route can act on it, and derive the authorisation, the guards and the write label from the same canonical form.

## References

Code, at the audited revision `63772fd872af472ced58b009499355f3430c2a86`:

* `apps/portal/src/routes/$name/ownership/transfer.tsx` — L16-19 (no validation), L27 (raw owner lookup), L104 (authorisation)
* `apps/portal/src/routes/$name.tsx` — L29 (`isValidEnsName` used only for the TLD redirect), L60-72 (sidebar rendered unconditionally)
* `apps/portal/src/routes/$name/index.tsx` — L442, L458 (the "Invalid name" component check)
* `apps/portal/src/utils/token/getLabel.ts` — L10-20
* `apps/portal/src/utils/ens/resolveEnsOwner.ts` — L45
* `apps/portal/src/features/transfer/hooks/useTransferName.ts` — L113, L132, L177
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts` — L16-19 (docstring), L57
* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts` — L60, L63-65
* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts` — L1-5, L22-41
* `apps/portal/src/features/transfer/components/SendNameForm.tsx` — L68, L69-72, L92-96
* `apps/portal/src/features/transfer/helpers/transferToken.ts` — L66
* `apps/portal/src/features/transaction-manager/.../TransactionStateContent.tsx` — L113
* `apps/portal/src/utils/token/isNameOrAddress.ts` — L11-13, L23-25

Also live in the deployed beta: `https://explorer.ens.dev/assets/getLabel-DwFSWtPI.js` (sha256 `ddd69afb93282492c5db1574718828994f65f1044db048f3e783dcfdfafdf7aa`, fetched 2026-08-28) is byte-for-byte the repo's `getLabel.ts` — minified body `function n(n){let r=e(t(n))[0];if(!r||r.error)throw r?.error;return String.fromCodePoint(...r.input)}`.

ENSIP-15 (ENS Name Normalization): <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/archcypher/b7a62f56754b7fa22f6ace6b422dfb86>

## Proof of Concept

Everything below runs against a **local anvil fork of Sepolia**. No transaction was broadcast to any live network and no live host was touched.

### Step by step

{% stepper %}
{% step %}

## Victim owns the genuine name

The victim owns `poctestens.eth`, with a resolver and (optionally) a subregistry.
{% endstep %}

{% step %}

## Attacker registers the decoy

The attacker registers `POCTESTENS.eth` through the same ENSv2 `ETHRegistrar` — a distinct on-chain name, `labelhash 0xc1e3293b…` versus the genuine `0xf779601d…` — and gives it its own resolver, and a subregistry if they want the second detach step to appear.
{% endstep %}

{% step %}

## Attacker pushes the decoy to the victim

The attacker calls `safeTransferFrom` to push the decoy to the victim. An ERC-1155 transfer to an EOA cannot be refused, so this needs no interaction from the victim at all.
{% endstep %}

{% step %}

## Victim opens the transfer page

The victim opens `/POCTESTENS.eth/ownership/transfer` — directly, or by clicking **Ownership** in the sidebar that renders beside the "Invalid name" screen at `/POCTESTENS.eth`.
{% endstep %}

{% step %}

## The page authorises the raw name while checking the normalised name

The page authorises: `getOwnerV2("POCTESTENS.eth")` returns the victim, so the transfer form renders. `useCanTransferName` then checks roles on `getLabel(...)` = `"poctestens"` — the victim's **genuine** name — and also passes.
{% endstep %}

{% step %}

## Victim enters a destination

The victim types a destination for the junk name. In this run: `0x000000000000000000000000000000000000dEaD`, a conventional burn address. They leave the options at their defaults.
{% endstep %}

{% step %}

## Explorer displays a decoy-labelled transaction plan

The Explorer's transaction-flow modal presents the plan. Every step is labelled with the **decoy**: `Detach resolver - POCTESTENS.eth`, `Transfer name - POCTESTENS.eth`. The victim approves each one.
{% endstep %}

{% step %}

## The genuine name is transferred

Read the result back: `poctestens.eth` — the genuine name — is now owned by `0x000…dEaD`, and `POCTESTENS.eth` — the decoy they were trying to dispose of — is still theirs.
{% endstep %}
{% endstepper %}

### Running it

```bash
anvil --fork-url <sepolia-rpc> --chain-id 11155111

# the script imports the app's own ensjs/viem, so it must run from inside that
# workspace package for pnpm to resolve them:
cp f2-wrong-token-transfer.mjs <repo>/apps/portal/
cd <repo>/apps/portal && node f2-wrong-token-transfer.mjs
# exits 0 on success, 1 on any failed assertion.  POC_RPC overrides the RPC URL.
```

`f2-wrong-token-transfer.mjs` computes the app's own gates rather than assuming them, and asserts every step:

```
=== victim opens /POCTESTENS.eth/ownership/transfer ===
  [RAW ] resolveEnsOwner("POCTESTENS.eth").owner   = 0xf39Fd6e5…92266  VICTIM
  [RAW ] transfer.tsx:104 isAddressEqual(...)      = true   -> AuthorizedTransfer renders
  [NORM] getLabel("POCTESTENS.eth")                = "poctestens"   <-- a DIFFERENT on-chain name
  [NORM] useCanTransferName -> hasRoles("poctestens") = true   -> SendNameForm renders
  => optionIsVisible = {"setEthAddress":false,"detachResolver":true,"detachRegistry":true}
  => buildTransferPlan(defaults) = ["detach-resolver","detach-registry","transfer-token"]
  => modal step titles: "Detach resolver - POCTESTENS.eth", "Detach registry - POCTESTENS.eth",
                        "Transfer name - POCTESTENS.eth"
  [NORM] tokenId in safeTransferFrom   = 111935725446151297825882641984720337851749344781780489356155160692956111831040
         tokenId of the name ON SCREEN =  87697738635922340809029406312496423196774001943699792348753437616037536202752

=== result ===
  poctestens.eth   owner 0xf39Fd6e5…92266 VICTIM -> 0x000…dEaD  BURN ADDRESS
  POCTESTENS.eth   owner 0xf39Fd6e5…92266 VICTIM -> 0xf39Fd6e5…92266 VICTIM

=== assertions ===
  PASS  authorised on the RAW name
  PASS  useCanTransferName passed on the NORMALISED label
  PASS  write label differs from the route name
  PASS  tokenId signed != tokenId of the name on screen
  PASS  plan ends in transfer-token
  PASS  genuine name left the victim
  PASS  decoy stayed with the victim
  PASS  genuine name is now at the burn address
```

### Browser walkthrough

The same flow was driven end-to-end in a real browser against the same fork (`f2-ui-capture.mjs`, Playwright; described in `F2-ui-evidence.txt`; full setup in `REPRO-F2.txt`, which needs no Docker). The eight captures are uploaded via the **Attachments** field.

`F2-ui-05-crop-flow-labelled-decoy.png` is the decisive frame — the Explorer's transaction-flow modal:

```
Transaction flow
  ->  Detach resolver - POCTESTENS.eth
  ->  Transfer name - POCTESTENS.eth
                                       [ Open wallet ]
```

Both steps name the decoy; both carry calldata keyed on `getLabel(name)` = `poctestens`. Read back through the app afterwards:

```
/poctestens.eth/ownership   ->  Owner  0x0000…dEaD    (F2-ui-07)  the genuine name, burned
/POCTESTENS.eth/ownership   ->  Owner  0xf39F…2266    (F2-ui-08)  the decoy, still the victim's
```

The page carries the app's own warning throughout: *"Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding."* The user did check the recipient address. The **name** was never the one on screen.

(The browser seed gives the decoy a resolver but no subregistry, so that run shows the two-step plan; the script above seeds a subregistry too and shows the three-step plan. Both end in the same transfer.)

### Files supplied in the linked secret gist

| File                                   | What it is                                                                                                                    |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `f2-wrong-token-transfer.mjs` / `.out` | **canonical PoC** and its output                                                                                              |
| `reversibility-probe.mjs` / `.out`     | evidence for the permanence paragraph: a live recipient inherits the roles and restores both the resolver and the subregistry |
| `F2-ui-setup-only.mjs`                 | seeds the fork for the browser walkthrough                                                                                    |
| `f2-ui-capture.mjs`                    | the Playwright walkthrough that produced the captures                                                                         |
| `F2-ui-evidence.txt`                   | what each capture shows, and the actors                                                                                       |
| `ui-stub-indexer.mjs`, `REPRO-F2.txt`  | 20-line GraphQL stub and the full repro guide                                                                                 |

### Files uploaded as attachments

`F2-ui-01-transfer-form.png` · `F2-ui-02-recipient-burn-address.png` · `F2-ui-03-steps.png` · `F2-ui-04-flow-labelled-decoy.png` · **`F2-ui-05-crop-flow-labelled-decoy.png`** · `F2-ui-06-flow-complete.png` · `F2-ui-07-aftermath-genuine-name-burned.png` · `F2-ui-08-aftermath-decoy-still-owned.png`
