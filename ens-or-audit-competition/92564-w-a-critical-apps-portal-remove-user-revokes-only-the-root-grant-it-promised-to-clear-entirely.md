# 92564 w a critical apps portal remove user revokes only the root grant it promised to clear entirely

## #92564 \[W\&A-Critical] apps/portal: Remove user revokes only the root grant it promised to clear entirely

**Submitted on Sep 10th 2026 at 17:25:24 UTC by @zaaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92564
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

Explorer's resolver Roles table shows one row per account, unioning every role that account holds across every resource on that resolver. Opening the row prints `Global roles (all names)`, and `Remove user` asks the administrator to confirm removing the user "from all roles". What the application then builds and asks the wallet to sign is exactly one revocation call, carrying exactly one scope. An account granted `Set Address` at the root and again on a single name is therefore only partly revoked: the administrator signs, the transaction succeeds, the sheet closes, and the account keeps a record-writing role. It uses it to point a name at an address of its choosing, and a second pass through the same `Remove user` flow submits byte-identical calldata and changes nothing. The competition Resources page names `63772fd872af472ced58b009499355f3430c2a86` as the audited revision; `cda79acaad59711b943fc68207ebb3f1d0ff8596` is the competition fork commit created from that revision — its commit message reads `fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86`, so both identifiers name the same tree. Every line cited below is read at `cda79aca`, and all four files cited below are byte-identical on the frozen branch tip `1c9b47f18fcddd2e864dfe385c4171061c9811ae` — `git diff cda79aca 1c9b47f1 --` over those paths is empty while the same diff over the whole tree reports four changed files, so the empty result is a fact about these paths rather than a broken command.

## Vulnerability Details

**Expected:** `Remove user` on an account revokes that account's roles on the root resource **and** on every name-scoped resource it holds, matching the dialog's promise.

**Actual:** one revocation call goes out, carrying one scope. At least one grant survives, the account keeps and uses the authority it confers, and running `Remove user` on that account a second time submits byte-identical calldata and clears nothing.

The invariant behind it is short. `AccountRoleGroup` represents an account's authorisations across **N** resources; `RemoveUserAction` carries exactly **one** `name`. The other resources cannot be named by the transaction that is built, so they survive — and that is why correcting the resource-to-name lookup would not fix the defect. It would only change which single resource the one call clears.

Which resource that is, in the environment this was reproduced in, is settled by the indexer, and the indexer is out of scope: the name-scoped resource the indexer reports does not match the resource the Portal computes, so `buildResourceToNameMap` finds no name, `roleName` falls back to `''`, and the call the flow builds is `revokeRootRoles`. That is branch selection, not the vulnerability. The vulnerability is that one call is built for N resources, and it is the same defect whichever resource the call happens to name.

`groupRolesByAccount` merges the resolver's role rows by account. Rows on different resources land in the same group, their decoded role bitmaps are concatenated into one `decodedRoles` array, and their resolved names are collected into a set (`apps/portal/src/lib/roles/resolverRoles.ts:193-215`). Nothing is lost in that grouping: both grants are present in the object the sheet renders.

One line into the sheet, that set collapses:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`. Everything downstream carries one name. The sheet's scope paragraph prints `Global roles (all names)` whenever the set contains `(root)`, with no mention of the other resources (`:391-397`, the literal string at `:394`). Its confirmation dialog reads, verbatim at `:445-448`:

```tsx
<DialogDescription>
  Are you sure you want to remove this user from all roles? This
  action cannot be undone.
