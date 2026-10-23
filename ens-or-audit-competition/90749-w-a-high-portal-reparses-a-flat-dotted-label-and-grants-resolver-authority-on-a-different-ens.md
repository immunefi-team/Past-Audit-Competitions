# 90749 w a high portal reparses a flat dotted label and grants resolver authority on a different ens name

**Submitted on Aug 26th 2026 at 21:30:26 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90749
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Subdomain takeover without already-connected wallet interaction

## Description

### Brief/Intro

The ENS Portal resolver-role editor discards the typed identity of the indexed ENSv2 node it operates on. It keeps only the node's dotted display string, then asks ENSJS to rebuild DNS bytes from that string when it prepares an `authorizeNameRoles()` transaction. An attacker can use the public registrar to register one raw label `alpha.beta` (a single label whose data contains a dot) under `.eth`, deliver it to a victim, and point it at the victim's resolver. The indexer represents that flat object as `alpha.beta.eth`, the same spelling as the victim's genuine hierarchical name `alpha.beta.eth` (label `alpha` under `beta.eth`). Portal shows the victim that one scalar, but ENSJS deterministically reparses it as the three DNS labels `alpha | beta | eth`. A victim who performs one ordinary Add User flow — entering the attacker's address, selecting the displayed name, checking only Set Address Manager, and confirming once — signs an authorization for the different genuine hierarchical name. The attacker, who previously had no authority on either object, receives `ROLE_SET_ADDR` on the victim's genuine name from Portal's own transaction and immediately rewrites that name's address record. This is an application-created wrong-resource authorization, not a display-only homograph issue.

### Vulnerability Details

The required invariant is: the exact object shown and selected in a transaction-building application must be the object encoded into the wallet request. A display scalar is not a reversible serialization when two byte-distinct registry/DNS objects produce the same text. Portal violates that invariant between selection and calldata construction.

#### Two different ENS objects share one Portal scalar

