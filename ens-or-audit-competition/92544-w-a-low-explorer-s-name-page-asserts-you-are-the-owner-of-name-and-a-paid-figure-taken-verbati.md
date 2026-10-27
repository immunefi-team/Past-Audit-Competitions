# 92544 w a low explorer s name page asserts you are the owner of name and a paid figure taken verbatim from the url query string on any name for any visitor beside an ownership free button&#x20;

**Submitted on Sep 10th 2026 at 15:31:38 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92544
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

`/$name/` is the Explorer's profile page and the landing page for every ENS name. It declares three search params and trusts all three:

```ts
// apps/portal/src/routes/$name/index.tsx:51-59
const validateNameSearch = (search: Record<string, unknown>): NameSearch => {
  if (search.registered !== true && search.registered !== 'true') return {}
  const duration = Number(search.duration)
  return {
    registered: true,
    duration: Number.isFinite(duration) ? duration : undefined,
    paid: typeof search.paid === 'string' ? search.paid : undefined,
  }
}
```

The only test is that `registered` equals `true`; `duration` is any finite number and `paid` any string. Nothing checks that a registration occurred, that this visitor made it, or that this visitor owns the name. On that basis the page renders:

```tsx
// apps/portal/src/features/register/components/RegistrationSuccessBanner.tsx:23-37
<p …>Congratulations!</p>
<p className="text-p">You are the owner of {name}</p>
<p className="text-h3">{formatRegistrationDuration(durationSeconds)}</p>
<p className="text-p">Registration</p>
<p className="text-h3">{paid}</p>
<p className="text-p">Paid</p>
```

So `https://<explorer>/attacker.eth/?registered=true&duration=315360000&paid=$0.00%20(free)` makes the Explorer tell the visitor, in the app's own voice and on the app's own origin, that they own a name they have never touched, for a term and a price the attacker chose.

Two facts turn that from a cosmetic lie into a funds-moving one:

1. **The app has the truth on the same screen and prints the opposite.** In the executed capture below the page simultaneously reads `You are the owner of poc-ens-support.eth`, `Owner … 0x0000…11Ac` (three rows lower, from `ownerQuery.data.owner`) and a connected wallet of `0xf39…266`.
2. **The only call to action on that page spends the visitor's money on that name.** `ExtendNameButton` carries no ownership condition (`routes/$name/index.tsx:374-379`) and `useCanExtend` no ownership term (`useCanExtend.ts:67`). Completing the flow sends `approve()` + `renew()` from the visitor's wallet and extends the **attacker's** registration.

Driven end to end through the real UI in headless Chromium against a local Anvil fork of Sepolia, starting from nothing but the attacker's link, the victim's wallet paid **8.021939 USDC** and the attacker's name's expiry moved from `Sep 10 2027` to `Sep 10 2028`. Ownership did not change.

### What is false here, and what is not

Stated up front, because it decides which impact row applies.

**Correct on every screen the victim sees:** the recipient is the deployment's real `ETHRegistrar` (`0xa885…a2Cc`), the intended recipient of a renewal; the calldata is `renew("poc-ens-support", …)`, exactly the name displayed; the amount is `$8.02` on the confirmation screen and `16043878` in the `approve()`, the true price. No address is substituted and no field of the calldata diverges from what is rendered.

**False:** an **ownership-status assertion about the visitor**, made by the application in its own voice on its own origin — *"You are the owner of X"* — together with a term and a "Paid" figure the attacker chose. That assertion is the premise on which the victim approves an otherwise correctly-described transaction.

So this is not the "wrong recipient or amount" case. It is the already-connected-wallet case: the app induced a transaction out of a connected wallet on an attacker-authored premise. That is RULES.md:903-904, and that is what this report claims; the Critical row is argued as an alternative below, not as the claim.

***

## Impact

Precondition on the victim: none beyond a wallet and opening a link. Precondition on the attacker: one ordinary `.eth` registration — 8.000021 USDC on the audited deployment — with no interaction with the victim, no approval, no NFT push and no on-chain relationship of any kind.

The chain: the attacker registers a plausible name (`ens-support.eth`, `ens-refund.eth`) and sends the victim `https://<explorer>/ens-support.eth/?registered=true&duration=315360000&paid=%240.00%20(free)` — the real Explorer, real domain, over TLS, nothing spoofed. Before any wallet is connected the page already reads **"Congratulations! You are the owner of ens-support.eth"**, **"9 years 11 months 27 days / Registration"**, **"$0.00 (free) / Paid"**. Once the victim connects — or is already connected, since the Explorer persists the connection — the page also offers **Extend**, and completing that flow builds `approve(ETHRegistrar, price × 2)` and `renew(label, duration, paymentToken, referrer)`, both `from` the victim. The victim's USDC is gone, the attacker's registration is a year longer, and the attacker's ownership is untouched.

