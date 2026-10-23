# 90859 w a insight the explorer s reverse resolution table prints forward match true and a primary name badge for a name it never forward verified letting any address present itself under a&#x20;

**Submitted on Aug 27th 2026 at 16:16:33 UTC by @Valisthea for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90859
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## 1. Summary

The Explorer's reverse-resolution page has a column headed **"Forward match"**. On any of the five L2 rows (Optimism, Arbitrum, Base, Linea, Scroll) where the address publishes no chain-specific reverse record, the row inherits the chain-agnostic `default.reverse` claim and prints a green check with the literal string `True`, plus a `Primary name` badge, without ever performing or consulting a forward match.

The ENSIP-19 bidirectional verification result is fetched and then discarded. The reverse registrars accept any UTF-8 string from any caller with no ownership check, which is precisely why ENSIP-19 mandates forward verification. So any address can claim any name, and the Explorer will affirm that claim as verified, alongside the claimed name's avatar.

The same table proves the point against itself, and it does so twice over. For one address and one identical string, the **Default** row prints **False** while every L2 row inheriting that same claim prints **True**. And the L2 rows that do carry their own record render honestly on the same screen, so the table demonstrates both the correct behaviour and the defective one side by side.

## 2. Impact

The reverse-resolution page is the surface a user opens to answer "who is this address". On every L2 row that inherits the chain-agnostic claim, it affirms a name-to-address pairing that ENS itself refuses to verify. In the capture below that is three of its seven rows, and an attacker who sets no chain-specific record anywhere reaches all five L2 rows.

An attacker binds a name they do not own, and never could, to their own address, and the page presents it with a green check, a "Primary name" badge, and the real name's avatar. That is the mechanism by which a viewer decides an address is who it claims to be.

This maps to the program's Critical bullet on the application displaying an incorrect name, though I am filing at Medium because no transaction is built from this page and no funds move as a direct result. It is an identity assertion the product is not entitled to make.

## 3. Root cause

Two code paths in the same hook treat the same ENSIP-19 `match` flag differently.

**The path that keeps it.** `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:103`

```ts
    forwardMatch: nameResult?.match ?? false,
```

That is inside `getL1ReverseRecord`, whose own docstring at `:66` states which record it reads:

```
 * - `0x80000000` → `default.reverse` (ENSIP-19 cross-chain fallback).
```

and which fetches at `:74-78`:

```ts
  let nameResult = await getName(client, {
    address,
    coinType,
    allowMismatch: true,
  })
```

**The path that discards it.** The same file, `:236-241`, fetches the record a second time to produce the value the L2 rows inherit, and keeps only the name:

```ts
  const defaultResult = await getName(l1Client, {
    address,
    coinType: DEFAULT_EVM_COIN_TYPE,
    allowMismatch: true,
  })
  const defaultName = defaultResult?.name ?? null
```

`defaultResult.match` is dropped on the floor.

**The renderer then hard-codes the verified state.** `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx`, inside the cell for the column headed `Forward match` (`:88`):

```tsx
      if (!name && defaultName && canInheritDefault(row.original.coinType)) {
        return (
          <div className="flex flex-row items-center gap-2">
            <Badge variant="outline" className="text-xs">
              <CheckCircle2 className="w-4 h-4" />
              <span>True</span>
            </Badge>
            <Badge variant="outline" className="text-xs">
              <SquareUser className="w-4 h-4" />
              <span>Primary name</span>
            </Badge>
```

(`:96` through `:106`.) `forwardMatch` is read into a local at `:93` and this branch never uses it.

The sidebar repeats the equivalence at `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:83`:

```ts
  const isPrimaryName = forwardMatch || isInheritingDefault
```

which drives the green banner at `ReverseResolutionSidebar.tsx:649`, `This is the primary name on {label}`, and renders the claimed name as an avatar-bearing badge.

## 4. The table contradicts itself, on one screen

`canInheritDefault` excludes the Default row and the Mainnet row. `columns.tsx:12-13`:

```ts
const canInheritDefault = (coinType: number) =>
  coinType !== DEFAULT_EVM_COIN_TYPE && coinType !== MAINNET_COIN_TYPE
```

So the Default row, which is the row that actually holds the record being inherited, falls through to the honest branch and renders `False` from `forwardMatch` (`columns.tsx:139`). For one address, one identical unverified string:

| Row                                           | Branch taken                             | "Forward match" cell         |
| --------------------------------------------- | ---------------------------------------- | ---------------------------- |
| Default (`0x80000000`)                        | honest branch, `forwardMatch === false`  | **False**                    |
| Mainnet (coin 60)                             | honest branch, excluded from inheritance | **False**                    |
| Any L2 row with its own chain-specific record | honest branch, `name` is non-null        | **False**                    |
| Any L2 row without one                        | inherit branch, `forwardMatch` unread    | **True** plus "Primary name" |

