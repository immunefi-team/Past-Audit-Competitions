# 91318 w a critical portal s remove all roles action revokes only one name leaving a delegate s global resolver authority active

**Submitted on Aug 31st 2026 at 15:06:31 UTC by @Pig46940 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91318
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

When a resolver delegate has both a global root role and a role for an individual name, Portal merges the two assignments into one account row, shows both scopes, and asks the owner to confirm removal “from all roles,” but it submits and mines only one name-scoped union revocation; the name role is removed while the root role remains, allowing the delegate to keep contract-wide authority and modify an unrelated name after the owner completes the advertised removal workflow.

## Vulnerability Details

The Permissioned Resolver stores authorization per resource. A root role applies to every name served by the resolver, while a name role applies only to one DNS-encoded name. Portal's Add user flow exposes that distinction: an empty selected name is labelled `Root (all nodes)`, and selecting a name creates a separate name-scoped assignment.

The failure occurs when Portal reads those assignments back. `groupRolesByAccount` groups records only by delegate account, unions their decoded role bits, and keeps their resource labels in one list:

```
root resource:          delegate D -> ROLE_SET_ADDR
mixed-scope-b.eth:      delegate D -> ROLE_SET_TEXT

grouped Portal row:     delegate D
displayed resources:    (root), mixed-scope-b.eth
displayed roles:        Set Address, Set Text
```

The sidebar then chooses one scope for the entire grouped row:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Because a non-root name is present, `roleName` becomes `mixed-scope-b.eth`. The removal action combines that one name with the unioned role list:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

Portal shows the owner this confirmation immediately before starting the transaction:

> Are you sure you want to remove this user from all roles? This action cannot be undone.

The transaction helper treats a non-empty name as a name-scoped operation, so the request becomes:

```solidity
authorizeNameRoles(
    dnsEncode("mixed-scope-b.eth"),
    ROLE_SET_ADDR | ROLE_SET_TEXT,
    delegate,
    false
)
```

That call can revoke bits only from the `mixed-scope-b.eth` resource. It cannot revoke the independent root assignment. Implementing the displayed all-role action requires one revocation for each represented resource, including:

```solidity
revokeRootRoles(ROLE_SET_ADDR, delegate)
authorizeNameRoles(dnsEncode("mixed-scope-b.eth"), ROLE_SET_TEXT, delegate, false)
```

The complete causal chain is:

```
independent root and name role records
→ account-only grouping unions role bits and resource labels
→ sidebar selects one non-root name for the whole row
→ owner confirms removal from all displayed roles
→ Portal submits one name-scoped union revocation
→ the root role remains active
→ the delegate can still modify unrelated names served by the resolver
```

The PoC demonstrates the following contradiction using the production Portal path and the official Permissioned Resolver implementation on a fresh local Anvil chain:

```yaml
Displayed resources:
  - (root)
  - mixed-scope-b.eth
Displayed roles:
  - Set Address
  - Set Text
Authorized action: remove this user from all roles
Wallet-bound function: authorizeNameRoles
Wallet-bound role bitmap: 17
Portal calldata equals submitted calldata: true
Portal calldata equals mined calldata: true
Before the transaction:
  root role bitmap: 1
  mixed-scope-b.eth role bitmap: 16
After the mined transaction:
  root role bitmap: 1
  mixed-scope-b.eth role bitmap: 0
Delegate's name-scoped role on unrelated-c.eth: 0
Delegate changes unrelated-c.eth address: zero address -> 0x4444444444444444444444444444444444444444
After the correct root revocation:
  root role bitmap: 0
  same delegate operation: reverted
```

The owner must approve the wallet request. That approval does not make the request equivalent to the action Portal presented: the UI does not disclose that the root assignment is omitted, and the encoded scope parameter addresses only one name.

## Impact Details

The selected published impact is:

> Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

This maps to `Modifying transaction arguments or parameters`. Portal asks the connected owner to remove the delegate from all roles represented by the row, but constructs a transaction whose resource argument is one individual name and omits the root-scoped revocation required to implement the confirmed action. Portal transforms an explicitly confirmed all-resource revocation into a transaction whose scope parameter targets only one name.

