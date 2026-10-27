# 89360 w a critical the name profile route resolves one url parameter as two ens names rendering an attacker s receiving address under the victim s genuine owner and expiry

**Submitted on Aug 18th 2026 at 19:35:17 UTC by @rev3rse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89360
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Attacker can make the Manager "direct users to transfer funds to an address other than the intended one", which your program overview names as priority one:

> "Our primary concern is any bug that could lead to a loss of user funds. In particular, vulnerabilities where the application could:
>
> * direct users to transfer funds to an address other than the intended one"

The verb is *direct*, and the Manager does the directing. The `/$name` route resolves a single URL path parameter as **two different ENS names** and renders both in one document. Three of the four profile services normalize the parameter first; `profileRecords.ts` does not. The identity block therefore shows the **victim's** real owner and expiry, while every content field, including the address under **"Main receiving address"**, comes from a node the **attacker** controls.

## Vulnerability Details

### Root cause

`profileRecords.ts` performs **zero ENS-name normalization**, while `profileOwner.ts` (3 references), `profileExpiry.ts` (2) and `profileRegistration.ts` (2) each call `normalizeEthName`, `normalizeEth2LdName` or `normalizeDnsName`. One directory, four services, one parameter, and the fourth omits the step.

`profileRecords.ts:44-49` keys the indexer on the **raw string**:

```ts
function fetchIndexerDomain(name: string): Promise<DomainQuery> {
  ...
  const promise = indexerClient
    .query<DomainQuery>(DomainDocument, { id: name })                      // L49  RAW
```

`profileOwner.ts:69-70` keys **the same document** on the namehash of the **normalized** name, with `const ethName = normalizeEthName(params.name)` at `profileOwner.ts:25`:

```ts
      indexerClient
        .query<DomainQuery>(DomainDocument, { id: namehash(ethName.name) }) // L70  NORMALIZED
```

and `profileRecords.ts:122-124` hands the raw string straight to ensjs:

```ts
  const records = yield* fromPromise(
    getRecords(client, {
      name,                                                                // L124 RAW
```

Two services, one document, two different keys. No reading in which the raw name is deliberate survives that. Nothing downstream rescues it either: ensjs `getRecords` faithfully resolves whatever string it is handed, which is correct library behaviour.

The route never canonicalises the parameter: `routes/$name/route.tsx:5-13` tests `name.length < 3 || !name.includes('.')` and nothing else, `index.tsx:52,55,173` passes that string to `<ProfileView name={name} />`, and `ProfileView.tsx:93-112` fans one `name` variable out to all four services. The render then splits the two answers: `ProfileDetails` shows Owner and Expires from the normalized lookup, while `ProfileCards` hands the raw-derived records to `ProfileAddressesSection.tsx:159,171-173`, whose `MainAddressCard` sits under "Main receiving address". `CopyableButton.tsx:47` puts the full address in the button `title` and `:37` writes it to the clipboard.

Your own code states the invariant, and names mixed case explicitly, at `NamesTable.tsx:84-85`:

```
// Use the canonical normalized name for the on-chain label — the display name
// may be unnormalized (e.g. mixed case) and would hash to the wrong label.
```

`.greptile/config.json` carries rule `normalize-ens-names`, requiring user-supplied ENS names to "pass through `normalize` from `viem/ens` before being used in contract calls or queries", scoped `apps/**/*.ts`. `profileRecords.ts` is inside that glob. Your automated reviewer was configured to catch this.

### Attack

{% stepper %}
{% step %}

## Victim owns `pocvictim….eth`

The victim owns `pocvictim….eth`.
{% endstep %}

{% step %}

## Register and configure the decoy

I register the title-cased decoy `Pocvictim….eth`, deploy a resolver through the app's own factory path, then `setAddr(node, 60, attackerEOA)` and set a `description` text record.
{% endstep %}

{% step %}

## Send the profile URL

I send the victim `https://<manager>/Pocvictim….eth`, the form a name takes in a tweet, a DM or a marketplace listing.
{% endstep %}

