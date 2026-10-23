# 89464 w a insight ens explorer reverse resolution certifies an unverified default reverse name as the verified l2 primary name

**Submitted on Aug 19th 2026 at 13:29:53 UTC by @mas4 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89464
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief/Intro

The Explorer's reverse-resolution view (`/addr/$address/reverse-resolution`) shows a green "This is the primary name on {chain}" certification per L2, taken from the address's inherited `default.reverse` name without a forward-resolution check. `DefaultReverseRegistrar.setName(string)` is permissionless and needs no forward record, so any address can set `default.reverse` to a name it does not own (`vitalik.eth`) and be certified as its L2 primary.

### Vulnerability Details

Verification requires bidirectionality: the name must forward-resolve to the address. The inherited-default path skips it.

```ts
const isInheritingDefault = !name && !!defaultName && !isL1   // line 80
const isPrimaryName = forwardMatch || isInheritingDefault     // line 83  <-- no forward check on default
```

[computeDisplayNameState.ts#L80-L83](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts#L80-L83)

The inherited `defaultName` is fetched with its match bit discarded (`allowMismatch: true`), and `isPrimaryName` drives the green banner. A name set *directly* on the L2 instead yields `isPrimaryName = false` and the red "does not resolve back" banner.

#### Exploitation

{% stepper %}
{% step %}

#### Set an unverified default reverse name

Attacker `A` calls `DefaultReverseRegistrar.setName("vitalik.eth")`. Permissionless, writes only `A`'s reverse slot, no ownership or forward record needed.
{% endstep %}

{% step %}

#### Leave L2 reverse records unset

`A` sets no L2 reverse records, so every L2 row inherits its L1 default.
{% endstep %}

{% step %}

#### Open the reverse-resolution page

Victim opens `/addr/A/reverse-resolution`.
{% endstep %}

{% step %}

#### Receive an incorrect primary-name certification

`forwardMatch` is false (no forward `addr(vitalik.eth)` record points to `A`) but `isInheritingDefault` is true, so `isPrimaryName` is true and `A` is certified as `vitalik.eth` on all five L2s.
{% endstep %}
{% endstepper %}

### Impact

The Explorer vouches for an attacker-controlled address as the verified L2 primary of a name it does not own, which can make that address pass as a trusted identity in payment-address or social-engineering confusion. Unauthenticated, one `setName` call, any name. Low: it is the reverse address-to-name surface, not the forward send path, which stays L1-anchored.

### Recommendation

Carry the inherited default's forward-match bit into the decision instead of discarding it: use the `.match` result of the `default.reverse` lookup in `useReverseResolution.ts` (drop `allowMismatch: true`) and gate:

```ts
const isPrimaryName = forwardMatch || (isInheritingDefault && defaultForwardMatch)
```

so an inherited-but-unverified default renders the same red / unverified state as a directly-set one.

## References

* [`computeDisplayNameState.ts#L80-L83`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts#L80-L83): forward check skipped
* [`useReverseResolution.ts#L236-L241`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts#L236-L241): default fetched `allowMismatch: true`
* [`ReverseResolutionSidebar.tsx#L645-L661`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx#L645-L661): green vs red banner
* ENSIP-19: L2 reverse resolution, `default.reverse` inheritance

## Proof of Concept

The test imports the real `computeDisplayNameState` and asserts the attacker's unverified inherited default is classified as primary on an L2 while the same name set directly is rejected.

```ts
// Reverse-resolution primary-name verification bypass.
// Place next to computeDisplayNameState.ts (apps/portal/src/utils/reverseResolution/) so the
// ./computeDisplayNameState import resolves to the real module, then:
//   pnpm --filter portal exec vitest run src/utils/reverseResolution/reversePrimaryBadge.test.ts --reporter=verbose
import { describe, expect, it } from 'vitest'
import { computeDisplayNameState } from './computeDisplayNameState'

describe('reverse-resolution primary-name verification bypass', () => {
  const optimismReverseChainId = 10 // an L2 reverse registrar chain id

  it('classifies an UNVERIFIED inherited default as the verified primary on L2 (the bug)', () => {
    // Attacker-set default.reverse for a name they do not own, no L2 record: the L2 row inherits it.
    const inherited = computeDisplayNameState({
      name: null,
      defaultName: 'vitalik.eth',
      forwardMatch: false,
      reverseRegistrarChainId: optimismReverseChainId,
    })
    expect(inherited.isInheritingDefault).toBe(true)
    expect(inherited.isPrimaryName).toBe(true) // renders the green "primary name" banner

    // The same name set directly is correctly rejected.
    const directlySet = computeDisplayNameState({
      name: 'vitalik.eth',
      defaultName: null,
      forwardMatch: false,
      reverseRegistrarChainId: optimismReverseChainId,
    })
    expect(directlySet.isPrimaryName).toBe(false)
  })
})
```

Run (Node v22 + pnpm v10), after copying the file next to `computeDisplayNameState.ts` in `apps/portal/src/utils/reverseResolution/`:

```
$ pnpm --filter portal exec vitest run src/utils/reverseResolution/reversePrimaryBadge.test.ts --reporter=verbose

 RUN  v4.1.5 /home/user/ens1-repo/apps/portal
 ✓ src/utils/reverseResolution/reversePrimaryBadge.test.ts > reverse-resolution primary-name verification bypass > classifies an UNVERIFIED inherited default as the verified primary on L2 (the bug) 1ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

The passing test confirms `isPrimaryName` is `true` for the unverified inherited default and `false` for the same name set directly, which is the condition that renders the green banner at `ReverseResolutionSidebar.tsx:645`.