</DialogDescription>
```

`handleRemoveUser` builds the pending action from that single name and the union of the roles (`:307-323`), and `onStart` passes both to the mutation (`:486-502`).

Only one resource can be expressed by the write path. `prepareRevokeResolverRolesTransaction` chooses `scope: 'root'` when the name is empty and `scope: 'name'` otherwise, and encodes one call either way — there is no loop over resources anywhere in the file:

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
)
const data = encodeFunctionData({ abi: writeParams.abi, functionName: writeParams.functionName, args: writeParams.args })
```

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-67`. Every run below took the `scope: 'root'` branch and submitted `revokeRootRoles`, leaving the name-scoped grant. Reading the other branch, an indexer that stored the resource whole would make the sheet emit `authorizeNameRoles(..., false)` and leave the **root** grant instead — a wider residue. That is a reading of these lines and not something this reproduction ran: no indexer reachable from here produces a matching resource, so the application never took that branch. `staging-graphql.ens.dev`, the app's default indexer, returns an empty `roles` set and `eacRolesChangeds` rows whose `resource` is null, so it offers nothing to compare either way.

**Missing control:** `handleRemoveUser` must build one revocation per resource in the group, not one per row, and the confirmation must name the scopes it covers. `AccountRoleGroup` already carries `roles`, the per-resource rows, so the resources are in hand at the point where they are discarded.

**Where:** `apps/portal/src/lib/roles/resolverRoles.ts:147-160, 193-215`; `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169, 307-323, 391-397, 445-448, 486-502`; `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-67`; `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:39, 186-206, 249-289`.

**Reachability.** That two-scope state is what the application's own `Add user` sheet produces. Its scope combobox offers `Root (all nodes)` — the empty string, `ResolverAddUserSheet.tsx:39` — alongside every node the resolver serves (`:249-289`), and `handleSubmit` grants against whichever is selected with no check for a grant the account already holds at another scope (`:186-206`). Two ordinary passes through that sheet, root then a name, build it, and a run that does exactly that is included below.

**Discriminating control:** two accounts go through the identical flow, in the same run, on the same resolver, with the same administrator wallet and the same dialog text. One variable differs — whether the account also holds a grant on a single name in addition to the root grant. Whichever account holds the extra grant keeps `Set Address` on that name after removal, writes to it, and the resolver returns the address it wrote. Its control, root grant only, is removed completely, disappears from the Roles table, and both of its post-removal writes revert. That pair has been run five times with a different pair of subject addresses each time, four on one Explorer instance and once on a second instance started from an independent checkout of the same commit, and the outcome was the same in each.

**Excluded-premise check:** the run needs no leaked key, no phishing, no wallet defect, no contract defect and no approval bypass. The residual authority is not an indexing artefact: it is read with `hasRoles` directly from the resolver contract and exercised by a transaction sent straight to that contract, and the single-call construction is visible in `revokeResolverRoles.ts` with no indexer in the path. Nothing is stubbed or intercepted anywhere in the reproduction: real Explorer, real Panoptes indexer, real Anvil Sepolia fork, real deployed `PermissionedResolver`.

**Relation to the published Known Issues.** R2-06 is the nearest entry — a revocation the UI reports as done that leaves on-chain authority alive — but it is the Rhinestone smart-account session key in `packages/smart-account`, where **no** on-chain revocation call is made at all and the permission simply expires. Here a revocation call is built, signed by the administrator and mined; it is narrower than the sheet's text. QA-01 and WEB-1257 concern role combinations that make later actions **error** after a name **transfer**; no transfer occurs here and nothing errors. Part 2 of that document states that a demonstrated security consequence of a role-combination defect — its own example is "a role combination that grants authority it should not, rather than merely erroring" — is a new finding.

## Impact Details

This matches the in-scope impact:

> Malicious interactions with an already-connected wallet, such as:
>
> * Modifying transaction arguments or parameters

The connected wallet is asked to sign a revocation whose arguments carry one scope while the screen that produced it states, in the same interaction, that the user is being removed from **all** roles. The administrator authorises the sentence and signs the parameters, and the two do not match. The narrowing is the application's own: it selects the single scope and hands the wallet those parameters.

**Attacker:** the account the administrator is deliberately removing. Nobody presses `Remove user` on an account they still trust — the ordinary case is a departing team member, a rotated key, or a delegate whose access is being withdrawn. It holds only the grants it was given, performs no action until after the administrator's transaction is mined, and needs no privilege it was not already handed.

**Victim:** the resolver administrator, who opens Roles, opens the account's row, reads `Global roles (all names)`, confirms a dialog that says the user is being removed from all roles, signs, and is shown a completed transaction.

**The measured result, as the run prints it.** Two accounts, one flow, one variable. Under test is an account holding `Set Address` at the root and again on `sable-seer-205.colonny.eth`; its control holds that role at the root only.

```
                              two-scope account                root-only control
roles before                  root=true  name=true             root=true  name=true *
scope paragraph in the sheet  Global roles (all names)         Global roles (all names)
confirmation dialog           remove this user from all roles  remove this user from all roles
transaction submitted         0xce156e82 revokeRootRoles(1,·)  0xce156e82 revokeRootRoles(1,·)
hasRootRoles(1, account)      false                            false
hasRoles(name, 1, account)    TRUE  — survived                 false — gone
still listed in Roles table   yes, scope column now empty      no
account setAddr on the name   SUCCEEDED                        reverted
addr(name, 60) reads back     0xdead00000000000000000000000000000000beef   0x  (empty)
account setAddr on any other  reverted                         reverted

* `hasRoles` on a name's resource returns true for a root holder, so both accounts read true
  before the removal. What separates them is the second grant, which only the test account has.
```

That account also stays in the Roles table after the removal, now with an empty scope column. That is visible to an administrator who goes back and looks, and it is the reason the next line matters: **the flow offers no way to finish the job.** A second pass through `Remove user` on the same account submits byte-identical calldata, the name-scoped grant is still held afterwards, and the account's next `setAddr` on the name succeeds again. Completing the revocation requires calling `authorizeNameRoles(dnsEncode(name), roles, account, false)` on the resolver outside the application.

**What the wrong arguments cost.** The administrator approves a transaction believing it ends the account's authority over the resolver's records. It ends that authority on one resource. On the resource that survives, `Set Address` is the role that decides what an ENS name points at, and the removed account exercised it: the resolver serving `sable-seer-205.colonny.eth` returns the address that account wrote. **Executed end-to-end:** yes — through the Explorer in a browser on an isolated Sepolia fork, from the `Add user` sheet that creates the state, through the `Remove user` dialog and the administrator's signature, to reading the name's address record back off the chain. Nothing was broadcast to a public network.

**Verified out of band:** role state before and after each removal came from `hasRootRoles` and `hasRoles` on the resolver contract, not from the application. The submitted calldata was read from the mined block by matching sender and recipient, and its selector decoded independently. The address record was read with `addr(namehash(name), 60)` on the resolver after the write was mined.

Three limits belong with that result. Repairing the residual grant takes one direct `authorizeNameRoles(..., false)` call from the resolver's root administrator, so this is not an irreversible loss — but it is not repairable inside the application, and an administrator who has just been told the user was removed from all roles has no reason to go looking. The demonstration ends at the resolver that serves the name's records rather than at a forward resolution walked from the registry, because on this fork `getResolver` returns null for the seeded names; what is shown is the record the resolver answers with, not a payment misdirected by it. And the state requires one account to hold grants at two scopes, which the `Add user` sheet builds in two ordinary passes with nothing in the flow warning against it, but which an administrator has to have done at some point — how often that happens in a live deployment is not something this reproduction can measure.

## References

* Audited commit, whose message maps it to the revision on the Resources page: <https://github.com/immunefi-team/audit-comp-ens/commit/cda79acaad59711b943fc68207ebb3f1d0ff8596>
* [`groupRolesByAccount` unions rows from different resources into one row](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L193-L215)
* [`buildResourceToNameMap` keys the lookup on `computeNameResource`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L143-L160)
* [`roleName` collapses the set of resources to one name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173)
* [The sheet prints `Global roles (all names)`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L397)
* [The dialog promises removal from all roles](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L466)
* [`handleRemoveUser` passes one name and the union of roles](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323)
* [`prepareRevokeResolverRolesTransaction` encodes exactly one call for one scope](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L75)
* [`Add user` offers root or a single node as the scope, with no check for an existing grant](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L186-L289)

### Proof of Concept

This reproduction runs in the Explorer itself: the browser loads `http://localhost:3003`, roles are managed from the resolver's Roles page, and every transaction is signed through the repository's own headless wallet fixture. There is no request interception, no GraphQL stub and no mock indexer — the page is answered by the Panoptes indexer that `pnpm e2e:infra:up` starts, and every assertion about authority is a direct contract read. Everything is local; nothing reaches a public network. `roles-sheet-and-dialog.png` is the browser capture of the sheet and dialog quoted in step 6, taken by `page.screenshot()` during that run.