{% step %}

## Victim sees mixed profile data

The victim reads the identity block, sees the real owner and the real expiry, copies the address under "Main receiving address", and pays me.
{% endstep %}
{% endstepper %}

## Impact Details

This is impact **C11**, quoted from your severity matrix:

> "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves"

The first bullet describes this literally. Avatar, header image, About description, socials, links and the Main receiving address all derive from the raw parameter, so **the attacker supplies the entire visible profile and the application credentials it with the victim's genuine owner and expiry.** The identity block is the one thing a careful user can check, it is truthful, and that is what makes the address beside it persuasive. Cost is one decoy registration, reusable against every visitor. The amount lost is chosen by whoever pays, so I state no figure.

One framing point, because it decides how this should be classified. This is **not** a lookalike or impersonation attack, and no homoglyph is involved. Under ENSIP-15 `Vitalik.eth` and `vitalik.eth` are **the same name**: normalization folds case, which is why every conformant ENS client resolves both to one node. The decoy exists on chain only because the registrar performs no normalization, so `keccak256("Vitalik")` and `keccak256("vitalik")` are distinct labels. The attacker therefore does not register a name that *resembles* the victim's. They register a string the protocol treats as *identical to it*, and the Manager then resolves identity from one and records from the other. The deception is not performed by anything the attacker controls; it is performed by this application, on your domain, using your own resolution code, and the victim's trust is in you rather than in the attacker.

## References

1. Program severity matrix, impact C11 (`critical`), "Causing the application to display or pre-fill an incorrect address, name, or transaction detail ... resolving or rendering an ENS name to an attacker-controlled address". Supports the severity claimed.
2. Program overview, "Prioritized vulnerabilities": "direct users to transfer funds to an address other than the intended one,". Supports why this class is your priority one.
3. Known-issues preamble carve-in, "new consequences of a listed root cause that materially change its severity", read against entry EXP-INPUT-005 (Medium, "display confusion ... rather than incorrect resolution"). Supports the dedup argument.
4. ENSIP-15, ENS Name Normalization, <https://docs.ens.domains/ensip/15>. Supports the claim that no correct validator rejects mixed case.
5. `apps/manager/src/features/profile/service/profileRecords.ts:49,95,123-124`. The defect: the raw name goes to the indexer and to `getRecords`.
6. `apps/manager/src/features/profile/service/profileOwner.ts:11,25,30,70`, `profileExpiry.ts:22,77`, `profileRegistration.ts:17,66`. The three siblings that normalize the same parameter.
7. `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-85` and `.greptile/config.json` rule `normalize-ens-names` (scope `apps/**/*.ts`). Your own statement of the invariant, naming mixed case.

## Link to Proof of Concept

<https://gist.github.com/almog-shachar/b116dc87e12581222b7fc365670d5d24>

## Proof of Concept

`poc.patch` adds one self-contained file, `apps/manager/src/__ens_poc__/profile-name-split.poc.test.tsx`. It mounts the shipped `<ProfileView name="Pocvictim….eth" />` with **one string prop** through the repo's own RTL harness (`@/utils/test-utils`) against a local anvil Sepolia fork, and asserts against the **rendered DOM**, not against variables. Names, resolvers and records are created on the fork through the registrar's own commit and register flow and the app's own `buildSetResolverCall`. **Local fork only. No live-chain write is performed.**

From a fresh clone:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install
git apply /path/to/poc.patch

anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 18547 &

