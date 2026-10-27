# 91097 w a critical explorer s remove user from all roles signs a one resource revoke leaving a former delegate able to redirect another ens name

**Submitted on Aug 29th 2026 at 16:23:40 UTC by @JAO for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91097
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

### Brief/Intro

The Explorer groups every resolver-role assignment held by the same account across independent ENS resources into one row. The row visibly lists all affected names and asks the owner to confirm that the user will be removed “from all roles”, but the transaction builder selects only the first name and emits one name-scoped revoke. A delegate with `ROLE_SET_ADDR` on `alice.eth` and `bob.eth` therefore remains authorized on one name after the advertised remove-all transaction succeeds. The executable PoC has that supposedly removed delegate monitor the confirmed revocation, change the surviving name's address in the next block, and divert a subsequent 1 ETH resolution-based payment.

This is not a claim that an external stranger can acquire an initial role. The delegate was legitimately authorized before the owner used the security-sensitive offboarding action. The violated boundary is the owner's explicit, confirmed revocation: after Explorer reports a remove-all transaction as successful, the former delegate retains authority that the application knew about but omitted from the transaction.

### Vulnerability Details

The audited application revision is:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

The primary PoC uses these fixed values in every layer:

| Item                     | Value                                        |
| ------------------------ | -------------------------------------------- |
| Resolver owner           | `0x1111111111111111111111111111111111111111` |
| Delegate                 | `0x2222222222222222222222222222222222222222` |
| Attacker destination     | `0x4444444444444444444444444444444444444444` |
| Intended payment address | `0x5555555555555555555555555555555555555555` |
| Payer                    | `0x6666666666666666666666666666666666666666` |
| Resources                | `alice.eth`, `bob.eth`                       |
| Role on each resource    | `ROLE_SET_ADDR`                              |

The affected role-management action is available to a normal administrator of that resolver; it does not require an ENS governance account. The route at `apps/portal/src/routes/resolver/$address/roles/index.tsx:41-50` sets `canManageRoles` when the connected account has root `ROLE_SET_ADDR` on the selected resolver, and passes that capability into the production roles table.

#### 1. The table deliberately merges independent resources

`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-104` passes the complete resolver role list to `groupRolesByAccount`.

`apps/portal/src/lib/roles/resolverRoles.ts:179-220` keys the result only by account. For every additional role assignment it appends both the assignment and the decoded role to the existing account row, and adds every resolved resource name to the same `resolvedNames` set:

```typescript
const account = role.account.toLowerCase()
const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

For the two grants above, the production function returns one row with:

```
resolvedNames = ["alice.eth", "bob.eth"]
decodedRoles = ["ROLE_SET_ADDR", "ROLE_SET_ADDR"]
```

This is not malformed indexer data. Both assignments are current and valid; the defect is the application's loss of the resource-to-role relationship while constructing an account-level action.

#### 2. The sidebar shows all resources but selects only one

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169` reads the merged values and reduces all resources to the first non-root name:

```typescript
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The rendered sheet identifies the row as:

```
Roles scoped to alice.eth, bob.eth
```

The removal dialog at `ResolverRolesSidebar.tsx:441-463` then states:

```
Are you sure you want to remove this user from all roles? This action cannot be undone.
```

`handleRemoveUser` at lines 307-322 nevertheless stores only `roleName` together with the union of the account row's decoded roles:

```typescript
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

