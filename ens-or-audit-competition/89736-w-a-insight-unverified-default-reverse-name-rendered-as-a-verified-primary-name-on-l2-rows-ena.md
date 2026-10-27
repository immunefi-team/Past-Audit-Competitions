# 89736 w a insight unverified default reverse name rendered as a verified primary name on l2 rows enabling identity spoofing

**Submitted on Aug 20th 2026 at 20:37:45 UTC by @Victor\_Von\_Doom for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89736
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's reverse-resolution page (/addr/

/reverse-resolution) displays ENS names that fail forward verification as if they were verified primary names. The hook that fetches the cross-chain default name (default.reverse, ENSIP-19) calls getName with allowMismatch: true and then discards the returned match flag. Every L2 row without its own reverse record inherits that name and gets stamped with a "True" forward-match badge plus a "Primary name" badge, and the sidebar shows a green "This is the primary name on " banner. Because any address can permissionlessly set any name as its own default.reverse (one transaction on the DefaultReverseRegistrar, no ownership check), an attacker can make the Explorer assert that their address is the verified primary name of vitalik.eth, or any other name they do not own. The page exists specifically so users can verify address-to-name correspondence, so a false "verified" badge here is a false identity assertion and a direct phishing enabler.

## Vulnerability Details

The bug is a chain of four steps. I verified each one in the audited revision (63772fd872af472ced58b009499355f3430c2a86) and reproduced the end result live.

1. The hook fetches the default name and throws away the verification result

apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241

```
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})
const defaultName = defaultResult?.name ?? null
```

allowMismatch: true tells ensjs to return the name even when UniversalResolver.reverse reverts with ReverseAddressMismatch, i.e. when the name does NOT resolve back to the address. I confirmed this in the pinned ensjs source (dist/actions/public/getName.js): on that revert with allowMismatch set, ensjs returns the name with match: false instead of null. The hook keeps only .name. The ReverseResolutionResult type carries forwardMatch and defaultName but has no defaultMatch field, so the verification state of the default name never reaches any renderer.

2. The table stamps inherited names as verified without checking anything

apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:96-112

```
if (!name && defaultName && canInheritDefault(row.original.coinType)) {
  return (
    ...
    <Badge ...><CheckCircle2 .../><span>True</span></Badge>
    <Badge ...><SquareUser .../><span>Primary name</span></Badge>
    ...
```

canInheritDefault excludes only the Default and Mainnet rows, so Optimism, Arbitrum, Base, Linea and Scroll all enter this branch. The branch never reads forwardMatch or any match state belonging to defaultName. The badges are unconditional.

3. The sidebar repeats the assertion

apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:79-83

```
const isInheritingDefault = !name && !!defaultName && !isL1
const isPrimaryName = forwardMatch || isInheritingDefault
```

An L2 row with name = null, defaultName = "vitalik.eth", forwardMatch = false comes out as isPrimaryName: true. The sidebar then renders a green banner "This is the primary name on " (ReverseResolutionSidebar.tsx:645-652) with no further guard.

4. The attacker's write is permissionless and msg.sender-scoped

The Sepolia DefaultReverseRegistrar is 0x4f382928805ba0e23b30cfb75fc9e848e82dfd47 (config.ts:34-35). setName(name) writes the caller's own default.reverse record. There is no ownership check on the name, which is exactly why forward verification exists. The attack is not overwriting a victim's record; it is the attacker pointing their own address's default.reverse at a name that does not resolve back to them.

Why this is not intended design

The inheritance mechanism itself (L2 rows falling back to default.reverse) is fine and is part of ENSIP-19. The defect is that the verification result is discarded and then asserted as True. Four observations:

* The legitimate case never needs allowMismatch: true. If a user owns the name they set as default.reverse, getName forward-verifies and returns match: true. The option is only required to surface names that fail verification, and then the flag saying "this failed" is dropped.
* On the same page, the Default row (which is the default.reverse record itself) shows the name with forwardMatch: False, because there the match flag is kept (useReverseResolution.ts:103). The L2 rows show the same name as True / Primary name. One name cannot simultaneously fail forward verification and be a verified primary name just because it is inherited.
* The sibling feature handles this correctly. useReverseMatch (forward-resolution) separates verified / pending / mismatch / unverifiable and never labels a mismatched name as verified.
* The repo test that locks isPrimaryName: true for inherited defaults (computeDisplayNameState.test.ts:91-105) tests a function that was never given the default name's match state. The flaw is upstream in the hook, which blinds the display layer.

Live reproduction

I ran this end to end on Sepolia against a local instance of the Explorer at the audited revision (public endpoints, no mocks):

1. From EOA 0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12 I called setName("vitalik.eth") on 0x4f382928805ba0e23b30cfb75fc9e848e82dfd47. TX: 0x5e3a032062570048cf06ebf3d66ee7bc4e50d84d59f6590971815f7cd4228dd5 (block 11530743, status success, gas used 46,798).
2. nameForAddr(attacker EOA) now returns "vitalik.eth", while vitalik.eth resolves to 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 on Sepolia. Forward verification fails.
3. /addr/0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12/reverse-resolution renders:
   * Default row: vitalik.eth, forward match False
   * Mainnet row: vitalik.eth, forward match False
   * Optimism, Base, Arbitrum, Linea, Scroll: vitalik.eth, forward match True, badge "Primary name"
4. The Optimism row sidebar shows the green banner "This is the primary name on Optimism" and Primary name: True.

Screenshots of steps 3 and 4 are attached. I also wrote a unit test that drives the real column renderers with the exact hook output shape (defaultName = "vitalik.eth", forwardMatch = false, L2 coin type) and asserts the True / Primary name badges render; it passes 3/3, with controls showing a genuinely verified record renders identically and L1 rows do not inherit.

## Impact Details

An attacker can make the Explorer display their own address as the verified primary name of any ENS name, including names belonging to well-known people, projects, or exchanges. The spoofed rows are visually identical to genuinely verified ones: same True badge, same Primary name badge, same green sidebar banner. There is nothing on the page that tells a viewer the name failed forward verification; the only hint is a small "Default" tag, and the Default row's own False badge sits two rows above five rows saying True.

The realistic abuse is phishing and social engineering. The Explorer is the tool users check before trusting an address. An attacker sends a victim a link to their address page (or a screenshot of it), the page shows vitalik.eth verified as the primary name on five networks, and the victim sends funds to the attacker's address believing it belongs to the name's owner. That maps to the in-scope impact "resolving or rendering an ENS name to an attacker-controlled address."

