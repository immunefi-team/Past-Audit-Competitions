# 92257 w a critical resolver role editor upgrades a name scoped grant to resolver root submitting a broader transaction than the operator selected

**Submitted on Sep 8th 2026 at 14:48:14 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92257
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Bug Description

**Brief.** Portal reconstructs the display name of an indexed resolver role by hashing the name and comparing the result against the resource emitted on chain. The reconstruction omits a mask, so it **never** matches for a name-scoped role. The row keeps the role but loses its name; the editor turns that failed *display* lookup into the empty string; and the transaction builder reads the empty string as a request for the resolver **root**. Adding a permission to a delegate who was authorised for one name therefore builds `grantRootRoles(role, delegate)` instead of `authorizeNameRoles(name, role, delegate, true)`. The operator's wallet is asked to approve a transaction whose authorisation domain is strictly broader than the one the interface offered, and the confirmation names no scope at all.

**Details (verified at HEAD).**

*The lookup key is computed without the part mask:*

* `apps/portal/src/lib/roles/resolverRoles.ts:138-141`:

  ```ts
  export const computeNameResource = (name: string): string =>
    keccak256(
      encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
    )
  ```

  Its own doc comment says it *"Mirrors `PermissionedResolverLib.resource(node, 0)` in Solidity"*. The Solidity resource reserves its **low four bytes** for the `part` field and zeroes them for a name-level role; this mirror keeps the full digest. For `resedit875836.eth`:

  ```
  Portal computes : 0x47349dbce904e72017a7f45c9cd6af6a8a989dd1c85ec094a33cd351fbcf5bb8
  chain / indexer : 0x47349dbce904e72017a7f45c9cd6af6a8a989dd1c85ec094a33cd35100000000
  ```

  The two agree on all 28 leading bytes. Zeroing the low four bytes of Portal's value reproduces the on-chain resource **exactly** — so the defect is precisely the missing mask, not a different preimage.

*The failed lookup silently drops the scope:*

* `resolverRoles.ts:204` — `if (resolvedName) existing.resolvedNames.add(resolvedName)`. No match, so nothing is added; the role itself is kept. The row is rendered with a blank Name cell.

*The editor then derives write authority from that missing display label:*

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`:

  ```ts
  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
  ```

  `resolvedNames` is empty, so `roleName` is `''`. It is stored as the transaction's `name` and carried into the write.

*And the empty string selects a different contract function:*

* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:47`:

  ```ts
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
  ```

  `scope: 'root'` and `scope: 'name'` are not two arguments to one call — they are **two different contract functions**, `grantRootRoles` and `authorizeNameRoles`, over two different authorisation domains.

*What the operator is shown before signing.* Captured verbatim from the confirmation modal during the run:

```
Transaction overview
Save resolver role changes
Not Started
Est. Cost
Estimating…
Start
Close
```

Neither the row nor the modal states a scope: the sidebar renders **neither** "Global roles (all names)" **nor** "Roles scoped to resedit875836.eth", and the modal names only a generic operation. Nothing between the tick and the signature discloses that root authority is about to be granted.

***

## Impact

**The unauthorised authority exists as soon as the transaction mines.** The operator, with a wallet already connected to Portal, ticks one permission on a row whose scope column is **blank**. Their belief that the row is name-scoped does not come from the interface, which states no scope anywhere: it comes from having created that role themselves, minutes earlier, through Portal's own *Add user* flow, against a name they picked from its combobox. Portal then builds a different transaction from the one the interface described — a different function, over a strictly broader authorisation domain — and puts it in front of the wallet under a label that names no scope. That is the "modifying transaction arguments or parameters / submitting malicious transactions" impact, and it needs no further act by anyone: measured immediately before the edit, `roles(0, delegate)` is `0`; once the edit mines, it is `1`.

The consequences that follow are measured, and are offered as corroboration rather than as the claim:

1. **Authority the operator never granted.** The delegate holds `RESOLVER_ROLE_SET_ADDR` over **every** name the resolver serves, not the one name they were delegated.
2. **A second name falls under their control.** In the PoC the delegate sets the ETH address of `regedit875836.eth`, a name never delegated to them and belonging to the same operator.
3. **Portal then resolves that name to them,** and a payer reading the address off Portal's own Address Resolution page sends 0.25 ETH, which arrives in the delegate's account.

The operator's interaction is the ordinary role-management workflow. No phishing, no malformed input, no browser defect, no malicious indexer, and no contract defect: the resolver enforces exactly the role Portal asked it to grant, and Panoptes returns exactly the resource the chain emitted.

***

## Risk Breakdown

