# 89705 w a insight missing recipient address validation in set primary name leads to ens name resolution hijacked to an attacker controlled address

**Submitted on Aug 20th 2026 at 17:09:04 UTC by @W3B\_SR\_0xki for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89705
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The reverse-resolution "Set primary name" action in the Explorer app signs a setAddr (forward-resolution) transaction using the address of the page currently being viewed, not the connected wallet's own address. Because anyone can permissionlessly set their own reverse-resolution string to an arbitrary name they do not own, an attacker can make their own address display a victim's real ENS name, then get that victim — the name's actual owner — to visit the attacker's address page and click the button. The victim's wallet ends up signing a legitimate-looking setAddr transaction that redirects their own name's ETH resolution to the attacker's address instead of completing their primary-name pairing as the UI implies. On a production deployment, anyone who subsequently sends funds to the victim's name — the core use case ENS exists for — would send them to the attacker instead, for as long as the redirection goes unnoticed.

## Vulnerability Details

Location: apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/

The request builder uses the page's own address as the write target, not the signer:

```typescript
// hooks/useReverseResolutionMutations.ts
const getForwardResolutionRequest = useCallback(
  (address: Address): SetForwardResolutionRequest => {
    return createSetForwardResolutionRequest({
      name: displayName,
      coinType,
      resolverAddress,
      targetAddress: address,   // the route's own address — never the connected wallet
    })
  },
  [displayName, coinType, resolverAddress],
)
```

ReverseResolutionSidebar.tsx calls it with the route param, and the button that triggers it has no identity check at all:

```typescript
const handleSetPrimaryNameStart = () => {
  if (!displayName) return
  const request = getForwardResolutionRequest(address) // `address` = route's $addr
  submitForwardResolution({ name: displayName, request })
}
...
<Button
  onClick={handleSetPrimaryName}
  disabled={
    !isConnected ||
    isEnsOwnerLoading ||
    isForwardResolutionPending ||
    isSwitchingChainForForward
  }
>
```

Button enablement is gated only by whether a real name is set on this row and not already the primary — again, no identity check:

```typescript
// utils/reverseResolution/computeDisplayNameState.ts
const canSetAsPrimary = !!displayName && !isPrimaryName && name !== null
```

The correct pattern exists elsewhere in the same codebase, on the sibling component that offers the reverse write instead of the forward write:

```typescript
// features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx
// Offered only when the connected wallet *is* this address (setName is msg.sender-scoped).
const canSetPrimaryName =
  !!connectedAddress &&
  !!data?.address &&
  isAddress(data.address, { strict: false }) &&
  isAddressEqual(connectedAddress, data.address)
```

This confirms what the reverse-resolution sidebar's check should look like — getForwardResolutionRequest just never asks it.

Exploit sequence:

Attacker calls setName("\<victim's real ENS name>") on the L1/coin-60 ReverseRegistrar for their own address — permissionless, standard ENS behavior, requires no relationship to the target name. /addr//reverse-resolution now displays the victim's name for that row, with forwardMatch: false. computeDisplayNameState returns canSetAsPrimary: true, and the button renders enabled. The victim (the name's real owner) is directed to that URL and clicks "Set primary name," reasonably believing they are completing their own name's primary pairing — a real, designed feature, not an unusual or off-workflow action. Their wallet is prompted to sign setAddr(namehash(victim's name), 60, \<attacker's address>). Since the victim genuinely owns the name, the resolver accepts the write.

Confirmed, reproduced on a Sepolia fork carrying the real registrar, resolver, and the target name's actual on-chain ownership (commit 1c9b47f18fcddd2e864dfe385c4171061c9811ae):

Victim 0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12 owns pocpr1a01fa9d5e7.eth, resolver 0x23194F875483cdc4998f433d56864Cb77a335D34.

Attacker 0x6C63F8E00E0d154742C5F1372231b0A7f68A97A2 calls setName("pocpr1a01fa9d5e7.eth") on 0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6 — succeeds.

Victim signs the exact request the sidebar builds for the attacker's route address: setAddr(0xc99507a99937d9855ab92a80217e85f5e71456ca315348ab51415f1667d387d6, 60, 0x6c63f8e00e0d154742c5f1372231b0a7f68a97a2) — succeeds.

pocpr1a01fa9d5e7.eth's ETH forward resolution: null → 0x6C63F8E00E0d154742C5F1372231b0A7f68A97A2, verbatim.

## Impact Details

The victim's name resolves to an attacker-controlled address for the affected coin type. getForwardResolutionRequest is the same call for every row this table renders — the missing check applies to any coin type the reverse-resolution table exposes, not only the ETH/coin-60 row demonstrated here; the other rows weren't separately tested and I'm not claiming them as confirmed, only as the same code path.

