# 91524 w a critical base58 case folding substitutes the solana copy target

**Submitted on Sep 2nd 2026 at 10:27:05 UTC by @Nyksx for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91524
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The profile receiving-address helpers compare every cryptocurrency address after lowercasing, causing distinct case-sensitive Base58 records to be grouped as one address. A Solana-labelled copy control can return a different, attacker-controlled address and cause an irreversible native SOL transfer to the wrong recipient.

## Finding Description

The Manager profile chooses one record as the main receiving address. If no ETH record exists, `getMainAddressCandidate()` selects the first address returned by the profile data. The UI then calls `getReceivingAddressChains()` to determine which chain icons belong on that main card and `getChainSpecificAddresses()` to decide which other records need separate cards.

Both grouping functions lowercase the address strings before comparing them. Case-insensitive comparison is appropriate for hexadecimal EVM addresses, but it is invalid for Base58 encodings such as IOST and Solana. Uppercase and lowercase letters represent different Base58 digits, so changing one letter can produce a distinct valid public key.

An ENS owner can set IOST coin type 291 to an attacker-controlled Solana public key beginning with `FVen...` and set the authoritative SOL coin type 501 record to a distinct valid value beginning with `Fven...`. With ETH absent, the IOST record becomes the main address. Lowercasing makes the two strings compare equal, so the Manager places the Solana icon on the IOST card and suppresses the actual SOL-specific card. The copy button still copies the IOST value.

The visible chain identity and copied recipient are therefore derived from different resolver records.

## Root Cause

The `getReceivingAddressChains()` computes `mainAddress.value.toLowerCase()` and groups every address whose lowercased value matches. `getChainSpecificAddresses()` repeats the same transformation to remove purported duplicates.

The implementation applies an EVM-specific normalization rule to all coin types. It never dispatches equality or canonicalization according to the record's coin type or encoding.

## Technical Walkthrough

{% stepper %}
{% step %}

## A malicious ENS owner leaves the ETH record unset.

{% endstep %}

{% step %}

## The owner sets IOST coin type 291

Set it to `FVen3X669xLzsi6N2V91DoiyzHzg1uAgqiT8jZ9nS96Z`, which is also an attacker-controlled Solana public key.
{% endstep %}

{% step %}

## The owner sets SOL coin type 501

Set it to `Fven3X669xLzsi6N2V91DoiyzHzg1uAgqiT8jZ9nS96Z`.
{% endstep %}

{% step %}

## The two Base58 strings differ by one letter's case

They decode to different 32-byte values.
{% endstep %}

{% step %}

## The IOST value becomes the main receiving address

Because ETH is absent and coin type 291 appears first, `getMainAddressCandidate()` selects the IOST value as the main receiving address.
{% endstep %}

{% step %}

## The Solana icon is associated with the IOST main card

`getReceivingAddressChains()` lowercases both values, treats them as equal, and associates the SOL icon with the IOST main card.
{% endstep %}

{% step %}

## The authoritative SOL record is removed

`getChainSpecificAddresses()` performs the same comparison and removes the authoritative SOL record from the separate-address list.
{% endstep %}

{% step %}

## A Solana-labelled option copies the IOST record

`ProfileAddressesSection` renders a Solana-labelled option whose `CopyableButton` value is the IOST record.
{% endstep %}

{% step %}

## A visitor sends SOL to the attacker-controlled address

A visitor who copies that option and signs a native SOL transfer sends to the attacker-controlled IOST value rather than the authoritative SOL record.
{% endstep %}
{% endstepper %}

## Impact Explanation

A user can be shown a Solana chain indicator while the associated copy action supplies a different Base58 address. If the user submits a native SOL transfer using that copied value, the funds are irreversibly credited to the attacker-controlled key. Exploitation requires a maliciously configured ENS profile and a user who relies on the affected copy control.

## Recommendation

Consider replacing global lowercase comparison with coin-type-aware equality. Compare Base58 records exactly or compare their decoded bytes, and apply case-insensitive normalization only to address formats whose specifications permit it. Keep each chain icon and copy target bound to the same authoritative resolver record, and add regression coverage for valid Base58 values that differ only by case.

## Proof of Concept

```ts
import { describe, expect, it } from 'vitest'
import type { ProfileRecords } from '@/features/profile/types'
import {
  getChainSpecificAddresses,
  getMainReceivingAddress,
  getReceivingAddressChains,
} from './ProfileView.helpers'

const makeRecords = (
  addresses: ProfileRecords['addresses'],
): ProfileRecords => ({
  addresses,
  base: {},
  contact: [],
  links: [],
  social: [],
  unknown: [],
  agentRegistrations: [],
})

const attackerAddress = 'FVen3X669xLzsi6N2V91DoiyzHzg1uAgqiT8jZ9nS96Z'
const authoritativeSolAddress = 'Fven3X669xLzsi6N2V91DoiyzHzg1uAgqiT8jZ9nS96Z'

describe('finding 903: case-sensitive Base58 addresses are conflated', () => {
  it('labels the attacker IOST value as Solana and suppresses the real SOL address', () => {
    const records = makeRecords([
      { coinType: 291, value: attackerAddress },
      { coinType: 501, value: authoritativeSolAddress },
    ])

    expect(getMainReceivingAddress(records)).toMatchObject({
      coinType: 291,
      value: attackerAddress,
    })
    expect(
      getReceivingAddressChains(records).map(({ coinType }) => coinType),
    ).toEqual([291, 501])
    expect(getChainSpecificAddresses(records)).toEqual([])
    expect(attackerAddress).not.toBe(authoritativeSolAddress)
  })

  it('keeps a visibly different benign SOL value on its own card', () => {
    const records = makeRecords([
      { coinType: 291, value: attackerAddress },
      { coinType: 501, value: `2${authoritativeSolAddress.slice(1)}` },
    ])

    expect(
      getReceivingAddressChains(records).map(({ coinType }) => coinType),
    ).toEqual([291])
    expect(getChainSpecificAddresses(records)).toEqual([
      expect.objectContaining({ coinType: 501 }),
    ])
  })
})
```