The consequence is security-sensitive rather than cosmetic. In the actual local EVM witness, the incorrectly scoped Portal transaction was mined, removed the name role, and left the delegate's global `ROLE_SET_ADDR` authority active. The delegate then used that retained root authority, without a role on the affected name, to change an unrelated name's address record. After the owner performed the correct root revocation, the identical delegate operation reverted and the record remained unchanged.

The following limitations are material:

* the delegate previously received legitimate authority;
* the owner initiates the removal and must approve the wallet request;
* the remaining root assignment may be rediscovered after refreshing the role view;
* the owner can recover by submitting a separate root-revocation transaction;
* the PoC executes only on a fresh local Anvil chain and does not contact or mutate a public chain; and
* no theft, irreversible loss, or public-production attack is claimed.

The report is limited to the connected-wallet scope mismatch, the mined local authorization state it produces, and the demonstrated ability of the retained delegate to modify another name until the omitted root revocation is completed.

## References

* [Add user models the empty name as the root resource](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L37-L39)
* [Add user separately selects the root resource or an individual name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L248-L284)
* [Role records are grouped by account and their role bits are unioned](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L175-L220)
* [The sidebar selects one non-root name for the grouped row](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L170)
* [The selected name and unioned roles form the removal action](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323)
* [All-role confirmation and transaction start](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L493)
* [The helper chooses root or name revocation from one name value](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)
* [Enhanced Access Control uses root-role fallback and separate root revocation](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/access-control/EnhancedAccessControl.sol#L19-L27)
* [Root roles are revoked through the dedicated root-resource function](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/access-control/EnhancedAccessControl.sol#L145-L166)
* [Permissioned Resolver name-role authorization](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L263-L289)

The current official Known Issues were reviewed. The closest entries concern smart-account session revocation (`R2-06`), transfer-related role combinations (`QA-01` and `WEB-1257`), and reused transaction IDs (`R3-07`). None describes account-only grouping of independent resolver resources, an all-role confirmation being converted into one name-scoped revocation, or the resulting retained root authority. The required repair is to preserve roles per resource and emit one revocation per original scope; the repairs described by those entries do not address this transformation.

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/6489c713fa22801d029a5da7d51cb317>

## Proof of Concept

This PoC mounts the production Portal role-removal flow, deploys the official contracts-v2 factory and Permissioned Resolver on fresh local Anvil, sends and mines Portal's exact calldata, proves the retained root authority through a real delegate record update, and proves the correct root-revocation control.

Repository and audited commit:

```
https://github.com/immunefi-team/audit-comp-ens.git
cda79acaad59711b943fc68207ebb3f1d0ff8596

Official upstream audited revision listed by Immunefi:
63772fd872af472ced58b009499355f3430c2a86

Authoritative Permissioned Resolver implementation used as the EVM oracle:
https://github.com/ensdomains/contracts-v2.git
8d1c89350729f87b3968cf41ef3d683951444a44
```

Environment used for the final runs:

```
Ubuntu under WSL
Node v22.16.0
pnpm v10.27.0 (repository-pinned)
Vitest v4.1.5
Foundry/Anvil v1.6.0-v1.7.0 (f83bad912a9dba7bf0371def1e70bb1896048356)
Solidity v0.8.27
```

## Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
git -C audit-comp-ens checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
cd audit-comp-ens
corepack enable
pnpm install --frozen-lockfile
cd ..

git clone https://github.com/ensdomains/contracts-v2.git
git -C contracts-v2 checkout 8d1c89350729f87b3968cf41ef3d683951444a44
git -C contracts-v2 submodule update --init --recursive
cd contracts-v2/contracts
forge build --use 0.8.27 --optimize true --optimizer-runs 1000 --evm-version prague --use-literal-content --build-info src/resolver/PermissionedResolver.sol lib/verifiable-factory/src/VerifiableFactory.sol
cd ../../audit-comp-ens
```

Download the exact PoC source from the following GitHub Gist:

[GitHub Gist — resolver-mixed-scope-remove-all.fresh-anvil.poc.test.tsx](https://gist.github.com/HiroyukiNaito/6489c713fa22801d029a5da7d51cb317)

Save it at:

```
apps/portal/src/features/resolver/components/resolver-mixed-scope-remove-all.fresh-anvil.poc.test.tsx
```

It can be downloaded directly from the repository root with:

```bash
curl -L \
  "https://gist.githubusercontent.com/HiroyukiNaito/6489c713fa22801d029a5da7d51cb317/raw/resolver-mixed-scope-remove-all.fresh-anvil.poc.test.tsx" \
  -o apps/portal/src/features/resolver/components/resolver-mixed-scope-remove-all.fresh-anvil.poc.test.tsx
```

## Reproduction steps

{% stepper %}
{% step %}

### Deploy the contracts and create a resolver

The test deploys the exact VerifiableFactory and Permissioned Resolver implementation and creates a resolver owned by the local victim account.
{% endstep %}

{% step %}

### Assign the delegate roles

It gives delegate D global `ROLE_SET_ADDR` and name-specific `ROLE_SET_TEXT` for `mixed-scope-b.eth` through real contract transactions.
{% endstep %}

{% step %}

### Mount the production Portal flow

It mounts the production Portal table, sidebar, confirmation dialog, transaction manager, EOA path, and Viem wallet request path.
{% endstep %}

{% step %}

### Confirm and submit the Portal removal transaction

It confirms the displayed `(root)` and name resources, accepts the exact “remove this user from all roles” dialog, forwards Portal's request unchanged to Anvil, and verifies the mined input byte-for-byte.
{% endstep %}

{% step %}

### Verify the retained root authority

It reads the authoritative post-state, then has D change the address record of unrelated name C using only the retained root role.
{% endstep %}

{% step %}

### Perform the correct root revocation

It performs the correct root revocation and proves that repeating D's operation reverts without changing C.
{% endstep %}

{% step %}

### Run scope controls

It runs root-only, name-only, and correct two-request mixed-scope controls.
{% endstep %}
{% endstepper %}

## Execution command

```bash
CONTRACTS_V2_ROOT="$(realpath ../contracts-v2)" corepack pnpm --dir apps/portal exec vitest run src/features/resolver/components/resolver-mixed-scope-remove-all.fresh-anvil.poc.test.tsx --reporter=verbose
```

Decisive output from two unchanged fresh-process runs:

```
Run 1: exit 0; Test Files 1 passed (1); Tests 1 passed (1); wall 8.72s
Run 2: exit 0; Test Files 1 passed (1); Tests 1 passed (1); wall 7.57s

Portal/submitted/mined calldata equality: true
Portal calldata SHA-256: 04b5117649592bc9a9d3f551bc77b35a6355cb0aff918ebf181891526067d9be
Buggy transaction receipt: success
State before: root=1, mixed-scope-b.eth=16
State after:  root=1, mixed-scope-b.eth=0

Retained-authority control:
  delegate name-C role: 0
  delegate root role: 1
  transaction receipt: success
  unrelated-c.eth address: 0x0000000000000000000000000000000000000000
                       -> 0x4444444444444444444444444444444444444444

Correct-root-revoke control:
  root after revoke: 0
  repeated delegate transaction: reverted
  unrelated-c.eth address: unchanged

Scope controls:
  root only: root=0, name=16
  name only: root=1, name=0
  correct mixed removal: two requests, root=0, name=0

External/non-loopback attempts: 0 in both runs
Public-chain reads: 0 in both runs
Public-chain writes: 0 in both runs
```

The test keeps the production resolver roles table and grouping, sidebar, confirmation dialog, transaction modal, resolver-role helper, transaction manager and state machine, EOA transport, Viem `sendTransaction`, final wallet request, official factory deployment, proxy creation, Permissioned Resolver authorization, mined transactions, and EVM state reads real.

The connected account and client hooks are supplied by the harness, and the wallet request is forwarded to a loopback Anvil provider rather than a third-party wallet. No real wallet signature, public RPC, deployed production service, public-chain transaction, or non-loopback request is used.