What I am not claiming: no direct fund loss from the application itself, no transaction pre-fill, no wallet compromise, and the attacker does not gain any control over the name they spoof. Scope-limiting factors, stated plainly: the Explorer currently runs on Sepolia, the false assertion lives on the reverse-resolution sub-tab (the main address header uses wagmi's useEnsName, which forward-verifies), and the surface is display-only. The severity comes from the false verification assertion itself, on the exact page whose job is verification, exploitable with one permissionless transaction and zero victim interaction.

## References

Code (audited revision 63772fd872af472ced58b009499355f3430c2a86):

* apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241 (allowMismatch: true, match flag discarded)
* apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:54-63, 96-112 (unconditional True / Primary name badges)
* apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:79-83 (isPrimaryName = forwardMatch || isInheritingDefault)
* apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:645-652 (green banner)
* apps/portal/src/features/reverse-resolution/config.ts:34-35 (Sepolia DefaultReverseRegistrar address)
* apps/portal/src/utils/reverseResolution/computeDisplayNameState.test.ts:91-105 (repo test locking the unverified inheritance)

Library semantics:

* @ensdomains/ensjs dist/actions/public/getName.js (allowMismatch returns name with match: false on ReverseAddressMismatch)

Spec:

* ENSIP-19 (default.reverse / cross-chain primary names): <https://docs.ens.domains/ensip/19>

On-chain proof (Sepolia):

* TX 0x5e3a032062570048cf06ebf3d66ee7bc4e50d84d59f6590971815f7cd4228dd5 (setName("vitalik.eth") from attacker EOA 0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12)

Attachments:

* BUG ENS.jpg (reverse-resolution table: Default/Mainnet show False, all five L2 rows show True / Primary name for the same unverified name)
* BUG ENS2.jpg (Optimism row sidebar: green banner "This is the primary name on Optimism", Primary name: True)

## Link to Proof of Concept

<https://gist.github.com/JultoshiBoy/1a3281d1d71a2d944669c58b21df07a5>

## Proof of Concept

## Proof of Concept

Overview: any address can set any ENS name as its own default.reverse record (permissionless, one transaction, no ownership check). The Explorer then renders that unverified name as a verified primary name on all L2 rows. Reproduced live on Sepolia against the audited revision.

Prerequisites

* An EOA with a small amount of Sepolia ETH (faucet: sepolia-faucet.com or similar). No special permissions needed.
* The Explorer app from the audited revision, or any deployment of it. It runs with no configuration against public Sepolia endpoints: pnpm install --frozen-lockfile && pnpm --filter portal dev, then open <http://localhost:3001>.

Step 1 — Claim a name you do not own as your default.reverse

From the attacker EOA, call setName("vitalik.eth") on the Sepolia DefaultReverseRegistrar at 0x4f382928805ba0e23b30cfb75fc9e848e82dfd47. This is the same call the app's own UI issues (AddressResolutionSidebar.tsx:545-553), so it can be sent through Etherscan's "Write Contract" tab, a wallet, or viem/cast.

I executed this from EOA 0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12:

TX: 0x5e3a032062570048cf06ebf3d66ee7bc4e50d84d59f6590971815f7cd4228dd5 Block: 11530743 Status: success Gas: 46,798

The registrar does not check whether the caller owns vitalik.eth. It never can: default.reverse is a per-address record keyed on msg.sender.

Step 2 — Confirm the on-chain state

nameForAddr(0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12) returns "vitalik.eth"

And confirm the forward record does not match:

vitalik.eth addr(60) on Sepolia = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 attacker EOA = 0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12

These differ, so forward verification fails. This is the exact condition ensjs reports as match: false.

Step 3 — Open the reverse-resolution page for the attacker address

Observed result (screenshot attached: BUG ENS.jpg):

Default vitalik.eth False Mainnet vitalik.eth False Optimism vitalik.eth True \[Primary name] \[Default] Base vitalik.eth True \[Primary name] \[Default] Arbitrum vitalik.eth True \[Primary name] \[Default] Linea vitalik.eth True \[Primary name] \[Default] Scroll vitalik.eth True \[Primary name] \[Default]

The name that failed forward verification is displayed as False on the Default and Mainnet rows, and as True with a Primary name badge on all five L2 rows. Same name, same page, contradictory verification verdicts.

Step 4 — Open the row detail sidebar

Click "More" on any L2 row (Optimism in my reproduction). The sidebar shows (screenshot attached: BUG ENS2.jpg):

* Green banner with checkmark: "This is the primary name on Optimism"
* Name: vitalik.eth
* Primary name: True

A viewer of this page has no indication that the name failed forward verification. The badges and banner are identical to those shown for a genuinely verified primary name.

Step 5 (optional) — Automated reproduction

I added a unit test that feeds the real column renderers the exact hook output shape (defaultName = "vitalik.eth", forwardMatch = false, L2 coin type) and asserts the True / Primary name badges render:

cd apps/portal npx vitest run src/features/reverse-resolution/components/ReverseResolutionTable/columns.poc.test.tsx

Test Files 1 passed (1) Tests 3 passed (3)

The two control tests show that a genuinely verified record renders the same badges (spoof is indistinguishable) and that L1 rows do not inherit.

Note on the sidebar step: the "More" button only renders when the connected wallet matches the displayed address, so a third-party visitor sees the table (step 3) directly. The sidebar (step 4) is what the address owner sees, and it repeats the same false assertion.