The same application also renders the identical badge, correctly gated, one tab over. `apps/portal/src/features/forward-resolution/components/AddressResolution/columns.tsx:145-150`:

```tsx
          {reverseMatch === 'verified' && (
            <Badge variant="outline" className="text-xs">
              <SquareUser className="size-4" />
              <span>Primary name</span>
            </Badge>
          )}
```

Same icon, same text, gated on a verification result. Two tabs of one product, one badge, one verified and one asserted.

The repository also records that it knows the reverse registrar is unauthenticated. `ReverseResolutionSidebar.tsx:454`:

```
// The reverse registrar's `setName(string)` accepts any UTF-8 string, so a
```

## Proof of Concept

## 5. Proof of concept

**No transaction is required. The condition already exists on Sepolia.**

### 5.1 The witness

Address `0x55e55c649895940826a852820d9e1a076ec47b09` has set `lfg.eth` as its `default.reverse` name. It does not own `lfg.eth`, and the name's forward record does not point back at it.

### 5.2 Confirm it from the chain, read-only

```bash
UR=0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe
RPC=https://ethereum-sepolia-rpc.publicnode.com

# 1. The resolver refuses to verify the pairing, and hands back the claimed name.
cast call $UR "reverse(bytes,uint256)(string,address,address)" \
  0x55e55c649895940826a852820d9e1a076ec47b09 2147483648 --rpc-url $RPC
```

Result:

```
execution reverted, data: 0xef9c03ce ... "lfg.eth"
```

That selector is not ambiguous:

```bash
cast sig "ReverseAddressMismatch(string,bytes)"
# 0xef9c03ce
```

`0xef9c03ce` is exactly the error `parseReverseAddressMismatchError` handles at `packages/ensjs/src/actions/public/getName.ts:140` in the pinned ensjs build (`e96662c`), and its first argument is the name. With `allowMismatch: true`, which is what the Explorer passes, `getName` returns `{ name: "lfg.eth", match: false }`. That is the value the Explorer keeps and the flag it drops.

```bash
# 2. The forward record of the claimed name does not point back.
cast call $UR "resolve(bytes,bytes)(bytes,address)" \
  <dns-encoded lfg.eth> $(cast calldata 'addr(bytes32)' $(cast namehash lfg.eth)) --rpc-url $RPC
```

Result: `0x0000000000000000000000000000000000000000`.

The claim is unverified by construction.

### 5.3 What the page actually renders

Text extracted from the running application at the URL in 5.4, on the audit revision, with no wallet connected. This is the attached screenshot, transcribed:

```
Reverse resolution

Network      Name                          Forward match

Default      lfg.eth                       False
Mainnet      lfg.eth                       False
Optimism     lfg.eth        [Default]      True   [Primary name]  [Default]
Base         baseprimaryname.eth           False
Arbitrum     l2eth.eth                     False
Linea        lfg.eth        [Default]      True   [Primary name]  [Default]
Scroll       lfg.eth        [Default]      True   [Primary name]  [Default]
```

**The same name, from the same single unverified claim, is reported False twice and True three times on one screen.** Section 5.2 established from the chain that the resolver refuses to pair this address with `lfg.eth`, and that `lfg.eth` resolves forward to the zero address. The False rows are the correct ones. The three True rows, each carrying a "Primary name" badge, are asserting a forward match that was never performed and that the chain contradicts.

The rest of this section derives that output from the source, so the render can be traced rather than taken on trust. The rendering is deterministic given three values, and each step below is the cited source.

{% stepper %}
{% step %}

### Step A. The Default row gets a name and a false flag.

`getL1ReverseRecord` is called for the Default row at coinType `0x80000000` (`useReverseResolution.ts:66` documents that mapping) and fetches with mismatches allowed (`:74-78`). The UniversalResolver reverts `ReverseAddressMismatch` carrying `lfg.eth`, shown in 5.2. ensjs converts that into `{ name: "lfg.eth", match: false }` (`getName.ts:140-148`). The row is therefore built at `useReverseResolution.ts:99` and `:103` as:

```
name         = "lfg.eth"
forwardMatch = false
```

{% endstep %}

{% step %}

### Step B. `defaultName` is set for every row.

The second fetch at `:236-241` returns the same record, keeps only the name, and it is spread onto every row:

```
defaultName = "lfg.eth"
```

{% endstep %}