What the page asserts against what is true, all four values read off the *same rendered page* in the executed capture:

| Rendered by the app                          | Source of that value                   | What is actually true                                                        |
| -------------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------- |
| `You are the owner of poc-ens-support.eth`   | `?registered=true` (`index.tsx:83-87`) | The connected wallet owns nothing. `Owner` = `0x0000…11Ac`                   |
| `9 years 11 months 27 days` / `Registration` | `?duration=315360000`                  | The registration is 1 year; the page's own `Expires` row says `Sep 10, 2027` |
| `$0.00 (free)` / `Paid`                      | `?paid=…`, rendered verbatim           | Nothing was paid by this visitor. The attacker paid 8.000021 USDC            |
| `Owner … 0x0000…11Ac`                        | `getEnsOwner(name)`, on-chain          | correct — and it is the attacker, not the visitor                            |

The victim then submits two transactions the app builds itself: `Approve USDC for v2 renewal` → `approve(0xa885…a2Cc, 16043878)` on MockUSDC (`useRenewalTransactions.ts:147-160` → `intents.ts:42-58`), and `Extend poc-ens-support.eth` → `renew("poc-ens-support", 31622400, 0x768F…67a39, 0x00…00)` on the `ETHRegistrar` (`useRenewalTransactions.ts:190-217`); both are decoded off the chain below. The victim's action is ordinary — the app produces this URL shape itself after a real registration (`useRegistrationSuccessRedirect.ts:39-45`), and clicking the page's one primary button is not an action outside a normal workflow.

### Why RULES.md:999 (reflected plain text injection) does not close this

The program excludes *"Reflected plain text injection (e.g. url parameters, path, etc.) — this does not exclude reflected HTML injection with or without JavaScript, or persistent plain text injection"* (RULES.md:999) — the cleanest close available against a finding that starts with a query string, so it is answered head-on.

**Conceded — `paid` and `duration` are the reflected half.** `$0.00 (free)` is the attacker's own string, printed verbatim from `?paid=` (`RegistrationSuccessBanner.tsx:35`), and `?duration=315360000` renders as `9 years 11 months 27 days`. If they were the whole finding, RULES.md:999 would apply and this would not have been filed.

**They are not the load-bearing half — `registered` is, and it is not text.** It is a **boolean flag**: nothing an attacker writes is rendered from it, and it merely selects between two of the app's own rendering branches (`index.tsx:83-87`, `:358-360`). The sentence that does the damage — *"You are the owner of {name}"* — is authored in the app's source at `RegistrationSuccessBanner.tsx:26`, and `{name}` is the route path. What the attacker controls is **whether the application makes its own ownership assertion about the visitor**, not what characters land on the page: a missing authorization check in `validateNameSearch`, a state defect rather than injected content.

**The exclusion's own boundary points the same way.** By expressly *not* excluding reflected HTML or persistent plain text, it aims at the case where echoing a string back is the entire impact. Here the impact is `approve()` + `renew()` from an already-connected wallet and 8.021939 USDC gone, executed on chain. And if the banner is nonetheless read as content modification, the row that fits is in scope: *"Injecting/modifying the static content … without JavaScript (reflected)"*, RULES.md:927, a **Medium**. Medium is the floor here, not zero.

### The extend modal does warn, and this report does not argue that away

Before any funds move, the victim clicks **I Understand** on this sentence (`ExtendNameDisclaimer.tsx:14-17`, captured verbatim in the PoC):

> Extending a name does not change the owner. Extending a name you do not own will not give you ownership of it.

That is a real warning: it states the exact proposition being exploited, it appears before the spend, and the victim dismisses it, after which four further screens display the name, term, recipient and amount correctly. **This report does not claim the victim was unwarned.** It rests on the display defect — the app asserted from an unauthenticated URL that the visitor owns the name, then submitted a transaction from that visitor's already-connected wallet.

One structural fact about that screen is worth stating anyway. `ExtendNameModal` **treats the disclaimer as the app's own "you do not own this" signal and skips it for owners**:

```tsx
// apps/portal/src/features/renew/components/ExtendNameModal.tsx:60-72
const isOwner = !!(address && ownerData?.owner && isAddressEqual(address, ownerData.owner))
useEffect(() => {
  if (open && isOwner) {
    setStep((current) => (current === 'disclaimer' ? 'settings' : current))
  }
}, [open, isOwner])
```

So one screen computes ownership correctly from chain state — and, finding the visitor is not the owner, shows the warning — while the screen behind it has already told that visitor from the URL that they *are* the owner. The warning is also generic: on none of the four screens does it say that *this* visitor is not the owner, or name the owner. Neither point is offered as proof the victim could not have known; both show the app holds the correct answer at the moment it prints the wrong one.

***

## Root cause

### The trusted branch: three values straight off the query string