#### 3. The sole wallet transaction addresses only `alice.eth`

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-74` has only two scopes: one root resource or one supplied name. With `roleName = alice.eth`, it encodes exactly one call:

```
authorizeNameRoles(dnsEncode("alice.eth"), ROLE_SET_ADDR, delegate, false)
```

The UI and logic PoCs independently produce the same calldata hash:

```
keccak256(calldata) = 0xf0c4c0bd9ef77b9401722879617d1d67a2532026fe6f315e9beef79b34b1870e
```

The sole calldata is:

```
0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000100000000000000000000000022222222222222222222222222222222222222220000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000b05616c6963650365746800000000000000000000000000000000000000000000
```

`bob.eth` is absent. After this call, `ROLE_SET_ADDR` is removed from `alice.eth` but remains unchanged on `bob.eth`.

#### 4. Post-transaction refresh can reveal, but does not repair, the residual grant

The remove mutation polls the indexer after confirmation (`ResolverRolesSidebar.tsx:278-286`; `pollForIndexerSync.ts:18-21,60-73`). The default schedule waits five seconds and performs five invalidations separated by three seconds. If the refreshed result is current, the table can show the remaining `bob.eth` assignment again.

This is an important limitation on stealth, but not a security control:

* the signed transaction has already succeeded with incomplete parameters;
* no atomic postcondition checks that all resources were revoked;
* no compensating transaction is submitted;
* the former delegate can monitor the confirmed revoke and act before the owner notices and signs a second transaction.

The PoC explicitly tests the refreshed state and expects the residual row to contain `bob.eth`; it does not depend on indexer lag or continued stale UI state.

#### 5. A former delegate acts independently after the normal owner workflow

The contract consequence harness uses the production `PermissionedResolver` at ENS v2 revision:

```
48b3e2d39513b9dd32ef1850877a29009bc807b9
```

It executes the exact Portal calldata above as the owner and proves:

```
alice.eth / ROLE_SET_ADDR / delegate = revoked
bob.eth   / ROLE_SET_ADDR / delegate = still authorized
```

It then advances one block. Acting as the delegate, it successfully calls the real resolver implementation to set `bob.eth` to the attacker destination. A payer resolves `bob.eth` only after that write and sends 1 ETH to the returned address. The assertions prove:

```
attacker balance       = 1 ETH
intended payee balance = 0 ETH
payer balance          = 0 ETH
```

No further victim action, browser compromise, stale page state, stale indexer response, leaked key, governance address, social engineering, or contract flaw is used after the ordinary remove-user transaction.

#### 6. This is distinct from intentional grant retention on name transfer

Public `ensdomains/contracts-v2` issue #418 discusses a different design: grants remain attached to a registry resource when ownership of an ERC-1155 name is transferred. ENS maintainers state in that thread that transfer-time retention is intentional and that enumerating existing grants requires indexed data.

This report does not challenge those transfer semantics and does not contain a name transfer. Here the same owner remains authorized over the same resolver, Explorer already receives every indexed resource assignment, and the owner explicitly selects **Remove user from all roles**. The application then discards the resource-to-role relationship and signs only one of the required revokes. The contracts correctly enforce that incomplete calldata.

### Attack Sequence

{% stepper %}
{% step %}

## Grant roles on two resources

An ordinary resolver administrator previously grants the same delegate `ROLE_SET_ADDR` on `alice.eth` and `bob.eth` in one shared permissioned resolver.
{% endstep %}

{% step %}

## End the delegate relationship

The relationship with that delegate ends or the delegate is suspected of compromise.
{% endstep %}

{% step %}

## Open the Explorer roles page

The owner opens the normal Explorer roles page. One delegate row visibly lists `alice.eth, bob.eth`.
{% endstep %}

{% step %}

## Confirm removal of all roles

The owner selects **Remove user** and confirms removal “from all roles”.
{% endstep %}

{% step %}

## Sign the incomplete revoke

Explorer prepares and the wallet signs the sole successful name-scoped revoke for `alice.eth`.
{% endstep %}

{% step %}

## Delegate redirects the remaining name

The delegate monitors the confirmed revocation. In the next block, it uses the surviving `bob.eth` role to replace that name's address with its own.
{% endstep %}

{% step %}

## Payer sends funds to the attacker

A later payer resolves `bob.eth` and sends funds to the attacker-controlled address.
{% endstep %}

{% step %}

## Residual role may appear after refresh

A refreshed Explorer may subsequently show the residual role, but detecting it does not recover a completed payment and still requires another owner transaction to revoke it.
{% endstep %}
{% endstepper %}

The prerequisite in the first step is explicit: this is not initial privilege escalation. It is persistence of a former delegate's authority after the normal interface confirmed that all of that account's roles were being removed.

### Impact Details

The primary in-scope impact is the connected wallet approving transaction parameters that do not implement the security-sensitive action the application presented. The row and confirmation cover every role represented by two ENS resources, while the sole transaction omits one resource entirely.

The concrete consequence is not theoretical. The contract PoC proves that the surviving production permission authorizes a post-revocation record change and that a subsequent resolution-based 1 ETH payment is transferred to the attacker. Correctly revoking both resources in the negative control causes the same attacker write to revert and preserves the intended payment address.

The former authorization does not make the post-revocation action legitimate. Before the owner's transaction the delegate is expected to have the role; after the advertised remove-all transaction it is expected to have none. The only authority used for the harmful next-block write is the assignment that Explorer omitted from the signed revocation.

This is an Explorer application defect. The ENS contract enforces the submitted scope correctly and is used only to demonstrate the consequence of the application-produced calldata; no contract or indexer vulnerability is claimed.

Secondary regression cases in the PoC files supplied through the Secret Gist show that the same account-only grouping also mishandles mixed root/name resources and fine-grained address resources. They are included as supporting evidence for the root cause, not as separate impact claims.

### References

* In-scope Explorer source: <https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal>
* Competition scope and exact Critical impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Competition information and rules: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* ENS EAC resource and revocation semantics: <https://docs.ens.domains/ensv2/enhanced-access-control/>
* Distinct transfer-retention design discussion (#418, expected transfer behavior): <https://github.com/ensdomains/contracts-v2/issues/418#issuecomment-5416414637>
* Distinct transfer-retention design discussion (#418, indexed enumeration): <https://github.com/ensdomains/contracts-v2/issues/418#issuecomment-5419175943>
* Resolver-role management authorization: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L41-L50>
* `ResolverRolesTable`: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L95-L139>
* Account-only grouping: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L175-L220>
* Single-name selection and removal: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L169>
* Confirmation and transaction preparation: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L510>
* Revoke calldata builder: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L28-L75>

## Link to Proof of Concept

<https://gist.github.com/jaooli-cyber/22d1ac0a2cafb010ba1a10db3015b09d>

## Proof of Concept

### What the three Secret Gist files prove

Secret Gist environment (`public: false`):\
<https://gist.github.com/jaooli-cyber/22d1ac0a2cafb010ba1a10db3015b09d>

{% stepper %}
{% step %}

## `multi-resource-resolver-role-revocation.ui.poc.test.tsx`

Renders the real `ResolverRolesSidebar`, verifies the two-name display and exact all-roles confirmation, then captures and decodes the sole production intent.
{% endstep %}

{% step %}

## `multi-resource-resolver-role-revocation.poc.test.ts`

Runs the real account grouping and revoke-intent builder; it proves the omitted resource, the post-refresh residual row, order independence, a correct single-resource control, and secondary root/fine-grained cases.
{% endstep %}

{% step %}

## `ResolverRoleRevocationEvm.t.sol`

Executes the exact Portal calldata against the real ENS v2 `PermissionedResolver`, performs the independently attacker-driven next-block address change and 1 ETH diversion, and includes the correct two-resource revocation control.
{% endstep %}
{% endstepper %}

SHA-256:

```
ad61e89302948dc63697579cc9598c5fa7b0e3def3450415b38b6de2a435c378  multi-resource-resolver-role-revocation.poc.test.ts
50dd47b818fcd274e5c0a834f01fd35e5c41f7e72d7157a6e95d65556af81fd1  multi-resource-resolver-role-revocation.ui.poc.test.tsx
e1e1c43d95facbd3dbb32002743cf9a5d259689bc671423648aef5ea4e8305c3  ResolverRoleRevocationEvm.t.sol
```

{% stepper %}
{% step %}

## Run the Explorer UI and calldata PoCs

Check out the frozen competition source:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm install --frozen-lockfile
```

