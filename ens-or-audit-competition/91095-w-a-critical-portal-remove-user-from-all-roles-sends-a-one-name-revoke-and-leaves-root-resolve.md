# 91095 w a critical portal remove user from all roles sends a one name revoke and leaves root resolver authority usable

**Submitted on Aug 29th 2026 at 15:55:22 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91095
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Portal “Remove user from all roles” sends a one-name revoke and leaves root resolver authority usable

### Prior report

This is the proof-complete replacement for closed report `#89611`. The prior closure accepted that Portal built a name-scoped call but said the in-scope proof did not demonstrate the consequence through the Explorer application. The attached E2E now runs the ordinary Portal flow, sends the real wallet transaction, verifies its exact calldata and receipt, proves the removed account uses the authority left behind, and verifies the resulting attacker address on Explorer's public Address Resolution page.

This report should replace `#89611`, not count as a second independent finding from the same researcher.

### Target

* Program: ENS Audit Competition
* Asset: Explorer app, `apps/portal`
* Upstream audited revision: `63772fd872af472ced58b009499355f3430c2a86`
* Tested frozen branch: `audit-comp-ready` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

### Summary

Portal groups one account's root-scoped and name-scoped resolver roles into one row. Because the row contains a root record, the UI labels it `Global roles (all names)`, and the removal confirmation says the account will be removed `from all roles`.

The transaction builder does not preserve that resource set. It unions every displayed role bit, chooses only the first non-root name, and sends one name-scoped `authorizeNameRoles(..., false)` call. The transaction succeeds, but it cannot revoke the root resource or any sibling-name resource.

The full E2E gives one target root `ROLE_SET_ADDR | ROLE_UPGRADE` and name-scoped `ROLE_SET_TEXT`, clicks the normal Portal Remove User flow, signs and mines the generated transaction, and proves:

* Explorer initially renders the owner's safe ETH address.
* The name role is removed.
* Both root roles remain.
* The removed account changes the name's ETH address using the retained root authority.
* Revoking the surviving root role does not restore the overwritten address.
* After a clean reload, Explorer renders the removed account's address instead of the safe address.

### Root cause

`apps/portal/src/lib/roles/resolverRoles.ts` groups records only by account. It appends role bits from every resource into one `decodedRoles` array and collects root/name labels into `resolvedNames`.

`ResolverRolesSidebar.tsx` then derives one transaction name:

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The same component labels any row containing `(root)` as `Global roles (all names)` and asks:

```
Are you sure you want to remove this user from all roles?
```

But its Remove action passes the union bitmap with only `roleName`. `prepareRevokeResolverRolesTransaction` uses root scope only when the name is empty; otherwise it encodes a name-scoped revoke:

```ts
name === ''
  ? { scope: 'root', roles, targetAccount: account, resolverAddress }
  : { scope: 'name', name, roles, targetAccount: account, resolverAddress }
```

There is no per-resource loop. One successful transaction therefore cannot perform the all-resource removal presented by the UI.

### Proof of concept

Files:

* `poc-resolver-role-collapse-ui-send.spec.ts`
* `playwright.portal-role-collapse.config.ts`
* `E2E-README.md`

Place them in the repository E2E paths shown by their imports and run from `e2e`:

```bash
PORTAL_APP_URL=http://127.0.0.1:3002 \
pnpm exec playwright test \
  --config projects/portal/playwright.portal-role-collapse.config.ts \
  --reporter=list
```

{% stepper %}
{% step %}

## Registers a real local ENS name and obtains its resolver.

{% endstep %}

{% step %}

## Grants the target real root and name-scoped roles on that resolver.

{% endstep %}

{% step %}

## Supplies those same records to the Portal role view.

{% endstep %}

{% step %}

## Opens the normal Roles page and clicks Remove User.

{% endstep %}

{% step %}

## Confirms the UI's all-roles wording.

{% endstep %}

{% step %}

## Authorizes the connected-wallet request and waits for the real transaction receipt.

{% endstep %}

{% step %}

## Decodes the mined calldata and verifies it is the wrong one-name union revoke.

{% endstep %}

{% step %}