| File:line                                  | Code                                                                                                                                                                                                                                     |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `routes/$name/index.tsx:45-49, 64`         | `type NameSearch = { registered?: boolean; duration?: number; paid?: string }`; `validateSearch: validateNameSearch`                                                                                                                     |
| `routes/$name/index.tsx:51-59`             | `validateNameSearch` — the only check is `search.registered !== true && search.registered !== 'true'`; `duration` is any finite `Number`, `paid` any `string`                                                                            |
| `routes/$name/index.tsx:83-87`             | <p><code>const { registered, duration, paid } = Route.useSearch()</code><br><code>const registrationBanner = registered === true && duration !== undefined && paid !== undefined ? { durationSeconds: duration, paid } : null</code></p> |
| `routes/$name/index.tsx:358-360`           | `{registrationBanner && (<RegistrationSuccessBanner name={name} {...registrationBanner} />)}`                                                                                                                                            |
| `RegistrationSuccessBanner.tsx:26, 31, 35` | `You are the owner of {name}`, `{formatRegistrationDuration(durationSeconds)}`, `{paid}` — quoted above; `{paid}` is the attacker's string rendered as the app's own "Paid" figure                                                       |

There is no producer-side authentication of any kind. The legitimate producer is a client-side navigation — `useRegistrationSuccessRedirect.ts:39-45` calls `navigate({ to: '/$name', params: { name }, search: { registered: true, duration: durationSeconds, paid }, replace: true })`. `search` is the address bar. Anyone can type it.

### The chain-derived branch, on the same page, in the same component

The same component holds the true owner at `routes/$name/index.tsx:91` (`useQuery(getEnsOwnerQueryOptions({ name }))`) and the true visitor at `:131` (`useConnection()`), and renders the owner as a row at `:393-397` (`Owner … 0x0000…11Ac`). Both operands the banner needs are in scope where its props are computed, and both are on the same screen. The banner consults neither.

### The unconditional spend button on the same page