**Prerequisites:** Docker, Node 22, pnpm 10.

1. **Owner/reviewer:** run `git clone`, `git checkout` and `pnpm install` to get the audited tree. The commit message of `cda79aca` reads `fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86`, the revision named on the Resources page.

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-resolver-repro
cd ens-resolver-repro
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
git log -1 --format=%s        # prints the line above
pnpm install --frozen-lockfile
```

2. **Owner/reviewer:** run `pnpm e2e:infra:up` to start the local Sepolia fork, the Panoptes indexer and the supporting services.

```sh
pnpm e2e:infra:up
```

If a service other than anvil and panoptes reports a missed health deadline, continue; this reproduction uses the chain, the indexer and the Explorer only.

3. **Owner/reviewer:** run `cp` and the funding script to apply the repository's own local environment, exactly as `.github/workflows/e2e.yml` does, then point the E2E project at port 3003.

```sh
cp apps/portal/.env.ci apps/portal/.env
cp e2e/.env.ci e2e/.env
sed -i 's|^PORTAL_APP_URL=.*|PORTAL_APP_URL=http://localhost:3003|' e2e/.env
bash e2e/infra/scripts/fund-account.sh 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

4. **Owner/reviewer:** run the Explorer on port 3003 and leave it running in this shell.

```sh
pnpm --filter portal exec vite --port 3003
```

Wait until `curl -sf http://localhost:3003 > /dev/null` succeeds, then use a second shell.

5. **Owner/reviewer:** run `cat` to add the reproduction spec.