## Reads the resolver state and proves the root bitmap survived.

{% endstep %}

{% step %}

## Sends a transaction from the removed account that changes the name's ETH address.

{% endstep %}

{% step %}

## Revokes the surviving root role and proves the overwritten address remains.

{% endstep %}

{% step %}

## Reloads the in-scope Explorer Address Resolution page and verifies it still displays the removed account's address.

The same page displayed the owner's safe address before removal.
{% endstep %}
{% endstepper %}

Canonical output:

```
[resolver-role-collapse] explorer-before=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[resolver-role-collapse] ui-tx=0x7ff522676ea39e74fe82b734c7457ab9f520db88a1b0b9884bc1ac5c70fae8ab3
[resolver-role-collapse] ui-row=Global roles (all names)
[resolver-role-collapse] ui-confirmation=remove from all roles
[resolver-role-collapse] ui-status=success
[resolver-role-collapse] name-role-after-ui=false
[resolver-role-collapse] root-roles-after-ui=true
[resolver-role-collapse] exploit-tx=0xd35c736e22310c04c5d760b64aa4d7f6819d40490ff31dc0021e98f2efae3dfe1
[resolver-role-collapse] final-address=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] root-roles-after-corrective-revoke=false
[resolver-role-collapse] address-after-corrective-revoke=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] explorer-after=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
1 passed (1.1m)
```

The mocked indexer response contains the exact grants already created on the local chain. For the public Address Resolution view, the mock supplies only an empty list of indexed record keys so the page can load; the ETH address value is resolved from the live local chain. The vulnerable UI, wallet request, calldata, mined transactions, role reads, address change, and both Explorer address displays are unmodified and real against the local ENS environment.

### Expected behavior

Removing a row presented as global/all roles must revoke every displayed account/resource pair, or the UI must require separate explicit removals for root and each name.

### Actual behavior

Portal sends one name-scoped revoke containing a union of unrelated resource bits. The transaction succeeds while root and sibling-resource authority remains usable.

### Impact and severity

The closest exact competition impact is the Critical connected-wallet row for modifying transaction arguments. The user approves an all-resource removal, but the in-scope app submits one-name calldata and leaves resolver-changing authority with the account it says was removed.

The target was previously an authorized delegate, but after the misleading successful revocation it can trigger the exploit at any time; there is no indexer race or requirement that the owner perform another action in a narrow window. The owner can recover, but revoking the surviving role does not undo the attacker's write: the owner must also notice and send a separate address-correction transaction. That recovery is a severity limitation and may justify a High assessment. This report does not claim direct theft or irreversible loss. It proves that a successful security revocation generated by Portal leaves material authority active, immediately usable, and rendered as the wrong address by Explorer even after the authority is finally revoked.

### Known-issue and duplicate boundary

No published ENS known issue describes this grouping-and-one-resource-revoke root. `QA-01` covers functional errors after role combinations and expressly leaves demonstrated unintended authority as a new consequence.

Immunefi matched the earlier report to private report `#89420`. ENS's live rules say duplicate bug submissions are valid. The match may affect chief-finder status and reward share, but it does not make the Portal bug a public known issue.

### Recommended fix

Keep roles keyed by both account and resource. For Remove, generate one exact revoke per root/name resource shown to the user. If the product wants a single grouped row, the confirmation must list every transaction and resource. A simpler safe design is one row per account/resource pair.

### Link to Proof of Concept

<https://gist.github.com/krutftw/13c2ca99c383abe35295929dac305ea8>

## Proof of Concept

This is the proof-complete replacement for closed report `#89611`. The earlier report's secret gist has been updated in place with the full UI E2E described below.

### Tested source

* Repository: `https://github.com/immunefi-team/audit-comp-ens`
* Branch: `audit-comp-ready`
* Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* In-scope asset: `apps/portal`

### Files in the secret gist

* `poc-resolver-role-collapse-ui-send.spec.ts`
* `playwright.portal-role-collapse.config.ts`
* `E2E-README.md`
* `TERMINAL-OUTPUT.txt`

The gist also retains the original focused unit and contract tests for comparison.

### File placement

