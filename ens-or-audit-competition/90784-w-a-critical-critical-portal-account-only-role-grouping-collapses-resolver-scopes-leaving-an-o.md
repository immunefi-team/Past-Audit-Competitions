# 90784 w a critical critical portal account only role grouping collapses resolver scopes leaving an offboarded collaborator able to redirect ens resolution

## #90784 \[W\&A-Critical] Critical: Portal account-only role grouping collapses resolver scopes, leaving an offboarded collaborator able to redirect ENS resolution

**Submitted on Aug 27th 2026 at 08:06:05 UTC by @evolutionstorm for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90784
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

### Description

### Summary

An ENS resolver owner can complete Portal's normal **Remove user** workflow, approve a transaction represented as removing a collaborator **from all roles**, and still leave that collaborator with root-scoped resolver authority.

Portal groups root and name-scoped resolver entries only by account. A mixed account is rendered as **Global roles (all names)**, but the removal action selects the first non-root name, unions every displayed role, and submits one name-scoped `authorizeNameRoles(..., false)` transaction. It never calls `revokeRootRoles`.

The transaction succeeds and clears the name-scoped grant while the root grant survives. In two fresh local-chain browser fixtures, a former non-admin collaborator retained root `SET_ADDR` after the owner approved the advertised removal and changed the victim name's ETH address to an attacker-controlled canary. A resource-aware repaired control removed both scopes and made the same write revert.

### Selected impact and severity

**Severity: Critical**

**Selected in-scope impact (verbatim):**

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves

This finding relies on the first example: **resolving an ENS name to an attacker-controlled address**. Portal represents complete offboarding but submits only a name-scoped revocation. The retained root `SET_ADDR` authority then lets the former collaborator change the victim name's resolution to an attacker-controlled address. The PoC directly measures the resulting resolver state: `addr(namehash(name), 60)` returns the attacker's canary after offboarding.

The misleading removal is the mechanism that produces that qualifying impact, not a separate claimed impact. Portal displays `Global roles (all names)` and asks the owner to remove the collaborator `from all roles`, but the transaction revokes only one name-scoped role. Root authority survives and enables the subsequent attacker-controlled resolution.

No production or third-party fund transfer was performed or is claimed. The demonstrated impact terminates at the program's expressly listed attacker-controlled ENS-resolution example.

The required owner action is a prompted, ordinary Portal offboarding workflow, not an unprompted or unusual action. The prior delegation requirement is stated as a precondition rather than treated as continuing authorization: after the owner approves explicit removal from all roles, the collaborator is a former operator and should retain no represented authority.

Reference: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

### Preconditions and attacker model

1. A resolver owner previously assigned a non-admin collaborator one ordinary root-scoped manager role and one ordinary name-scoped manager role on the same resolver.
2. The resolver overview contains both valid role entries for that account.
3. The owner later uses Portal's normal **Remove user** workflow to offboard the collaborator.

This is a natural configuration supported by Portal itself. Its **Add user** sheet lets an owner select either `Root (all nodes)` or any individual resolver name before assigning manager roles ([`ResolverAddUserSheet.tsx:126-135,186-211,248-285`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L126-L135)). Repeating that ordinary flow for the same collaborator at root and name scope creates the mixed row; no direct contract call or malformed indexer record is needed.

No governance key, victim key, indexer control, role-admin bit, browser defect, or third-party compromise is required. A contract control confirms that the ordinary root manager cannot self-grant the second name-scoped role.

### Root cause

1. [`useResolverOverview.ts:90-98`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L90-L98) receives each role's `account`, `resource`, and `roleBitmap` correctly.
2. [`resolverRoles.ts:179-220`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220) groups only by lowercased account. It merges roles and resource names from independent resources into one `AccountRoleGroup`.
3. [`ResolverRolesTable.tsx:102-104`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L104) passes that mixed row directly to the sidebar.
4. [`ResolverRolesSidebar.tsx:165-193`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L193) retains the unioned roles but collapses the resource list to one name:

   ```ts
   const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
   ```