```sh
cat > e2e/projects/portal/tests/resolver-remove-user.spec.ts <<'TS'
/**
 * Remove user on an account whose grants span two resources, with a root-only
 * control, plus the diagnostic that records why the sheet picked the root scope.
 *
 * For each account it records, in one table:
 *   - the resource the indexer reports for each of the account's role rows
 *   - computeNameResource(NAME), the app's own formula (lib/roles/resolverRoles.ts:150-160)
 *   - the resourceToNameMap lookup for each of those resources, built the way
 *     buildResourceToNameMap builds it, with the ROOT resource as a positive control
 *   - the scope the sheet displayed
 *   - the selector of the calldata the app generated
 *
 */
import { mkdirSync, writeFileSync } from 'node:fs'
import { resolve } from 'node:path'
import { type Address, encodeFunctionData, encodePacked, keccak256, parseAbi, toHex } from 'viem'
import { namehash } from 'viem/ens'
import { connectWithHeadlessWallet, expect, test } from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient, testClient, walletClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const APP = 'http://localhost:3003'
const RESOLVER = '0x9f449af3d099540e68e3ee32e033438ded568610' as Address
const ROOT_ADMIN = '0xa569696dbf9191441d045891aadcf47a919cbc1c' as Address
const ADMIN = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as Address
const TWO_SCOPE = '0x71bE63f3384f5fb98995898A86B02Fb2426c5788' as Address // anvil #12
const ROOT_ONLY = '0xFABB0ac9d68B0B445fB7357272Ff202C5651694a' as Address // anvil #13

const NAME = 'sable-seer-205.colonny.eth'
const OTHER = 'silver-market-204.colonny.eth'
const ATTACKER_ADDR = '0xdEaD00000000000000000000000000000000BEEf'
const ROOT_RESOURCE = `0x${'00'.repeat(32)}` as const
const ROLE_SET_ADDR = 1n
const ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111n
const RUNTIME = resolve(import.meta.dirname, 'runtime')

const abi = parseAbi([
  'function grantRootRoles(uint256 roleBitmap, address account)',
  'function revokeRootRoles(uint256 roleBitmap, address account)',
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function setAddr(bytes32 node, uint256 coinType, bytes value)',
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
])

const dnsEncode = (name: string) => {
  const out: number[] = []
  for (const l of name.split('.')) { const b = Buffer.from(l, 'utf8'); out.push(b.length, ...b) }
  out.push(0)
  return toHex(Uint8Array.from(out))
}
/** apps/portal/src/lib/roles/resolverRoles.ts computeNameResource */
const computeNameResource = (name: string) =>
  keccak256(encodePacked(['bytes32', 'bytes32'], [namehash(name), `0x${'00'.repeat(32)}`]))

/** Every transaction this harness sends itself, so the app's own transaction can
 *  never be confused with one of ours when scanning the chain for it. */
const harnessSent = new Set<string>()

const sendAs = async (from: Address, data: `0x${string}`) => {
  const hash = await walletClient.sendTransaction({ account: from, to: RESOLVER, data, chain: null })
  harnessSent.add(hash.toLowerCase())
  const r = await publicClient.waitForTransactionReceipt({ hash })
  if (r.status !== 'success') throw new Error(`reverted ${hash}`)
  return hash
}
const tryWrite = async (from: Address, data: `0x${string}`) => {
  await testClient.impersonateAccount({ address: from })
  await testClient.setBalance({ address: from, value: 10n ** 20n })
  try {
    const hash = await walletClient.sendTransaction({ account: from, to: RESOLVER, data, chain: null })
    harnessSent.add(hash.toLowerCase())
    const r = await publicClient.waitForTransactionReceipt({ hash })
    return { mined: true, succeeded: r.status === 'success', hash }
  } catch (e) {
    return { mined: false, succeeded: false, hash: null, error: String((e as Error).message).slice(0, 200) }
  } finally {
    await testClient.stopImpersonatingAccount({ address: from })
  }
}

test('remove user: one revocation call, one scope, the other grant survives', async ({ page, wallet }) => {
  mkdirSync(RUNTIME, { recursive: true })
  const nameRes = computeNameResource(NAME)
  const out: any = {
    startedAt: new Date().toISOString(), app: APP, resolver: RESOLVER, name: NAME,
    computeNameResource: nameRes, rootResource: ROOT_RESOURCE, accounts: {},
  }

  await testClient.impersonateAccount({ address: ROOT_ADMIN })
  await testClient.setBalance({ address: ROOT_ADMIN, value: 10n ** 20n })
  for (const a of [ADMIN, TWO_SCOPE, ROOT_ONLY]) await testClient.setBalance({ address: a, value: 10n ** 20n })
  await sendAs(ROOT_ADMIN, encodeFunctionData({ abi, functionName: 'grantRootRoles', args: [ALL_ROLES, ADMIN] }))
  await testClient.stopImpersonatingAccount({ address: ROOT_ADMIN })

  for (const s of [TWO_SCOPE, ROOT_ONLY]) {
    await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'revokeRootRoles', args: [ALL_ROLES, s] }))
    for (const n of [NAME, OTHER])
      await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'authorizeNameRoles', args: [dnsEncode(n), ALL_ROLES, s, false] }))
  }
  await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, '0x'] }))
  await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(OTHER), 60n, '0x'] }))

  // two-scope account: root + one name.  root-only account: the control.
  await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'grantRootRoles', args: [ROLE_SET_ADDR, TWO_SCOPE] }))
  await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'authorizeNameRoles', args: [dnsEncode(NAME), ROLE_SET_ADDR, TWO_SCOPE, true] }))
  await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'grantRootRoles', args: [ROLE_SET_ADDR, ROOT_ONLY] }))

  const gql = async (query: string, variables: any) => {
    const r = await fetch('http://127.0.0.1:5655/graphql', {
      method: 'POST', headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ query, variables }),
    })
    return (await r.json()) as any
  }
  const overview = async () =>
    (await gql('query($id:String!){resolver(id:$id){roles{account resource roleBitmap} nodes{name}}}', { id: RESOLVER.toLowerCase() })).data.resolver
  const rowsFor = async (acct: string) => (await overview()).roles.filter((x: any) => x.account.toLowerCase() === acct.toLowerCase())

  await expect.poll(async () => (await rowsFor(TWO_SCOPE)).length, { timeout: 120_000, intervals: [2000] }).toBeGreaterThanOrEqual(2)
  await expect.poll(async () => (await rowsFor(ROOT_ONLY)).length, { timeout: 120_000, intervals: [2000] }).toBeGreaterThanOrEqual(1)

  // buildResourceToNameMap, reproduced from the same node list the page receives
  const ov = await overview()
  const resourceToNameMap = new Map<string, string>()
  resourceToNameMap.set(ROOT_RESOURCE, '(root)')
  for (const n of ov.nodes as { name: string }[])
    if (n.name) resourceToNameMap.set(computeNameResource(n.name).toLowerCase(), n.name)
  out.resourceToNameMapSize = resourceToNameMap.size
  out.nodeCount = (ov.nodes as any[]).length
  // positive control for the lookup mechanism
  out.lookupControlRoot = resourceToNameMap.get(ROOT_RESOURCE) ?? null
  out.lookupOfComputedNameResource = resourceToNameMap.get(nameRes.toLowerCase()) ?? null

  await page.setViewportSize({ width: 1400, height: 1600 })
  await page.goto(`${APP}/resolver/${RESOLVER}/roles`)
  await connectWithHeadlessWallet(page, wallet)
  await expect(page.getByRole('heading', { name: 'Roles', exact: true })).toBeVisible({ timeout: 30_000 })

  const short = (a: Address) => `${a.slice(0, 6)}…${a.slice(-4)}`

  const removeVia = async (subject: Address, tag: string) => {
    await page.goto(`${APP}/resolver/${RESOLVER}/roles`)
    await expect(page.getByRole('heading', { name: 'Roles', exact: true })).toBeVisible({ timeout: 30_000 })
    const row = page.getByRole('row').filter({ hasText: short(subject) })
    await expect(row).toHaveCount(1, { timeout: 30_000 })
    const rowText = await row.innerText()
    await row.getByRole('button', { name: 'Edit user roles' }).click()
    await expect(page.getByRole('button', { name: 'Remove user' })).toBeVisible({ timeout: 15_000 })
    const globalLabel = await page.getByText('Global roles (all names)', { exact: true }).count()
    const scopedTo = await page.getByText(/^Roles scoped to /).allInnerTexts().catch(() => [])
    await page.screenshot({ path: resolve(RUNTIME, `sheet-${tag}.png`), fullPage: true })
    await page.getByRole('button', { name: 'Remove user' }).click()
    const confirmText = await page.getByRole('dialog').filter({ hasText: 'Remove user' }).innerText().catch(() => '')
    await page.waitForTimeout(1200)
    await page.screenshot({ path: resolve(RUNTIME, `confirm-${tag}.png`), fullPage: true })
    const fromBlock = await publicClient.getBlockNumber()
    await page.getByRole('button', { name: 'Remove', exact: true }).click()
    await page.getByRole('button', { name: 'Start', exact: true }).click({ timeout: 30_000 })
    const ow = page.getByRole('button', { name: 'Open wallet', exact: true })
    if (await ow.isVisible().catch(() => false)) await ow.click()
    await authorizeTransaction(wallet, 60_000)
    const findTx = async () => {
      const head = await publicClient.getBlockNumber()
      for (let n = fromBlock + 1n; n <= head; n++) {
        const b = await publicClient.getBlock({ blockNumber: n, includeTransactions: true })
        for (const t of b.transactions as any[])
          if (
            t.from?.toLowerCase() === ADMIN.toLowerCase() &&
            t.to?.toLowerCase() === RESOLVER.toLowerCase() &&
            !harnessSent.has((t.hash as string).toLowerCase())
          )
            return { hash: t.hash, data: t.input as string }
      }
      return null
    }
    await expect.poll(async () => Boolean(await findTx()), { timeout: 60_000, intervals: [2000] }).toBe(true)
    return { rowText, globalLabel, scopedTo, confirmText, tx: (await findTx())! }
  }

  const measure = async (subject: Address) => ({
    rootSetAddr: await publicClient.readContract({ address: RESOLVER, abi, functionName: 'hasRootRoles', args: [ROLE_SET_ADDR, subject] }),
    nameSetAddr: await publicClient.readContract({ address: RESOLVER, abi, functionName: 'hasRoles', args: [BigInt(nameRes), ROLE_SET_ADDR, subject] }),
  })

  for (const [tag, subject] of [['twoScope', TWO_SCOPE], ['rootOnly', ROOT_ONLY]] as const) {
    const indexerRows = await rowsFor(subject)
    const before = await measure(subject)
    const ui = await removeVia(subject, tag)
    await page.waitForTimeout(4000)
    const after = await measure(subject)
    const write = await tryWrite(subject, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, ATTACKER_ADDR] }))
    const resolved = await publicClient.readContract({ address: RESOLVER, abi, functionName: 'addr', args: [namehash(NAME), 60n] })
    const writeOther = await tryWrite(subject, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(OTHER), 60n, ATTACKER_ADDR] }))
    out.accounts[tag] = {
      subject,
      indexerRows: indexerRows.map((r: any) => ({
        resource: r.resource,
        equalsComputeNameResource: r.resource.toLowerCase() === nameRes.toLowerCase(),
        resourceToNameMapLookup: resourceToNameMap.get(r.resource.toLowerCase()) ?? null,
      })),
      scopeShownInSheet: ui.globalLabel ? 'Global roles (all names)' : (ui.scopedTo[0] ?? '(none)'),
      nameColumnInRow: ui.rowText,
      confirmationText: ui.confirmText,
      selector: ui.tx.data.slice(0, 10),
      calldata: ui.tx.data,
      txHash: ui.tx.hash,
      before, after,
      setAddrOnScopedName: write, addrReadsBack: resolved, setAddrOnUnrelatedName: writeOther,
    }
    if (write.succeeded) await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, '0x'] }))
    if (writeOther.succeeded) await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(OTHER), 60n, '0x'] }))
  }

  // Does a second pass through the same flow finish the job?
  {
    const second = await removeVia(TWO_SCOPE, 'twoScope-pass2')
    const after = await measure(TWO_SCOPE)
    const write = await tryWrite(TWO_SCOPE, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, ATTACKER_ADDR] }))
    const resolved = await publicClient.readContract({ address: RESOLVER, abi, functionName: 'addr', args: [namehash(NAME), 60n] })
    out.secondRemovalPass = {
      subject: TWO_SCOPE,
      calldata: second.tx.data,
      identicalToFirstPass: second.tx.data === out.accounts.twoScope.calldata,
      txHash: second.tx.hash, after,
      setAddrOnScopedName: write, addrReadsBack: resolved,
    }
    if (write.succeeded) await sendAs(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, '0x'] }))
  }

  out.finishedAt = new Date().toISOString()
  writeFileSync(resolve(RUNTIME, 'remove-user-evidence.json'), `${JSON.stringify(out, null, 2)}\n`)
  console.log(JSON.stringify(out, null, 2))
})
TS
```