| File:line                                              | Code                                                                                                             |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `routes/$name/index.tsx:374-379`                       | `{resolvedProtocolVersion !== 'ENSv1' && (<ExtendNameButton … />)}` — the only condition is the protocol version |
| `features/renew/components/ExtendNameButton.tsx:34,46` | `const { canExtend } = useCanExtend({ name, protocolVersion })` … `if (!canExtend) return null`                  |
| `features/renew/hooks/useCanExtend.ts:67`              | \`const canExtend = isExtendable2LD(selectedName) && (isV2                                                       |
| `features/renew/utils/nameExtension.ts:61-74`          | `isExtendable2LD` — a `.eth` 2LD inside its grace window. No ownership term.                                     |

That the ENS contracts allow anyone to renew any name is correct and is not the defect. The defect is that the page pairs that permissionless spend with an attacker-authored assertion that the spender owns the name.

***

## Preempting the likely closure arguments

**"The extend modal warns non-owners."** It does, and that is conceded in full above. The finding is not that the victim was unwarned; it is that the application asserted ownership it had not checked, from a URL, while holding the correct answer in the same component scope, and then submitted a transaction from the visitor's already-connected wallet.

**"Renewal is permissionless; anyone can renew anyone's name."** Correct, and that is the premise, not the defect. The registrar renewed the name it was told to and charged the caller. The defect is that `apps/portal` pairs that permissionless spend with its own attacker-supplied assertion that the name is the visitor's, while holding the contradicting on-chain owner in the same scope (`index.tsx:91`).

**"It's only a display bug / no funds are stolen."** Funds move: 8.021939 USDC left the victim's wallet in `0x68dc2584…` and `0x3769e66c…`, unrecoverably — `renew()` has no refund path. RULES.md:903-904 does not require the funds to reach the attacker, only a malicious transaction submitted from an already-connected wallet.

**"The search params come from the app's own redirect, so they are trusted."** `useRegistrationSuccessRedirect.ts:39-45` navigates with `search: { registered: true, duration, paid }`, and `search` is the address bar: no token, nonce, signature, origin check or chain read between the URL and the assertion.

**"The victim has to click."** The false display costs zero clicks and needs no wallet — captured below with the header reading `Connect` — because nothing in `index.tsx:83-87` or `RegistrationSuccessBanner.tsx` reads the connection. The click is how it becomes a transaction.

**"Sepolia is a testnet."** It is where the audited code runs, and nothing in the defect is network-dependent: `validateNameSearch`, the banner, `useCanExtend` and `useRenewalTransactions` have no such branch.

**"An attacker could just build a phishing site."** Not *this* one: the assertion is served by the genuine Explorer, on the genuine origin, over the genuine certificate, alongside genuinely-correct on-chain data, and the wallet prompt is the real app's.

### The Critical row, and why this report does not lead with it

RULES.md:891-892 is *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient"*, illustrated by *"resolving or rendering an ENS name to an attacker-controlled address"* and *"showing the wrong recipient or amount on a transaction the user then approves"*. Neither example is met, and the report does not pretend otherwise: as conceded above, the displayed address, the displayed name, the recipient and the amount are all **correct**. What is incorrect is an **ownership-status assertion about the visitor**, plus a "Registration"/"Paid" pair describing a *past* registration.

The row is argued in the alternative only on this reading: those values are transaction details the application displays, they are false, and the user submits a transaction on the strength of them. A reviewer who accepts it lands on Critical; one who does not still lands on RULES.md:903-904 (High), and at worst on RULES.md:927 (Medium). None of the three readings is zero.

### #91314 was closed on an objection that lands on this PoC verbatim — here is the answer

Report **#91314** (`apps/portal`, a victim paying to renew a name they do not own) was **closed**, with this reason on the record:

> The report correctly identifies that the V1 names reader omits an ownership check that its sibling readers include … However, the submitted proof stops short of showing the claimed impact: **the USDC spend it measures is produced by the test harness itself driving the victim account through the renewal flow, not by demonstrating that a real user would be induced to select and approve payment for a name they don't own.** The wallet prompt a victim would see displays the correct name and amount, so the gap between a misleading list entry and an actual financial loss to a real victim is not bridged by the evidence provided.

That objection applies to the *mechanics* here too: a harness drives the victim account, and the wallet prompt does display the correct name and amount — conceded, and precisely why this report claims RULES.md:903-904 rather than the wrong-recipient row. The difference is what the objection turns on: **the inducement**. In #91314 it was a *list entry* — a name in a "your names" table because of an ensjs relation default — and the user had to decide, unprompted, to renew a row they had no reason to act on. Here there is nothing to select: the victim arrives on **one name's page** from a link, and the app states in its own authored sentence (`RegistrationSuccessBanner.tsx:26`) **"You are the owner of poc-ens-support.eth"**, above a fabricated term and "Paid" figure, with `Extend` as the only call to action.

The harness clicks the app's own buttons in the app's own order — no injected state, no console calls, no direct contract calls. Every click is one a user makes; what this report supplies that #91314 could not is the application's own sentence telling the user to make them.

### This is the #91851/#91722/#91837 family — and it supplies the element ENS said was missing

ENS has treated #91851, #91722 and #91837 as one family and graded it **Insight** on a stated basis (fgeorgescu, on #91851, Sep 8):

> On classification, we are applying the same reading we applied to #91722 and #91837, which reach the same end state. The transaction sets a reverse record on the victim's own address naming a third party's name. **No funds move, no name changes owner**, and forward resolution of the victim's names is unchanged.

and, on #91837:

> No name changes owner, expiry or records. Not a hijack or brick. **No funds move and no approval is granted.** Forward resolution is unchanged.

Both statements are precise, and both are satisfied on the opposite side here:

| ENS's stated test                          | This finding                                                                                                                                                                                   |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "No funds move"                            | **8.021939 USDC left the victim's wallet**, in `0x68dc2584…` and `0x3769e66c…`. Executed and decoded from the chain, not narrated. Not recoverable: `renew()` has no refund path.              |
| "No approval is granted"                   | An **`approve(ETHRegistrar, 16043878)`** was submitted from the victim's wallet — `0x5e33665b…` / `0x29233252…` — for exactly `2 ×` the charge, built by `useRenewalTransactions.ts:147-160`.  |
| "No name changes owner, expiry or records" | The **expiry of a third party's name moved +31 622 400 s**, paid for by the victim. Ownership is indeed unchanged — which is why the attacker's gain is a free year rather than a stolen name. |
| "Forward resolution is unchanged"          | Agreed, and irrelevant: nothing here depends on resolution.                                                                                                                                    |

The family was discounted because the induced transaction had no cost to the victim and no effect on any name. Here it has both — which, on ENS's own reasoning, is the element whose absence produced the Insight grade.

***

## Not a duplicate

Against the submissions already filed in this competition:

| Report                                                                                                        | Why this is not it                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **#91851** (`/$name/address` offers "Set primary name" for an unrelated name)                                 | A missing authorization predicate on a *write*: `canSetPrimaryName` (`AddressResolutionSidebar.tsx:454-458`) checks wallet-equality but not authority over the name, and the harm is a `setName()`. This is a fabricated *read* — `validateNameSearch` (`routes/$name/index.tsx:51-59`) accepts an unauthenticated URL claim of ownership — and the harm is `approve()` + `renew()`. Neither fix touches the other's files. |
| **#89441** (`allowMismatch` certifies an unverified reverse record)                                           | Read-path bug in `features/reverse-resolution/`, about ENSIP-3 verification. No search params, no banner, no spend.                                                                                                                                                                                                                                                                                                         |
| **#91314** (`useV1NamesForAddress` includes `resolvedAddress`; **closed** — answered above)                   | Closest in *spirit*, but its root cause is an over-broad ensjs relation filter in a dashboard query. Here nothing is queried, and its fix (`resolvedAddress: false`) leaves this fully live.                                                                                                                                                                                                                                |
| **#92043 / #90190 / #90163** (renewal sends the wrong label)                                                  | Raw-vs-normalized divergences in the *calldata*. Here the label is correct — `renew("poc-ens-support", …)` is exactly the name displayed.                                                                                                                                                                                                                                                                                   |
| **#90307** (uint64 expiry crashes the names page)                                                             | A crash on a read of on-chain data. Here `duration` comes from the URL and does not crash — it renders, which is worse.                                                                                                                                                                                                                                                                                                     |
| **#89270 / #90161 / #91837 / #91900 / #91306 / #89279 / #89444 / #89445 / #89460 / #89462 / #91838 / #91839** | Different apps (`apps/manager`), subsystems (`workers`, `packages/migration`, `packages/smart-account`, `packages/transaction-manager`), or files. None involve `/$name/` search params, the banner, or the extend flow.                                                                                                                                                                                                    |

Against the published Known Issues: **QA-05** (RULES.md:786-790) is a *liveness* defect renewing your own name, where this renewal succeeds on someone else's; **QA-10** (811-815, avatars in bulk renewal) and **WEB-1304** (835, NameWrapper expiry staleness) touch neither ownership display nor search params; **EXP-INPUT-003** concerns the persisted Zustand/xstate store rather than TanStack Router search params, and its stated consequence is a shape-parsing failure, where here the shapes are valid and the *claim they encode* is unauthenticated; **EXP-INPUT-005** self-scopes to "homograph-style display confusion … rather than incorrect resolution", and no deceptive characters are involved; **R2-02** (no CSP) is not the vector, since the banner comes from the app's own legitimate code path; **SEC-MGR-008 / 011** are `apps/manager` debug routes and **R3-01…R3-08** transaction-manager defects, while the transaction manager here reported both transactions accurately.

**WEB-626** (RULES.md:826, a grace-period owner shown a modal implying the name is someone else's) deserves its own line: it is the exact **inverse** of this finding, and its fix — suppressing the disclaimer for grace-period owners — makes this one strictly worse, by removing the only caution the flow shows.

The known-issues document states its audits "focused on application source directories" and were not exhaustive. `validateNameSearch` and `RegistrationSuccessBanner.tsx` appear nowhere in it.

***

## Recommendation

1. **Do not let the URL assert ownership.** Both operands are already in scope. In `routes/$name/index.tsx:83-87`:

```diff
-  const registrationBanner =
-    registered === true && duration !== undefined && paid !== undefined
-      ? { durationSeconds: duration, paid }
-      : null
+  // A URL cannot prove a registration. Only render the success banner when the
+  // connected wallet is in fact the on-chain owner of this name.
+  const isConnectedOwner =
+    !!connectedAddress &&
+    !!ownerQuery.data?.owner &&
+    isAddressEqual(connectedAddress, ownerQuery.data.owner)
+  const registrationBanner =
+    isConnectedOwner &&
+    registered === true && duration !== undefined && paid !== undefined
+      ? { durationSeconds: duration, paid }
+      : null
```

(`connectedAddress` is read at `:131` and `isAddressEqual` imported at `:3`; move the `useConnection()` call above the banner computation.)

2. **Do not render an attacker-supplied string as a monetary figure.** `paid` should be a number the page formats itself, and both `paid` and `duration` cross-checked against `getV2RegistrationData` — the page already fetches those fields (`useV2RegistrationData.ts:34-44`) and renders them as `Registered` / `Expires`. Better still, drop the params and derive the banner from the registration the page can already see.
3. **Tell the truth on the extend surface.** `ExtendNameModal` already computes `isOwner` (`:60-64`); when it is false, say so concretely on the settings and confirmation screens — `You are not the owner of ${name}. It is owned by ${ownerDisplay}.` — not on one screen the user has already dismissed.
4. **Gate the CTA, or label it.** `ExtendNameButton` (`useCanExtend.ts:67`) may legitimately stay open to non-owners — gifting a renewal is a real use — but it should render as an explicitly third-party action (`Extend for owner`), never as the primary button on a page carrying an unverified ownership banner.
5. **Audit the other unauthenticated search params.** `/register?name=` is the only other one, and its `validateSearch` (`routes/register/index.tsx:50-56`) does a `trim()` / `!== '.eth'` check only — the real validation runs downstream, where `RegisterName.tsx:42` passes the value through `validateRegistrableEthName`. That two-stage pattern is what `/$name/` should follow, with `getEnsOwner` as the authority.

## Proof of Concept

## Proof of Concept

Everything below was executed. No live network was written to; the fork performs reads against Sepolia and all writes are local.

### Environment

```
checkout : /Users/bp/x/ens/5   (audit-comp-ready @ 1c9b47f)
fork     : anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
                 --chain-id 11155111 --port 8623 --silent
