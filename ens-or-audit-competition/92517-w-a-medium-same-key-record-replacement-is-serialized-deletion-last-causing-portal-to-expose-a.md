# 92517 w a medium same key record replacement is serialized deletion last causing portal to expose a revoked delegate s address as the base recipient

**Submitted on Sep 10th 2026 at 12:14:20 UTC by @kyan for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92517
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Portal serializes a same-key Base address replacement as `set replacement` followed by `clear`, so the replacement is immediately erased. On PermissionedResolver, the missing Base record then falls back to a historical Default address previously set by a resolver delegate who has since been revoked. Portal displays and copies that former delegate's address as the Base recipient, causing a user who relies on the displayed value to send funds to an unintended recipient.

## Vulnerability Details

The assessed asset is `apps/portal` at competition revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The vulnerable flow spans these production paths:

* `apps/portal/src/features/records/hooks/useEditRecordsState.ts` retains deletion of the existing Base record and addition of its replacement as separate pending changes.
* `apps/portal/src/features/records/helpers/transformPendingChanges.ts` serializes additions before deletions and does not coalesce operations that refer to the same `(name, coinType)` key.
* `apps/portal/src/features/records/utils/saveRecords.ts` forwards that ordered list to the resolver write.
* `apps/portal/src/features/profile/components/CoinRecord.tsx` exposes the resolved value through Portal's copy-recipient action.

For one logical Base address key, the exact production transformer returns:

```ts
[
  { coin: BASE_COIN, value: replacementC },
  { coin: BASE_COIN, value: '' },
]
```

The resulting owner-signed resolver operations are therefore:

```
setAddr(Base, replacement C)
setAddr(Base, empty)
```

The second call erases the value written by the first. PermissionedResolver then applies its intended fallback behavior and resolves Base to the retained Default address A.

The demonstrated security boundary is:

1. A resolver delegate legitimately sets Default=A while authorized.
2. The owner revokes the delegate; the PoC confirms that the delegate's attempted post-revocation Default and Base writes both fail.
3. The owner has an explicit Base=B record, which initially shadows Default=A.
4. Through Portal, the owner deletes Base=B, adds Base=C, and signs the generated save transaction.
5. Portal's deletion-last ordering removes C, resolution falls back to A, and Portal displays and copies A as the Base recipient.

This is not a resolver authorization bypass: the revoked delegate performs no post-revocation write. The vulnerability is that Portal constructs a destructive owner-signed batch which makes the delegate's historical value operational again. The cause-changing control reverses only the two same-key operations; clearing first and setting C second leaves C as the resolved and copied recipient. Inline editing, which emits one setter, is also safe.

PermissionedResolver fallback persistence is intentional. The defect is Portal's failure to reduce same-key pending changes to one final state before constructing calldata. The current published known issues and the closest public issue/PR searches do not describe this pending-record identity and deletion-last serialization failure.

Recommended remediation is to canonicalize pending changes by `(name, coinType)` and emit exactly one final operation for each logical record. Alternatively, deletion must precede replacement. The final confirmation should display the resulting record state, and regression tests should cover delete/re-add of the same key, inline replacement, different-key changes, fallback behavior, and a revoked historical delegate.

## Impact Details

This maps directly to the selected in-scope impact: **causing the application to display or pre-fill an incorrect address so that a user sends funds to an unintended recipient**.

After the owner signs what appears to be a Base address replacement, Portal displays and copies A—the revoked former delegate's address—instead of intended replacement C. A payer relying on Portal can consequently transfer funds to A. The deployed-code PoC demonstrates a `0.01 ETH` payment on a pinned Base Sepolia fork using only the address copied from Portal. In the order-only control, Portal copies C, C receives `0.01 ETH`, and A receives zero.

The demonstrated lower-bound loss is therefore `0.01 ETH`. The realistic per-occurrence loss is the amount a user independently chooses to send after relying on the incorrect Portal recipient; this report does not claim autonomous theft, unlimited loss, or a protocol-wide TVL amount. Exploitation requires the historical delegate state, an owner performing the ordinary same-key delete/re-add flow, and a payer subsequently relying on the displayed or copied address. The former delegate does not need an active role or a post-revocation transaction.

## References

* Affected asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Pending-change state: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useEditRecordsState.ts>
* Vulnerable transformer: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/transformPendingChanges.ts>
* Resolver write path: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/utils/saveRecords.ts>
* Recipient component: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/CoinRecord.tsx>

## Link to Proof of Concept

<https://gist.github.com/erenyeager1483/e91da3a1746c38028c33932063023334>

## Proof of Concept

## 3. Proof of Concept

The attached secret Gist contains the supporting runnable source, locked dependency files, placement map, clean PASS log, state/receipt JSON, mounted production-component HTML, and fidelity attestation. The written steps below are the PoC; the Gist supplies the supporting files rather than replacing these instructions.

1. Check out competition revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
2. Follow `poc-README.md` to recreate the flat Gist files and start localhost-only forks:
   * Sepolia block `11549180`, hash `0xf122ea293f1371712bd27a5253f03dfd14884c931f7120832e8d2d9cd514ed79`.
   * Base Sepolia block `46631358`, hash `0x63a9fa4cde8381616e8175cf888028d202af5fef9de725426f4c6ce162030f8a`.
3. Run the two Vitest files with the frozen target mounted read-only.
4. Observe four passing tests and the following state chain:
   * the former delegate's post-revocation Default and Base writes fail;
   * Base=B initially shadows Default=A;
   * exact Portal serialization emits Base=C followed by Base=empty;
   * deployed resolver resolution returns A;
   * exact `CoinRecord -> EntityActionCopy` displays and copies A;
   * a Base Sepolia fork payer using only that copied value credits A by `0.01 ETH`.
5. Run the order-only control. Clearing Base before setting C makes Portal copy C; C receives `0.01 ETH` and A receives zero.

The preserved `R2-final-clean-20260910.raw.log` records two files and four tests passing. `resolver-consumer-base-fork-upgrade.json` contains the receipts, role checks, resolver output, copied value, payment deltas, recovery, and cause-changing control. `mechanism-unit-control.json` contains safe controls.

No `mockCall`, storage/code write, fabricated API/indexer response, database insertion, impersonation, forced outcome, or public transaction supplies a necessary condition. Fresh accounts, registration/proxy setup, local funding, commitment-age advancement, and clipboard observation are non-causal fixtures.

The PoC does not claim a complete deployed remote-indexer/editor/profile route or an extension wallet, and it uses fresh fork fixture state rather than an existing user's public name. The load-bearing reducer, transformer, deployed resolver, production recipient component, and Base Sepolia payment execute.
