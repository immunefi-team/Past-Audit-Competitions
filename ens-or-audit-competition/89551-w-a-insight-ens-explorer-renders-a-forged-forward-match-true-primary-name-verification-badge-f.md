# 89551 w a insight ens explorer renders a forged forward match true primary name verification badge for an attacker s unverified default reverse name&#x20;

**Submitted on Aug 19th 2026 at 20:29:18 UTC by @Cryptor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89551
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer's reverse-resolution view looks up an address and shows the ENS name(s) that resolve back to it, alongside a **"Forward match"** column. That column's green `✓ True` + `Primary name` badge is the app's trust signal that the name's forward `addr()` record actually points back to the looked-up address — the check that distinguishes a genuine primary name from an unverified claim.

For rows that inherit the account's **`default.reverse`** name (the ENSIP-19 cross-chain fallback, coin type `0x80000000`), that badge is hardcoded to `True` and `Primary name` **without performing the forward-match check**, and the resolution deliberately discards the match result. Because a `default.reverse` record is settable by anyone for their own address with no ownership of the claimed name, an attacker can make the Explorer present any name (for example a well-known one) against the attacker's own address, decorated with a forged "forward-verified primary name" badge.

## Vulnerability Details

**1. The `default.reverse` name is resolved with mismatches allowed, and the forward-match flag is thrown away.**

`useReverseResolution.ts`, lines 236–241:

```ts
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE, // 0x80000000 — the default.reverse record
  allowMismatch: true,             // return the name even if forward addr() does NOT point back
})
const defaultName = defaultResult?.name ?? null // keeps only .name; .match is discarded
```

`getName(..., { allowMismatch: true })` returns `{ name, match }`, where `match` is the forward confirmation that the name's `addr()` record resolves back to `address`. Only `.name` is retained; the `.match` flag is never carried onto the row.

**2. The table hardcodes the positive verification badge for inherited-default rows.**

`columns.tsx`, the `forwardMatch` column cell, lines 96–112:

```tsx
if (!name && defaultName && canInheritDefault(row.original.coinType)) {
  return (
    <div ...>
      <Badge ...><CheckCircle2 .../><span>True</span></Badge>       {/* not verified */}
      <Badge ...><SquareUser .../><span>Primary name</span></Badge> {/* not verified */}
      <Badge ...>Default</Badge>
    </div>
  )
}
```

The own-name branch of the same cell (lines 125–138) correctly gates the `True` / `Primary name` badges on `row.original.forwardMatch`. The inherited-default branch does not — it asserts them unconditionally.

**3. The source value is fully attacker-controlled and unauthenticated.**

`default.reverse` (coin type `0x80000000`) is a per-address record any account can set for its own address via the DefaultReverseRegistrar; setting it does not require owning, or being authorized on, the name it points to. So `defaultName` is an arbitrary attacker-chosen string, rendered as a forward-verified primary name.

## Impact

The "Forward match" column exists specifically to tell a user whether a name↔address pairing is trustworthy — the forward record confirming the reverse claim. For inherited-`default.reverse` rows that signal is forged: the Explorer displays a name the address's owner does not own, and whose forward record does not point back, next to a green `✓ True` and `Primary name` badge indistinguishable from a genuinely verified pairing.

An unprivileged attacker (any address holder, with no name ownership and no separate step required to make the badge appear) points their `default.reverse` at a name a victim trusts. When the victim looks that address up in the Explorer — the normal use of the tool — they see the address presented as the forward-verified primary of that name across all L2 rows. A user who relies on this verification badge to conclude "this address is the canonical/verified address for `<name>`" is deceived about the name↔address association, which is the precondition for sending funds or approving a transaction to the wrong recipient. This is the program's wrong-address impact family — "resolving or rendering an ENS name to an attacker-controlled address." The "Forward match" column is a verification control whose sole purpose is to confirm a name↔address pairing before a user trusts it; the bug makes the application affirmatively assert a false confirmation (`✓ True / Primary name`) for an attacker-controlled address claiming a name the attacker does not own, with no privilege, no name ownership, and no social-engineering step required to make the false verification appear.