```
e2e/projects/portal/tests/poc-resolver-role-collapse-ui-send.spec.ts
e2e/projects/portal/playwright.portal-role-collapse.config.ts
```

### Start the repository test environment

From the repository root:

```bash
pnpm install --frozen-lockfile
pnpm --filter @ens-apps/e2e infra:up
pnpm --filter portal dev --host 127.0.0.1 --port 3002
```

The repository's `e2e/README.md` documents the same local Sepolia/Anvil test stack. Leave Portal running, then in another shell run:

```bash
cd e2e

PORTAL_APP_URL=http://127.0.0.1:3002 \
ANVIL_RPC_URL=http://127.0.0.1:8545 \
pnpm exec playwright test \
  --config projects/portal/playwright.portal-role-collapse.config.ts \
  --reporter=list
```

### What the test does

{% stepper %}
{% step %}

## Registers a real local `.eth` name owned by the connected wallet and sets its safe ETH address to the owner.

{% endstep %}

{% step %}

## Grants a second account root `ROLE_SET_ADDR | ROLE_UPGRADE` and name-scoped `ROLE_SET_TEXT` on the real local resolver.

{% endstep %}

{% step %}

## Loads the public Explorer Address Resolution page and verifies it renders the owner's safe address.

{% endstep %}

{% step %}

## Loads the normal Portal resolver Roles page with indexer role rows that exactly mirror those on-chain grants.

{% endstep %}

{% step %}

## Verifies Portal groups them into one `Global roles (all names)` row.

{% endstep %}

{% step %}

## Clicks Remove User and verifies the confirmation says the account will be removed `from all roles`.

{% endstep %}

{% step %}

## Authorizes the real connected-wallet request and waits for the transaction to mine.

{% endstep %}

{% step %}

## Finds and decodes the mined transaction.

It is `authorizeNameRoles(name, unionBitmap, target, false)`, not a root revoke or a per-resource revoke sequence.
{% endstep %}

{% step %}

## Reads the resolver and proves the name role is gone while root `ROLE_SET_ADDR | ROLE_UPGRADE` remains.

{% endstep %}

{% step %}

## Sends a transaction from the removed account using the surviving root `ROLE_SET_ADDR` and changes the name's ETH address to that account.

{% endstep %}

{% step %}

## Revokes the surviving root roles from the removed account and proves the already-overwritten address remains.

{% endstep %}

{% step %}

## Performs a clean Explorer reload and verifies the Address Resolution page renders the removed account's address instead of the original safe address.

{% endstep %}
{% endstepper %}

### Fresh result

```
[resolver-role-collapse] ui-row=Global roles (all names)
[resolver-role-collapse] ui-confirmation=remove from all roles
[resolver-role-collapse] name=portal-role-collapse-1788017743.eth
[resolver-role-collapse] resolver=0x85735fA91280962dcf564BAdfb5AbfBB616055C5
[resolver-role-collapse] target=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] explorer-before=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[resolver-role-collapse] ui-tx=0x7ff522676ea39e74fe82b734c7457ab9f520db88a1b0b9884bc1ac5c70fae8ab3
[resolver-role-collapse] ui-status=success
[resolver-role-collapse] name-role-after-ui=false
[resolver-role-collapse] root-roles-after-ui=true
[resolver-role-collapse] exploit-tx=0xd35c736e22310c04c5d760b64aa4d7f6819d40490ff31dc0021e98f2efae3dfe1
[resolver-role-collapse] final-address=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] root-roles-after-corrective-revoke=false
[resolver-role-collapse] address-after-corrective-revoke=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] explorer-after=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
1 passed (1.1m)
```

### Mock boundary

The role indexer response is mocked only to supply the same root and name-scoped grants already created on the local chain. For the Explorer address page it supplies an empty record-key inventory, not an address value. Portal's grouping and confirmation, the wallet request, both mined transactions, role state, the address write, the corrective role revoke, and both Explorer address displays are real against the local ENS environment.

The exploit does not rely on indexer lag. Once Portal's successful removal transaction leaves the root role active, the removed delegate can act at any time without another owner action. The owner can recover, but must both revoke the surviving authority and separately correct the overwritten address.