Copy the two TypeScript PoC files from the linked Secret Gist to:

```
apps/portal/src/features/resolver/components/multi-resource-resolver-role-revocation.poc.test.ts
apps/portal/src/features/resolver/components/multi-resource-resolver-role-revocation.ui.poc.test.tsx
```

Run:

```bash
pnpm --dir apps/portal exec vitest run \
  src/features/resolver/components/multi-resource-resolver-role-revocation.poc.test.ts \
  src/features/resolver/components/multi-resource-resolver-role-revocation.ui.poc.test.tsx \
  --reporter=verbose
```

Expected summary:

```
Test Files  2 passed (2)
Tests       6 passed (6)
```

The primary tests prove, in order:

1. The real grouping function produces one row for two independent names.
2. The real sheet shows both names and says “remove ... from all roles”.
3. The real intent contains one `authorizeNameRoles` revoke for `alice.eth`.
4. Its calldata hash is `0xf0c4c0bd9ef77b9401722879617d1d67a2532026fe6f315e9beef79b34b1870e`.
5. `bob.eth` remains authorized and reappears if current data is regrouped.
6. Reversing input order changes which name is revoked but always leaves one.
   {% endstep %}

{% step %}

## Run the contract consequence and negative control

Check out the tested ENS v2 revision with dependencies:

```bash
git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9
```

Copy the Solidity PoC file from the linked Secret Gist to:

```
test/unit/resolver/ResolverRoleRevocationEvm.t.sol
```

Run the primary consequence and correct-revocation control:

```bash
forge test \
  --match-path test/unit/resolver/ResolverRoleRevocationEvm.t.sol \
  --match-test 'test_(removeAllUiCallLeavesSecondNameAuthorityLiveAndDivertsPayment|correctPerResourceRevocationBlocksTheSameAttack)' \
  -vv
```

Expected result:

```
[PASS] test_correctPerResourceRevocationBlocksTheSameAttack()
[PASS] test_removeAllUiCallLeavesSecondNameAuthorityLiveAndDivertsPayment()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

To run all primary and secondary cases:

```bash
forge test \
  --match-path test/unit/resolver/ResolverRoleRevocationEvm.t.sol \
  -vv
```

Expected summary:

```
4 tests passed, 0 failed, 0 skipped
```

{% endstep %}
{% endstepper %}

### Validation performed by the researcher

```
Targeted Explorer PoCs:       6 passed, 0 failed
Explorer TypeScript check:    passed
Biome check:                  passed
Full Explorer regression:     151 files / 1455 tests passed
Contract consequence PoCs:    4 passed, 0 failed
Solidity formatting:          passed
Full contracts regression:    45 suites / 935 tests passed
```

The Explorer suite emits baseline happy-dom/Reown aborted remote-fetch messages during teardown; all commands exit with status 0.
