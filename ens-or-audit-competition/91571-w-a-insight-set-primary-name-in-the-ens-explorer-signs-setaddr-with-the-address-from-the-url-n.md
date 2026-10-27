# 91571 w a insight set primary name in the ens explorer signs setaddr with the address from the url not the connected wallet so an attacker can redirect where any ens name resolves

**Submitted on Sep 2nd 2026 at 18:31:42 UTC by @enspire for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91571
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief/Intro

The Explorer's address page at `/addr/<address>/reverse-resolution` lets you view and, when the record doesn't match, fix an address's reverse resolution with a "Set primary name" button. The button reads a name string off that reverse record and offers to write it as the primary name, but the address it actually writes into the name's resolver is the page's own URL parameter, never checked against the wallet that's about to sign. Since anyone can point their own reverse record at any name string they like, an attacker can make `bob.eth` show up on their own address page, send Bob a link to it, and have Bob's own wallet, doing exactly what the button says, point `bob.eth`'s ETH address record at the attacker.

### Vulnerability Details

The address comes straight from the route, with no ownership check anywhere downstream.

`apps/portal/src/routes/addr/$addr/reverse-resolution.tsx:41`

```ts
const { addr: address } = Route.useParams() as { addr: Address }
```

The row's display name comes from an unverified reverse lookup. `getName` is called with `allowMismatch: true`, so a name is returned even when its forward record doesn't point back to this address.

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:74-78`

```ts
let nameResult = await getName(client, {
  address,
  coinType,
  allowMismatch: true,
})
```

Whether the "Set primary name" button appears is decided by `computeDisplayNameState`, which only looks at whether the name forward-matches. It never takes a wallet address as input at all.

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:88`

```ts
const canSetAsPrimary = !!displayName && !isPrimaryName && name !== null
```

Clicking the button builds the write request from the page's `address`, and that's the exact value the sidebar received as a prop straight from the route.

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:758`

```ts
const request = getForwardResolutionRequest(address)
```

`createSetForwardResolutionRequest` takes that address as `targetAddress` and hands it to ensjs's `setAddrParameters`, which builds `setAddr(namehash(name), coinType, targetAddress)` against the name's own resolver. Nothing in this function checks who is calling it or whether `targetAddress` has anything to do with them.

`packages/l2-primary/src/utils/setForwardResolution.ts:55-64`

```ts
const setAddr = setAddrParameters({
  name,
  coin: coinType,
  value: targetAddress,
})

return {
  address: resolverAddress,
  ...setAddr,
}
```

And the confirmation dialog the user sees before signing only ever renders two strings, the step title and a template that interpolates the name, never the address.

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:752-753`

```ts
title: 'Set primary name',
transactionName: `Set primary name to ${displayName}`,
```

Put together: `setName(string)` on the reverse registrar is a public function with no argument validation, keyed to `msg.sender` on-chain, so any address can claim any name string as its reverse record. The Explorer reads that unverified claim, decides whether to offer a fix based only on whether it matches forward resolution, and if the user acts on it, writes whichever address happens to be in the URL into the name's resolver. The one thing that would make this safe, checking that the URL's address is the address about to sign, is checked nowhere in this chain.

Walking through it with names: Bob owns `bob.eth` and holds resolver-write authority on it. An attacker calls `ReverseRegistrar.setName("bob.eth")` from their own address, call it `0xAAA...`. The attacker sends Bob a link to `https://app.ens.domains/addr/0xAAA.../reverse-resolution`, maybe framed as "check what's set for this address" or embedded somewhere Bob would open it while managing his ENS names. Bob opens it, sees a row that says `bob.eth` next to a red "doesn't resolve back to this name" banner, and a "Set primary name" button. He clicks it, gets a wallet prompt for a transaction titled "Set primary name to bob.eth," and signs. `bob.eth`'s `addr(60)` record now points at `0xAAA...`.

### Impact Details

There's no bounded "one cycle" cost the way a DeFi exploit has a closed-form number, since what's at risk is whatever gets sent to `bob.eth` after the redirect and before Bob notices. ENS names are routinely used as human-readable payment addresses precisely because people trust that `bob.eth` resolves to Bob, so the realistic loss scales with how much the victim's name is used that way, not with any single transaction. The write is also silent from the victim's side: nothing tells Bob his address record changed unless he goes and checks it, and the app's own confirmation step actively withholds the one piece of information (the target address) that would have let him catch it.

I picked "unintended recipient" over "direct theft of user funds" on purpose. The transaction itself moves nothing. What it does is repoint where future payments to `bob.eth` land, and that needs a third party, someone paying Bob, to actually turn into a loss. "Taking state-modifying authenticated actions on behalf of other users without any interaction by that user" doesn't fit either, since Bob has to open the link and sign the transaction himself. What actually happens is the application causing Bob to submit a transaction to an unintended recipient by pre-filling the wrong address, and that impact's own listed example, resolving a name to an attacker-controlled address, describes the end state precisely.

Reachability doesn't need any privilege beyond what each party already has. The attacker calls a public, unauthenticated function on the reverse registrar. Bob needs nothing more than the ordinary resolver-write authority he already holds over his own name, the same authority he'd use for any legitimate record update. And clicking "Set primary name" from a shared address page is the button's actual, intended, in-workflow function. Nothing about the exploit steps outside what the feature is built to do.

The mitigating detail worth stating plainly: the sidebar does show the attacker's truncated address in a badge, under the red mismatch banner, before the button is clicked. That's a real signal a careful user could catch. It just isn't in the one place designed for a final check, the confirmation dialog, which shows only the name.

## References