6. **Resolver admin (the connected wallet `0xf39F…2266`), driven by the spec:** run `pnpm --filter @ens-apps/e2e test:portal` to execute it. It grants `Set Address` to a test account at the root scope and again on `sable-seer-205.colonny.eth`, and to a control account at the root scope only. For each it then opens the Roles table, clicks `Edit user roles`, reads the scope paragraph — `Global roles (all names)` — clicks `Remove user`, reads the confirmation dialog — `Are you sure you want to remove this user from all roles? This action cannot be undone.` — clicks `Remove`, `Start`, and signs.

```sh
pnpm --filter @ens-apps/e2e test:portal resolver-remove-user.spec.ts
```

7. **Attacker account (the account the admin just removed), driven by the spec:** after each removal is mined the spec sends `setAddr(namehash('sable-seer-205.colonny.eth'), 60, 0xdEaD…BEEf)` from that account to the resolver, reads `addr` back, and sends the same call for an unrelated name.

```sh
cat e2e/projects/portal/tests/runtime/remove-user-evidence.json
```

8. **Owner/reviewer:** read the output. One variable separates the two accounts — whether the account also holds a grant on a single name.

```
                              two-scope account                root-only control
roles before                  root=true  name=true             root=true  name=true *
scope paragraph in the sheet  Global roles (all names)         Global roles (all names)
confirmation dialog           remove this user from all roles  remove this user from all roles
transaction submitted         0xce156e82 revokeRootRoles(1,·)  0xce156e82 revokeRootRoles(1,·)
hasRootRoles(1, account)      false                            false
hasRoles(name, 1, account)    TRUE  — survived                 false — gone
still listed in Roles table   yes, scope column now empty      no
account setAddr on the name   SUCCEEDED                        reverted
addr(name, 60) reads back     0xdead00000000000000000000000000000000beef   0x  (empty)
account setAddr on any other  reverted                         reverted

* `hasRoles` on a name's resource returns true for a root holder, so both read true before the
  removal; what separates them is the second grant, which only the test account has.
```

