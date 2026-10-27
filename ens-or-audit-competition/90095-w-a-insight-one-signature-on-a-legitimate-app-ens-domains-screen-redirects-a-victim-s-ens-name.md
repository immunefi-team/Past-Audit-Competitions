# 90095 w a insight one signature on a legitimate app ens domains screen redirects a victim s ens name to the attacker the app builds the redirect from an attacker chosen unverified record and ne&#x20;

**Submitted on Aug 22nd 2026 at 23:36:19 UTC by @GaelicThunder for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90095
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Impact sentence

One signature by the victim — clicked inside the Portal's own normal "set primary name" workflow, on the legitimate app domain — repoints the Ethereum-address (coin type 60) resolution of `victimname.eth` to the attacker's address, because the app uses an attacker-chosen, **unverified** reverse record as the *write-target parameter* of a `setAddr` transaction, offers that write to **any connected wallet regardless of who owns the page being viewed**, and never shows the receiving address anywhere in its human-readable confirmation.

## Precondition — stated up front

**The attack requires exactly one signature by the victim (the name's owner). Everything up to that signature is permissionless and fully attacker-controlled; everything after it is immediate and silent.** We state this first because it is the load-bearing fact for both severity and scope: it is why we submit at High rather than Critical, and it is the fact a triager will weigh against the out-of-scope rule — argued from both sides below.

After the signature there is no second step, no delay and no attacker dependency: forward resolution for the coin type simply resolves to the attacker. The victim retains ownership and can manually restore their records once they notice, so we do **not** claim permanence — the loss window covers payments, exchanges or contract interactions that resolve the name's Ethereum address between that signature and the victim's notice.

## The attack, as an attacker performs it

Every pre-signature step is an ordinary, permissionless operation:

1. Attacker EOA calls `ReverseRegistrar.setName("victimname.eth")` (L1, and/or the L2 reverse registrar). Both registrars are msg.sender-keyed and accept any string — the app's own comment: *"both accept `setName(string)` for any UTF-8 string (V1 name, V2 name, subname, DNS-imported, even a non-existent name), so we don't gate the input on existence or protocol version at all"* (`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/hooks/useReverseResolutionMutations.ts:79-83`). On-chain proof below.
2. For the deception layer (step 4), the attacker additionally calls `DefaultReverseRegistrar.setName("victimname.eth")` — the same msg.sender-keyed `setName(string)` primitive class, likewise callable by any EOA. The on-chain proofs below demonstrate this primitive on both the L1 `ReverseRegistrar` and the app-pinned ENSIP-19 `DefaultReverseRegistrar` (Transcript C, read back through the app's own `allowMismatch:true` path).
3. Victim opens `/addr/<ATTACKER>/reverse-resolution`. This is a first-class public explorer route for *any* address; the header reads only "Reverse resolution" (`apps/portal/src/routes/addr/$addr/reverse-resolution.tsx:108-127`) — the page never states whose records are being shown. No attacker-crafted URL is needed to get there: the public Recent Activity table links registration actors to their address pages, and every address page carries an enabled "Reverse Resolution" sidebar tab (route-graph evidence in the Scope rule section below).
4. The page renders a self-consistent lie: rows without their own record inherit the spoofed default and show hardcoded **"Forward match: True" + "Primary name"** badges (`.../ReverseResolutionTable/columns.tsx:96-112` — the row's actual `forwardMatch=false` is ignored), plus the green banner *"This is the primary name on \<chain>"* in the row detail panel (`.../ReverseResolutionSidebar.tsx:645-652`, driven by `isPrimaryName = forwardMatch || isInheritingDefault`, `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:83`).
5. On the row holding the attacker's own record with the victim's name, the sidebar shows the honest red mismatch banner — **and directly below it, the fix**: `canSetAsPrimary === true` renders the **"Set primary name"** button for **any connected wallet** (`ReverseResolutionSidebar.tsx:619-640`; disabled only on not-connected/pending state).
6. Victim clicks → confirm modal titled "Set primary name", transactionName *"Set primary name to victimname.eth"* (`ReverseResolutionSidebar.tsx:751-753`). **The receiving address appears nowhere at app level** — not in the title, not in the confirmation text, not in the page header.
7. Victim signs. The submitted transaction is `setAddr(namehash("victimname.eth"), coinType, <ATTACKER>)` against the victim name's own resolver — built from the **route parameter** as `targetAddress` (`useReverseResolutionMutations.ts:121-131`, invoked at `ReverseResolutionSidebar.tsx:758`). The victim is the authorized signer (they own the name), so resolver authorization passes and the write succeeds.
8. A dapp, exchange or payment flow resolving `victimname.eth` for coin type 60 now receives the attacker's Ethereum address.

### Why the missing gate is consequential, not cosmetic

The sibling surface does it right: `apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx:450-458` gates its set-primary action on `isAddressEqual(connectedAddress, data.address)` with the explicit rationale *"Offered only when the connected wallet \*is\* this address (setName is msg.sender-scoped)"*. The reverse sidebar's write is **not** msg.sender-scoped — `setAddr` takes an arbitrary `targetAddress` — which is precisely why it needed the identity check and doesn't have one. The gate function `computeDisplayNameState` takes no viewer input at all; the check (`computeDisplayNameState.ts:88`) is only `canSetAsPrimary = !!displayName && !isPrimaryName && name !== null`.

## Root cause chain (file:line, all verified at HEAD `1c9b47f`)

1. **Unverified read.** `useReverseResolution.ts:77` and `:239` read reverse records with `allowMismatch: true`; `:236-241` discards the `match` field and spreads the raw name onto every row, including rows with no record of their own.
2. **False verification display.** `columns.tsx:96-112` hardcodes "True / Primary name" badges onto inherited-default rows while ignoring `row.original.forwardMatch`; `computeDisplayNameState.ts:83` (`isPrimaryName = forwardMatch || isInheritingDefault`) classifies a spoofed inherit as the verified primary. The helper's own unit test asserts this as intended behaviour — the design trusts `default.reverse` as if it were written by the name's owner. It is writable by anyone (proof below).
3. **No identity gate on a cross-user write.** `computeDisplayNameState.ts:88` has no connected-account input; `ReverseResolutionSidebar.tsx:619-640` gates the button on connection state only.
4. **Write parameterized by the viewed route address and the unverified name.** `useReverseResolutionMutations.ts:121-131` builds `createSetForwardResolutionRequest({ name: displayName /* unverified reverse row */, coinType, resolverAddress, targetAddress: address /* route param */ })`.
5. **Receiving address absent from the human confirmation.** `ReverseResolutionSidebar.tsx:751-753` + `routes/addr/$addr/reverse-resolution.tsx:108-127`.

The sponsor's own guidance describes the correct behavior — the official ENS blog (*How ENS Normalization Works*, ens.domains): *"Before displaying a primary name, an app verifies that the claimed name is valid and normalized, then performs forward resolution to confirm that it points back to the same address… This prevents an arbitrary name claim from being presented as a verified ENS identity."* Here the data layer explicitly requests the opposite (`allowMismatch: true`), the UI presents the arbitrary claim as verified (hardcoded True/badges/banner), and the arbitrary claim then parameterizes a fund-steering transaction.

## Link to Proof of Concept

<https://gist.github.com/GaelicThunder/22d51ef525b1e3babd6893f3914de0e4>

## Proof of Concept

## Proof of concept — executed

**Level 1 — the app's code paths (15/15 vitest tests, 5 files, fresh run 2026-08-22, `poc/output.txt`).** They execute the real application code — `computeDisplayNameState`, `createSetForwardResolutionRequest` from `@ens-apps/l2-primary/utils`, `prepareSetForwardResolutionTransaction` from the portal's own transaction-manager helpers, and the shipped column definitions rendered through `react-dom/server` — not re-implementations:

| File                                                | Tests | What it proves                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------------------------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `zz-poc-unverified-reverse-setprimary.poc.test.tsx` | 2/2   | (a) `canSetAsPrimary=true` for the mismatched unverified row, with the deciding function provably taking no viewer input; (b) decoding the exact request the flow submits gives `args == [namehash('victimname.eth'), 60n, ATTACKER]` — victim's node, attacker's address.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `zz-poc-real-sidebar-render.test.tsx`               | 3/3   | **The REAL shipped `ReverseResolutionSidebar` rendered server-side** for the mismatched row with a connected wallet: R1 — the honest red mismatch banner AND the ungated "Set primary name" button both render from real JSX; R2 — the data handed to the confirm modal names only the target name (`title='Set primary name'`, `transactionName='Set primary name to victimname.eth'`) and contains no receiving address — not even truncated hex, no `'0x'` substring at all; R3 — across the entire rendered surface the receiving address appears in no full form (checksummed/lowercase/bare hex all absent); the sole remnant is exactly ONE occurrence of the disclosed 5+4 truncation (`0xf39...2266`, the Primary-name row's address badge), outside the modal payload. Data-layer hooks are inert stubs (disclosed in-file); every asserted string is constructed inside the component under test. |
| `zz-poc-chained-spoof-and-write.test.tsx`           | 3/3   | The chained page state: (a) **row-scoped assertions** on the real rendered columns — the Base Sepolia L2 row itself contains "Primary name"+True while its own Default row and the attacker's own-record row each contain False (the fabrication is located to specific rows, not merely present somewhere in the table); (b) the same page yields `canSetAsPrimary=true` and builds `setAddr(namehash('victimname.eth'), 60, ATTACKER)`; (c) coherence — the name displayed as verified elsewhere IS the write target, and the receiving address is invisible app-side.                                                                                                                                                                                                                                                                                                                                     |
| `zz-poc-spoofed-default-reverse.test.tsx`           | 2/2   | The deception substrate alone: shipped column definitions render hardcoded "True / Primary name" for an inherited default whose forward match is false, while the honest False shows on the source row.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `zz-poc-wallet-popup-decode.poc.test.ts`            | 5/5   | Executes the shipped request builders and proves the wire-format facts used below: the transaction carries no ABI alongside the EIP-1193 request; the readable ENS name becomes a `bytes32` namehash; the target address is encoded in the third `bytes` argument; and a genuine self-service request differs in that target word. This does **not** claim that every wallet hides the address: a wallet with the correct ABI or simulation may expose it.                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

**Level 2 — the write primitive, executed on-chain** (anvil fork of Sepolia, chainId 11155111, real deployed contracts from the app's own ensjs config; transcript with verbatim commands and outputs: `poc/transcript-reverseregistrar-setname.txt`). A plain role-less EOA claims `victimname.eth` as its own reverse record on the L1 `ReverseRegistrar` (`0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6`), which stores it verbatim — including an UNREGISTERED name:

```
$ cast call $REVREG "node(address)(bytes32)" $ATTACKER
0x79c28201d2a1bfe6ce2b93152486b2950f5803f962642f1c46c6543761058bf0
$ cast call 0x000…0dE1 "recordExists(bytes32)(bool)" $NODE
false                                    # ← attacker holds no reverse record

$ cast send --unlocked $REVREG "setName(string)" "victimname.eth" --from $ATTACKER
status       0x1  (success)
txHash       0x11e5bc252b399a6258db9f8d51681864aca0f2b545bf7b8e542a108118b5f2fd

$ cast call 0xe996…49b5 "name(bytes32)(string)" 0x79c2…8bf0     # readback
"victimname.eth"                         # ← stored verbatim, returned to the app

$ cast call 0x000…0dE1 "owner(bytes32)(address)" $(cast namehash victimname.eth)
0x0000000000000000000000000000000000000000   # ← claimed name doesn't even exist
```

The same primitive holds on the **app-pinned ENSIP-19 DefaultReverseRegistrar** (`0x4f382928805ba0e23b30cfb75fc9e848e82dfd47` — hardcoded by the app itself at `reverse-resolution/config.ts:34-35`; transcript with verbatim commands and outputs: `poc/transcript-default-revreg-setname.txt`). There a NON-controller EOA (zero roles, owns nothing) claims `victimname.eth`, and the record is read back through the app's exact read path (`useReverseResolution.ts:236-240` → ensjs `getName` with `coinType = 0x80000000`, `allowMismatch: true`) — before the write that read is a bare revert (no record → app shows nothing); after it the UniversalResolver returns custom error `ReverseAddressMismatch(string,bytes)` carrying `"victimname.eth"` — precisely the error ensjs's `parseReverseAddressMismatchError` converts into `{name, match:false}`, which is the unverified name the app then spreads onto the rows:

```
$ cast send --unlocked $DRR $DATA --from $CALLER   # DATA = setName("victimname.eth"), prebuilt calldata
status               1 (success)
transactionHash      0x4007fc4e82a192f202d96dbc542ef3e5db0a4ef531a1e527b00cccef61ebd784

$ cast call $DRR "nameForAddr(address)(string)" $CALLER
"victimname.eth"

$ cast call $UNIVERSAL_RESOLVER "reverse(bytes,uint256)(string,address,address)" $CALLER 0x80000000
Error: … execution reverted: custom error 0xef9c03ce: …76696374696d6e616d652e657468…
#      0xef9c03ce = ReverseAddressMismatch(string,bytes); args decode to ("victimname.eth", …)
```

**Level 3 — THE LOSS, EXECUTED END-TO-END ON-CHAIN** (transcript with verbatim commands and outputs: `poc/transcript-loss-executed.txt`). This closes the loop from signature to impact — nothing below is narrated:

1. The VICTIM (`0x7099…79C8`) registers `victimname.eth` through the production v2 `ETHRegistrarController` commit/reveal flow and points it at their own dedicated resolver (deployed via the deployed VerifiableFactory, mirroring the project's own e2e fixture).
2. The victim signs **ONE** transaction — the exact request shape the app's button builds:

```
$ cast send --unlocked $RESOLVER \
    $(cast calldata "setAddr(bytes32,uint256,bytes)" \
      0x253f32e0d26c4cb51eaf649b50a6d60fa0f22f65475c06d44f45c788f8032586 \
      60 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266) --from $VICTIM
status   0x1  (success)          # ← resolver authorization passed: signer owns the name
txHash   0x1818b00fdf48496d130153a2a63f3ac8a41331a7e40bc669f3e40be3cc12a729
```

3. Forward resolution now returns the ATTACKER — including through the ENSIP-10 UniversalResolver that conforming dapps actually call:

```
$ cast call $RESOLVER "addr(bytes32,uint256)(bytes)" $NODE 60
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266            # ← ATTACKER

$ cast call $UNIVERSAL_RESOLVER "resolve(bytes,bytes)(bytes,address)" \
    0x0a76696374696d6e616d650365746800 \
    0x3b3b57de253f32e0d26c4cb51eaf649b50a6d60fa0f22f65475c06d44f45c788f8032586
#   ^ dnsEncode("victimname.eth")     ^ addr(bytes32) selector + namehash(victimname.eth)
0x000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266   # returned bytes: ATTACKER
0x20eA25A0559Af7C27ee9e2D1DE5daDB1d02a846F                          # resolved by: the victim's resolver
```

### Level 3b — current fresh-fork authorization control

The canonical standalone runner (`poc/run-transcripts.sh`) recreates the entire state from a fresh Sepolia fork and records the result in `poc/rerun-out/transcript-rerun-current.txt`. The separate clean-tree application suite executes and decodes the shipped `createSetForwardResolutionRequest` and `prepareSetForwardResolutionTransaction` builders, asserting that they produce `setAddr(namehash(name), 60, routeAddress)`. The runner then executes those same contract arguments across the real authorization boundary:

1. **The record pays the VICTIM before the attack** — the current output records `[PASS] DECISIVE D1: BEFORE record resolves to VICTIM`, so the write demonstrably repoints live resolution away from its rightful target, not from an empty record.
2. **Mined authorization negative control:** the designated attacker EOA, which has no resolver role in this setup, sends the same resolver address and calldata later sent by the victim; the transaction is mined with status 0 and leaves the record unchanged. The victim owner then sends that same call and it succeeds. This demonstrates that resolver authorization works and that this is not a direct contract-auth bypass. It does not claim that a separately authorized delegate could not write; it proves that the tested zero-role attacker cannot perform the write without obtaining an authorized victim signature.
3. **The production builders are executed independently** by the 15/15 clean-tree suite, which decodes their output and asserts the exact function, namehash, coin type and route-address arguments used by this flow.
4. **AFTER is read through ENSIP-10**: `UniversalResolver.resolve()` returns the attacker with `resolvedBy` = the victim's resolver.

Harness plumbing disclosed: mock-USDC mint to the victim, `anvil_setCode` on canonical Anvil accounts (Sepolia has EOF bytecode at those well-known addresses), local harness funding, and a time advance past `MIN_COMMITMENT_AGE`. The project's own fixture uses the same account-code clearing at `makeName.ts:209`. `poc/run-transcripts.sh` performs and labels every setup step on a fresh fork before registering the target name through the production controller. None grants a protocol role. The attacker-signed transaction is intentionally broadcast, mined and reverted.

### Evidence scope — stated plainly

What is executed: the gating logic, the production transaction builders (decoded calldata with every argument asserted), the REAL rendered sidebar surface (component-level SSR of the shipped source), the permissionless write primitive, **and the resolution-redirection chain on-chain — one victim signature flips ENSIP-10 forward resolution of the victim's name from the victim's own record to the attacker** (`poc/rerun-out/transcript-rerun-current.txt`: the record pays the victim before the attack, an attacker-signed byte-identical calldata request is MINED and REVERTED as the negative control, and the victim's signature on that same call performs the flip). What is asserted-by-construction rather than driven through a live browser+wallet: the victim physically clicking the button on `/addr/<attacker>/reverse-resolution`. That final step is deterministic given the executed artifacts — the click opens the modal whose exact bound strings we captured from the real component render, the button's handler builds the request whose calldata we decoded. Level 3b proves that the corresponding resolver call succeeds when sent by the victim owner and flips forward resolution from the victim's record to the attacker. In the same setup, the identical call request sent by the tested zero-role attacker is mined and reverts. The control establishes the authorization boundary; the finding is that the UI induces the authorized owner to submit the wrong target, not that the attacker can bypass resolver authorization directly. We chose honesty over inflation: this is code-path-executed plus on-chain-executed evidence; only the physical click is not screen-recorded.

## Scope rule — argued from both sides

The program excludes: *"Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows."*

**Why this finding is in scope:** the victim performs a *prompted*, *in-app* action that **is** the normal app workflow. "Set primary name" on the reverse-resolution panel is exactly what that button exists for; the flow is view → offered button → app's own confirm modal → signature, with no dev tools, no console, no other-tab edits. Compare the shape that the exclusion is *for* (an out-of-band user edit mid-migration, unrelated to the flow): here nothing outside the app's own rendered workflow is required. And the flagship impact language — resolving an ENS name to an attacker-controlled address — inherently contemplates user-side signatures; a reading of the rule that excluded any UI-mediated signature would leave that flagship category empty. The sponsor's own catalogue treats UI-trust bugs as reportable (EXP-INPUT-005, Medium), not as out-of-scope.

**The other side, honestly:** a triager may still argue that setting a primary name while viewing an unfamiliar address is outside the user's intended self-service workflow. If the attack depended on an attacker-supplied URL or instructions, that would also raise the program's phishing/social-engineering boundary. This report does not rely on either: the application exposes an ordinary, entirely in-app route to third-party address pages.

**No attacker-supplied link is required for route reachability.** The Portal homepage always renders its public **Recent Activity** table (`apps/portal/src/routes/index.tsx:18-54`). That table requests 15 global events without an account or address filter and refreshes every 30 seconds (`features/dashboard/hooks/useRecentActivity.ts:25-50,58-63`). Registration events map their `owner` field to an address actor (`features/dashboard/utils/formatActivityEvent.ts:29-39,122-129`), and the table renders that actor as a clickable address badge (`features/dashboard/components/RecentActivityTable.tsx:32-33,84-89`). Address badges navigate to `/addr/$addr` (`components/EntityBadge.tsx:266-275`).

A concrete in-app path is therefore:

1. The attacker performs an ordinary registration, producing a registration event whose "Registered by" actor is the attacker's address.
2. A user browsing the Portal homepage can click that actor in Recent Activity and arrive at `/addr/<attacker>`.
3. Every valid address page mounts `AddrSidebar` (`routes/addr/$addr.tsx:20-24`), whose enabled **Reverse Resolution** item unconditionally links to `/addr/$addr/reverse-resolution` (`components/AddrSidebar.tsx:39-66,115-139`).

This proves organic route reachability through the application's own rendered navigation; it does not claim that every user will choose that actor or sign the subsequent transaction. The residual scope risk is exactly that user-motivation question. Once the user does inspect the route, however, the application itself renders the cross-user action, opens its own confirmation modal, and constructs the write. The missing viewer-versus-route-address gate is load-bearing: the sibling forward-resolution surface performs that check, while this surface does not.

## Boundary against known issues

* **WEB-1301** (Manager address view shows misleading values) — different app, different surface, read-only wrong values, no spoofing primitive and no write.
* **EXP-INPUT-005** (validators accept inputs normalization rejects — homograph *display confusion*) — different mechanism entirely; our attack needs **no confusable characters** (`victimname.eth` is byte-exact), and it ends in an incorrect *write*, not confusion.
* **WEB-1257 / QA-01** — transfer/role-combination axes, unrelated.
* **QA-06** (subname still shown as primary after deletion/stale resolution, profile throws on open) — a stale-*display* defect after a name stops resolving; read-only confusion with an error-page symptom. Our finding needs no deletion and no stale state: the redirected name actively resolves at signature time, and the impact is a live cross-user *write*, not a display.
* No published issue covers: consuming `allowMismatch` reverse rows as transaction parameters, the absent identity gate on a cross-user write, or the hardcoded verification badges on inherited `default.reverse` records.

## On "working as designed"

The helper's own unit test (`computeDisplayNameState.test.ts:91-105`) asserts the inherited-default → `isPrimaryName=true` behavior, so a triager may call this working as designed. Two answers. First, design intent cannot relocate the defect: what the unit test codifies is precisely the unverified-trust rule the sponsor's own blog quote says apps must not perform — a test asserting insecure behavior documents the bug, it does not excuse it. Second, the team's own sibling surface proves this isn't an accepted trade-off: `AddressResolutionSidebar.tsx:450-458` gates its set-primary action on `isAddressEqual(connectedAddress, data.address)` with the in-code rationale *"Offered only when the connected wallet \*is\* this address"* — an identity gate that `computeDisplayNameState` structurally cannot provide because it takes no viewer input at all. One of the two surfaces encodes the correct standard; the other lacks it on a write that is strictly more dangerous.

## What reaches the wallet — and what this proof does not claim

We executed the shipped request builders (`createSetForwardResolutionRequest` → `prepareSetForwardResolutionTransaction`) and decoded the resulting request (`poc/zz-poc-wallet-popup-decode.poc.test.ts`, 5/5). The application submits a zero-value call to the victim name's resolver whose calldata is `setAddr(namehash(displayName), coinType, routeAddress)`.

The application's own confirmation is unambiguous but incomplete: it says "Set primary name to " and does not display `routeAddress` (`ReverseResolutionSidebar.tsx:751-763`). The route address is nevertheless present in calldata as the third `bytes` argument. A wallet with the relevant ABI, transaction simulation, or additional contract context may decode and surface that address; a careful user inspecting fully decoded calldata can therefore detect the mismatch. We do **not** claim that every wallet conceals the destination or that the transaction is impossible to distinguish.

Two narrower wire-format facts are demonstrated:

* the request sent by this application carries no ABI alongside the transaction envelope (`helpers/setForwardResolution.ts:43-59`);
* the human-readable ENS name is not present in calldata: the first argument is its `bytes32` namehash, while the target address is encoded in the third argument.

Thus the app modal supplies the name but omits the destination, while the raw transaction supplies the destination and a namehash rather than a readable name. A sufficiently capable wallet or expert review can mitigate the risk. The security consequence does not depend on universal wallet opacity: the Portal still offers a cross-user action without the identity gate and describes an app-built wrong-target write as an ordinary "Set primary name" operation.

## Honest limits

* The sidebar's "Primary name" row does show the viewed address truncated (5+4 chars, `ReverseResolutionSidebar.tsx:700-710`) — a weak signal that exists somewhere on the page but is absent from the modal and page header where the signing decision happens.
* A wallet's decoded-calldata or simulation view may reveal the attacker's address and could make the mismatch detectable. The raw call carries the name as a `bytes32` namehash rather than readable text; richer off-wallet context may still map it back to a known name. The app confirmation itself shows the name but omits the address.
* The red mismatch banner is honest — but the button immediately below offers the "fix", which *is* the harmful write when the viewer is not the owner; in the chained form the surrounding page corroborates the spoofed identity with fake-verified claims.
* One victim signature required; post-signature loss is the redirection window until the victim notices and restores records (reversible by the owner — no permanence claimed).

## Recommendation

1. Gate the set-primary action on `isAddressEqual(connectedAddress, routeAddress)`, mirroring `AddressResolutionSidebar.tsx:450-458`. Counterfactual check: this single line kills the attack — a victim viewing the attacker's page (connected as themselves) never sees the button, and the attacker gains nothing by viewing their own page.
2. Render the receiving address in the TransactionModal title/confirmation text.
3. Stop discarding the `match` field on `default.reverse` reads; carry it through and render the real match status instead of hardcoded True badges.
4. Adopt the sponsor's own stated order — verify before displaying a primary name, per the blog quote above.