app      : pnpm --filter portal exec vite dev --port 9623     # the package is named `portal`
browser  : playwright chromium (headless)
```

PoC scripts, all shipped, all at the **repo root** `.poc/` except the state reader:

```
.poc/91851b-setup.mjs         attacker registers one ordinary name; victim is funded
.poc/91851b-indexer-stub.mjs  30-line stand-in for the ENS indexer (port 9624)
.poc/91851b-render.mjs        control render vs attacker-link render
.poc/91851b-drive.mjs         drives the whole flow in headless Chromium
.poc/91851b-txs.mjs           walks the fork's tail blocks, decodes the victim's txs
apps/portal/.poc-91851b-state.mjs   owner / expiry / USDC via the app's own ensjs stack
```

Every script honours `POC_RPC` (and `POC_BASE` for the drive), so the PoC runs on any port pair. `apps/portal/.env.local`:

```
VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8623
VITE_SEPOLIA_RPC_URL_SERVER=http://127.0.0.1:8623
VITE_USE_MOCK_WALLET=true
VITE_MOCK_ACCOUNT=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:9624/
```

`VITE_USE_MOCK_WALLET` is the repo's own Playwright harness (`src/lib/mockWallet.mock.ts:21`): it forwards `eth_sendTransaction` to the chain's RPC so a headless browser can sign without a popup, changing *who signs*, not what the app builds or renders. `VITE_INDEXER_GRAPHQL_URL` points at the stub, which answers one query — `getRegistrationAndExpiry` in `useV2RegistrationData.ts:34-44` — with the fork-registered name's true registration date and expiry, and `{ domains: [] }` otherwise; the production indexer returns that row for every registered name, and nothing else in the flow reads it.

Roles: `ATTACKER = 0x00000000000000000000000000000000000a11Ac`, `VICTIM = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` (the mock wallet's account, i.e. the connected wallet in the browser).

### Setup — the attacker registers one ordinary name (`.poc/91851b-setup.mjs`)

```js
const ETH_REGISTRAR = getAddress('0xa88553F454b77203B0D036A05c894d555EAAa2Cc')
const USDC          = getAddress('0x768F42455A2D082E23ceeF7d51e5787C82d67a39')
const ATTACKER      = getAddress('0x00000000000000000000000000000000000a11ac')
const VICTIM        = getAddress('0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266')