All pinned to commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` on `audit-comp-ready`.

* [`reverse-resolution.tsx#L41`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/addr/%24addr/reverse-resolution.tsx#L41) — URL param becomes `address`, no ownership check
* [`useReverseResolution.ts#L74-L78`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts#L74-L78) — `getName(..., allowMismatch: true)`
* [`computeDisplayNameState.ts#L88`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts#L88) — `canSetAsPrimary`, no wallet argument
* [`ReverseResolutionSidebar.tsx#L758`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx#L758) — `getForwardResolutionRequest(address)`
* [`ReverseResolutionSidebar.tsx#L752-L753`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx#L752-L753) — confirmation dialog title/transactionName
* [`setForwardResolution.ts#L28-L65`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/l2-primary/src/utils/setForwardResolution.ts#L28-L65) — `createSetForwardResolutionRequest`, builds `setAddr`
* [`useReverseResolutionMutations.ts#L79-L83`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/hooks/useReverseResolutionMutations.ts#L79-L83) — comment confirming `setName` accepts any UTF-8 string for any V1/V2/subname/DNS-imported name
* `node_modules/@ensdomains/ensjs-abi/dist/reverseRegistrar.js` — `setName(string)` ABI, single argument, no access control; installed dependency, not vendored in-tree
* `node_modules/@ensdomains/ensjs/dist/utils/coders/setAddr.js` — `setAddrParameters`, builds `setAddr(namehash(name), coinType, value)`; installed dependency, not vendored in-tree

## Link to Proof of Concept

<https://gist.github.com/ensp1re/5f172102011cd61e69372940b386d02f>

## Proof of Concept

This is a plain vitest unit test against the real, unmodified functions the app calls. No fork, no RPC, no deployed contracts needed, since the whole bug lives in which address string gets passed into a calldata encoder before a wallet ever gets involved. It includes a control case alongside the exploit case: same functions, same shapes, only the target address differs.

### Repository

```
https://github.com/immunefi-team/audit-comp-ens
```

### Reviewed commit

```
1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

This is the current head of `audit-comp-ready`, the branch the Explorer app asset targets.

PoC file, attached alongside this doc: `SetPrimaryName.audit.test.ts`

Secret gist (README plus the PoC file):

```
https://gist.github.com/ensp1re/5f172102011cd61e69372940b386d02f
```

{% stepper %}
{% step %}

### Clone and check out the reviewed commit

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

{% endstep %}

{% step %}

### Install dependencies

```bash
pnpm install --frozen-lockfile
```

{% endstep %}

{% step %}

### Retrieve the PoC test file

```bash
gh gist view 5f172102011cd61e69372940b386d02f -f SetPrimaryName.audit.test.ts > apps/portal/src/SetPrimaryName.audit.test.ts
```

{% endstep %}

{% step %}

### Run the test

```bash
pnpm --filter portal test src/SetPrimaryName.audit.test.ts
```

{% endstep %}
{% endstepper %}

No RPC or archive access is needed anywhere in this.

What actually happened running it, pulled straight from the gist rather than a local copy:

```
> portal@ test audit-comp-ens/apps/portal
> vitest run src/SetPrimaryName.audit.test.ts

 RUN  v4.1.5 audit-comp-ens/apps/portal

 Test Files  1 passed (1)
      Tests  5 passed (5)
```

Control case, decoded:

```
request.address === RESOLVER
decoded.functionName === 'setAddr'
decoded.args[0] === namehash('bob.eth')
decoded.args[1] === 60n
decoded.args[2] === BOB          // Bob's own address, correct
```

Exploit case, decoded:

```
request.address === RESOLVER     // same resolver as the control
decoded.functionName === 'setAddr'
decoded.args[0] === namehash('bob.eth')
decoded.args[1] === 60n
decoded.args[2] === ATTACKER     // not BOB
```

### What the tests show

The control models a completely ordinary, intended use of this exact feature: Bob calls `setName("bob.eth")` on his own address, hasn't set forward resolution yet, so `canSetAsPrimary` is true, and clicking the button correctly writes his own address into `bob.eth`. That passes and produces `decoded.args[2] === BOB`.

The exploit changes exactly one thing: the address in the URL is the attacker's, not Bob's, because the attacker called the identical `setName("bob.eth")` from their own account. Every other input is unchanged, `computeDisplayNameState` returns the same `canSetAsPrimary: true`, and the write goes to the same resolver through the same `setAddr` call. The only difference in the decoded calldata is `args[2]`, which comes out as the attacker's address instead of Bob's. Nothing in the code path distinguished the two cases, because nothing in it ever looks at who is about to sign.

The third test in the exploit block confirms the app gives Bob nothing to catch this with at the moment that matters: the confirmation dialog's `title` and `transactionName` strings are byte-identical between the control and the exploit, both just say "Set primary name to bob.eth."

### Code walkthrough

The address that ends up in the calldata is the route param, unchanged, all the way down:

```ts
// apps/portal/src/routes/addr/$addr/reverse-resolution.tsx:41
const { addr: address } = Route.useParams() as { addr: Address }
```

```ts
// ReverseResolutionSidebar.tsx:758
const request = getForwardResolutionRequest(address)
```

```ts
// packages/l2-primary/src/utils/setForwardResolution.ts:55-64
const setAddr = setAddrParameters({
  name,
  coin: coinType,
  value: targetAddress,
})

return {
  address: resolverAddress,
  ...setAddr,
}
```

`setAddrParameters` (from `@ensdomains/ensjs`) is what actually produces the `setAddr(namehash(name), coinType, value)` triple the test decodes. This ran against the real, unmodified target at the pinned commit. Nothing in the app or its dependencies was changed to produce this result.