cd apps/manager
pnpm vitest run src/__ens_poc__/profile-name-split.poc.test.tsx
```

Sharing gist link here again: <https://gist.github.com/almog-shachar/b116dc87e12581222b7fc365670d5d24>

Fork at latest; pinning `--fork-block-number` needs an archive RPC. `POC_FORK_RPC` overrides the URL. The run writes `src/__ens_poc__/poc-run.log` plus the rendered subtree as `rendered-split.html` and `rendered-split.txt`.

Result: `Test Files 1 passed (1)`, `Tests 5 passed (5)`. Verbatim from `poc-run.log`:

```
------------------------------------------------------------------------------------------------
ONE ROUTE PARAMETER:  /Pocvictimh2qfjj.eth
rendered by:          <ProfileView name="Pocvictimh2qfjj.eth" />   (ONE prop)
------------------------------------------------------------------------------------------------
RENDERED DOM — identity region (ProfileDetails.tsx)
   name badge   : "Pocvictimh2qfjj.eth"
   Owner  row   : "key_verticalOwner0x1e73...43B7"
   Registered   : "(absent)"
   Expires row  : "historyExpiresAUG.18.2027AUGUST 18, 2027"
------------------------------------------------------------------------------------------------
RENDERED DOM — address region (ProfileAddressesSection.tsx)
   "Main receiving addressPocvictimh2qfjj.eth0xa7E6...1396"
   button title : "0xa7E67cCab0e1C1E9A9A675c6febbeb407A131396"
RENDERED DOM — About region (ProfileAbout.tsx, raw-name text record)
   description  : "Send payment to the address below."
------------------------------------------------------------------------------------------------
   victim   EOA = 0x1e73A4B0d0bee941eD4B9d77Ba54E10E5B1843B7   -> truncated "0x1e73...43B7"
   attacker EOA = 0xa7E67cCab0e1C1E9A9A675c6febbeb407A131396   -> truncated "0xa7E6...1396"
   namehash("pocvictimh2qfjj.eth") = 0x0b373d429c0454ea03d0daa451b9ed4f3f1fc1e8412ac1631717a493fce3c82b
   namehash("Pocvictimh2qfjj.eth") = 0xa58f3233e73bd77f8058366d3a82d60ef50c1d38a2038efebf749fd1f62c4257
------------------------------------------------------------------------------------------------
```

Owner and Expires are the victim's. The Main receiving address, its copy-button title and the About text are the attacker's. One prop, one rendered document, both halves on the same screen.

`outerHTML` of the address region, abridged (full text in `rendered-split.txt`):

```html
<h3 class="mb-3 text-ens-quartz-600 text-sm leading-normal">Main receiving address</h3>
<button ... title="0xa7E67cCab0e1C1E9A9A675c6febbeb407A131396" type="button">
  ... <span ...>Pocvictimh2qfjj.eth</span> ... <span ...>0xa7E6...1396</span>
```

Your services, called directly rather than ensjs, same transcript:

```
getProfileRecords("Pocvictimh2qfjj.eth")  addr(60) = 0xa7E67cCab0e1C1E9A9A675c6febbeb407A131396
getProfileRecords("pocvictimh2qfjj.eth")  addr(60) = 0x1e73A4B0d0bee941eD4B9d77Ba54E10E5B1843B7
indexer queries issued (id variable verbatim):
   Domain  id = "Pocvictimh2qfjj.eth"
   Domain  id = "pocvictimh2qfjj.eth"

getOwner       ("Pocvictimh2qfjj.eth") -> 0x1e73A4B0d0bee941eD4B9d77Ba54E10E5B1843B7 (protocol v2)
getExpiry      ("Pocvictimh2qfjj.eth") -> 1818616121
indexer ids seen: []
```

Those recorded `id` values are the direct observation of `profileRecords.ts:49`: raw strings, never a namehash. On the identity path the indexer is never consulted, which proves the harness did not produce the owner result.

**Positive control:** the benign name renders **its own** address; `getByTitle(victim)` resolves and `getByTitle(attacker)` does not.

**Negative control:** a name with no resolver and no `addr` renders **no address card at all** (`ProfileAddressesSection.tsx:163` returns `null`), so nothing is inherited from a previous render.

`getRegistration` errors for the decoy **and** for the genuine name on this fixture, so the `Registered` row renders for neither, as the transcript shows. **I claim Owner and Expires only.**