const [base, premium] = await pub.readContract({
  address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'getRegisterPrice',
  args: [LABEL, DURATION, USDC] })
const price = base + premium
await send(ATTACKER, USDC, encodeFunctionData({ abi: erc20Abi, functionName: 'mint',    args: [ATTACKER, price * 4n] }))
await send(ATTACKER, USDC, encodeFunctionData({ abi: erc20Abi, functionName: 'approve', args: [ETH_REGISTRAR, price * 4n] }))
const commitment = await pub.readContract({ address: ETH_REGISTRAR, abi: registrarAbi,
  functionName: 'makeCommitment',
  args: [LABEL, ATTACKER, SECRET, zeroAddress, zeroAddress, DURATION, REFERRER] })
await send(ATTACKER, ETH_REGISTRAR, encodeFunctionData({ abi: registrarAbi, functionName: 'commit', args: [commitment] }))
await rpc('evm_increaseTime', [Number(await pub.readContract({ address: ETH_REGISTRAR, abi: registrarAbi,
  functionName: 'MIN_COMMITMENT_AGE' })) + 60])
await rpc('evm_mine', [])
await send(ATTACKER, ETH_REGISTRAR, encodeFunctionData({ abi: registrarAbi, functionName: 'register',
  args: [LABEL, ATTACKER, SECRET, zeroAddress, zeroAddress, DURATION, USDC, REFERRER] }))

// final step: the victim is an ordinary holder of the payment token and nothing else.
await send(VICTIM, USDC, encodeFunctionData({ abi: erc20Abi, functionName: 'mint', args: [VICTIM, 1_000_000_000n] }))
console.log('victim USDC:', (await pub.readContract({
  address: USDC, abi: erc20Abi, functionName: 'balanceOf', args: [VICTIM] })).toString())