{% step %}

### Step C. The inheriting L2 rows have no name of their own.

`getL2ReverseRecord` reads `nameForAddr` on each L2 reverse registrar. For this witness the address set no record on Optimism, Linea or Scroll, so those three reads return empty and the function returns `createEmptyResult(network)` at `useReverseResolution.ts:56-57`:

```
name         = null
forwardMatch = false
defaultName  = "lfg.eth"   (from step B)
```

{% endstep %}

{% step %}

### Step D. The branch each row takes.

`columns.tsx:96` is the first test in the cell:

```tsx
      if (!name && defaultName && canInheritDefault(row.original.coinType)) {
```

* **Default row:** `name` is `"lfg.eth"`, so `!name` is false. The test fails. It also fails the second test at `:114` (`if (!name)`). It reaches the ternary at `:127`, `forwardMatch` is `false`, and it renders the `XCircle` badge reading **False** at `:139-142`.
* **Mainnet row:** excluded from inheritance by `canInheritDefault` (`:12-13`), same outcome.
* **Optimism, Linea, Scroll:** `!name` is true, `defaultName` is truthy, and `canInheritDefault` is true because their coin types are neither `DEFAULT_EVM_COIN_TYPE` nor `MAINNET_COIN_TYPE`. The branch fires and renders, unconditionally, the `CheckCircle2` badge reading **True** at `:101` and the `SquareUser` badge reading **Primary name** at `:105`.
* **Base and Arbitrum:** this witness *does* publish a chain-specific record on those two (`baseprimaryname.eth` and `l2eth.eth`), so `name` is non-null, `:96` fails, and each row falls through to the honest ternary at `:127` and renders **False**.

That last bullet is the control, and it is the strongest thing in this report. The same table, in the same render, shows the code producing an accurate answer whenever it has a real value to check, and a hard-coded `True` whenever it does not. The defect is not that the page cannot tell: it is that the one branch with nothing to verify asserts verification anyway.
{% endstep %}
{% endstepper %}

**Result**, matching the render in 5.3 row for row:

```
Default    lfg.eth               Forward match: False    <- holds the claim, reports it honestly
Mainnet    lfg.eth               Forward match: False    <- excluded from inheritance
Optimism   lfg.eth  [Default]    Forward match: True   + Primary name    <- inherit branch
Base       baseprimaryname.eth   Forward match: False    <- has its own record, honest branch
Arbitrum   l2eth.eth             Forward match: False    <- has its own record, honest branch
Linea      lfg.eth  [Default]    Forward match: True   + Primary name    <- inherit branch
Scroll     lfg.eth  [Default]    Forward match: True   + Primary name    <- inherit branch
```

`forwardMatch` is read into a local at `:93` and the branch that fires never uses it. Note that the three rows asserting `True` are exactly the three showing the inherited `lfg.eth`, which is the one string the chain refuses to verify, while the two rows showing a name of their own report `False`. The page is accurate everywhere it has something to check, and confidently wrong in the one place it has nothing.

### 5.4 Verify it in one click

```
https://<explorer-host>/addr/0x55e55c649895940826a852820d9e1a076ec47b09/reverse-resolution
```

No wallet, no connection, no authentication. The route validates only that the parameter is an address. The chain state above was re-confirmed on 2026-08-27 and is not under my control, so if the address's record changes, section 5.5 recreates an equivalent witness in one transaction.

To reproduce the attached capture from a clean checkout of the audit revision:

```bash
pnpm install --frozen-lockfile
cd apps/portal
VITE_SEPOLIA_RPC_URL=https://ethereum-sepolia-rpc.publicnode.com pnpm dev --port 5174
# then open
# http://localhost:5174/addr/0x55e55c649895940826a852820d9e1a076ec47b09/reverse-resolution
```

I supply the RPC endpoint explicitly rather than using the one committed in the repository, because the program's rules forbid exercising credentials recovered from the repo. `VITE_SEPOLIA_RPC_URL` is the override the application already provides for this (`apps/portal/src/lib/wagmi.ts:27-28`).

### 5.5 Recreating the witness, if the on-chain state changes

If you would rather demonstrate against a specific name, one transaction from any throwaway address creates a fresh witness:

```bash
cast send 0x4f382928805ba0e23b30cfb75fc9e848e82dfd47 \
  "setName(string)" "<any-registered-name-you-do-not-own>.eth" \
  --rpc-url $RPC --private-key $THROWAWAY
```

The registrar address is the one the application itself writes to, at `apps/portal/src/features/reverse-resolution/config.ts:34-35`. It keys on `msg.sender` and performs no ownership check on the string.