## Proof of Concept

## Proof of Concept

### Setup (per the program Resources page)

Requirements: Node **v22**, pnpm **v10**. Audited revision: commit `63772fd872af472ced58b009499355f3430c2a86` (`immunefi-team/audit-comp-ens`, fork commit `cda79ac`).

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 63772fd872af472ced58b009499355f3430c2a86   # or its fork commit cda79ac
corepack enable
pnpm install --frozen-lockfile
```

### A. Coded PoC (deterministic; drives the real column cell)

Create the file `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.poc.test.tsx` with exactly this content:

```tsx
import type { Row } from '@tanstack/react-table'
import { render, screen } from '@testing-library/react'
import { describe, expect, it } from 'vitest'
import type { ReverseResolutionResult } from '../../hooks/useReverseResolution'
import { columns } from './columns'

const L2_COIN_TYPE = 0x8000000a // Optimism ENSIP-11 coin type -> canInheritDefault === true

const forwardMatchColumn = columns.find(
  (c) => (c as { accessorKey?: string }).accessorKey === 'forwardMatch',
)!
const renderForwardMatchCell = (original: Partial<ReverseResolutionResult>) => {
  const cell = (forwardMatchColumn as { cell: (ctx: unknown) => JSX.Element }).cell
  return render(<>{cell({ row: { original } as Row<ReverseResolutionResult> })}</>)
}

describe('P-1: forged forward-match badge for an unverified default.reverse name', () => {
  it('renders "True" + "Primary name" for an inherited default.reverse name even though forwardMatch is false', () => {
    renderForwardMatchCell({
      coinType: L2_COIN_TYPE,
      name: '',                    // this L2 has no own reverse record -> inherits the default
      defaultName: 'vitalik.eth',  // attacker set their default.reverse to a name they do NOT own
      forwardMatch: false,         // vitalik.eth's forward addr() does NOT point to the attacker
    })
    expect(screen.getByText('True')).toBeTruthy()
    expect(screen.getByText('Primary name')).toBeTruthy()
  })

  it('control: a real own-name row with forwardMatch=false correctly shows "False"', () => {
    renderForwardMatchCell({ coinType: L2_COIN_TYPE, name: 'bob.eth', defaultName: null, forwardMatch: false })
    expect(screen.getByText('False')).toBeTruthy()
  })
})
```

Run:

```
pnpm --filter portal exec vitest run \
  src/features/reverse-resolution/components/ReverseResolutionTable/columns.poc.test.tsx
```

Result (both pass): the first test proves the cell renders the `True` / `Primary name` verification badges for an attacker-controlled, unverified `default.reverse` name (`forwardMatch: false`); the control proves the same cell renders `False` for a genuine row — so the badge is forged only on the inherited-default path.

### B. End-to-end reproduction (browser + Sepolia)

Prerequisites: an attacker EOA on Sepolia, and the running Explorer (`pnpm --filter portal dev`, `http://localhost:3001` — no configuration needed per the Resources page). Choose any name the attacker does **not** own — call it `victim-name.eth`.

1. From the attacker EOA on Sepolia, set the account's **default reverse** name to `victim-name.eth` via the DefaultReverseRegistrar set-name entrypoint (ENSIP-19, coin type `0x80000000`). This writes only the attacker's own `default.reverse` record; it does not touch `victim-name.eth`.
2. Do not set a coin-60 `addr.reverse` record, and confirm `victim-name.eth`'s forward `addr()` does not point to the attacker (it will not — the attacker does not own it).
3. Open the Explorer's reverse-resolution view for the attacker's address.
4. Observe: every L2 row without its own reverse record displays `victim-name.eth` with a green **`✓ True` (Forward match)** badge and a **`Primary name`** badge — asserting a forward confirmation that was never performed and would fail.
5. For contrast, set a legitimate own name whose forward `addr()` matches, and note the identical badge only appears there after the real `forwardMatch` check — confirming the inherited-default path skips it.