```

Captured output of the original run:

```
fork: http://127.0.0.1:8623 | block 11669628n
register price: 8.000021 USDC
registered poc-ens-support.eth to ATTACKER 0x00000000000000000000000000000000000a11Ac
```

The victim's 1000 USDC mint is the script's last step; there is no approval and no transaction between attacker and victim at any point. It makes the `1997838208` "before" figure decompose exactly — `997838208`, the real Sepolia balance of `0xf39F…266` at the fork block, `+ 1000000000` — as confirmed on the re-run below.

### State before, read through the app's own ensjs stack

`apps/portal/.poc-91851b-state.mjs` reads `getExpiry` / `getOwner` from `@ensdomains/ensjs/public/v2` over `extendChainWithEns` — the stack the app itself uses — and prints:

```json
{
  "name": "poc-ens-support.eth",
  "owner": "0x00000000000000000000000000000000000a11Ac",
  "expiry": "1820513809",
  "usdcOf": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "usdc": "1997838208"
}
```

### The flow, driven through the real UI (`.poc/91851b-drive.mjs`)

The script opens the attacker's URL in headless Chromium and clicks only the app's own buttons: `Extend` → `I Understand` → `Next` → the USDC row → `Confirm` → `Start` → `Open wallet`. It also serves the app's two public-Sepolia failover RPCs from the fork, since viem's `fallback()` transport would otherwise read from real Sepolia, where the fork-registered name does not exist.

```js
const URL_ = `${BASE}/${NAME}?registered=true&duration=315360000&paid=%240.00%20(free)`
…
await page.goto(URL_, { waitUntil: 'domcontentloaded' })
```

**Step 1 — the page as rendered (`innerText` of `<body>`).** The elision is the app's own chrome — search box, sidebar toggle, section nav — plus one line named rather than hidden: the Explorer's *"Notice: ENS v2 is in active development…"* banner and `Dismiss` button, which sits directly above the fabricated banner (quoted in full in the re-run capture) and warns about testnet state resets, not ownership.

```
  | poc-ens-support.eth
  | [… 9 section-nav rows elided: Address Resolution … History …]
  | 0xf39…266                     <-- the connected wallet
  | Congratulations!
  | You are the owner of poc-ens-support.eth
  | 9 years 11 months 27 days
  | Registration
  | $0.00 (free)
  | Paid
  | poc-ens-support.eth
  | Extend                        <-- the only CTA on the page
  | poc-ens-support.eth
  | Expires
  | Sep 10, 2027 2:15am           <-- one year, not ten
  | Registered
  | Sep 10, 2026 2:15am
  | Owner
  | Address
  | Copy
  | 0x0000…11Ac                   <-- the real owner: the ATTACKER
  | [… 11 rows elided: Parent / Resolver / Subregistry / Protocol …]
```

A control render of the same page **without** the query string, from the same script, contains none of it:

```
control contains "Congratulations!": false
attack  contains "Congratulations!": true
attack  contains "You are the owner of": true
attack  contains "$0.00 (free)": true
attack  contains "Extend": true
```

**Step 2 — after clicking `Extend`, the modal's first screen, verbatim:**

```
  | Extending a name does not change the owner. Extending a name you do not own will not give you ownership of it.
  | I Understand
  | Close
```

**Step 4 — the settings screen, verbatim:**

```
  | Extend name
  | poc-ens-support.eth
  | For
  | Pick by date
  | 1.00 year
  …
  | Extension:
  | 1 year
  | New expiry:
  | Sep 10, 2028
  | Price:
  | $8.01/year
  | Total:
  | $8.02
  | Next
```

**Step 6 — the confirmation screen, verbatim:**

```
  | Confirm extension
  | poc-ens-support.eth
  | Extension
  | 1 year
  | 366 days
  | New expiry
  | Sep 10, 2028
  | in 731 days
  | Total cost
  | $8.02
  | Select payment method
  | USDC
  | 1,997.838
  | available
  | DAI
  | 1,000
  | available
  | Confirm
```

**Step 7 — the transaction modal, verbatim:**

```
  | Transaction flow
  | Approve USDC for v2 renewal
  | Extend poc-ens-support.eth
  | Open wallet