**Which clause I rely on, and what I deliberately do not lean on.** I rely on **"Malicious interactions with an already-connected wallet — Modifying transaction arguments or parameters — Substituting contract addresses — Submitting malicious transactions."** Portal substitutes `grantRootRoles` for `authorizeNameRoles` and widens the authorisation domain, then presents it for signature without naming the scope. I do **not** rest the finding on the display clause (*"resolving or rendering an ENS name to an attacker-controlled address"*): once the over-broad grant is on chain, Portal's later rendering of the redirected name is faithful to chain state, and I would rather say so than claim otherwise. The funds measurement is included to show the consequence is real, not to carry the eligibility argument.

**The application creates the authority; it does not fail to remove one.** `roles(0, delegate)` is measured at `0` immediately before the operator touches the interface — asserted in the PoC, not assumed — and at `1` once the edit mines. The delegate does not *still hold* anything: root authority over every name this resolver serves did not exist until Portal composed the transaction that created it. That distinction matters, because it is the whole of the finding.

**The application is the instrument here, not a bystander.** The harmful state change is a transaction that Portal composed and labelled, and that was broadcast once approved. It exists on chain because Portal put it there. Nothing about this finding depends on a third party choosing to act afterwards, and nothing about it depends on a defect in the contracts or the indexer — both behave correctly throughout, and the fix touches neither.

**How this differs from my own earlier report on the registry role editor.** I flag the overlap rather than let triage find it. That report concerned a *missing orchestration edge*: a mixed grant+revoke plan in `RegistryEditUserSheet` never starts its revoke step, so a requested write is never broadcast. Here the resolver editor **starts and mines the only transaction it built** — the defect is that it chose the wrong authorisation resource and the broader selector. Different component, different mechanism, and the counterfactual holds both ways: repairing the registry editor's step graph does not change `computeNameResource`, and correcting the resource mask does not create the missing edge.

**Not a Known Issue.** Checked against *ENS Apps: Known Security Issues* (2026-08-14): all 25 Part 1 findings, all 10 QA items, both tracker lists. No entry concerns resource-identifier reconstruction, scope derivation from a display string, or selector choice in the resolver role editor.

* **R3-07** is about a **reused transaction id** causing a completed transaction to skip archiving, history and telemetry. This finding involves no reuse and no skipped bookkeeping: one transaction is built, signed, mined and recorded — it is simply the wrong one.
* **QA-01 / WEB-1257** concerns **transfers** and follow-up actions that error afterwards. There is no transfer here and nothing errors.
* **WEB-716** (no way to replace an existing subregistry) is a missing action, not a mis-scoped one.

**Part 2's own carve-out describes this finding.** The document states that a functional defect becomes a new finding when it has a demonstrated security consequence, giving as its example *"a role combination that grants authority it should not, rather than merely erroring."* Here the application grants authority it should not — root over every name — and nothing errors.

**Anticipated objection: the operator signed it.** They did — twice, and both times deliberately — and that is the substance of the finding rather than an answer to it. The second approval is the one that carried root scope, and nothing in its path said so: the row carried a blank scope column, the sidebar displayed neither the delegated name nor "Global roles (all names)", and the confirmation modal read only "Save resolver role changes". At no point between selecting a permission and approving the transaction does the interface disclose that the authorisation domain is the resolver root. An operator cannot withhold consent from a scope they are never shown.

**Honest bounds.** Measured on an isolated local fork of the real deployed Sepolia v2 contracts, in the unmodified Explorer UI, against the repository's own Panoptes API — no shim, stub or substituted endpoint. Not covered: whether the same mask omission affects other resource-derived lookups elsewhere in the app; I checked only the resolver roles path. No affected production source file was modified; the audited commit is `1c9b47f`.

***

## Recommendation

**Do not derive authorisation scope from a display string.** In order of importance:

1. **Correct the resource computation.** `computeNameResource` must mask the low four bytes it currently leaves as digest, so that it actually mirrors `PermissionedResolverLib.resource(node, part)`. A single unit test comparing it against a resource read back from a deployed resolver would have caught this.
2. **Fail closed on an unresolved resource.** If a non-root resource cannot be mapped to a name, disable mutation on that row and say so. Defaulting an unknown scope to *root* inverts the safe direction: the fallback should be the narrowest authority, never the broadest.
3. **Carry the authoritative resource through the edit plan.** Represent a row as `(account, resource, scope)` and pass the resource identifier itself into the transaction builder, so the write cannot be re-derived from anything rendered.
4. **Do not merge roles from different resources into one editable bitmap.** `groupRolesByAccount` currently collapses several resources into one row, which is what forces a single string to stand for a set.
5. **Name the scope in the confirmation.** "Save resolver role changes" should read as the scope being granted — one name, or every name — and a test should assert that the decoded selector and resource match the row that was edited.

***

## References

