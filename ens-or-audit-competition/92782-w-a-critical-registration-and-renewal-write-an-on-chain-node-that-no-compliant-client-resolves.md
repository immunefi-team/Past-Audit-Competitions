# 92782 w a critical registration and renewal write an on chain node that no compliant client resolves

**Submitted on Sep 12th 2026 at 08:33:07 UTC by @andreeaaa for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92782
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## 1. Summary

`apps/manager` validates registration input with a hand-written character blacklist instead of ENS normalization (ENSIP-15 / UTS-46). The label reaches `register()` unmodified, so the `namehash` written on chain differs from the `namehash` every compliant ENS client computes for the same visible string.

The manager already ships ENSIP-15 and uses it elsewhere. `apps/manager/src/features/profile/service/profileName.ts:1` imports `normalize` from `viem/ens` (viem vendors `@adraffy/ens-normalize`) and calls it at `:18`. It is invoked on the profile route and not on the registration route.

The same monorepo also ships a correct registration validator: `apps/portal/src/utils/token/nameValidation.ts:11` → `getLabelRegistrationError` (`apps/portal/src/utils/token/isNormalized.ts:69`), built on `ens_normalize`. The two validators disagree on **20 of 35** tested inputs.

Two defect classes, neither requiring an attacker:

* **Permanent.** A user types an interior underscore (`ali_ce`), a double hyphen (`ab--cd`) or Arabic tatweel (`aliـce`). Every guard passes, the registrar sells the name, and `ens_normalize` **throws** — no canonical form exists, so no rescue registration exists either.
* **Recoverable.** **207 of the 3,944 fully-qualified sequences in Unicode's RGI emoji set have the form `<base> U+FE0F`** — the form a keyboard emits. ENSIP-15 strips the `U+FE0F`, so the user receives a different name from the one displayed and priced.

***

## 2. Invariant broken

For any name accepted by a registration or renewal flow, the label written on chain and the name ENS normalization resolves must denote the same name.

***

## 3. Locations

### 3.1 The defective guard

`apps/manager/src/features/register-v2/utils/name-parser.ts`:

* line **11**: `const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/\`
* line **33**: `export const parseName = (`
* line **46**: `const normalized = name.trim().toLowerCase()`
* line **73**: `if ([...labels, label, tld].some((part) => INVALID_LABEL_CHARS.test(part))) {`

`parseName` performs trim, lowercase, whitespace check, `..` check, TLD/sub-label split, and the blacklist at line 73. There is no normalization step. The ≥3 length rule lives elsewhere, at `apps/manager/src/routes/register/$name.tsx:40` and `nameUtils.ts:124`.

### 3.2 Renewal inherits it

`apps/manager/src/features/renew/utils/renewableName.ts:25-26`:

```ts
export const parseRenewableName = (name: string) =>
  parseName(name).andThen((parsedName) => {
```

### 3.3 Route loader to calldata

| step          | location                                                                                                                                                                | effect on the label                    |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| route loader  | `apps/manager/src/routes/register/$name.tsx:24` — `const parsedName = parseName(name)`                                                                                  | blacklist only                         |
| loader output | `apps/manager/src/routes/register/$name.tsx:57` — `label: parsedName.value.label`                                                                                       | passes through                         |
| XState actors | `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:322, 793, 922, 973` — `const label = cleanLabel(input.name)`                         | —                                      |
| `cleanLabel`  | `registration.hca.actors.ts:498` — `const cleanLabel = (name: string): string => name.replace(/\.eth$/, '')`                                                            | strips `.eth`                          |
| batch build   | `packages/smart-account/src/providers/rhinestone/registration-calls.ts:211` `buildRevealBatch`, `:214` `const node = namehash(name)`, `:286` `functionName: 'register'` | raw label → `namehash`, → `register()` |

viem's `namehash` does not normalize; it expects an already-normalized name.

### 3.4 The correct validator in the same repository

`apps/portal/src/utils/token/isNormalized.ts:16-21`:

```ts
const isNormalizedLabel = (label: string) => {
  try {
    return ens_normalize(label) === label
  } catch {
    return false
  }
}
```

Its comment at `isNormalized.ts:63` describes `[labelhash]`-form labels as *"a known ENS spoofing"* vector.

### 3.5 Why the session policy does not intercept it

`registration-calls.ts:238-247` documents that `HCAOwnerAndSessionValidator._checkResolverDeployment` reconstructs and `keccak256`-compares the calldata of step 1 only, `deployProxy(PERMITTED_RESOLVER_IMPL, salt, initialize(account, ALL_ROLES, []))`. That salt is `computeResolverSalt(params.hca)` (`manifest.ts:223`), whose parameters are `{id, owner, version}`. The label is not an input to it, so the one content hash in that policy cannot depend on the label.

The commit leg and the reveal leg derive the label from the same `cleanLabel(input.name)` (`registration.hca.actors.ts:793` for commit, `:322`/`:922`/`:973` for the reveal path). `:800` records that *"the commitment binds label/wallet/secret/resolver/duration — the reveal must reuse ALL of them"*. Both legs carry the identical raw label, so the commit/reveal binding has nothing to reject: a mismatch is the only condition it detects.

***

## 4. Evidence of absence

```
$ grep -rn "ens-normalize\|ensNormalize\|ENSIP-15" apps/manager/src/features/register-v2 --include="*.ts"
(no output, exit 1)

$ grep -rn "normalize(" apps/manager/src/features/register-v2 --include="*.ts" --include="*.tsx"
(no output, exit 1)

$ grep -rn "ens-normalize\|ensNormalize" packages/transaction-manager/src --include="*.ts" | grep -v test
(no output, exit 1)

$ grep -rn "normalize(" packages/transaction-manager/src --include="*.ts" | grep -v test
(no output, exit 1)

$ grep -c "adraffy" packages/transaction-manager/package.json
0
```

ENSIP-15 is used in the manager only on the profile route (`profileName.ts:18,48`, `setPrimaryName.ts:49`).

***

## Proof of Concept

## 5. Proof of concept

Every harness drives the real guards in the app's order — `normalizeDomainNameFromUrl` → `validateENSName` → `normalizeQuery`/`toLabel` → `parseName` → `parseRenewableName` → `buildRevealBatch` — then ABI-decodes `register(...)`. All assertions are `expect()` calls. Place the files in `apps/manager/src/` and run from `apps/manager`.

On-chain figures are quoted in the payment token the app itself resolves via `getDestinationContracts(sepolia.id)`: registrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, token `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`.

### 5.1 Permanent class

**Files:** `poc-ens-permanent.test.ts` (`Tests 5 passed (5)`), `poc-ens-permanent-onchain.test.ts` (`Tests 1 passed (1)`).

Central assertion — ENSIP-15 produces no output at all for the label the manager just built:

```ts
expect(threw, 'ENSIP-15 DEBE rechazar: no existe canonico').toBe(true)
```

```
caso guion-bajo
  el usuario teclea  : ali_ce
  validateENSName    : PASA
  parseName          : PASA
  parseRenewableName : PASA
  register() label   : ali_ce
  nodo ESCRITO       : 0x09e813561baa1d9e88b69dff35ef09fb5181694ae248f10e370dd62afe1c95bb
  ENSIP-15           : LANZA -> Invalid label "ali_ce": underscore allowed only at start
  nombre canonico    : NO EXISTE -> irrecuperable

caso doble-guion
  ENSIP-15           : LANZA -> Invalid label "ab--cd": invalid label extension: "ab--"

caso tatweel
  ENSIP-15           : LANZA -> disallowed character U+0640
```

Live registrar:

```
  ali_ce         disponible=true   precio=8.00   canonico=NO EXISTE
  ab--cd         disponible=true   precio=8.00   canonico=NO EXISTE
  ali<640>ce     disponible=true   precio=8.00   canonico=NO EXISTE
```

Control: `alice` passes the same harness and `normalize('alice.eth') === 'alice.eth'`.

### 5.2 The registrar does not validate normalization on the write path

`isAvailable` and `getRegisterPrice` are `view` functions. Two further read-only `eth_call`s establish that `register()` carries no label validation either:

```
makeCommitment (view) accepts every label:
  ali_ce          -> 0xd95481db5a54637914e2fcb40e51bba002f193f4dc4e0d62b736c02a7851c294
  ab--cd          -> 0x48317c28a76417afe40896f642d67ed3a26e85e6ac36be373d26a0bb98e14eed
  tatweel         -> 0x74e24ff803a4c73e5aa5fcbcd2a59a0d129d7e278d1715c6ae2c10ece8730530
  amor❤️          -> 0x93aa2212ccf38d1fd6a93490fc4d342a8f0e93e980b67c549dbd9f5df10a95e2
  trianguloverde  -> 0x8b2da7551ac40c4a7e3920975457394c19fcc3680ba4bdd03fd51ab5dcb9b922   <- canonical CONTROL

register() eth_call, raw revert data:
  ali_ce          -> 0x0cb9df3f d95481db5a54... 0000...015180
  ab--cd          -> 0x0cb9df3f 48317c28a764... 0000...015180
  tatweel         -> 0x0cb9df3f 74e24ff803a4... 0000...015180
  amor+FE0F       -> 0x0cb9df3f 93aa2212ccf3... 0000...015180
  trianguloverde  -> 0x0cb9df3f 8b2da7551ac4... 0000...015180   <- canonical CONTROL
```

Every call reverts with the same selector `0x0cb9df3f`, whose first argument is that label's own `makeCommitment` hash and whose second word is `0x15180` = 86 400 s: the commit/reveal age precondition. A registrar that rejected non-normalizable labels would revert the three permanent cases with a different selector from the canonical control. It does not.

### 5.3 The consequence class is already populated on mainnet

ENS Registry `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`, read-only `eth_call`:

```
=== canonical, registered ===
  vitalik.eth           owner=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045   normalize -> vitalik.eth
  ens.eth               owner=0xb6E040C9ECAaE172a89bD561c5F73e1C48d28cd9   normalize -> ens.eth
       => a compliant client normalizes and queries successfully

=== permanent class, registered ===
  vitalik_buterin.eth   owner=0x9e076aCE1Fe609535a0396a0b45BFA8F948E14b7
                        normalize THROWS -> Invalid label "vitalik_buterin": underscore allowed only at start
       => the name exists and has an owner; a compliant client throws at the
          normalization step and never emits the lookup
```

A registered canonical name resolves. A registered interior-underscore name cannot be queried at all. Querying the indexer ENS itself serves returns **21** such interior-underscore registrations with live owners and active expiries.

### 5.4 Recoverable class

**File:** `poc-ens-fe0f.test.ts` — `Tests 13 passed (13)`: 8 defect cases, 4 counterfactual controls.

```ts
expect(r.searchError).toBeNull()                    // search guard passes
expect(r.renewAccepted).toBe(true)                  // renewal guard passes
expect(r.onChainLabel).toBe(typed)                  // raw label reaches register()
expect(r.priceQueryLabel).toBe(typed)               // pricing uses the raw string
expect(canonical).not.toBe(`${typed}.eth`)          // ENSIP-15 maps it elsewhere
expect(writtenNode).not.toBe(resolvedNode)          // the finding
```

```
caso amor
  Unicode           : 2764 FE0F ; fully-qualified (2764 ; unqualified)
  el usuario teclea : amor<2764><FE0F>
  validateENSName   : PASA
  parseName         : PASA
  parseRenewableName: PASA
  register() label  : amor<2764><FE0F>
  nodo ESCRITO      : 0x825dba559aa8efbb3963a2e520f37a8c4c25c1ec38febabc236bb2a18d62dbc6
  ENSIP-15 resuelve : amor<2764>.eth
  nodo RESUELTO     : 0x3477ca1615a2522e06f6d6fd19268c138e8cd382a7bead97e954237ab8be0b0f
```

Controls `alice`, `ali-ce`, `cohete🚀` (U+1F680, `Emoji_Presentation=Yes`) and `café` (NFC) each produce `namehash(written) === namehash(resolved)`.

### 5.5 Which emoji are affected

From `emoji-test.txt` **Version 17.0, 2025-08-04** (`https://unicode.org/Public/emoji/17.0/emoji-test.txt`, attached as `unicode-emoji-test.txt`):

| emoji                                                                                  | fully-qualified form | bare form     |
| -------------------------------------------------------------------------------------- | -------------------- | ------------- |
| ❤️ U+2764, ☀️ U+2600, ✈️ U+2708, ❄️ U+2744, ☮️ U+262E, ✂️ U+2702, ⚠️ U+26A0, ✔️ U+2714 | `<base> FE0F`        | `unqualified` |
| 🚀 U+1F680, ⚡ U+26A1, ☕ U+2615                                                         | `<base>` bare        | —             |

The file's own header states it *"provides data for testing which emoji forms should be in keyboards"*. **207 of 3,944** fully-qualified RGI sequences are `<base> U+FE0F`: for those, the keyboard's output is the FE0F form and the bare form is `unqualified`. The corpus uses only emoji from that set; `Emoji_Presentation=Yes` emoji are used as controls.

### 5.6 Differential test, manager vs portal

`poc-r6-manager.test.ts` + `poc-r6-portal.test.ts`, cross-referenced by `r6_diff2.py`:

* **20 of 35** corpus inputs: the manager accepts what the portal rejects and emits a non-canonical label. Uppercase inputs are excluded from the count — `parseName` lowercases at line 46, so its output matches the canonical form.
* Of the 28 inputs the manager accepts: 8 produce a label ENSIP-15 maps to a different name; **11** produce a label ENSIP-15 cannot normalize at all.

### 5.7 Pricing tier

For `sol☀️` and `paz☮️` the registrar counts `U+FE0F` as an extra character, so the FE0F variant is quoted in the cheaper length tier (8.00) while the canonical 4-character name is quoted at 160.00. The quote shown belongs to a longer name than the one the user is buying.

***

## 6. Impact

### 6.1 Permanent class

A user types an underscore, a double hyphen or Arabic tatweel. `apps/manager` accepts it through every guard, the raw label reaches `register()`, and the registrar sells it. `ens_normalize` throws on that label, so:

* a client following ENSIP-15 (ethers v6 normalizes internally; viem requires it by documentation) throws at the normalization step and never emits the lookup — the name cannot be queried, let alone resolved;
* no canonical name exists to register as a rescue.

Permanence here is a property of ENSIP-15, not of any particular transaction: where no canonical form exists, no rescue registration exists. §5.3 shows the resulting state already on mainnet — an owned name that no compliant client can query.

The permanent class is **11 labels** in the tested corpus (§5.6). Tatweel is routine Arabic typesetting; an underscore and a double hyphen are keys on every keyboard.

### 6.2 Recoverable class

The user registers a node that differs from the one any compliant client computes. The name cannot be resolved by ENSIP-15-compliant clients, is rejected as invalid by `apps/portal` in the same repository, and resolves — for anyone typing the visually identical string — to a different node a third party may own.

### 6.3 Both classes

The renewal flow shares the defect through `parseRenewableName`. The quoted price can belong to a different length tier than the canonical name (§5.7). Neither class requires an attacker: the input is what a standard keyboard emits.

***

## 7. Duplicate analysis

**`EXP-INPUT-005`** (known issue, Medium) — published text: validators *"accept homoglyphs, bidi and control characters"*, impact bounded to *"homograph-style display confusion **in our UI rather than incorrect resolution**"*.

The permanent class sits outside that entry on two independent axes.

**Axis 1 — character class.** EXP-INPUT-005 enumerates homoglyphs, bidi and control characters. An interior **underscore** (`ali_ce`) is none of the three. An interior **double hyphen** (`ab--cd`) is none of the three. These labels are not covered by the known issue's own enumeration.

**Axis 2 — consequence.** EXP-INPUT-005 bounds itself to display confusion *"rather than incorrect resolution"*. This is incorrect resolution: the written node and the resolvable node are different values, persisted on chain outside the app, and §5.3 demonstrates the resulting unresolvable state on mainnet.

Searched: the program's published "Public Disclosure of Known Issues" document. The audited tree contains no `audits/` directory and a single squashed commit, so neither offers a corpus to search.

***

## 8. Remediation

1. Call `normalize` from `viem/ens` — already a manager dependency, already used at `profileName.ts:18` — inside `parseName` (`name-parser.ts:33`). Alternatively reuse `validateRegistrableEthName` from `apps/portal` so the monorepo has one registration validator.
2. Normalize in `normalizeQuery` (`nameUtils.ts:31`) and `normalizeDomainNameFromUrl` (`utils/domain.ts:27`) so availability and pricing query the name that will be registered.
3. Where normalization changes the string, show the canonical name before the user commits; where it throws, surface the error.
4. Apply the same to renewal, which inherits the defect through `parseRenewableName`.

### 8.1 Fix implemented and verified

Patch attached as `fix-name-parser-ensip15.patch`: import `normalize` from `viem/ens` and canonicalize per label, leaving TLD handling untouched; a label that cannot be normalized returns the existing `INVALID_CHARACTER` error.

| check                                        | result                                                                                                       |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| PoC with the fix applied                     | `Tests 9 failed \| 4 passed (13)` — the 8 defect cases fail as intended, the 4 canonical controls still pass |
| `npx vitest run`, whole `apps/manager` suite | **`Test Files 174 passed (174)`, `Tests 1533 passed \| 1 skipped`**                                          |

No regressions, including `name-parser.test.ts` and its `my-name🚀.xyz` case: U+1F680 is `Emoji_Presentation=Yes` and normalizes to itself, and canonicalizing per label leaves the TLD path untouched.

***

## 9. Reproduction

```
cp poc-ens-permanent.test.ts poc-ens-permanent-onchain.test.ts poc-ens-fe0f.test.ts apps/manager/src/
cp poc-r6-portal.test.ts apps/portal/src/
cd apps/manager && npx vitest run src/poc-ens-permanent.test.ts src/poc-ens-fe0f.test.ts
cd apps/manager && npx vitest run src/poc-ens-permanent-onchain.test.ts
cd apps/portal  && npx vitest run src/poc-r6-portal.test.ts
```