9. **Resolver admin, driven by the spec:** the same spec then runs `Remove user` a second time on the two-scope account, through the identical Explorer flow. Calldata from that pass is compared byte for byte with the first.

```
identical to the first pass   true
hasRoles(name, 1, account)    true
account setAddr again         SUCCEEDED
addr(name, 60) reads back     0xdead00000000000000000000000000000000beef
```

10. **Owner/reviewer:** read the mapping block in the same file, which records why the scope the sheet chose was the root one. Because the shipped indexer stores the resource with its low four bytes zeroed, the lookup misses and `roleName` falls back to `''`. That block carries its own positive control — on the identical map the root resource and the computed name resource both resolve — so the miss is the four zeroed bytes and not a broken map. This decides which single resource the one call names; it is not the defect.

```
computeNameResource("sable-seer-205.colonny.eth")
  0x7699da6cf09c185a1adf1b448f858466d800587433b5e6426157c89886479f98
resource the indexer reports for that grant
  0x7699da6cf09c185a1adf1b448f858466d800587433b5e6426157c89800000000
                                                            ^^^^^^^^ low four bytes zeroed
resourceToNameMapLookup       null
lookup control, root resource "(root)"
lookup control, computed name "sable-seer-205.colonny.eth"
scope the sheet displayed     Global roles (all names)
selector the app generated    0xce156e82  revokeRootRoles
```

## Supporting evidence

**The pair is stable.** This two-account pair has been run five times, with a different pair of subject addresses each time and the same outcome in each: four on one Explorer instance and once on a second instance started from an independent checkout of the same commit, with the grants placed by direct contract calls in four of them and by the Explorer's own `Add user` sheet in the fifth. In every run the control was fully removed and its later writes reverted.

**The two-scope state is what the application's own sheet builds.** This second spec makes both grants by driving `Add user` — one pass with the scope combobox left on `Root (all nodes)`, one with a node selected — then removes through the same `Remove user` flow. No chain state is seeded for the grants under test and no operator configuration is involved.