5. The sidebar labels any row containing root as `Global roles (all names)` at [`ResolverRolesSidebar.tsx:389-396`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L396) and asks for confirmation to remove the user `from all roles` at [`:441-465`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L465).
6. [`ResolverRolesSidebar.tsx:307-323`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) and [`:467-501`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L467-L501) construct one remove action using the single `roleName` and the union of every displayed role.
7. [`prepareResolverRolesIntent.ts:38-47`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L38-L47) forwards that lossy action.
8. [`revokeResolverRoles.ts:45-74`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L74) selects root scope only when `name === ''`. Because a mixed row contains a non-root name, it always creates a single name-scoped revoke.

The production test [`resolverRoles.test.ts:185-197`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.test.ts#L185-L197), `should merge the same account across resources`, explicitly requires this grouping. It asserts only the combined row and role union; it does not exercise editing or removal and therefore does not establish that collapsing those resources into one transaction is intended. The reported branch is that untested destructive consumer of the otherwise expected grouping.

Official ENS documentation preserves the boundary that Portal loses: root roles apply contract-wide, resource roles are specific, and root revocation uses the distinct `revokeRootRoles` operation. Portal correctly retains those separate operations in its transaction helper but chooses only one after combining the resources.

References: [Enhanced Access Control](https://docs.ens.domains/ensv2/enhanced-access-control/) and [Permissioned Resolver](https://docs.ens.domains/ensv2/permissioned-resolver/).

### Reviewed operation versus submitted transaction

Portal renders:

```
Global roles (all names)
Remove user
Are you sure you want to remove this user from all roles?
```

For the browser witness—root `SET_ADDR` plus name-scoped `SET_TEXT`—the signed transaction starts with selector `0xbbd9abb5` and decodes as:

```
authorizeNameRoles(
  dnsName,
  0x11,
  collaborator,
  false
)
```

The correct operation requires one transaction per represented resource:

```
revokeRootRoles(0x01, collaborator)
authorizeNameRoles(dnsName, 0x10, collaborator, false)
```

### Demonstrated effect

Both fresh browser fixtures produced the same state transition:

```
before: root SET_ADDR = 1, name SET_TEXT = 16
owner approves Portal remove transaction
after:  root SET_ADDR = 1, name SET_TEXT = 0
former collaborator setAddr(namehash(name), 60, canary) succeeds
addr(namehash(name), 60) == canary
```

The former collaborator can therefore redirect the victim name's ETH resolution after explicit offboarding. Root scope is contract-wide, so the same role authorizes record writes for every node whose records are served by that resolver.

The implementation also generalizes beyond the two-resource witness. For a collaborator represented at root, name A, and name B, `resolvedNames.find(...)` selects only the first non-root name. Portal submits one revoke for that name; root and the other name remain untouched. The browser PoC demonstrates root plus one name, while this multiple-name consequence follows directly from the same production selection at `ResolverRolesSidebar.tsx:165-170`.

### Measured attack cost

After the owner completes the ordinary removal transaction, exploitation requires one attacker-signed `setAddr` transaction. The two independent browser fixtures each measured that transaction at exactly **64,542 gas**.

```
1 transaction x 64,542 gas = 64,542 total gas
64,542 gas x 1 gwei = 0.000064542 ETH
0.000064542 ETH x $2,486.61/ETH = approximately $0.16
```

The USD illustration uses the ETH/USD price shown for 2026-08-27 by [Investing.com](https://ng.investing.com/crypto/ethereum/historical-data). At the fixtures' measured effective gas prices of approximately 1.004-1.035 gwei, the transaction cost was approximately **$0.16-$0.17** at that reference price. At that cost, the former collaborator can redirect the affected victim name; because the retained role is root-scoped, it authorizes record writes for every name served by that resolver. The owner's 40,938-gas removal is the victim's normal offboarding action, not an attacker expense. No attack-scale extrapolation is used.

### PoC boundaries and two independent witnesses

The unlisted Gist contains three complementary proofs:

1. **Rendered component and intent witness:** production grouping and `ResolverRolesSidebar` are rendered; the real removal clicks invoke the production intent builder. Root `SET_TEXT = 0x10` and name `SET_ADDR = 0x01` are used deliberately.
2. **Production helper/local-contract witness:** the same inverse role ordering executes against a local PermissionedResolver. The name role becomes zero, root `SET_TEXT` remains, and the former collaborator changes a text record.
3. **Real Portal/local-chain browser witness:** root `SET_ADDR = 0x01` and name `SET_TEXT = 0x10` are used. The unmodified Portal, transaction manager, wallet signing, receipts, contract storage, and address readback are real. This witness proves address redirection twice.

Using opposite role orderings shows the defect is loss of the resource boundary, not special behavior of one role bit.

The browser proof intercepts only `getResolverOverview` because the frozen E2E indexer fixture does not expose the generated resolver-role feed. The supplied response mirrors the root and name grants created on-chain and uses the exact shape consumed by the production query. Every other GraphQL request falls through unchanged. No indexer vulnerability or indexer control is claimed; the indexer is only a reachability fixture, while all security-relevant writes, receipts, role storage, and resolution effects come from local Anvil.

### Negative and repaired controls

#### Three-part proof matrix

| Required proof                                           | Test and observed result                                                                                                                                    |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Baseline consults the scope discriminator                | A root-only Portal row resolves `roleName` to the empty string, selects `revokeRootRoles`, and clears the root bitmap.                                      |
| Correct resource-aware enforcement blocks the actor      | Explicit root and name revocations leave both bitmaps zero; the identical former-collaborator record write then reverts with `EACUnauthorizedAccountRoles`. |
| Mixed-resource condition defeats the represented removal | Root plus name entries merge into one row; Portal submits one name-scoped revoke, root remains `1`, and the former collaborator's `setAddr` succeeds.       |

* A root-only Portal row selects `revokeRootRoles` and clears the root bitmap.
* Explicit root plus name revokes leave both bitmaps zero.
* After that repair, the identical former-collaborator record write reverts.
* An ordinary root manager cannot self-seed the name-scoped grant.
* The formal model proves name-only removal of union `0x11` leaves root bit `0x01`, while resource-aware removal clears it.
* No mainnet, public testnet, or third-party transaction is performed.

### Novelty and known-issue delta

#### Published ENS Apps known-issues record (2026-08-14)

The published record states that reports sharing a listed root cause are duplicates, but it also expressly preserves as new a demonstrated security consequence of a functional issue, giving as its example **“a role combination that grants authority it should not, rather than merely erroring.”** This finding matches that carveout directly and does not share a listed root cause.

The closest disclosed entries are materially different:

| Entry            | Disclosed behavior                                                                                             | Why this finding is different                                                                                                                                          |
| ---------------- | -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| QA-01 / WEB-1257 | Certain role combinations after a name or parent transfer cause errors in later record or primary-name actions | No transfer occurs here. The ordinary removal transaction succeeds, but Portal silently revokes only one resource and preserves authority the owner expressly removed. |
| WEB-1305         | Transferring V2 roles to a V1 name remains pending                                                             | No V1 name, role transfer, or pending transaction is involved.                                                                                                         |
| R2-06            | Smart-account session revocation clears only local state while an on-chain co-owner permission remains         | This finding uses no smart account or session. It constructs the wrong resolver authorization call because Portal merged independent root/name resources.              |
| WEB-716          | The UI does not offer replacement of an existing subregistry                                                   | No registry deployment or replacement is involved.                                                                                                                     |

No listed entry describes account-only grouping of resolver roles, collapse of multiple resources into one removal action, or retention of root resolver authority after Portal confirms removal **from all roles**.

#### Public contracts-v2 issue #418

Issue #418 concerns contract transfer logic: ERC-1155 ownership moves while third-party EAC grants remain stored. Its root cause is that contract `_update` transfers only the owner's roles. This finding performs no ownership transfer. Its root cause is Portal's account-only grouping and its lossy construction of an explicit removal transaction.

Reference: <https://github.com/ensdomains/contracts-v2/issues/418>

This finding is also unrelated to R3-07 transaction-ID reuse. It uses no transaction restoration, retry, or history state.

### Recommended fix

Treat `(account, resource)` as the editable and removable identity. Prefer one row per resource. If account-grouped presentation is retained, construct one explicit revocation per underlying entry:

* `revokeRootRoles` for the root entry;
* `authorizeNameRoles(..., false)` for each name entry;
* fail closed for unknown or unresolved resource identifiers.

The confirmation and transaction modal should enumerate every resource and role being removed. Re-read authoritative entries before building the sequence, and add regression tests for root+name, multiple-name, duplicate-role, unknown-resource, partial-failure, and root-only rows.

### Link to Proof of Concept

<https://gist.github.com/evolutionstorm/b0775bb11c51d3f1efc1d534ce54be73>

### Proof of Concept

## Resolver mixed-scope removal PoC

The complete self-contained PoC is in this unlisted GitHub Gist:

<https://gist.github.com/evolutionstorm/b0775bb11c51d3f1efc1d534ce54be73>

This Gist reproduces an ENS Explorer/Portal authorization-scope error at the audited revision `63772fd872af472ced58b009499355f3430c2a86`.

The same collaborator holds one root-scoped resolver role and one name-scoped role. Portal renders one combined row as `Global roles (all names)` and asks the owner to remove the collaborator `from all roles`, but the signed transaction is one name-scoped `authorizeNameRoles(..., false)` call. The root role survives.

All execution is local. The browser proof uses the unmodified Portal, its real transaction manager, a headless test wallet, and a local PermissionedResolver. It intercepts only `getResolverOverview` to supply the two role entries created on-chain because the frozen E2E indexer fixture does not expose that role feed. All other GraphQL operations fall through unchanged.

### Layout

* `poc_component.test.tsx`: production grouping/sidebar/intent proof and a root-only control.
* `poc_onchain.test.ts`: production helper against the local resolver, including the inverse-role witness, self-seeding control, retained write, and repair.
* `poc_browser.spec.ts`: focused real-Portal/local-chain proof with two fresh names, address redirection, root-only control, and resource-aware repair.
* `mixed_scope_revoke.smt2`: formal scope-preservation control.

### Prerequisites

* Node 22.22, pnpm 10.27, Docker, and Z3.

`target/` is the local checkout name used by this PoC; it is not a directory in the ENS repository. Create a clean workspace, clone the complete competition repository into that local directory, pin it to the tested competition commit, install its dependencies, and clone this Gist beside it:

```bash
mkdir ens-resolver-removal-repro
cd ens-resolver-removal-repro

git clone --branch audit-comp-ready --single-branch \
  https://github.com/immunefi-team/audit-comp-ens.git target
git -C target checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm --dir target install --frozen-lockfile

git clone https://gist.github.com/b0775bb11c51d3f1efc1d534ce54be73.git \
  resolver-remove-all-poc
```

The expected layout is:

```
audit-comp-ens/
├── target/
└── resolver-remove-all-poc/
```

### Reproduce

{% stepper %}
{% step %}

### Start the local ENS fixture

```bash
cd target
pnpm e2e:infra:up
cd ..
```

{% endstep %}

{% step %}

### Run the production component and intent proof

```bash
pnpm --dir target/apps/portal exec vitest run \
  --config ../../../resolver-remove-all-poc/vitest.component.config.ts \
  --reporter=verbose
```

Expected: `2 passed`.
{% endstep %}

{% step %}

### Run the production helper/local-contract proof

```bash
pnpm --dir target/apps/portal exec vitest run \
  --config ../../../resolver-remove-all-poc/vitest.onchain.config.ts \
  --reporter=verbose
```

Expected: `4 passed`.
{% endstep %}

{% step %}

### Run the focused browser proof

```bash
pnpm --dir target/e2e exec playwright test \
  --config ../../resolver-remove-all-poc/playwright.config.ts \
  --grep "resolver remove-all UI revokes only one displayed scope"
```

Expected: `1 passed`. The test creates and checks two independent name fixtures.
{% endstep %}

{% step %}

### Run the formal control

```bash
z3 resolver-remove-all-poc/mixed_scope_revoke.smt2
```

Expected:

```
sat
((displayedUnion #x0000000000000000000000000000000000000000000000000000000000000011)
 (submittedBitmap #x0000000000000000000000000000000000000000000000000000000000000011)
 (rootAfterBug #x0000000000000000000000000000000000000000000000000000000000000001)
 (nameAfterBug #x0000000000000000000000000000000000000000000000000000000000000000)
 (rootAfterRepair #x0000000000000000000000000000000000000000000000000000000000000000))
```

{% endstep %}

{% step %}

### Clean up

```bash
cd target
pnpm e2e:infra:down
```

{% endstep %}
{% endstepper %}