```

No screen in the flow names the owner, or states that the connected wallet is not the owner.

### What actually landed on chain

Pulled from the fork by `.poc/91851b-txs.mjs`, which walks the tail blocks and decodes every transaction sent `from` the victim. The original run used an unversioned copy of the same script; the shipped one's output on a fresh fork is quoted in the re-run below.

```json
{
  "block": 11669635,
  "hash": "0x5e33665b335084cb98a3bd62aebd992830f1824d8ffedb1f1987f49c04c7a41c",
  "from": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
  "to":   "0x768f42455a2d082e23ceef7d51e5787c82d67a39",
  "fn":   "approve",
  "args": ["0xa88553F454b77203B0D036A05c894d555EAAa2Cc", "16043878"]
}
{
  "block": 11669636,
  "hash": "0x68dc2584116d9c8c8e1e5c49e5a7a99116526222b1a3f3661bf6d478c4c986cc",
  "from": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
  "to":   "0xa88553f454b77203b0d036a05c894d555eaaa2cc",
  "fn":   "renew",
  "args": ["poc-ens-support", "31622400",
           "0x768F42455A2D082E23ceeF7d51e5787C82d67a39",
           "0x0000000000000000000000000000000000000000000000000000000000000000"]
}
```

### State after

Same reader, after the flow:

| Quantity                       | Before                     | After                      | Delta                         |
| ------------------------------ | -------------------------- | -------------------------- | ----------------------------- |
| Victim USDC (6 dp)             | `1997838208`               | `1989816269`               | **−8.021939 USDC**            |
| Attacker's name expiry         | `1820513809` (Sep 10 2027) | `1852136209` (Sep 10 2028) | **+31 622 400 s = +366 days** |
| Owner of `poc-ens-support.eth` | `0x…a11Ac`                 | `0x…a11Ac`                 | unchanged                     |

The victim paid, the attacker's asset appreciated, the victim received nothing — and every number came off the fork after the browser drove the app's own buttons.

### Independent re-run — and the false banner with **no wallet connected at all**

The whole PoC was re-run from scratch on a second fork (`--port 8654`, Vite on `9654`, indexer stub on `9655`, fork block `11670103`) using only the shipped scripts. The name is `poc-ens-support-2.eth` on this run, because `poc-ens-support` is no longer available on Sepolia; nothing else differs.

**(1) The banner needs no wallet.** This leg set `VITE_USE_MOCK_WALLET=false`, so the app registers no connector and no wallet is or can be connected. The page still renders the banner in full. `innerText` of `<body>`, with nothing whatever elided from the top of the page down to the true owner's address:

```
  | ⌘K
  | Search
  | Search for ENS names or Ethereum addresses
  | Toggle Sidebar
  | poc-ens-support-2.eth
  | Address Resolution
  | Records
  | Resolver
  | Ownership
  | Roles
  | Subnames
  | Registry
  | Token info
  | History
  | Connect                       <-- no wallet is connected, and none can be
  | Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026.
  | Dismiss
  | Congratulations!
  | You are the owner of poc-ens-support-2.eth
  | 9 years 11 months 27 days
  | Registration
  | $0.00 (free)
  | Paid
  | poc-ens-support-2.eth
  | Extend
  | poc-ens-support-2.eth
  | Expires
  | Sep 10, 2027 3:56am
  | Registered
  | Sep 10, 2026 3:56am
  | Owner
  | Address
  | Copy
  | 0x0000…11Ac                   <-- the real owner: the ATTACKER
  | [… 22 further rows elided below the owner: Parent / Resolver / Subregistry / Protocol / Subnames / Roles / Records / HISTORY …]

control contains "Congratulations!": false
attack  contains "Congratulations!": true
attack  contains "Connect": true
attack  contains "0x0000…11Ac": true
```

The header reads `Connect`. There is no connected wallet, no session and no signature anywhere in this render, and the application still asserts that the visitor owns the name — three rows above the true owner's address. The app's own development notice, the page's only caution, sits directly above the fabricated banner and is about testnet state resets, not ownership.

**(2) The spend reproduces to the exact wei.** The second leg turned the mock wallet back on and let `.poc/91851b-drive.mjs` click the app's own buttons. Balances, through `apps/portal/.poc-91851b-state.mjs`:

```
after the attacker registers, before the victim is funded :  usdc  997838208   <- real Sepolia balance
after the victim is minted 1000 USDC                      :  usdc 1997838208   <- the "before" figure
after the browser drove Extend -> … -> Open wallet         :  usdc 1989816269   <- -8.021939 USDC
```

Expiry `1820519766 → 1852142166` = **+31 622 400 s**, owner `0x…a11Ac` unchanged (`getOwner` through the app's own stack, before and after). Absolute timestamps differ from the original run because the fork block differs; every delta is identical.

The shipped block-walker (`POC_RPC=http://127.0.0.1:8654 node .poc/91851b-txs.mjs 40`) reports three transactions from the victim: the funding `mint(0xf39F…266, 1000000000)` at block `11670114` (`0x9275af90…`), reported so the balance arithmetic is fully accounted for, then the two the application submitted — `approve` with args `["0xa88553F454b77203B0D036A05c894d555EAAa2Cc", "16043878"]` at `11670115` (`0x29233252…`) and `renew` with args `["poc-ens-support-2", "31622400", "0x768F42455A2D082E23ceeF7d51e5787C82d67a39", "0x00…00"]` at `11670116` (`0x3769e66c…`). The decode is identical to the original run apart from the label and the hashes, and `16043878` is exactly `2 × 8 021 939`, matching `tokenPrice * 2n` at `useRenewalTransactions.ts:147-160`.

Two artifact defects in the original run are fixed in the shipped scripts: `91851b-drive.mjs` no longer ends in `ReferenceError: rpcCalls is not defined` (the chain evidence is read back by `91851b-txs.mjs`, not in the browser), and `91851b-setup.mjs` now funds the victim and no longer calls `available()` on the name it just registered. Both runs exited 0. Screenshots: `.poc/91851b-6-nowallet.png`, `.poc/91851b-r1-banner.png` … `-r5-final.png`.

***