## 6. Why this is not a duplicate

**QA-06** ("If a subname is set as primary while still resolving and is then deleted or stops resolving, it is still shown as the primary name") concerns the **viewer's own** previously-set name lingering in the interface after it stops resolving. Here a third party who owns nothing binds an arbitrary name to their own address, and the root cause is a discarded ENSIP-19 `match` flag plus a hard-coded verified badge. The program states that a demonstrated security consequence of a known QA behaviour is a new finding, and impersonating an arbitrary name behind a green check, on every network that inherits the claim, is that consequence.

**WEB-382**, referenced inside the very file this report cites (`ReverseResolutionSidebar.tsx:455-456`: "a user can point their address at a name that was never registered (see WEB-382). We surface this as a warning rather than gating the write"), is the **write** side. It is a considered product decision to warn rather than block when *you* set *your own* reverse name. This report is the **read** side: a third party's unverified claim rendered to *other people* as forward-matched. No warning of any kind appears on the page a viewer sees, and a write-side warning is not a control over what the read-side table asserts.

**EXP-INPUT-005** requires confusable characters. This requires none: the victim's exact, correctly normalized, genuinely registered name is displayed as verified. In fact the Explorer's own call rejects unnormalized names on this path, because it does not pass `allowUnnormalized` and `getName.ts:153` returns `null` for them.

**EXP-4337-002**, **EXP-4337-003** and **SEC-TXM-002** are transaction-construction issues in the transaction manager. This page builds no transaction.

## 7. Recommended fix

Carry the flag through instead of discarding it, and gate the badge on it.

```ts
// useReverseResolution.ts, alongside the existing defaultName
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})
const defaultName = defaultResult?.name ?? null
const defaultForwardMatch = defaultResult?.match ?? false   // add this
// spread { defaultName, defaultForwardMatch } onto each row
```

```tsx
// columns.tsx, inherit-default branch
if (!name && defaultName && canInheritDefault(row.original.coinType)) {
  return row.original.defaultForwardMatch
    ? (/* True + Primary name + Default */)
    : (/* False + Default */)
}
```

```ts
// computeDisplayNameState.ts
const isPrimaryName = forwardMatch || (isInheritingDefault && defaultForwardMatch)
```

The cleanest version is to reuse the classifier the sibling feature already has. `apps/portal/src/features/forward-resolution/components/hooks/useReverseMatch.ts` already models exactly this concern with a four-state verdict and already handles the `ReverseAddressMismatch` case. The reverse-resolution table should consume it rather than reimplement half of it.

**Regression test:** an address whose `default.reverse` claim does not forward-resolve back must render `False` on every row, not only on the Default row. Address `0x55e55c649895940826a852820d9e1a076ec47b09` is a ready-made fixture on Sepolia.

## 8. Full citation index

| Claim                                             | Location                                                                                       |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Verified flag kept on the L1 path                 | `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:103`                |
| That path reads `default.reverse` at `0x80000000` | `.../useReverseResolution.ts:66`                                                               |
| The fetch, with mismatches allowed                | `.../useReverseResolution.ts:74-78`                                                            |
| Second fetch, flag discarded                      | `.../useReverseResolution.ts:236-241`                                                          |
| Column heading                                    | `.../ReverseResolutionTable/columns.tsx:88`                                                    |
| `forwardMatch` read but unused in the branch      | `.../columns.tsx:93`                                                                           |
| Hard-coded True and Primary name                  | `.../columns.tsx:96-106`                                                                       |
| Honest branch renders False                       | `.../columns.tsx:139`                                                                          |
| Default and Mainnet excluded from inheritance     | `.../columns.tsx:12-13`                                                                        |
| Sidebar equates inherited with primary            | `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:83`                        |
| Green banner                                      | `.../ReverseResolutionTable/ReverseResolutionSidebar.tsx:649`                                  |
| Repo knows the registrar is unauthenticated       | `.../ReverseResolutionSidebar.tsx:454`                                                         |
| WEB-382 reference                                 | `.../ReverseResolutionSidebar.tsx:455-456`                                                     |
| The same badge, gated, in the sibling feature     | `apps/portal/src/features/forward-resolution/components/AddressResolution/columns.tsx:145-150` |
| DefaultReverseRegistrar address                   | `apps/portal/src/features/reverse-resolution/config.ts:34-35`                                  |
| Library converts the revert into `match: false`   | ensjs `e96662c`, `packages/ensjs/src/actions/public/getName.ts:140-148`                        |
| Unnormalized names rejected on this path          | ensjs `e96662c`, `.../getName.ts:153`                                                          |
