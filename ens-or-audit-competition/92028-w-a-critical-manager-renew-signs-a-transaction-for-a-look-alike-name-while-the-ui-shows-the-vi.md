# 92028 w a critical manager renew signs a transaction for a look alike name while the ui shows the victim s real name

**Submitted on Sep 6th 2026 at 13:20:09 UTC by @PrincessBottle23 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92028
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager builds the on-chain `renew(label, ...)` calldata from the raw URL label without ENSIP-15 normalization, while its eligibility gate reads on-chain state for the *normalized* name, and no route checks whether the connected wallet owns the name. An attacker registers a look-alike such as `nick` + U+200B (zero-width space), which renders identically to `nick.eth`, and sends the victim an ordinary in-app link. The page shows "Renew nick.eth" with the real name's price and expiry and no ownership warning, and the wallet signs a transaction that renews the attacker's node. The victim pays to extend the attacker's registration; their real name stays unrenewed and, if it was expiring, can lapse.

## Vulnerability Details

Items 1 and 2 are the integrity primitive: the same user input is a normalized identity on the read path and a raw identity on the write path. Item 3 is the amplifier: it removes the only control that could have surfaced the mismatch to the victim.

**1. No ENSIP-15 normalization.** `parseName` (`name-parser.ts:33-82`), the only input check on the renew, register, and profile routes, does `trim()`, `toLowerCase()`, and an ASCII blocklist:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
const normalized = name.trim().toLowerCase()
if (/\s/.test(normalized)) return ParseNameError.err('SPACE_NOT_ALLOWED')
...
return ok({ subLabels: labels, label, tld })
```

U+00AD, U+200B, and U+2060 pass, contribute no glyph, and are mapped away by `ens_normalize`. `apps/portal` in the same repository already checks `ens_normalize(label) === label` (`isNormalized.ts:18`); the Manager does not.

**2. The gate reads the normalized name; the display and calldata use the raw label.**

```ts
// profileExpiry.ts:77,88 (route gate): normalized
const ethName = normalizeEth2LdName(name)            // viem normalize() inside
... getOwner({ name: ethName.name })
// PricingDomainHeader.tsx:9 (heading): raw
const name = `${label}.eth`
// renewalUi.machine.ts:349 -> registration.actors.ts:1025-1029 (calldata): raw
label: submission.label
args: [label.replace('.eth', ''), duration, paymentToken, REFERER_ADDRESS]   // renew(...)
```

For `nick\u200B.eth` the gate and the page use the real `nick.eth` (owner, expiry, price) while the wallet signs `renew("nick\u200B", ...)`, node `0xb8ad7bd6...`, not `0x05a67c0e...`.

**3. No ownership check on the path.** The `/renew/$name` loader (`routes/renew/$name.tsx:19-64`) gates only on parse, protocol, grace period, and `canRenewV2Name`; `/renew-v1/$name` reads `profileOwnerQuery` but uses only `.protocol`. `features/renew/workflow/` has no `isOwner` or `ownerOf` reference. The app's "You won't gain ownership" dialog exists but is wired only on the profile page's Renew button (`ProfileRenewAction.tsx:66`), and even there it does not protect the victim:

```ts
// ProfileView.tsx:100,119: isOwner is derived from profileOwnerQuery(name), which normalizes (profileOwner.ts:25)
const isOwner = useIsOwner({ owner })   // owner of the REAL nick.eth == victim -> true -> no dialog
// ProfileRenewAction.tsx:83: raw name into /renew/$name
params={{ name }}
```

The dialog fires only for a bystander who owns neither name.

**Attack.**

1. Attacker registers `nick\u200B.eth` under the same protocol as the victim's `nick.eth` (the register route runs the same `parseName`).
2. Attacker sends the victim `/renew/nick%E2%80%8B.eth` or `/nick%E2%80%8B.eth`.
3. The page shows `nick.eth`, the real name's price and expiry, and no dialog. The victim signs; the attacker's node is renewed and the real name is untouched.
4. The success screen shows "Renewal Complete!" for `nick.eth`; "Back to profile" links to the raw look-alike (`RenewalDetails.tsx:116`).

Cyrillic and bidi characters make viem's `normalize()` throw and are rejected on the renew routes; BOM and NBSP are removed by `trim()`. The three invisible-format characters above are the reliable set.

**Remediation.** Reject `normalize(input) !== input` in the `/$name`, `/renew/*`, and `/register/*` loaders, as `apps/portal` does. An ownership check only helps if it compares against the owner of `namehash(rawLabel + '.eth')`, the node the calldata targets; the existing normalized `isOwner` cannot catch this case.

## Impact Details

The boundary violated is not wallet cryptography but the application's transaction-intent integrity: the app displays, prices, and confirms one ENS node while constructing calldata for another, and requests the signature itself.

Per exploit: the connected wallet signs a `renew` whose `label` targets a node other than the one the app displayed, priced, and confirmed, on the app's own signature request, with its own third-party warning suppressed. The victim's selected renewal payment is applied to the attacker-controlled look-alike rather than the displayed name, and the victim's real name is not renewed while the UI says it was. As a downstream consequence, a victim who renews because the name is expiring has no reason to act again; once the grace period passes, the name enters premium auction and can be taken by anyone, a permanent loss of the name and everything bound to it.

Scope facts: the fee goes to the ENS registrar, not the attacker's wallet (the attacker's gain is a free renewal of the squat, denial of the victim's renewal, and a chance at the lapsed name). The victim must open a first-party Manager link and proceed; the page shows only the real name's data and, on the profile page, the victim as owner, so there is nothing to notice. No takeover of the real name's records or ownership is possible: every mutation is authorized per node on-chain. On a fork of the deployed Sepolia ENSv2 contracts the full chain executes end to end (attacker registers `nick\u200B.eth`; the victim's renew extends that node by 365 days for $8.000021 USDC while the real `nick.eth` expiry moves 0 seconds); the registrar prices the zero-width label as available while the real name reverts `NameNotAvailable`.

**Distinction from known issues.** **EXP-INPUT-005 (Medium)** states: *"ENS name validators accept inputs that ENS normalization later rejects or transforms - homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot - causing homograph-style display confusion in the UI rather than incorrect resolution."* Its scope is display confusion and explicitly excludes incorrect resolution. This finding is the excluded case: the connected wallet signs a `renew` whose `label` resolves to a different on-chain node than the one displayed and priced. The forked execution against the deployed Sepolia contracts shows the victim's payment extending the attacker's node by 365 days while the displayed name's expiry moves 0 seconds. That is incorrect resolution and a misdirected transaction, mapping to the in-scope Critical impacts "malicious interactions with an already-connected wallet: modifying transaction arguments" and "display an incorrect name or transaction detail so that a user submits a transaction to an unintended recipient." While the input-acceptance primitive (Defect 1) overlaps EXP-INPUT-005, the read-normalized / write-raw split and the missing ownership check (Defects 2 and 3) are what turn that primitive into a signed transaction against the wrong node and are not described by it.

**WEB-626 (High)** concerns the opposite warning failure: a false-positive third-party-renewal warning shown when the user actually owns the name. Here, the warning is falsely suppressed for the victim because `isOwner` resolves the look-alike to the victim's real `nick.eth`, allowing the victim to proceed without the warning on a name they do not own.

## References

Commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` on `immunefi-team/audit-comp-ens`.

* `parseName`: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L33-L82>
* `/renew/$name` loader: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L18-L64>
* Gate normalizes: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L73-L88>
* Calldata from raw label: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1020-L1031>
* Dialog wired only on the profile button: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx#L66-L84>
* Profile `isOwner` from the normalized owner: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileView.tsx#L99-L119> and <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileOwner.ts#L24-L25>
* In-repo baseline (`apps/portal`): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNormalized.ts#L16-L22>
* ENSIP-15: <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/0xSHKWON/37f8b2e9f0ad68f529f9a1a6f2bd241e>

## Proof of Concept

## Proof of Concept

Scripts, recorded outputs, and the screenshot are in the linked gist. `\u200B` below denotes the zero-width space (`%E2%80%8B` in URLs); U+00AD and U+2060 behave the same. Commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### On-chain, end to end (fork of the real Sepolia ENSv2 deployment)

`onchain-fork-exploit.sh` forks Sepolia at the addresses the app uses (`ETHRegistrar 0xa88553F4...`, `PermissionedRegistry 0xBDC85dD5...`, `USDC 0x768F4245...`), has the attacker register `nick\u200B.eth`, then has the victim renew it. Recorded output (`onchain-fork-exploit-output.txt`):

```
getRegisterPrice("nick")        -> reverted NameNotAvailable("nick")   # the real name is registered (the victim's)
getRegisterPrice("nick+U200B")  -> 8000021 0                           # the look-alike is available and priced at 8.000021 USDC

registered. getExpiry(look-alike) = 1820234667

BEFORE  look-alike expiry=1820234667  nick expiry=1801817044  victim USDC=2000000000
AFTER   look-alike expiry=1851770667  nick expiry=1801817044  victim USDC=1991999979

look-alike (attacker) expiry delta = 31536000 s   # +365 days, funded by the victim
nick.eth   (victim)   expiry delta = 0 s          # the victim's real name is untouched
victim USDC spent                  = 8000021 uUSDC # $8.000021
```

This settles the two facts a triager would probe: the deployed registrar imposes no normalization and prices/accepts the zero-width label (so the attacker can register it), and a `renew` carrying the raw label extends the attacker's node while the victim's real name does not move. `makeCommitment` on the live contract also returns different commitments for `nick` and `nick\u200B`, i.e. the contract commits to the raw label bytes.

### In the running app

`pnpm install && pnpm --filter manager dev`, commit as above.

1. Victim owns `nick.eth`. Attacker (any other wallet) registers `nick\u200B.eth` via `/register/nick%E2%80%8B.eth`; the register route runs the same `parseName`, so it is accepted (same protocol as the victim's name).
2. Victim opens `/renew/nick%E2%80%8B.eth`, or the profile `/nick%E2%80%8B.eth` and clicks Renew. The heading reads `nick.eth`; the price and expiry are the real name's; the "You won't gain ownership" dialog does not appear. On the profile page the victim is even shown as the owner, because `profileOwnerQuery` normalizes the name before the on-chain lookup.
3. Continue to the wallet and decode the pending calldata. `renew(label, ...)` carries `label` = `6e69636b e2808b` (`nick\u200B`), targeting `namehash("nick\u200B.eth")` = `0xb8ad7bd6...`, not `namehash("nick.eth")` = `0x05a67c0e...`.
4. Control: a third wallet that owns neither name opens `/nick%E2%80%8B.eth` and does get the dialog. The guard keys on the normalized name's owner, so it fires for bystanders and never for the victim.

### Supporting (no wallet needed)

`node core.mjs` (real `viem@2.56.3` + `@adraffy/ens-normalize@1.11.1`) shows the split deterministically:

```
LEGIT: label="nick.eth"  node=0x05a67c0ee82964c4f7394cdd47fee7f4d9503a23c09c38341779ea012afe6e00
[U+200B]  parseName.label = "nick\u200B"  (survives: true)   DISPLAYED = "nick\u200B.eth"
          CALLDATA node   = 0xb8ad7bd667c7862c80e98ce0194b98b4f15be232cf3912e998021655cea6c12e
          GATE node       = 0x05a67c0ee82964c4f7394cdd47fee7f4d9503a23c09c38341779ea012afe6e00  (canonical "nick.eth")
```

`render.png`: `nick.eth` and the three look-alikes all render at 116.48 px in Chromium. `renew-redirect-proof.txt`: greps showing no owner check in `features/renew/workflow` or `routes/renew/$name.tsx`, and `ProfileView.tsx:119` deriving `isOwner` from the normalized owner.