This is a redirection of future receipts, not a withdrawal of an existing balance — no transaction here moves funds already held by the victim, which is why I haven't also selected "Direct theft of user funds." The loss is unbounded and compounding instead: every payment sent to the name — ENS's primary purpose — is misdirected for as long as the hijack goes unnoticed, and nothing in the app surfaces or warns about the change. Scope of loss tracks how the name is used (personal address vs. a business's public receiving address) and how long it takes the victim to notice.

Recovery is possible — the victim retains full ownership of the name and resolver throughout, and can correct the record via the properly-gated flow (AddressResolutionSidebar, connected as themselves) once they realize something is wrong. There's just no mechanism that tells them to look.

Attacker cost is a single permissionless setName call and knowledge of the victim's ENS name, which is public by definition. No privilege, no compromise of the victim's wallet or keys required — only that the victim be directed to the attacker's address page and click a button that does exactly what it's designed to do, just against the wrong target.

## Proof of Concept

## Proof of Concept

### Environment

Reproduced on a **Sepolia fork**, not a synthetic/mocked environment — the fork carries the real `ReverseRegistrar`, the real resolver, and the real on-chain ownership state of the target name, forked from live Sepolia at the anchor commit's deployment. Only the ETH balances are local. This satisfies the requirement of a live devnet fork as opposed to a unit-test harness: every contract interacted with is the genuine deployed bytecode the app itself points at, not a mock or reimplementation.

Independently re-verified before relying on it: `namehash("pocpr1a01fa9d5e7.eth") = 0xc99507a99937d9855ab92a80217e85f5e71456ca315348ab51415f1667d387d6` — computed separately, matches the value in the signed calldata below. The resolver address is a genuine deployed contract (EIP-1167 minimal proxy) confirmed live on real Sepolia, and real Sepolia currently still shows this name's coin-60 address as unset — confirming the fork test never touched the live network.

### Actors & contracts

| Role                                              | Address                                      |
| ------------------------------------------------- | -------------------------------------------- |
| A — victim, real owner of the name                | `0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12` |
| B — attacker                                      | `0x6C63F8E00E0d154742C5F1372231b0A7f68A97A2` |
| Target name                                       | `pocpr1a01fa9d5e7.eth`                       |
| Name's resolver                                   | `0x23194F875483cdc4998f433d56864Cb77a335D34` |
| Coin-60 ReverseRegistrar (same one the app reads) | `0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6` |

### Setup

```bash
anvil --fork-url <sepolia-rpc> --port 8545 --chain-id 11155111
 
cast rpc anvil_setBalance 0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12 \
  0x56BC75E2D63100000 --rpc-url http://127.0.0.1:8545
 
cast rpc anvil_setBalance 0x6C63F8E00E0d154742C5F1372231b0A7f68A97A2 \
  0x56BC75E2D63100000 --rpc-url http://127.0.0.1:8545
```

### Reproduction

**Step 1 — attacker sets a colliding reverse record.** Permissionless; requires no relationship to the target name.

```ts
// B, via ensjs
setPrimaryNameWriteParameters(...) // → setName("pocpr1a01fa9d5e7.eth")
// sent to the coin-60 ReverseRegistrar
```

**Step 2 — victim signs the exact payload the app's "Set primary name" button builds**, using the app's own request/encode functions, not a hand-rolled equivalent:

```ts
// A, via the app's own code path
createSetForwardResolutionRequest({
  name: 'pocpr1a01fa9d5e7.eth',
  coinType: 60,
  resolverAddress: '0x23194F875483cdc4998f433d56864Cb77a335D34',
  targetAddress: '0x6C63F8E00E0d154742C5F1372231b0A7f68A97A2', // B — the open page's address
})
// encoded identically to prepareSetForwardResolutionTransaction
```

### Results

|                                        | Before | After                                            |
| -------------------------------------- | ------ | ------------------------------------------------ |
| `pocpr1a01fa9d5e7.eth` → ETH (coin 60) | `null` | `0x6C63F8E00E0d154742C5F1372231b0A7f68A97A2` (B) |

```
B setName:  0xc4c6d6dfe250ef5aaaebf00d3228e93b6cfe0c4f0e93e6451dedef95ab296dcf — success
A setAddr:  0x265aef2cbdb22e243b90af679fe1bb2ded89b34d27b828c96cf4a748ef5795e6 — success
  to:    0x23194F875483cdc4998f433d56864Cb77a335D34
  call:  setAddr(
           0xc99507a99937d9855ab92a80217e85f5e71456ca315348ab51415f1667d387d6,
           60,
           0x6c63f8e00e0d154742c5f1372231b0a7f68a97a2
         )
```

A signed a resolver write pointing their own name at B. The wallet-facing label was "Set primary name to pocpr1a01fa9d5e7.eth" — the address in the payload was never A's.

### Maps directly to the real UI flow

1. B sets their reverse record to A's name (Step 1 above).
2. A, connected as themselves, opens `/addr/0x6C63…97A2/reverse-resolution`.
3. The mainnet row shows `canSetAsPrimary === true` — button renders enabled on `isConnected` alone.
4. Click → wallet prompt → signs the `setAddr` above. No devtools, no non-standard client, no unprompted action outside the app's own designed flow.

**Fix:** gate both button enablement and `getForwardResolutionRequest`'s target on `isAddressEqual(connectedAddress, routeAddress)` — the same check `AddressResolutionSidebar.tsx:452-458` already applies to its own primary-name action.
