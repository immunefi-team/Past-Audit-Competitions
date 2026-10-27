# 92925 w a critical the manager profile page normalizes the name for the owner query but not for the records query resolving an ens name to an attacker controlled address

**Submitted on Sep 13th 2026 at 09:00:54 UTC by @bicktivism for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92925
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

**In-scope impact** — this Critical row of the Impacts in Scope table, quoted in full. (The table’s rows carry no numbers on the page, so it is cited by its text.)

> "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves"

The first sub-bullet is this finding, in the programme's own words — *resolving or rendering an ENS name to an attacker-controlled address*. The impostor name resolves, through the app's own `getOwner`, to a node the attacker owns.

**The published exclusion this has to clear** is known issue EXP-INPUT-005, quoted in full rather than in part, because half of it describes the input here:

> "EXP-INPUT-005 (Medium) — ENS name validators accept inputs that ENS normalization later rejects or transforms — homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot — causing homograph-style display confusion in the UI rather than incorrect resolution."

(The scope page states the same entry at greater length: "Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution." The two wordings make the same claim about impact, and it is that claim this report answers.)

The first sentence does cover the input: U+200B is an "other control character" and ENSIP-15 transforms it away. The exclusion then makes a claim about the *impact* — display confusion in the UI, not incorrect resolution — and that claim is what fails here. On the fork, `getOwner(impostor)` returns **B** while the page renders **A**, and `addr(60)` on the impostor node reads back **B**. The reader is shown one party's ownership beside another party's receiving address. That is incorrect resolution, which is the thing the exclusion says this class does not do.

**Asset:** Manager app. **Repo** `immunefi-team/audit-comp-ens`, branch `audit-comp-ready`, **commit** `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. **Files:** `apps/manager/src/routes/$name/index.tsx`, `apps/manager/src/features/profile/service/profileRecords.ts`, `apps/manager/src/features/profile/service/profileOwner.ts`.

***

## Brief/Intro

The Manager profile route loads two queries **from the same raw URL parameter**, and they disagree about what that parameter means. `profileOwner.ts:24` normalises it first (`normalizeEthName` → viem `normalize`); `profileRecords.ts:123` passes it **raw** to `getRecords`. For any label containing a default-ignorable code point — `U+200B` is the clearest — `normalize` folds it away, so the owner half silently retargets to a **different name** while the records half stays on the one the visitor typed.

The result on screen: the page shows **the real name's owner and registration dates** next to **the impostor's address records**. The reassurance and the payment detail come from two different names, and the page presents them as one identity.

Neither half of EXP-INPUT-005's carve-out survives, and not by argument: there is no visual difference for a user to notice, and resolution is exactly what diverges.

**Who can do this, and whose money is at risk.** The attacker is anyone. No ENS role, no admin, no registrar privilege and no action by the real name's owner is required at any step: the impostor registers an ordinary, available label the way any visitor would, and Part 1 of the proof below is reproduced with no wallet, no key and no registration at all. The party who loses is the visitor who reads the page — they see the genuine owner and the genuine registration dates, treat that as confirmation they are on the right name, and send funds to the address the impostor controls. The value taken belonged to that visitor before the attack; nothing here depends on a privileged party acting, correctly or otherwise.

***

## Vulnerability Details

`apps/manager/src/routes/$name/index.tsx` — one input, two queries:

```js
const [profileRecords, ownerData] = await Promise.all([
  queryClient.ensureQueryData(profileRecordsQuery(name)),
  queryClient.fetchQuery(profileOwnerQuery(name)),
])
```

| half        | call site                                                                 | what it resolves        |
| ----------- | ------------------------------------------------------------------------- | ----------------------- |
| **OWNER**   | `profileOwner.ts:24` → `normalizeEthName(params.name)` → viem `normalize` | the **normalised** name |
| **RECORDS** | `profileRecords.ts:123-124` → `getRecords(client, { name, … })`           | the **raw** name        |

Driven on the shipped modules through a recording transport, so the name each half asks about is **observed in the request bytes** rather than computed. The ENS/DNS wire encoding carries a length prefix per label, so the two packets differ in their first byte and neither is a substring of the other:

```
POSITIVE "vitalik.eth"
  owner side  normalizeEthName -> "vitalik.eth"
  records side eth_calls recorded: 4
  its DNS packet 07766974616c696b0365746800 on the wire: true