|                                           | S (attacker-created decoy)               | T (victim's genuine name)            |
| ----------------------------------------- | ---------------------------------------- | ------------------------------------ |
| Structure                                 | one flat label `alpha.beta` under `.eth` | label `alpha` under `beta.eth`       |
| DNS wire bytes                            | `0x0a616c7068612e626574610365746800`     | `0x05616c70686104626574610365746800` |
| Registry token / namehash / role resource | distinct                                 | distinct                             |
| Displayed in Portal as                    | `alpha.beta.eth`                         | `alpha.beta.eth`                     |

The internal dot is data inside S's single raw registry label; it is a DNS label boundary in T. The indexer referenced by the target's e2e configuration constructs `label + "." + parent` when indexing a registration, and its `names` table upserts by `(chain, namehash)`, so the flat object's row is what the resolver overview exposes under the spelling `alpha.beta.eth`. The indexer is not claimed as the vulnerable asset; it only supplies attacker-created public-chain state. The security failure is that the in-scope transaction builder discards the node's typed identity and reconstructs a different authorization object.

#### Demonstrated attack path

Attacker preparation (public operations over no one's assets, paid normally):

{% stepper %}
{% step %}

## The victim normally owns the genuine hierarchical `alpha.beta.eth` and administers its active PermissionedResolver R.

{% endstep %}

{% step %}

## The attacker uses the public `ETHRegistrar.commit()` / `register()` flow to register the raw label `alpha.beta` under `.eth`, assigns it to the victim, and sets R as its resolver.

{% endstep %}

{% step %}

## The indexer row for the resolver overview now exposes this flat object under the spelling `alpha.beta.eth`.

{% endstep %}

{% step %}

## The attacker asks the victim to grant Set Address Manager on the newly delivered object so the attacker can manage it.

{% endstep %}
{% endstepper %}

The victim then performs one normal product flow:

{% stepper %}
{% step %}

## In the Portal Roles page, the connected victim clicks Add User, enters the attacker's address, selects the displayed `alpha.beta.eth`, selects only Set Address Manager, and confirms once.

The confirmation reads `Grant resolver roles for alpha.beta.eth`.
{% endstep %}

{% step %}

## Portal submits `authorizeNameRoles(T_dns, ROLE_SET_ADDR, attacker, true)` — the hierarchical DNS packet, not the flat packet of the displayed object.

{% endstep %}

{% step %}

## The transaction succeeds.

The attacker immediately calls `setAddr` for T, and T now resolves to the attacker-controlled address.
{% endstep %}
{% endstepper %}

There is no failed transaction, Retry, race, stale-indexer assumption, malicious resolver, compromised key, privileged initial role, or post-failure victim action. The attacker's registration is preparation; the victim-side exploit is a single documented Add User flow with one successful wallet approval.

#### Root cause

1. `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:33-38` models each resolver node with only `id`, `name`, `owner`, and `resolver`.
2. `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:75-85` requests none of the available `labelName`, `labelhash`, `tokenId`, `canonicalId`, `normalizedName`, or `isNormalized` fields, and obtains no registry address or DNS bytes.
3. `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:127` stores `selectedNode` as a string; `:186` projects every node to `n.name` before selection; `:205` snapshots only `{ name, account, roles }` for the transaction; `:321` presents that same scalar in the confirmation text; `:327-334` sends the scalar into the grant transaction builder. No registry address, token identity, label vector, resource, or canonical DNS packet remains with which the application could encode S.
4. `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-60` passes any non-empty selected string to ENSJS with `scope: 'name'`.
5. The pinned ENSJS builder uses `packetToBytes(name)` (`apps/portal/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantResolverRoles.ts:96-98`) and builds `authorizeNameRoles` at `:124-135`.
6. Viem's `packetToBytes` splits every internal dot (`apps/portal/node_modules/viem/utils/ens/packetToBytes.ts:38`), deterministically turning the scalar `"alpha.beta.eth"` into `0x05 alpha | 0x04 beta | 0x03 eth | 0x00`. It cannot recover the selected flat-label packet `0x0a alpha.beta | 0x03 eth | 0x00`.
7. The EOA transport preserves the prepared `to` and `data` unchanged into the wallet request (`packages/transaction-manager/src/actors/eoa-transport.actor.ts:52-60`).

The resulting dataflow is:

```
selected node object (flat label "alpha.beta" under .eth)
  -> retained as display string "alpha.beta.eth"
  -> packetToBytes("alpha.beta.eth") = alpha | beta | eth
  -> authorizeNameRoles(0x05616c70686104626574610365746800, ROLE_SET_ADDR, attacker, true)
  -> grant lands on the genuine hierarchical name T
```

#### Runtime proof

The end-to-end PoC ran the real frozen Portal UI against real contracts-v2 deployments on a local Anvil chain (Sepolia chain ID, no Sepolia RPC writes). The target-provided mock-wallet connector only supplied the connected victim EOA; no transaction logic was patched. The PoC intercepts only the out-of-scope GraphQL indexer response, which it supplies from the reviewed indexer image's shipped schema and upsert SQL; the Portal UI, transaction helper, pinned ENSJS/Viem encoder, transaction manager, wallet RPC, mined receipt, on-chain role state, and attacker write are all real.

The concrete witness was:

```
displayed/selected object:  alpha.beta.eth (flat label S)
wallet requests sent:       1 (eth_sendTransaction)
captured calldata selector: 0xbbd9abb5 authorizeNameRoles(bytes,uint256,address,bool)
calldata DNS packet:        0x05616c70686104626574610365746800  (T: alpha | beta | eth)
calldata roles:             1 (ROLE_SET_ADDR)
calldata grantee:           0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (attacker)
confirmation text:          Grant resolver roles for alpha.beta.eth
local transaction:          0xdbcddc6037c50abb7c1b5d949452b27a231daf6e7f482ee69b09ccf8a30bb2d6
receipt:                    success (0x1)
Portal page errors:         0
```

The captured browser request byte-equals the mined transaction input. It contains the hierarchical DNS packet and does not contain the selected flat-label packet.

The terminal state was:

| State                                        | Before victim approval | After successful receipt |
| -------------------------------------------- | ---------------------: | -----------------------: |
| Attacker `ROLE_SET_ADDR` on T (genuine name) |                  false |                     true |
| Attacker `ROLE_SET_ADDR` on S (flat decoy)   |                  false |                    false |
| Attacker `setAddr(T)`                        |                reverts |                 succeeds |
| `addr(T)`                                    |                      — |         attacker address |
| `addr(S)`                                    |                      — |                     zero |

#### Controls

The same run proves two controls:

1. After the victim revokes the T role, the attacker's `setAddr(T)` reverts again — the resolver was not permissive by default.
2. A grant built with the correct flat DNS packet sets `ROLE_SET_ADDR` on S and leaves T untouched, and the attacker can then write S. This proves the incorrect DNS boundary, not the role mechanics, caused the T-only capability.

#### Why the GraphQL boundary does not mock the vulnerability

The only intercepted layer is the out-of-scope indexer response, which reproduces the reviewed indexer image's documented behavior (shipped schema, `(chain, namehash)` upsert, `label + "." + parent` construction). The in-scope root is independent of indexer behavior: Portal retains only a display string and rebuilds DNS bytes from it, so even a perfect indexer returning two distinct rows with the same spelling could not be distinguished by this code path. Every layer after the response — Portal UI, transaction builder, encoder, transaction manager, wallet RPC, chain state, and attacker write — executes for real.

#### Recommended remediation

{% stepper %}
{% step %}

## Extend resolver node queries and types to carry stable typed identity.

Include registry address, token/canonical ID, raw label or label vector, namehash, resolver, and canonical DNS wire bytes.
{% endstep %}

{% step %}

## Store the selected node object or an opaque stable ID, not `node.name`.

{% endstep %}

{% step %}

## Build authorization calldata from the selected object's canonical DNS bytes or resource, never by reparsing display text.

{% endstep %}

{% step %}

## Before opening the confirmation and again before submission, verify that the typed node still belongs to the selected resolver and that its display text round-trips to the same DNS packet.

{% endstep %}

{% step %}

## Reject ambiguous or non-round-trippable rows, and show enough registry/label identity for the user to distinguish same-spelling objects.

{% endstep %}

{% step %}

## Add a regression containing both flat label `alpha.beta` and hierarchical `alpha.beta.eth`.

Selecting either must never authorize the other.
{% endstep %}
{% endstepper %}

### Impact Details

Recommended selected impacts: **Malicious interactions with an already-connected wallet (modifying transaction arguments or parameters / submitting malicious transactions) — Critical**, with **Subdomain takeover without already-connected wallet interaction — High** as the demonstrated outcome mapping.

Portal's own transaction changes the authorization target before the wallet submission: the UI displays and the user selects object S, while the encoded calldata authorizes object T. The application affirmatively modifies the decisive transaction argument (the DNS name packet that determines the authorization resource) and submits it through the connected wallet. This is not the user misreading a screen: the string the user sees is the string the application chose to encode, and the encoding is what changes the object.

The application also creates authority the attacker did not possess before the victim's action. The pre-state proves the attacker held no role and could not write either resource; Portal's single transaction sets `ROLE_SET_ADDR` on T for the attacker, and the attacker immediately redirects the victim's genuine name to an attacker-controlled address. The takeover of T's resolution requires no further wallet interaction from the victim, matching the High outcome row; the wrong-argument wallet submission matches the Critical malicious-interaction row.

The owner can revoke the attacker's role through the same Roles UI after discovery, so this report does not claim permanent irreversibility. The created capability is real and exclusive to the attacker until discovery, the resolution redirect is immediate, and nothing in the product warns that the grant landed on a different object than the one displayed. No fund theft, NFT transfer, or incorrect-resolution purchase is needed for the impact claim.

#### Why this is not public known issue EXP-INPUT-005

The public disclosure of `EXP-INPUT-005` covers name validators accepting inputs that normalization later rejects or transforms — homoglyphs, bidi/control characters, and a single leading or trailing dot — and explicitly limits the practical impact to homograph-style UI confusion "rather than incorrect resolution." This finding differs in ingress, root cause, sink, and terminal impact:

| Dimension                 | Public EXP-INPUT-005                                             | This finding                                                                                                                       |
| ------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Ingress                   | a name-input validator accepts deceptive text                    | an already-registered on-chain object is delivered to the victim and opened in the resolver Roles page; no input validator is used |
| Deceptive character class | homoglyphs, bidi/control characters, one leading or trailing dot | one interior dot as data inside a raw on-chain label, which normalization-aware parsing treats as a DNS boundary                   |
| Root                      | validation and normalization disagree                            | the application drops typed node identity and reparses a display string into a different DNS object                                |
| Sink                      | UI display confusion                                             | a successful `authorizeNameRoles` grant on a different name's authorization resource                                               |
| Terminal impact           | publicly described as display-only                               | attacker gains `ROLE_SET_ADDR` on the victim's genuine name and redirects its resolution                                           |
| Sufficient narrow fix     | align/reject deceptive validator inputs                          | carry typed object identity from query to calldata; never reparse display text                                                     |

I cannot inspect the private `EXP-INPUT-005` ticket or proposed patch, so this comparison is limited to the program's public description. Private reports and private ticket breadth are likewise unavailable, so I cannot guarantee there is no private duplicate.

### References

Scope, source, and public-fix check:

* In-scope asset: `apps/portal` (Explorer app / Websites & Apps).
* Frozen competition source executed by the PoC: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
* Public `audit-comp-ready` head checked on 2026-08-27: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
* The public delta after the frozen revision contains only Hono/package-manifest and lockfile changes; no public typed-identity or grant-construction fix exists.
* The four decisive Portal files executed by the PoC (`useResolverOverview.ts`, `ResolverAddUserSheet.tsx`, `grantResolverRoles.ts`, and the ENSJS grant builder) match frozen competition source.

Links:

* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* In-scope Portal: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Resolver overview query/model: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/hooks/useResolverOverview.ts>
* Add User sheet (selection, confirmation, dispatch): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx>
* Grant helper: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts>
* Transaction manager (in-scope package): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>

## Link to Proof of Concept

<https://gist.github.com/Jornason/c11ad170da791309fff3f0e16474038d>

## Proof of Concept

### What this PoC proves

The linked Gist bundle drives the real frozen Portal UI through one ordinary Add User flow against real contracts-v2 deployments on a local Anvil chain. It proves that a victim who selects the displayed object `alpha.beta.eth` and only the Set Address Manager permission produces exactly one Portal wallet request, that the request authorizes the different genuine hierarchical name, that the transaction succeeds, and that the attacker immediately gains and uses `ROLE_SET_ADDR` on the victim's genuine name.

All writes are local. No Sepolia transaction is broadcast, no production user or site is contacted, and no real user key is used. The chain uses Sepolia's chain ID only so the Portal's chain guard accepts it.

### Environment and reproduction

Prerequisites: macOS or Linux; Foundry (`anvil`, `forge`, `cast`); Node 22 with pnpm 10; Python 3 with Playwright and Chromium; a contracts-v2 checkout (the fixture deploys its real registry/registrar/resolver sources); and the frozen competition source checkout containing `apps/portal` with dependencies installed.

From the Gist bundle root:

```bash
# 1. Point the fixture project at the contracts/ dir of a contracts-v2 checkout
CONTRACTS_V2_ROOT=/path/to/contracts-v2/contracts ./setup-foundry-toml.sh

# 2. Run the whole pipeline (dynamic localhost ports; writes artifacts/)
ENS_FROZEN_SOURCE_ROOT=/path/to/frozen-competition-source \
python3 run_continuous_portal_anvil.py
```

`ENS_FROZEN_SOURCE_ROOT` must contain `apps/portal`. Tool paths can be overridden with `ANVIL_BIN`, `FORGE_BIN`, `CAST_BIN`, and `NODE_BIN_DIR`; otherwise the runner uses `PATH`.

### Expected oracles

The runner exits non-zero on any failed assertion. A passing run prints `status: PASS` and writes `artifacts/continuous-portal-anvil/run-*/continuous-result.json` with:

```
before.nestedRole = false            (attacker has no ROLE_SET_ADDR on T)
before.flatRole = false              (attacker has no ROLE_SET_ADDR on S)
before.attackerSetAddrDenied = true
wallet eth_sendTransaction count = 1
Portal page errors = 0
receipt status = 0x1
afterPortal.nestedRole = true        (Portal's own transaction created the T grant)
afterPortal.flatRole = false
afterPortal.nestedAddr = attacker    (T now resolves to the attacker)
afterPortal.flatAddr = zero
controls.setAddrDeniedAfterRevoke = true
controls.nestedRoleAfterFlatGrant = false
controls.flatRoleAfterFlatGrant = true
```

### Captured exploit steps and retained witness

The retained witness of the executed reference run is `continuous-result.json` in the bundle, with screenshots `01-resolver-roles-page.png` through `05-transaction-complete.png`:

{% stepper %}
{% step %}

## The fixture deploys real contracts-v2 registries, registrar, price oracle, payment token, and resolver; creates the genuine hierarchical name T for the victim; and the attacker publicly commits and registers the flat label `alpha.beta` to the victim.

See `fixture.json`, `flat-registration-receipt.json`.
{% endstep %}

{% step %}

## The attacker is proven to start with no role on either object and a reverting `setAddr(T)`.

{% endstep %}

{% step %}

## The frozen Portal starts with its target-provided mock wallet connected as the victim.

The runner serves the reviewed indexer row at the external GraphQL boundary and drives the real accessible UI controls: Add User, attacker address, displayed `alpha.beta.eth`, only Set Address Manager. See `02-add-user-selection.png` (attached to this report).
{% endstep %}

{% step %}

## The confirmation reads `Grant resolver roles for alpha.beta.eth`.

See `04-transaction-confirmation.png` (attached).
{% endstep %}

{% step %}

## The victim confirms once.

Exactly one `eth_sendTransaction` is captured, decoded as `authorizeNameRoles(0x05616c70686104626574610365746800, ROLE_SET_ADDR, attacker, true)`. The captured request byte-equals the transaction mined in a local block, receipt `0x1`. Portal reports success for the displayed object. See `05-transaction-complete.png` (attached).
{% endstep %}

{% step %}

## The runner asserts the T-only role delta, executes the attacker's immediate `setAddr(T)` to the attacker address, then runs the revocation control and the correct-flat-DNS control.

The revocation control verifies the write is denied again. The correct-flat-DNS control verifies a grant on S only, with T untouched.
{% endstep %}
{% endstepper %}

The reference local transaction is `0xdbcddc6037c50abb7c1b5d949452b27a231daf6e7f482ee69b09ccf8a30bb2d6`; it is an Anvil transaction, not a Sepolia transaction. Dynamic values such as ports, timestamps, the fixture secret, and transaction hashes change between runs; the assertions above are the stable oracles.

### Attached code and integrity

The complete bundle — runner, Foundry deploy script, `foundry.toml.template` with setup helper, retained witness JSON, five screenshots, the indexer collision reproduction SQL, and `SHA256SUMS.txt` covering every file — is in the linked secret Gist: <https://gist.github.com/Jornason/c11ad170da791309fff3f0e16474038d> — the manifest verifies in the Gist root as-is. Publication sanitization replaced local absolute paths and internal run tags in file names and three JSON string fields (`poc` identity, artifact directory, one dev-server stack trace prefix); no evidence-bearing value (addresses, calldata, DNS packets, role state, transaction hash, receipt) was altered.