* `apps/portal/src/lib/roles/resolverRoles.ts:138-141` (`computeNameResource`, the missing part mask), `:204` (the unresolved name is silently dropped)
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169` (`resolvedNames.find(...) ?? ''`)
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:47` (`name === ''` selects `scope: 'root'`)
* `@ensdomains/ensjs` `wallet/v2` — `grantResolverRolesWriteParameters`, where `scope: 'root'` maps to `grantRootRoles` and `scope: 'name'` to `authorizeNameRoles`
* `@ensdomains/ensjs` resolver roles — `RESOLVER_ROLE_SET_ADDR = 1n << 0n` (`0x1`), `RESOLVER_ROLE_SET_TEXT = 1n << 4n` (`0x10`)

***

## Proof of Concept

## Proof of Concept

All testing is local (competition rule): an isolated local Anvil fork of the **real deployed Sepolia v2 contracts** (`:8546`, keyless public endpoint), the repository's **own Panoptes indexer and API** (`:5658`), and the unmodified Explorer served by its own dev server, with the project's `headless-web3-provider` fixture. **No indexer shim, stub or substituted response was used at any point.** Nothing was sent to Sepolia; no production system, third-party account or wallet was touched.

```
operator (Portal's mock wallet, resolver admin) : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
delegate / attacker                             : 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
payer                                           : 0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65
PermissionedResolver                            : 0x11fb930cB0a7493E5bF8a51bE5a64AeB642DaA37
name delegated to the attacker                  : resedit875836.eth
name never delegated to them                    : regedit875836.eth   (same resolver, same owner)
```

### Result

```json
{"delegatedName":"resedit875836.eth",
 "attackedName":"regedit875836.eth",
 "resolver":"0x11fb930cB0a7493E5bF8a51bE5a64AeB642DaA37",
 "displayedScope":"(blank)",
 "transaction":{"functionName":"grantRootRoles","args":["1","0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC"]},
 "rootRoleAfterEdit":"1",
 "attackerAddressRendered":"0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
 "attackerGainWei":"250000000000000000"}
```

### Assertions, in order

1. **Clean start.** The delegate's root and name-scoped roles are revoked on chain and `roles(0, delegate)` is asserted to be `0` before the UI is touched.
2. **The scoped role is created through Portal's own UI.** The operator uses **Add user**, picks `resedit875836.eth` from the name combobox, ticks **Set Text**, and approves. This is a name-scoped grant made by the application itself — not a fixture write.
3. **The indexer holds the real resource.** Panoptes returns both resolver nodes and the non-root role resource, `0x47349d…00000000`.
4. **The row shows no scope.** The sidebar renders neither "Global roles (all names)" (asserted `toHaveCount(0)`) nor "Roles scoped to resedit875836.eth" (asserted `toHaveCount(0)`).
5. **The operator adds one permission.** *Set Address* → Manager is unchecked before, *Set Text* → Manager is checked; the operator ticks *Set Address* and saves. The confirmation modal reads only `Save resolver role changes`.
6. **Exactly one resolver write occurs in the measured window, and it is the wrong one:** `grantRootRoles(1, 0x3C44…)`. Not `authorizeNameRoles`.
7. **On-chain authority is root.** `roles(0, delegate)` becomes `1` — `RESOLVER_ROLE_SET_ADDR` over every name the resolver serves.
8. **A name that was never delegated falls under their control.** The delegate calls `setAddr` for `regedit875836.eth` and it succeeds.
9. **Portal renders their address for that name,** on `/regedit875836.eth/address`.
10. **Funds follow.** The payer sends 0.25 ETH to the address read out of that page's Mainnet *Record* cell; the delegate's balance rises by exactly `250000000000000000` wei. Across the whole PoC the operator approves exactly **two** transactions — the *Add user* grant that creates the name-scoped role, and the edit that adds one permission — and both are ones they initiated deliberately. `wallet.getPendingRequestCount(SendTransaction)` is `0` after the edit: no further signature is requested at any point, and none is needed for the delegate to hold root.

### Reproduction

1. Start the isolated stack — anvil forked on `:8546`, plus the repository's Panoptes indexer and API on `:5658`.
2. `node setup-t32-full-ui.mjs` — registers `resedit875836.eth` and `regedit875836.eth` to the operator and deploys the shared PermissionedResolver. (Deployment is CREATE2 with a fixed salt, so the resolver address is reproducible.)
3. Start the Explorer with `VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5658/graphql` and `VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8546`.
4. Run the spec:

   ```
   PORTAL_APP_URL=http://localhost:<port> ANVIL_RPC_URL=http://127.0.0.1:8546 \
     npx playwright test --config=projects/portal/playwright.config.ts \
     projects/portal/tests/t32-full-ui.spec.ts
   ```

Result: **1 passed**, archived verbatim in `T32-FINAL-RUN.txt`. To see the substitution by hand, add any permission to a resolver row whose Name cell is blank and decode the resulting transaction: the selector will be `grantRootRoles`.