NEGATIVE "café.eth"
  owner side  normalizeEthName -> "café.eth"
  records side asks for packet 05636166c3a90365746800 -> true
  owner and records agree: true
ARM      "vitalik​.eth"
  url label utf8            : 766974616c696be2808b2e657468
  OWNER   normalizeEthName  : "vitalik.eth"   <- the REAL name, folded
  RECORDS eth_calls recorded: 4
  RECORDS asks for RAW      : 0a766974616c696be2808b0365746800 -> true
  RECORDS asks for CLEAN    : 07766974616c696b0365746800 -> false
  first recorded calldata   : 0x9061b9230000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000
  *** ONE PAGE, TWO NAMES: owner = "vitalik.eth" , records = "vitalik​.eth" ***
```

The `café` row is what makes this a defect rather than a curiosity: **ordinary non-ASCII does not split the page.** The mechanism is default-ignorable code points specifically.

**This is not an `@ensdomains/ensjs` bug.** ensjs DNS-encodes the string it is handed. ENS's own documentation puts the duty on the integrator — *"You MUST normalize a name before you attempt to create a namehash! If you don't, then the hash you get may be incorrect."* (`docs.ens.domains/resolution/names`) — and `profileOwner.ts:24` performs exactly that normalisation **twelve lines away in the same loader**. Two in-scope call sites, one input, two answers.

***

### Independent reproduction

Reproduced by a second researcher on a **separate fork**, with different labels, different owner addresses, an independently written harness and a **symmetric indexer stub** — one that returns `{"domain":null}` for every id identically, so it cannot prefer one name over the other. Their stub logged the ids the app requested: on the impostor page it was handed the **raw impostor string** and returned nothing, and the page still printed the clean name's owner. Six checks with controls, no disagreement with the results above. That rules out the indexer fixture as the source of the owner value.

***

### Recommendation

Normalise **once, at the earliest point**, and pass the normalised name to both halves. Concretely: in `routes/$name/index.tsx`, resolve the route parameter through `normalizeEthName` before either query is constructed, and have `profileRecordsQuery` take the normalised value; render the not-found fallback for a name that fails to normalise rather than querying it raw. This matches the Security Alliance ENS framework's rule — *"always normalize ENS names before creating namehash, labelhash, or DNS-encoding"*, at *"the earliest possible point in your ENS handling logic"*.

**Note:** this fix does **not** close the register route, where `parseName` (`features/register-v2/utils/name-parser.ts`) applies a JavaScript `\s` test and no ENS normalisation, so the Manager will still sell these labels. That is a separate defect with a separate fix.

***

### Known prior art, disclosed here

The **class** is published: *"Beyond Visual Confusion: Understanding How Inconsistencies in ENS Normalization Facilitate Homoglyph Attacks"*, Huang, Chintapalli, Wang and Gu, Proceedings of the ACM Web Conference 2025, doi `10.1145/3696410.3714675` — inconsistent normalisation **between ENS clients**. ENSIP-15 exists in part to prevent zero-width insertion.

**What that work does not cover** is this: a divergence **inside a single view of a single client**, between two call sites twelve lines apart in one route loader. We found no known-issues row or prior report describing an owner/records split, or any divergence between two resolution paths in one page.

***

### Limits, stated here rather than left to be found

* **The registration is on a fork of deployed Sepolia, not on live Sepolia.** Every contract is the real deployed one — the fork is taken at whatever Sepolia head it is run against, 11,694,577 for the registration transcript attached here and 11,694,591 for the ownership probe — and the calldata is the app's own, but the transactions exist only in a local fork. We **did not attempt** a live-Sepolia broadcast and we **are not claiming** one.
* **Both names in the demonstration are ours.** The impostor imitates a nonce label registered seconds earlier, not anyone's identity — deliberately, so the mechanism is shown without registering an impostor of a real name. Against a real name the display is identical; only the victim changes.
* **Two earlier findings of ours were withdrawn after being refuted by our own browser runs**: the Explorer's register route *does* validate, in `RegisterName.tsx:87`. We report that here because it is the reason we tested the Manager separately rather than assuming the two apps behave alike.

***

## Impact Details

**An attacker — any unauthenticated address, no role, no privilege — calls the Manager's register flow for a label one invisible code point away from a name a third party owns, pays the ordinary 8.000021 USDC for the year, then calls `setAddr` to point its records at themselves, and thereafter the Manager profile page for that impostor displays the real name's owner and dates beside the attacker's receiving address.** A victim who reads that page and pays it, pays the attacker.

**The user is never shown an address complete enough to check, and the only one they can obtain is the attacker's.** The page truncates every address it displays to four-and-four. On the impostor page the rendered text contains `0x1eF5...A9D3` (the real owner) and `0x00F9...5695` (the attacker) and **no full address at all** — a truncated A and a truncated B are the same visual shape, so there is nothing to compare. The one place a complete address exists is the copy-to-clipboard control's `title`, and on the impostor page that is `BUTTON[title="0x00F9d912DF6198d94581F6Bb51bdf40b4AD95695"]` — the attacker's — while the same control on the clean page carries the real owner's:

```
  THE ARM - the registered impostor
    carries A (the owner shown on the page) : NO
    carries B (the attacker's address)      : ['BUTTON[title="0x00F9d912DF6198d94581F6Bb51bdf40b4AD95695"]']

    *** HOVERING THE COPY BUTTON RETURNS THE ATTACKER'S ADDRESS IN FULL. ***
    The page names A as Owner and hands the careful user B.

  DOES ANYTHING MARK THE IMPOSTOR AS DIFFERENT?
    extra arias        on the impostor: []
    extra alts         on the impostor: []
    extra decorated    on the impostor: []
    extra pseudo       on the impostor: []
    extra flagClasses  on the impostor: []
    attributes carrying U+200B verbatim  : 1
```

**Hovering or copying to verify the address before paying is the delivery mechanism.**

**No signal at any layer.** Not in the pixels (163.69 px vs 163.69 px, with a one-character control at 171.13 px). Not in the price (8000021, byte-identical to what the registrar quotes for a fresh label nobody wants). Not in the markup: `title`, `aria-label`, `alt`, `::before`/`::after`, `text-decoration` and class names were harvested off the live page and **nothing extra appears on the impostor** — with the harvester's own control finding 61 attributes on the clean page, so the harvester was shown able to say otherwise. Not in the accessible tree either: `IMG[alt="…<U+200B>.eth pattern"]` carries the invisible code point verbatim, so a screen-reader user gets no signal.

**Quiet by construction.** List views use `domains`/`where.name`, which the indexer's own schema documents as **default-hiding non-normalised names**, while the profile page uses exact get. A registered impostor appears nowhere a user or an operator would browse — only where a victim is sent.

**Cost to the attacker:** 8.000021 USDC per year plus gas, repeatable per name and per year. **Recoverability:** a payment to the attacker's address is an ordinary transfer to an address they control — no sweep, no emergency-withdraw, no admin setter, no retry returns it. Fixing the validator stops new impostor sales but **cannot undo** registrations already made.

***

## References

**Code under test**

* Repo `immunefi-team/audit-comp-ens`, branch `audit-comp-ready`, commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* `apps/manager/src/routes/$name/index.tsx` — the single input that feeds both queries
* `apps/manager/src/features/profile/service/profileOwner.ts` — normalises (`normalizeEthName`, line 25)
* `apps/manager/src/features/profile/service/profileRecords.ts` — does not normalise (`getRecords(client, { name })`)

**Standards**

* ENSIP-15, ENS Name Normalization Standard — <https://docs.ens.domains/ensip/15>
* `@adraffy/ens-normalize`, the reference implementation the apps build against

**Programme documents quoted in this report**

* Impacts in Scope — <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Known Issues, EXP-INPUT-005 — <https://immunefi.com/audit-competition/audit-competition-ens/information/>

## Link to Proof of Concept

<https://gist.github.com/bilguunbicktivism/65c6472b0e4506a27571ff8f6d20c24f>

## Proof of Concept

## Proof of Concept

Two parts. Part 1 shows the split with no wallet, no key and no registration, and is the faster check. Part 2 fills the impostor's records so the two addresses appear side by side. Every block below is copied verbatim from a run transcript.

The shortest runnable check, needing only the clone and about five minutes:

```bash
git clone <audit-comp-ens> && cd ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install
cd apps/manager && VITE_INDEXER_GRAPHQL_URL=https://staging-graphql.ens.dev/ npx vite dev --port 5302 --host 127.0.0.1
# then open, and compare the Owner line on the first two:
#   http://127.0.0.1:5302/increasableness.eth
#   http://127.0.0.1:5302/increasableness%E2%80%8B.eth      <- same Owner, name does not exist
#   http://127.0.0.1:5302/zzq9x7v4nope00000.eth             <- control: no Owner at all
```

The scripts that produced every transcript quoted here are attached: `drive/drive25_profile_split.py` (the wire capture), `drive/drive29_dom_signals.py` (the attribute harvest), `src/__poc_register.test.ts` (the fork registration) and `src/__probe_owner_of.test.ts` (which node owns what). Each prints its reach gate and both controls before it reports any arm.

### Part 1 — the split, with no wallet, no key and no registration (about 5 minutes)

1. Clone the audited commit and install:

   ```bash
   git clone <audit-comp-ens> && cd ens
   git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
   pnpm install
   ```
2. Start the Manager against a reachable indexer:

   ```bash
   cd apps/manager
   VITE_INDEXER_GRAPHQL_URL=https://staging-graphql.ens.dev/ npx vite dev --port 5302 --host 127.0.0.1
   ```

   This variable is required. With the default local indexer unreachable, `fetchIndexerDomain` throws **before** `getRecords`, so every profile - including valid ones - renders "Error loading profile" and the defect is hidden.
3. Pick any registered name, asking the indexer for one so you are not relying on ours:

   ```bash
   curl -s https://staging-graphql.ens.dev/ -H 'content-type: application/json'      -d '{"query":"{ domains(first: 3) { id name } }"}'
   ```
4. Open these three URLs and compare the **Owner** line on the first two:

| # | URL                                           | what you see                                                  |
| - | --------------------------------------------- | ------------------------------------------------------------- |
| 1 | `http://127.0.0.1:5302/<name>.eth`            | the name, its **Owner**, Registered/Expires, its address      |
| 2 | `http://127.0.0.1:5302/<label>%E2%80%8B.eth`  | **the same Owner and dates — for a name that does not exist** |
| 3 | `http://127.0.0.1:5302/zzq9x7v4nope00000.eth` | the registration screen, **no Owner at all**                  |

5. Read row 3 as the control that makes rows 1 and 2 mean something: an unregistered name that imitates nothing does **not** inherit an owner. Only the zero-width twin does.

Observed with `increasableness.eth`:

```
  RENDERED TEXT (what a reader is actually shown):
   POSITIVE "arrow_drop_down search CONNECT login increasableness.eth key_vertical Owner bulletless.eth calendar_clock Registered SEPTEMBER 6, 2026 history Expires SEPTEMBER 6, 2027 About Addresses Main receiving address increasableness.eth 0xA0Cc...A6D9 favorite ios_share RENEW NAME double_arrow - TanStack Router"
   UNREG    "arrow_drop_down search CONNECT login arrow_back BACK Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026. zzq9x7v4nope00000.eth Starter Try it out 1 year $8.01/year $8.01 base price Committed Make it yours 3 years $5.51/year $16.52 base price Most popular Long-term identity Best yearly price 6 years $4.50/year $27.02 base price Register to date September 6, 2029 calendar_month Registering for 3 years expiring on September 6, 2029 TOTAL $ 1 6 . 5 2 USD Save $ 7 . 5 1 Stables accepted CONNECT TO REGISTER - TanStack Router"
   ARM      "arrow_drop_down search CONNECT login increasableness\u200b.eth key_vertical Owner bulletless.eth calendar_clock Registered SEPTEMBER 6, 2026 history Expires SEPTEMBER 6, 2027 About favorite ios_share RENEW NAME double_arrow - TanStack Router"
```

### Part 2 — the complete attack, with the impostor's records filled

Runs against a **fork of deployed Sepolia** — the real registrar, resolver implementation and payment token. Nothing is redeployed; the register calldata is built with the app's own `encodeRegisterCall`.

6. Fork deployed Sepolia:

   ```bash
   anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8545 --host 127.0.0.1
   ```
7. Register a clean throwaway label and its zero-width twin to two different owners, then set `addr(60)` on each:

   ```bash
   cd apps/manager && npx vitest run src/__poc_register.test.ts
   ```
8. Point the app at the fork and open both profiles:

   ```bash
   VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545    VITE_INDEXER_GRAPHQL_URL=https://staging-graphql.ens.dev/      npx vite dev --port 5303 --host 127.0.0.1
   ```

The test registers a clean throwaway label to **A** and its `U+200B` twin to **B**, sets `addr(60)`, and prints its own controls — **BEFORE** both labels AVAILABLE at 8000021, **AFTER** both TAKEN. (Use freshly generated accounts: anvil's default test accounts are EIP-7702 delegated on deployed Sepolia, so `register` reverts `ERC1155InvalidReceiver`.) Then open both profiles:

```
  BEFORE CONTROL — both labels must be AVAILABLE
    clean   : AVAILABLE base=8000021
    impostor: AVAILABLE base=8000021

  REGISTERING CLEAN -> A
    resolver deploy   : success -> 0x6C333bb86B40D436e77758C36256da8FC0c2B5C2
    commit            : success
    waited            : 60s (fork time advanced)
    approve           : success (price 8000021)
    REGISTER          : success  tx 0xc34a8c747ee5d999581a2643a48251006ea5c401ad214e6848da8993d9809d12

  REGISTERING IMPOSTOR -> B
    resolver deploy   : success -> 0x6015a9771d9402E61e6acc10E83B21f8D2Bd75A3
    commit            : success
    waited            : 60s (fork time advanced)
    approve           : success (price 8000021)
    REGISTER          : success  tx 0x5ecd9cb6e295b8fc8f17ad554b44d783be6d5fd189ff734bd1934a061e24934d

  AFTER CONTROL — both labels must now be TAKEN
    clean   : TAKEN
    impostor: TAKEN

  RENDERED TEXT (what a reader is actually shown):
   POSITIVE "arrow_drop_down search CONNECT login zzpoc7a787d.eth key_vertical Owner 0x1eF5...A9D3 history Expires SEPTEMBER 6, 2027 About Addresses Main receiving address zzpoc7a787d.eth 0x1eF5...A9D3 favorite ios_share RENEW NAME double_arrow - TanStack Router"
   UNREG    "arrow_drop_down search CONNECT login arrow_back BACK Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026. zzq9x7v4nope00000.eth Starter Try it out 1 year $8.01/year $8.01 base price Committed Make it yours 3 years $5.51/year $16.52 base price Most popular Long-term identity Best yearly price 6 years $4.50/year $27.02 base price Register to date September 6, 2029 calendar_month Registering for 3 years expiring on September 6, 2029 TOTAL $ 1 6 . 5 2 USD Save $ 7 . 5 1 Stables accepted CONNECT TO REGISTER - TanStack Router"
   ARM      "arrow_drop_down search CONNECT login zzpoc7a787d\u200b.eth key_vertical Owner 0x1eF5...A9D3 history Expires SEPTEMBER 6, 2027 About Addresses Main receiving address zzpoc7a787d\u200b.eth 0x00F9...5695 favorite ios_share RENEW NAME double_arrow - TanStack Router"
```

**The Owner line and the receiving address on the ARM row are two different parties.**

### Which query feeds the Owner field — decided, not argued

With an unregistered impostor this is undecidable: nobody owns it, so both candidate queries agree. Registering it **to a different address** decides it with no change to the application, using the same `getOwner` from `@ensdomains/ensjs/public/v2` that `profileOwner.ts` calls:

```
  CONTROLS, this run
    POSITIVE  getOwner(clean)        : 0x1eF512247511A16c64452c0F56E34886d2AEA9D3
    NEGATIVE  getOwner(unregistered) : 0x0000000000000000000000000000000000000000
    -> the oracle can name an owner and can decline to.

  THE ARM
    getOwner(impostor)               : 0x00F9d912DF6198d94581F6Bb51bdf40b4AD95695
    impostor node belongs to B       : true
    impostor node does NOT belong to A: true
```

The impostor node has an owner of its own on chain, and it is **B**. The page renders **A**. A field reading the impostor's own node would have shown B.

***

**The labels and transaction hashes above are generated fresh on every run.** The harness builds a throwaway nonce label (`zzpoc<nonce>`) and its zero-width twin, and registers them from freshly generated accounts, so a reader who re-runs it will get different labels, different resolver addresses and different transaction hashes. What must reproduce is the shape: both labels available at the same price beforehand, both taken afterwards, two different owners, two different nodes, and an Owner line that does not match the address line. This transcript was re-driven end to end on 2026-09-13 and every control held.