```sh
cat > e2e/projects/portal/tests/resolver-add-then-remove.spec.ts <<'TS'
/**
 * the two-scope account with the two-resource state built ENTIRELY through the Explorer's own
 * "Add user" sheet — no seeded chain state, no direct contract call, no operator
 * configuration. The sheet's scope combobox offers "Root (all nodes)" and every
 * node the resolver serves, and nothing stops the same account being added twice.
 *
  * two-scope account : Add user @ Root, then Add user @ sable-seer-205.colonny.eth
 * root-only control : Add user @ Root only — identical flow, one variable
 *
 * Then: Remove user, measure, and let the subject write.
 */
import { mkdirSync, writeFileSync } from 'node:fs'
import { resolve } from 'node:path'
import { type Address, encodeFunctionData, encodePacked, keccak256, parseAbi } from 'viem'
import { namehash } from 'viem/ens'
import { connectWithHeadlessWallet, expect, test } from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient, testClient, walletClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const APP = 'http://localhost:3003'
const RESOLVER = '0x9f449af3d099540e68e3ee32e033438ded568610' as Address
const ADMIN = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as Address
const TWO_SCOPE = '0xC0FFEe0000000000000000000000000000000001' as Address
const ROOT_ONLY = '0xc0Ffee0000000000000000000000000000000002' as Address

const NAME = 'sable-seer-205.colonny.eth'
const OTHER = 'silver-market-204.colonny.eth'
const ATTACKER_ADDR = '0xdEaD00000000000000000000000000000000BEEf'
const RUNTIME = resolve(import.meta.dirname, 'runtime')

const abi = parseAbi([
  'function revokeRootRoles(uint256 roleBitmap, address account)',
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function setAddr(bytes32 node, uint256 coinType, bytes value)',
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
])
const appResource = (name: string) =>
  keccak256(encodePacked(['bytes32', 'bytes32'], [namehash(name), `0x${'00'.repeat(32)}`]))

const impersonatedWrite = async (from: Address, data: `0x${string}`) => {
  await testClient.impersonateAccount({ address: from })
  await testClient.setBalance({ address: from, value: 10n ** 20n })
  try {
    const hash = await walletClient.sendTransaction({ account: from, to: RESOLVER, data, chain: null })
    const r = await publicClient.waitForTransactionReceipt({ hash })
    return { mined: true, succeeded: r.status === 'success', hash }
  } catch (e) {
    return { mined: false, succeeded: false, hash: null, error: String((e as Error).message).slice(0, 200) }
  } finally {
    await testClient.stopImpersonatingAccount({ address: from })
  }
}

test('state built through Add user, then Remove user leaves one grant', async ({ page, wallet }) => {
  mkdirSync(RUNTIME, { recursive: true })
  const nameRes = appResource(NAME)
  const out: any = { startedAt: new Date().toISOString(), app: APP, resolver: RESOLVER, name: NAME, arms: {} }

  await page.setViewportSize({ width: 1400, height: 1600 })
  await testClient.setBalance({ address: ADMIN, value: 10n ** 20n })
  // Test cleanup only: strip any grant a previous aborted run left behind, so the
  // grants under test are the ones the Add user sheet makes below.
  const ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111n
  const dnsEncode = (name: string) => {
    const o: number[] = []
    for (const l of name.split('.')) { const b = Buffer.from(l, 'utf8'); o.push(b.length, ...b) }
    o.push(0)
    return `0x${Buffer.from(Uint8Array.from(o)).toString('hex')}` as `0x${string}`
  }
  for (const s of [TWO_SCOPE, ROOT_ONLY]) {
    await impersonatedWrite(ADMIN, encodeFunctionData({ abi, functionName: 'revokeRootRoles', args: [ALL_ROLES, s] }))
    await impersonatedWrite(ADMIN, encodeFunctionData({ abi, functionName: 'authorizeNameRoles', args: [dnsEncode(NAME), ALL_ROLES, s, false] }))
  }
  await page.goto(`${APP}/resolver/${RESOLVER}/roles`)
  await connectWithHeadlessWallet(page, wallet)
  await expect(page.getByRole('heading', { name: 'Roles', exact: true })).toBeVisible({ timeout: 30_000 })

  const short = (a: Address) => `${a.slice(0, 6)}…${a.slice(-4)}`

  /** One pass through the Explorer's "Add user" sheet. scope='' means Root. */
  const addUser = async (subject: Address, scope: string, tag: string) => {
    await page.goto(`${APP}/resolver/${RESOLVER}/roles`)
    await expect(page.getByRole('heading', { name: 'Roles', exact: true })).toBeVisible({ timeout: 30_000 })
    await page.getByRole('button', { name: 'Add user' }).click()
    await expect(page.getByRole('heading', { name: 'Add user', exact: true })).toBeVisible({ timeout: 15_000 })
    await page.locator('#user').fill(subject)
    const combo = page.getByPlaceholder('Root (all nodes)')
    if (scope !== '') {
      // The combobox lists every node the resolver serves and does not filter as
      // you type, so the option is picked with the keyboard: first ArrowDown
      // highlights "Root (all nodes)", the second the first node in the list.
      await combo.click()
      await page.waitForTimeout(1200)
      const options = await page.getByRole('option').allInnerTexts()
      const idx = options.findIndex((o) => o.startsWith(scope))
      expect(idx, `option for ${scope} present`).toBeGreaterThan(0)
      // the popup opens with the first option already highlighted
      for (let i = 1; i <= idx; i++) await combo.press('ArrowDown')
      await combo.press('Enter')
      await page.waitForTimeout(500)
    }
    await page.locator('#add-ROLE_SET_ADDR-manager').click()
    await page.screenshot({ path: resolve(RUNTIME, `ui-${tag}-add-${scope === '' ? 'root' : 'name'}.png`), fullPage: true })
    const fromBlock = await publicClient.getBlockNumber()
    // Submit the Add-user form the way a keyboard user does: Enter in the text
    // input. The Save button is the form's only submit button, and this avoids
    // the router devtools badge that overlays the bottom-right of the viewport.
    await expect(page.getByRole('button', { name: 'Save', exact: true })).toBeEnabled({ timeout: 15_000 })
    await page.locator('#user').press('Enter')
    await page.getByRole('button', { name: 'Start', exact: true }).click({ timeout: 30_000 })
    const ow = page.getByRole('button', { name: 'Open wallet', exact: true })
    if (await ow.isVisible().catch(() => false)) await ow.click()
    await authorizeTransaction(wallet, 60_000)
    const findTx = async () => {
      const head = await publicClient.getBlockNumber()
      for (let n = fromBlock + 1n; n <= head; n++) {
        const b = await publicClient.getBlock({ blockNumber: n, includeTransactions: true })
        for (const t of b.transactions as any[])
          if (t.from?.toLowerCase() === ADMIN.toLowerCase() && t.to?.toLowerCase() === RESOLVER.toLowerCase())
            return { hash: t.hash, data: t.input as string }
      }
      return null
    }
    await expect.poll(async () => Boolean(await findTx()), { timeout: 60_000, intervals: [2000] }).toBe(true)
    const tx = (await findTx())!
    return { selector: tx.data.slice(0, 10), data: tx.data, hash: tx.hash }
  }

  const removeUser = async (subject: Address, tag: string) => {
    await page.goto(`${APP}/resolver/${RESOLVER}/roles`)
    await expect(page.getByRole('heading', { name: 'Roles', exact: true })).toBeVisible({ timeout: 30_000 })
    const row = page.getByRole('row').filter({ hasText: short(subject) })
    await expect(row).toHaveCount(1, { timeout: 30_000 })
    const rowText = await row.innerText()
    await row.getByRole('button', { name: 'Edit user roles' }).click()
    await expect(page.getByRole('button', { name: 'Remove user' })).toBeVisible({ timeout: 15_000 })
    const globalLabel = await page.getByText('Global roles (all names)', { exact: true }).count()
    const scopedTo = await page.getByText(/^Roles scoped to /).allInnerTexts().catch(() => [])
    await page.screenshot({ path: resolve(RUNTIME, `ui-${tag}-sidebar.png`), fullPage: true })
    await page.getByRole('button', { name: 'Remove user' }).click()
    const confirmText = await page.getByRole('dialog').filter({ hasText: 'Remove user' }).innerText().catch(() => '')
    await page.screenshot({ path: resolve(RUNTIME, `ui-${tag}-confirm.png`), fullPage: true })
    const fromBlock = await publicClient.getBlockNumber()
    await page.getByRole('button', { name: 'Remove', exact: true }).click()
    await page.getByRole('button', { name: 'Start', exact: true }).click({ timeout: 30_000 })
    const ow = page.getByRole('button', { name: 'Open wallet', exact: true })
    if (await ow.isVisible().catch(() => false)) await ow.click()
    await authorizeTransaction(wallet, 60_000)
    const findTx = async () => {
      const head = await publicClient.getBlockNumber()
      for (let n = fromBlock + 1n; n <= head; n++) {
        const b = await publicClient.getBlock({ blockNumber: n, includeTransactions: true })
        for (const t of b.transactions as any[])
          if (t.from?.toLowerCase() === ADMIN.toLowerCase() && t.to?.toLowerCase() === RESOLVER.toLowerCase())
            return { hash: t.hash, data: t.input as string }
      }
      return null
    }
    await expect.poll(async () => Boolean(await findTx()), { timeout: 60_000, intervals: [2000] }).toBe(true)
    const tx = (await findTx())!
    await page.waitForTimeout(2500)
    await page.screenshot({ path: resolve(RUNTIME, `ui-${tag}-submitted.png`), fullPage: true })
    return { rowText, globalLabel, scopedTo, confirmText, tx }
  }

  const measure = async (subject: Address) => ({
    rootSetAddr: await publicClient.readContract({ address: RESOLVER, abi, functionName: 'hasRootRoles', args: [1n, subject] }),
    nameSetAddr: await publicClient.readContract({ address: RESOLVER, abi, functionName: 'hasRoles', args: [BigInt(nameRes), 1n, subject] }),
  })
  const indexerRows = async (acct: string) => {
    const r = await fetch('http://127.0.0.1:5655/graphql', {
      method: 'POST', headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ query: 'query($id:String!){resolver(id:$id){roles{account resource roleBitmap}}}', variables: { id: RESOLVER.toLowerCase() } }),
    })
    const j: any = await r.json()
    return ((j.data?.resolver?.roles ?? []) as any[]).filter((x) => x.account.toLowerCase() === acct.toLowerCase())
  }

  for (const [tag, subject, scopes] of [
    ['twoScope', TWO_SCOPE, ['', NAME]],
    ['rootOnly', ROOT_ONLY, ['']],
  ] as const) {
    const grants: any[] = []
    for (const s of scopes) grants.push({ scope: s === '' ? '(root)' : s, ...(await addUser(subject, s, tag)) })
    await expect.poll(async () => (await indexerRows(subject)).length, { timeout: 120_000, intervals: [2000] }).toBeGreaterThanOrEqual(scopes.length)
    const before = await measure(subject)
    if (scopes.length === 2) expect(before, 'two-resource state built through the sheet').toEqual({ rootSetAddr: true, nameSetAddr: true })
    const ui = await removeUser(subject, tag)
    await page.waitForTimeout(4000)
    const after = await measure(subject)
    const write = await impersonatedWrite(subject, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, ATTACKER_ADDR] }))
    const resolved = await publicClient.readContract({ address: RESOLVER, abi, functionName: 'addr', args: [namehash(NAME), 60n] })
    const writeOther = await impersonatedWrite(subject, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(OTHER), 60n, ATTACKER_ADDR] }))
    out.arms[tag] = {
      subject, grantsMadeThroughAddUserSheet: grants, before,
      ui: { rowText: ui.rowText, globalLabelShown: ui.globalLabel, scopedToText: ui.scopedTo, confirmationText: ui.confirmText },
      submittedSelector: ui.tx.data.slice(0, 10), submittedData: ui.tx.data, submittedHash: ui.tx.hash,
      after, indexerRowsAfter: await indexerRows(subject),
      writeOnScopedName: write, nameResolvesTo: resolved, writeOnOtherName: writeOther,
    }
    if (write.succeeded) await impersonatedWrite(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(NAME), 60n, '0x'] }))
    if (writeOther.succeeded) await impersonatedWrite(ADMIN, encodeFunctionData({ abi, functionName: 'setAddr', args: [namehash(OTHER), 60n, '0x'] }))
  }

  out.finishedAt = new Date().toISOString()
  writeFileSync(resolve(RUNTIME, 'add-then-remove-evidence.json'), `${JSON.stringify(out, null, 2)}\n`)
  console.log(JSON.stringify(out, null, 2))
})
TS
pnpm --filter @ens-apps/e2e test:portal resolver-add-then-remove.spec.ts
cat e2e/projects/portal/tests/runtime/add-then-remove-evidence.json
```

Both of the admin's grants come from the sheet — `grantRootRoles` (`0x072d5d77`) for the root pass, `authorizeNameRoles` (`0xbbd9abb5`) for the node pass — the removal submits a single `revokeRootRoles` (`0xce156e82`), and afterwards `hasRoles` is still true on the node's resource. That account's `setAddr` on the name succeeds and `addr` reads back `0xdead…beef`; the control account, added only at the root scope, has the identical write revert and leaves the record empty.
