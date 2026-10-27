# 92465 w a critical owner controller role confusion in the explorer s address resolution fallback resolveensowner leads to irreversible transfer of the user s ens name to an unintended recipient

**Submitted on Sep 10th 2026 at 01:44:44 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92465
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Type an ENS name into the Explorer's transfer Recipient field and, if that name sets no ETH address record, the app silently substitutes the name's stale **ENS v1 registry controller** and then executes `safeTransferFrom` of your name to it. In ENS v1 the registry controller and the ERC-721 registrant are two different roles, and after a marketplace sale the controller stays with the seller until the buyer calls `reclaim()`, which no marketplace does for them.

The transfer form tells the sender *"Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding."* A sender who does exactly that - opens the recipient name in the same app, is shown the substituted address under the labels **"Owner"** and **"Manager"**. The app's own code, in three other files, says that address is *not* the owner: `useMigrationStatus.ts:35` calls the registry `owner` *"only a last-resort fallback"* behind the registrant, and `MigrateRegistryPrompt.tsx:17` says the holder is *"wrappedOwner/registrant, NOT the registry controller"*. The resolver behind the Recipient field takes the branch those files reject, and never reads the registrant at all.

## Vulnerability Details

### 1. In ENS v1 a name has two owners, and the app knows it

The **registrant** holds the ERC-721 on the BaseRegistrar. The **controller** is the address in the ENS Registry, returned by `registry.owner(node)`. Moving the NFT does not touch the registry, so after a sale the seller stays controller until the buyer calls `reclaim()`. This is the default outcome of buying a name, and ENS's own migration code models it: `classifyNames.ts:162-167` lifts `registry.owner` into `managerAddress` precisely when it differs from the registrant.

### 2. The substitution

`resolveAddressOrName` (`apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`) resolves a typed name through the UniversalResolver. When the name has no `addr(60)` record that returns `null`, and `:29-34` substitutes `getEnsOwner` instead:

```ts
    let resolvedAddress = resolved

    // Fallback for names that do not set an address record:
    // use current ENS owner address so the role can still be granted.
    if (!resolvedAddress) {
      const ownerResult = await getEnsOwner({ name: nameOrAddress })
      if (ownerResult.isOk()) {
        resolvedAddress = ownerResult.value?.owner ?? null
      }
    }
```

`getEnsOwner` reaches `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-101`, whose v1 branch returns the registry controller:

```ts
  const v1Owner = await getOwnerV1(client, { name })
  if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
    return {
      owner: v1Owner.owner,
      registryAddress: v1EthRegistry,
      protocolVersion: 'ENSv1',
    }
  }
```

`getOwnerV1` returns **both** roles — ensjs documents `owner` as *"the controller of the records"* and `registrant` as the NFT holder. `ResolvedEnsOwner` (`resolveEnsOwner.ts:20-24`) has no `registrant` field, so the correct answer is fetched and discarded on the same line.

### 3. This is the part that makes it a defect and not a documented convenience

The app answers "who holds this name?" in two places, with **opposite precedence**, and states in its own words which one is wrong.

`apps/portal/src/features/migration/hooks/useMigrationStatus.ts:31-36` — the docblock:

> "migratability is then evaluated against the name's own V1 token holder — wrappedOwner while the wrap is live, otherwise the Base Registrar registrant (**the registry `owner` is the controller, which may be a different address, so it is only a last-resort fallback**)."

`useMigrationStatus.ts:77-81` — and it implements exactly that:

```ts
  const holderCandidate =
    address ??
    domain.wrappedOwner?.id ??
    domain.registrant?.id ??
    domain.owner?.id
```

`apps/portal/src/features/registry/components/v2/MigrateRegistryPrompt.tsx:16-18`:

> "evaluated against the name's own V1 token holder (**wrappedOwner/registrant — NOT the registry controller**)"

`apps/portal/src/routes/addr/$addr/names.tsx:185-186` — the names list, which assigns the two roles to two different columns:

```ts
          // For unwrapped names: registrant is Owner, registry owner is Manager
          owner: relation.registrant || relation.wrappedOwner,
          manager: relation.owner || relation.wrappedOwner,
```

`resolveEnsOwner.ts:95-101` is the one path that inverts this: it returns the value the other three call the *Manager* / *last-resort fallback*, and it is the path wired to the Recipient field and to `safeTransferFrom`.

### 4. It reaches every name/address field in the app

`apps/portal/src/features/address/queries/getResolvedAddress.ts:26-31` rewraps `resolveAddressOrName` with none of the roles-sheet scoping the comment at `addUser.handlers.ts:27` was written for, and `apps/portal/src/features/address/hooks/useAddressResolution.ts:43-45` states its own reach:

> "Shared by every "name or address" input in the app (transfer recipient, add-user sheets, …)."

`grep -rn "useAddressResolution(" apps/portal/src` returns five call sites and `apps/manager/src` returns none:

* `features/transfer/components/SendNameForm.tsx:81` — the transfer **recipient** (this report)
* `routes/$name/create-subname.tsx:109` — the new subname's **owner** (see *Blast radius*)
* `features/roles/components/RolesAddUserSheet.tsx:73`
* `features/resolver/components/ResolverAddUserSheet.tsx:141`
* `features/registry/components/v2/RegistryAddUserSheet.tsx:66`

### 5. What the sender is shown

`SendNameForm.tsx:128-134` passes a `resolvedContent` override whose `RecipientPreview` (`:253-284`) renders the address with `truncate={false}` **plus the resolved address's own primary name**. That primary name is the only thing on the screen that could contradict the substitution, and `:271-275` drops it when there is none:

```tsx
        {match(primaryName)
          .with(P.string.minLength(1), (value) => (
            <CopyableRecord value={value} textClassName="text-foreground" />
          ))
          .otherwise(() => null)}
```

The substituted controllers do not have one. Measured on the fork for all three qualifying names (`evidence-mined/probes/reverse.mjs`):

| name           | substituted controller                       | its primary name |
| -------------- | -------------------------------------------- | ---------------- |
| `tkn.eth`      | `0x741d627F48C2Db692C59679634e0e3131E489d08` | `null`           |
| `prime.eth`    | `0x3a872f8FED4421E7d5BE5c98Ab5Ea0e0245169A0` | `null`           |
| `kowalski.eth` | `0xC1FF759da5f4f33a63007b668c60074a2a71D56B` | `null`           |

So the card renders a bare hex address and nothing else. `evidence-mined/transfer/T1-transfer-form.png` is the shipped form after typing `tkn.eth`. The harness prints every non-empty line of `<main>`; this is all of it, verbatim from `evidence-mined/transfer/run.log`:

```
[app] --- what the transfer form says about the recipient ---
[app]   Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026.
[app]   Dismiss
[app]   Back
[app]   Transfer ownership
[app]   Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding.
[app]   Recipient
[app]   0x741d627F48C2Db692C59679634e0e3131E489d08
[app]   Detach the resolver
[app]   Detaches this name’s resolver so it stops resolving to your records entirely. The recipient starts clean and sets up their own.
[app]   Transfer name
[app] --- end ---
[app] form mentions the stale controller in full: true
[app] form mentions the registrant at all:       false
```

### 6. The app satisfies its own verification demand against the wrong value

The warning says to check the recipient address. The trigger for the whole defect is *"the name has no ETH address record"* — so in every other ENS client `tkn.eth` resolves to nothing, and there is no second source to check against. Inside this app there is one, and it agrees with the substitution.

`apps/portal/src/routes/$name/ownership/index.tsx:128-132` renders `resolveEnsOwner`'s output under the label `Owner`:

```tsx
        <Owner
          asRow
          label={grace.isInGrace ? 'Previous owner' : 'Owner'}
          owner={data.owner}
        />
```

and `apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:53-58` reads the same registry slot directly and renders it at `:90` under `label="Manager"`:

```ts
  } = useReadContract({
    abi,
    functionName: 'owner',
    args: [namehash(name)],
    address: ensRegistryAddress,
  })
```

Loaded live in the shipped app (`evidence-mined/corroborate/C-ownership.png`, `C-overview.png`, text in `corroborate/run.log`):

```
=== http://localhost:5199/tkn.eth/ownership ===
   Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026.
   Dismiss
   Ownership
   Expires
   Feb 2, 2027 7:21pm
   Registered
   Feb 3, 2024 7:21pm
   Owner
   Address
   Copy
   0x741d…9d08
   Manager
   Address
   Copy
   0x741d…9d08
   Parent
   Name
   Copy
   eth
```

That is the head of the page's text, verbatim and unelided; the HISTORY table follows it, and the whole capture - both routes is in `corroborate/run.log`.

`0x3cDB9bC0…`, the address that actually holds `tkn.eth`, is **not shown under any ownership label** on either route. (It does appear lower down in the History table's "From" column, as the sender of the 2024 registration transactions — stated for precision; it is never presented as the name's owner or manager.)

### 7. From the field to the mined write

`SendNameForm.tsx:81` holds the resolution; `:106-109` hands `recipient` to `startTransfer`, which runs the plan from `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-43` and ends in `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71` — a bare `erc1155Abi.safeTransferFrom`, with no receiver validation anywhere in `apps/portal/src/features/transfer/` (`grep` for `getCode`, `onERC1155`, `supportsInterface`, `isContract`, `bytecode` returns nothing). The only downstream guards are `SendNameForm.tsx:87-89`, which reject the self and zero addresses. Neither tests where the address came from.

`buildTransferPlan.ts:1-4` states the ordering, and the detach options default to on (`SendNameForm.tsx:69-72`):

> "Config steps run before the token moves, since the sender loses the roles to make them once the ERC-1155 token transfers. The token transfer is last."

## Impact Details

user sending an ENS v2 name to a recipient identified by an ENS name that sets no ETH address record irreversibly loses that name to a third party: the Explorer silently substitutes the typed name's stale ENS v1 registry controller (after an ordinary sale, the seller) for its actual holder, pre-fills it as the recipient, corroborates it under its own "Owner" and "Manager" labels, and `safeTransferFrom`s to it after which both recovery paths, the victim re-transferring the token and calling unregister are irreversible.

**Impact row** *Critical*: "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow"

The second bullet is this bug. The flow is the transfer flow, the recipient the sender designated is the holder of the typed name, and the app sent the token to a different address. Mined, with the balance delta above.

**Second row** *Critical*: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves"

Both halves of the governing clause hold. The app pre-filled an incorrect address, the stale v1 controller its own three other files call the Manager and not the owner, and the user submitted a transaction to an unintended recipient. The bullet's word *attacker-controlled* does not gate the clause, and I do not claim it.

Holding the controller role on `tkn.eth` grants no authority over the sender's v2 name. `setResolver` and `setAddr` are gated on `registry.owner(node)` for that node alone. The token this moves sits on a different registry, and it does not come back.

## Blast radius with same primitive

`routes/$name/create-subname.tsx:109` runs the same hook for the new subname's **owner**, and passes no `resolvedContent` override, so it prints the shared truncated status line from `AddressNameInput.tsx:60-69` `Resolved: 0x741d…9d08`, eight hex characters, byte-identical in wording, colour and position to a genuine `addr(60)` resolution. Driven the same way, the app mined `0xa10299576c520d85c5948fd6487e5d298b92683c48f5847447f22a345ce490ff`, registering `sub1788995329147.ownerfb-1788995323.eth` to the substituted controller with `DEFAULT_ROLE_BITMAP` (all roles) on registry `0x055767e795846b0a0022ee9b5de1cb48bb9e8197`.

**I claim no impact row for it,** because I measured the recovery and it is revocable: the parent name's owner can call `unregister(tokenId)` on their own subregistry and take the subname back in one transaction (`"parentOwnerCanUnregister": "SUCCEEDS"`). It is the same root cause on a materially weaker sink, and it is included so the fix covers it.

The three add-user sheets (`RolesAddUserSheet.tsx:73`, `ResolverAddUserSheet.tsx:141`, `RegistryAddUserSheet.tsx:66`) take the same truncated label into revocable role grants. Also no row claimed.

## References

**The substitution**

* `apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40` — `resolveAddressOrName`
* `apps/portal/src/features/roles/helpers/addUser.handlers.ts:27-34` — the fallback, and the comment scoping it to role grants
* `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-101` — the v1 branch returns the registry controller
* `apps/portal/src/utils/ens/resolveEnsOwner.ts:20-24` — `ResolvedEnsOwner` has no `registrant` field

**The app's own model, which this path inverts**

* `apps/portal/src/features/migration/hooks/useMigrationStatus.ts:31-36` — *"the registry `owner` is the controller … only a last-resort fallback"*
* `apps/portal/src/features/migration/hooks/useMigrationStatus.ts:77-81` — `wrappedOwner ?? registrant ?? owner`
* `apps/portal/src/features/registry/components/v2/MigrateRegistryPrompt.tsx:16-18` — *"wrappedOwner/registrant — NOT the registry controller"*
* `apps/portal/src/routes/addr/$addr/names.tsx:185-186` — *"registrant is Owner, registry owner is Manager"*
* `packages/migration/src/service/classifyNames.ts:162-167` — ENS's own `managerAddress`

**How it reached the recipient field**

* `apps/portal/src/features/address/queries/getResolvedAddress.ts:13-31` — the generic wrapper and its docblock
* `apps/portal/src/features/address/hooks/useAddressResolution.ts:43-45` — *"Shared by every "name or address" input in the app"*
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:81` — the recipient's resolution
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:128-134,253-284` — `RecipientPreview`, full address plus the resolved address's own primary name
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:87-89` — the only downstream guards: self and zero address

**What the app tells the sender when they check**

* `apps/portal/src/features/transfer/components/SendNameForm.tsx:113-117` — *"check the recipient address before proceeding"*
* `apps/portal/src/routes/$name/ownership/index.tsx:128-132` — the substituted value rendered as `Owner`
* `apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:53-58,90` — the same registry slot rendered as `Manager`

**From field to mined transaction**

* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:1-4,22-43` — the ordering, and why the token moves last
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:69-72` — detach options default to on
* `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71` — `safeTransferFrom`, no receiver check

**Why the contract-recipient variant is not a permanence claim**

* `contracts-v2` `contracts/src/registry/PermissionedRegistry.sol:465` — `_mint(owner, tokenId, 1, "")`
* `contracts-v2` `contracts/src/erc1155/ERC1155Singleton.sol:313-319`, `:224-245` — `_updateWithAcceptanceCheck`, not overridden anywhere in `src/`
* Deployed logic behind the Sepolia proxy `0xE2B9Ce5b…` is `UserRegistry` at `0x624a25d67B59D587752EbEc8DdeD8827dAe52050` (Sourcify `exact_match`), which overrides neither `register` nor `_mint`

## Proof of Concept

## Proof of Concept

ran end-to-end against a live application.

attack step by step:

## The attack, step by step

No attacker, no privileged access, no contract to deploy. Steps 1-3 are the state an ordinary name sale leaves on chain. Steps 4-9 are one honest user session in the shipped app.

**Setup — what a normal sale leaves behind**

1. Alice registers `tkn.eth` in ENS v1. She holds two different roles at once: the **registrant** (the ERC-721 on the BaseRegistrar) and the **controller** (the address in the ENS Registry).
2. Alice sells the name to Bob. The NFT moves to Bob. The ENS Registry is not touched so no marketplace calls `reclaim()` for the buyer, so **Alice is still the controller** while Bob holds the name. This is the normal outcome of buying a name, not a mistake by anyone.
3. `tkn.eth` sets no ETH address record, so `addr(60)` returns nothing.

**The loss**

4. Carol owns an ENS v2 name and wants to send it to Bob. She opens her name → **Ownership** → **Transfer**.
5. She types `tkn.eth` into **Recipient**. There is no address record, so instead of refusing, the app quietly substitutes `registry.owner(tkn.eth)` — **Alice's stale address** — and shows it as the resolved recipient. Nothing on the card says a substitution happened, and the address has no primary name to contradict it.
6. The form tells her *"Make sure you check the recipient address before proceeding."* She does. She opens `/tkn.eth` in the same app, and it prints Alice's address under **Owner** and again under **Manager**. Bob's address is not shown under any ownership label. The check the app demanded confirms the wrong address.
7. She clicks **Transfer name** and signs the prompts the app raises.
8. `safeTransferFrom` mines. **Carol's name now belongs to Alice** — the seller of an unrelated name, who did nothing. Bob receives nothing.
9. Carol cannot undo it. Re-transferring the token reverts (she is no longer the owner) and `unregister` reverts. Getting the name back depends entirely on Alice volunteering to send it.

```bash
# 1. Fork live Sepolia (the anvil image pinned in e2e/infra/docker-compose.yml)
docker run -d --name ofb-anvil -p 8545:8545 --entrypoint anvil \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  --fork-url <YOUR_SEPOLIA_RPC> --chain-id 11155111 --port 8545 --host 0.0.0.0

# 2. Point the Explorer at the fork and serve it. The repo's own /rpc proxy is
#    already wired to 127.0.0.1:8545 (apps/portal/vite.config.ts:20-26).
echo 'VITE_SEPOLIA_RPC_URL=/rpc' > apps/portal/.env.local
(cd apps/portal && npx vite --port 5199 --strictPort) &

# 3. Save the file below as e2e/projects/portal/tests/owner-fallback-transfer.spec.ts and run it
cd e2e
PORTAL_APP_URL=http://localhost:5199 ANVIL_RPC_URL=http://127.0.0.1:8545 OUT=hunt/out-transfer \
  npx playwright test --config projects/portal/playwright.config.ts owner-fallback-transfer
```

The test **passes** when the victim's name lands on the substituted address: it asserts `movedTo == 0x741d627F…`, `balanceOf(victim) == 0`, `balanceOf(substituted controller) == 1`, `balanceOf(tkn.eth's real registrant) == 0`. It prints every line quoted below to the console and takes three full-page screenshots. Wall clock is \~35 s of test plus the 61 s `MIN_COMMITMENT_AGE` wait the repo's `makeName` fixture needs to stage the victim's name.

Nothing in the file is specific to the victim's name: `makeName` mints a fresh 2LD to the connected Anvil account each run, so the only chain state the PoC relies on is `tkn.eth`'s, which is live Sepolia's and is inherited by the fork.

### The PoC — `owner-fallback-transfer.spec.ts`

```ts
/**
 * The irreversible leg.
 *
 * The Explorer's transfer form runs the same `useAddressResolution` hook as the
 * create-subname Owner field. Type a name with no `addr(60)` record and the
 * recipient becomes that name's stale v1 registry *controller*. Here the sink
 * is `safeTransferFrom` on the victim's existing v2 2LD — a token the victim
 * cannot get back.
 *
 * Everything harmful is performed by the shipped app through its own buttons
 * and its own TransactionModal, driven by the repo's own e2e helper. The
 * harness connects the wallet, types the recipient a user types, and authorizes
 * the prompts the app raises.
 */
import fs from 'node:fs'
import type { Web3ProviderBackend } from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import { createPublicClient, http, parseAbi } from 'viem'
import { sepolia } from 'viem/chains'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://localhost:5199'
const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const OUT = process.env.OUT ?? 'hunt/out-transfer'

/** The recipient the user types. Live Sepolia state, inherited by the fork. */
const TYPED_NAME = 'tkn.eth'
/** tkn.eth's stale v1 registry controller — a plain EOA, no primary name. */
const STALE_CONTROLLER = '0x741d627F48C2Db692C59679634e0e3131E489d08'
/** tkn.eth's BaseRegistrar registrant — the party the sender means. */
const REGISTRANT = '0x3cDB9bC080Efe321A797E7286d302B90227cc6De'

const fork = createPublicClient({
  chain: { ...sepolia, rpcUrls: { default: { http: [ANVIL_RPC_URL] } } },
  transport: http(ANVIL_RPC_URL),
})

const erc1155Abi = parseAbi([
  'function balanceOf(address,uint256) view returns (uint256)',
  'function ownerOf(uint256) view returns (address)',
  'function safeTransferFrom(address,address,uint256,uint256,bytes)',
])

async function driveTransactionsToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  successTxIds: string[],
  timeoutMs = 300_000,
): Promise<void> {
  const transactionDialog = page.locator('[data-slot="dialog-content"]')
  await expect(transactionDialog).toBeVisible({ timeout: 30_000 })

  const succeeded = new Set<string>()
  const onConsole = (msg: { text(): string }) => {
    const text = msg.text()
    for (const id of successTxIds) {
      if (text.includes(`Transaction ${id} state: success`)) succeeded.add(id)
    }
  }
  page.on('console', onConsole)

  try {
    const deadline = Date.now() + timeoutMs
    while (Date.now() < deadline) {
      const openWalletButton = transactionDialog.getByRole('button', {
        name: /open wallet/i,
      })
      if (await openWalletButton.isVisible().catch(() => false)) {
        await openWalletButton.click()
        await authorizeTransaction(wallet, 60_000)
        await page.waitForTimeout(500)
        continue
      }

      const waitingButton = transactionDialog.getByRole('button', {
        name: /^Waiting\.\.\.$/i,
      })
      if (await waitingButton.isVisible().catch(() => false)) {
        const iconWalletButton = waitingButton.locator(
          'xpath=preceding-sibling::button[1]',
        )
        if (await iconWalletButton.isVisible().catch(() => false)) {
          await iconWalletButton.click()
          await authorizeTransaction(wallet, 60_000)
          await page.waitForTimeout(500)
          continue
        }
      }

      const primaryButton = transactionDialog.getByRole('button', {
        name: /^(Start|Next|Done)$/i,
      })
      if (
        (await primaryButton.isVisible().catch(() => false)) &&
        (await primaryButton.isEnabled().catch(() => false))
      ) {
        await primaryButton.click()
        await page.waitForTimeout(500)
        continue
      }

      if (succeeded.size === successTxIds.length) break
      await page.waitForTimeout(1_000)
    }
  } finally {
    page.off('console', onConsole)
  }

  expect(succeeded.size).toBe(successTxIds.length)
}

test.describe('Explorer owner-fallback: name transfer', () => {
  test('mines safeTransferFrom of the victim’s own v2 name to the typed name’s stale controller', async ({
    portalPage: page,
    wallet,
    accounts,
    makeName,
  }) => {
    test.setTimeout(900_000)

    const user = accounts.getAddress('user')
    const log: string[] = []
    const say = (s: string) => {
      log.push(s)
      console.log(s)
    }
    fs.mkdirSync(OUT, { recursive: true })

    say(`[fork] block ${await fork.getBlockNumber()}`)
    await connectWithHeadlessWallet(page, wallet)
    say(`[app] connected as ${user}`)

    // ── 1. The victim's real asset ──────────────────────────────────────
    const name = await makeName({ label: 'ofbxfer', owner: 'user' })
    say(`[stage] the victim's own v2 2LD: ${name}`)

    // ── 2. Open the app's own transfer form ─────────────────────────────
    await page.goto(`${PORTAL_APP_URL}/${name}/ownership`)
    const transferLink = page.getByRole('link', { name: 'Transfer' })
    await transferLink.waitFor({ state: 'visible', timeout: 90_000 })
    await transferLink.click()
    await expect(
      page.getByRole('heading', { name: 'Transfer ownership' }),
    ).toBeVisible({ timeout: 30_000 })

    // ── 3. Type the recipient ───────────────────────────────────────────
    await page.getByPlaceholder('ENS name or address').fill(TYPED_NAME)
    await page.waitForTimeout(6_000)
    await page.screenshot({
      path: `${OUT}/T1-transfer-form.png`,
      fullPage: true,
    })

    // Exactly what the app shows the sender about the recipient.
    const formText = await page.locator('main').innerText()
    say('[app] --- what the transfer form says about the recipient ---')
    for (const line of formText.split('\n').map((l) => l.trim()).filter(Boolean)) {
      say(`[app]   ${line}`)
    }
    say('[app] --- end ---')
    say(`[app] form mentions the stale controller in full: ${formText.includes(STALE_CONTROLLER)}`)
    say(`[app] form mentions the registrant at all:       ${formText.includes(REGISTRANT)}`)

    // ── 4. Pre-state ────────────────────────────────────────────────────
    const beforeBlock = await fork.getBlockNumber()

    // ── 5. The app executes the transfer ────────────────────────────────
    const transferButton = page.getByRole('button', { name: 'Transfer name' })
    await expect(transferButton).toBeEnabled({ timeout: 30_000 })
    await transferButton.click()
    // Detach options default to all-on, so the plan is
    // detach-resolver -> detach-registry -> transfer-token.
    await driveTransactionsToSuccess(page, wallet, [
      `transfer-${name}-transfer-token`,
    ])
    say('[app] transfer mined')
    await page.screenshot({ path: `${OUT}/T2-after-mine.png`, fullPage: true })

    // ── 6. Post-state, straight off the fork ────────────────────────────
    const moves = await fork.getLogs({
      event: {
        type: 'event',
        name: 'TransferSingle',
        inputs: [
          { indexed: true, name: 'operator', type: 'address' },
          { indexed: true, name: 'from', type: 'address' },
          { indexed: true, name: 'to', type: 'address' },
          { indexed: false, name: 'id', type: 'uint256' },
          { indexed: false, name: 'value', type: 'uint256' },
        ],
      },
      args: { from: user as `0x${string}` },
      fromBlock: beforeBlock,
      toBlock: 'latest',
    })
    if (moves.length === 0) {
      throw new Error('No ERC-1155 transfer out of the victim since ' + beforeBlock)
    }
    const move = moves[moves.length - 1]
    const movedTo = move.args.to as string
    const tokenId = move.args.id as bigint
    say(`[chain] token id   ${tokenId}`)
    say(`[chain] registry   ${move.address}`)
    say(`[chain] moved FROM ${move.args.from}   (the victim)`)
    say(`[chain] moved TO   ${movedTo}`)
    say(`[chain] tx         ${move.transactionHash}`)

    const balVictim = await fork.readContract({
      address: move.address,
      abi: erc1155Abi,
      functionName: 'balanceOf',
      args: [user as `0x${string}`, tokenId],
    })
    const balController = await fork.readContract({
      address: move.address,
      abi: erc1155Abi,
      functionName: 'balanceOf',
      args: [STALE_CONTROLLER as `0x${string}`, tokenId],
    })
    const balRegistrant = await fork.readContract({
      address: move.address,
      abi: erc1155Abi,
      functionName: 'balanceOf',
      args: [REGISTRANT as `0x${string}`, tokenId],
    })
    say(`[chain] balanceOf(victim)     = ${balVictim}`)
    say(`[chain] balanceOf(controller) = ${balController}`)
    say(`[chain] balanceOf(registrant) = ${balRegistrant}`)

    // ── 7. Can the victim get it back? ──────────────────────────────────
    let recovery = 'UNMEASURED'
    try {
      await fork.simulateContract({
        address: move.address,
        abi: erc1155Abi,
        functionName: 'safeTransferFrom',
        args: [
          STALE_CONTROLLER as `0x${string}`,
          user as `0x${string}`,
          tokenId,
          1n,
          '0x',
        ],
        account: user as `0x${string}`,
      })
      recovery = 'SUCCEEDS — the victim can pull the name back'
    } catch (e) {
      recovery = `REVERTS — ${((e as Error).message ?? '').split(String.fromCharCode(10))[0]}`
    }
    say(`[chain] victim attempting to reclaim the token: ${recovery}`)

    let unreg = 'UNMEASURED'
    try {
      await fork.simulateContract({
        address: move.address,
        abi: parseAbi(['function unregister(uint256 anyId)']),
        functionName: 'unregister',
        args: [tokenId],
        account: user as `0x${string}`,
      })
      unreg = 'SUCCEEDS'
    } catch (e) {
      unreg = `REVERTS — ${((e as Error).message ?? '').split(String.fromCharCode(10))[0]}`
    }
    say(`[chain] victim calling unregister on the .eth registry: ${unreg}`)

    // ── 8. What the app now says the name's owner is ────────────────────
    await page.goto(`${PORTAL_APP_URL}/${name}`)
    await page.waitForTimeout(8_000)
    await page.screenshot({ path: `${OUT}/T3-name-page.png`, fullPage: true })
    const short = (a: string) => `${a.slice(0, 6)}…${a.slice(-4)}`
    const nameBody = await page.locator('main').innerText().catch(() => '')
    say(`[app] name page names the controller ${short(STALE_CONTROLLER)}: ${nameBody.includes(short(STALE_CONTROLLER))}`)
    say(`[app] name page names the victim     ${short(user)}: ${nameBody.includes(short(user))}`)

    fs.writeFileSync(`${OUT}/run.log`, log.join(String.fromCharCode(10)))
    fs.writeFileSync(
      `${OUT}/result.json`,
      JSON.stringify(
        {
          typedName: TYPED_NAME,
          victimName: name,
          registry: move.address,
          tokenId: tokenId.toString(),
          movedTo,
          txHash: move.transactionHash,
          staleController: STALE_CONTROLLER,
          registrant: REGISTRANT,
          victim: user,
          balanceOfVictim: balVictim.toString(),
          balanceOfController: balController.toString(),
          balanceOfRegistrant: balRegistrant.toString(),
          victimCanReclaimToken: recovery,
          victimCanUnregister: unreg,
          formShowedFullControllerAddress: formText.includes(STALE_CONTROLLER),
          formShowedRegistrant: formText.includes(REGISTRANT),
        },
        null,
        2,
      ),
    )

    expect(movedTo.toLowerCase()).toBe(STALE_CONTROLLER.toLowerCase())
    expect(balVictim).toBe(0n)
    expect(balController).toBe(1n)
    expect(balRegistrant).toBe(0n)
  })
})
```

### Recorded output

Verbatim console of the run that produced the transaction cited above:

```

Running 1 test using 1 worker

[fork] block 11667805
[app] connected as 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[makeName] registering ofbxfer-1788995489.eth (duration=2419200s, desiredGapPastExpiry=0s)
[makeName] waiting 61s for MIN_COMMITMENT_AGE
[makeName] ✅ registered ofbxfer-1788995489.eth
[time] sync — browser clock set to 2026-09-09T23:11:18.000Z
[makeName] ready: ofbxfer-1788995489.eth
[stage] the victim's own v2 2LD: ofbxfer-1788995489.eth
[app] --- what the transfer form says about the recipient ---
[app]   Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026.
[app]   Dismiss
[app]   Back
[app]   Transfer ownership
[app]   Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding.
[app]   Recipient
[app]   0x741d627F48C2Db692C59679634e0e3131E489d08
[app]   Detach the resolver
[app]   Detaches this name’s resolver so it stops resolving to your records entirely. The recipient starts clean and sets up their own.
[app]   Transfer name
[app] --- end ---
[app] form mentions the stale controller in full: true
[app] form mentions the registrant at all:       false
[app] transfer mined
[chain] token id   85135093676918858441316441638259934445902223169298981865600238687246264303616
[chain] registry   0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2
[chain] moved FROM 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   (the victim)
[chain] moved TO   0x741d627F48C2Db692C59679634e0e3131E489d08
[chain] tx         0x9e91b1b9f6da62380056866f95c2e5168d57ec48753af236c1f21b303cdcab42
[chain] balanceOf(victim)     = 0
[chain] balanceOf(controller) = 1
[chain] balanceOf(registrant) = 0
[chain] victim attempting to reclaim the token: REVERTS — The contract function "safeTransferFrom" reverted with the following signature:
[chain] victim calling unregister on the .eth registry: REVERTS — The contract function "unregister" reverted with the following signature:
[app] name page names the controller 0x741d…9d08: true
[app] name page names the victim     0xf39F…2266: false
  ok 1 [portal-e2e] › projects\portal\tests\owner-fallback-transfer.spec.ts:118:3 › Explorer owner-fallback: name transfer › mines safeTransferFrom of the victim’s own v2 name to the typed name’s stale controller (24.2s)

  1 passed (33.1s)
```

Screenshots from the same run: `T1-transfer-form.png` (the shipped form after typing `tkn.eth`), `T2-after-mine.png` (the app the moment the transfer mines) and `T3-name-page.png` (the name's own page afterwards, printing the substituted controller as its owner).

The machine-readable summary the same run writes, verbatim:

```json
{
  "typedName": "tkn.eth",
  "victimName": "ofbxfer-1788995489.eth",
  "registry": "0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2",
  "tokenId": "85135093676918858441316441638259934445902223169298981865600238687246264303616",
  "movedTo": "0x741d627F48C2Db692C59679634e0e3131E489d08",
  "txHash": "0x9e91b1b9f6da62380056866f95c2e5168d57ec48753af236c1f21b303cdcab42",
  "staleController": "0x741d627F48C2Db692C59679634e0e3131E489d08",
  "registrant": "0x3cDB9bC080Efe321A797E7286d302B90227cc6De",
  "victim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "balanceOfVictim": "0",
  "balanceOfController": "1",
  "balanceOfRegistrant": "0",
  "victimCanReclaimToken": "REVERTS — The contract function \"safeTransferFrom\" reverted with the following signature:",
  "victimCanUnregister": "REVERTS — The contract function \"unregister\" reverted with the following signature:",
  "formShowedFullControllerAddress": true,
  "formShowedRegistrant": false
}
```

### The sender's check — step 6, a second read-only spec

Same harness, no transactions: it connects the wallet, opens `/tkn.eth` and `/tkn.eth/ownership`, screenshots both and prints every non-empty line of `<main>`. Save it beside the first spec as `owner-fallback-corroborate.spec.ts` and run it the same way, with `OUT=hunt/out-corroborate … owner-fallback-corroborate`.

```ts
import fs from 'node:fs'
import { expect, test, connectWithHeadlessWallet } from '../../../fixtures/playwright.portal.fixture.js'

const APP = process.env.PORTAL_APP_URL ?? 'http://localhost:5199'
const OUT = process.env.OUT ?? 'hunt/out-corroborate'
const NAME = 'tkn.eth'
const CONTROLLER = '0x741d627F48C2Db692C59679634e0e3131E489d08'
const REGISTRANT = '0x3cDB9bC080Efe321A797E7286d302B90227cc6De'
const short = (a: string) => `${a.slice(0, 6)}…${a.slice(-4)}`

test('what the app says tkn.eth’s owner is', async ({ portalPage: page, wallet }) => {
  test.setTimeout(300_000)
  fs.mkdirSync(OUT, { recursive: true })
  const log: string[] = []
  const say = (s: string) => { log.push(s); console.log(s) }
  await connectWithHeadlessWallet(page, wallet)

  for (const route of ['', '/ownership']) {
    await page.goto(`${APP}/${NAME}${route}`)
    await page.waitForTimeout(12_000)
    const tag = route === '' ? 'overview' : 'ownership'
    await page.screenshot({ path: `${OUT}/C-${tag}.png`, fullPage: true })
    const body = await page.locator('main').innerText().catch(() => '')
    say(`=== ${APP}/${NAME}${route} ===`)
    for (const l of body.split('\n').map(x => x.trim()).filter(Boolean)) say(`   ${l}`)
    say(`  >> shows controller ${short(CONTROLLER)} : ${body.includes(short(CONTROLLER)) || body.includes(CONTROLLER)}`)
    say(`  >> shows registrant ${short(REGISTRANT)} : ${body.includes(short(REGISTRANT)) || body.includes(REGISTRANT)}`)
    say('')
  }
  fs.writeFileSync(`${OUT}/run.log`, log.join(String.fromCharCode(10)))
})
```

Its full output — both routes, every line, verbatim; the excerpt in *Vulnerability Details §6* is the head of the second block. Screenshots `C-overview.png` and `C-ownership.png` are the same two pages:

```
=== http://localhost:5199/tkn.eth ===
   Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026.
   Dismiss
   tkn.eth
   tkn.eth
   Expires
   Feb 2, 2027 7:21pm
   Registered
   Feb 3, 2024 7:21pm
   Owner
   Address
   Copy
   0x741d…9d08
   Parent
   Name
   Copy
   eth
   Resolver
   Resolver
   Copy
   0xae66…b2Ba
   Protocol
   ENSv1
   Subnames
   4
   Records set
   1
   HISTORY
   Full history
   Date
   Transaction
   From
   1
   Feb 21, 2024 9:05am
   Copy
   Etherscan
   0x6c9d…6836
   Address
   Copy
   0x741d…9d08
   1
   Feb 21, 2024 9:00am
   Copy
   Etherscan
   0x876e…2230
   Address
   Copy
   0x741d…9d08
   1
   Feb 21, 2024 8:53am
   Copy
   Etherscan
   0xf8ca…64df
   Address
   Copy
   0x741d…9d08
   1
   Feb 21, 2024 8:49am
   Copy
   Etherscan
   0x0e98…f049
   Address
   Copy
   0x741d…9d08
   1
   Feb 6, 2024 2:19am
   Copy
   Etherscan
   0x82e0…eb22
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 8:59pm
   Copy
   Etherscan
   0xb2a6…f61b
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 3:18pm
   Copy
   Etherscan
   0x0b6e…4a16
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 3:09pm
   Copy
   Etherscan
   0x0774…3030
   Address
   Copy
   0x741d…9d08
   1
   Feb 4, 2024 2:48pm
   Copy
   Etherscan
   0x4b8f…30ba
   Address
   Copy
   0x3cdb…c6de
  >> shows controller 0x741d…9d08 : true
  >> shows registrant 0x3cDB…c6De : false

=== http://localhost:5199/tkn.eth/ownership ===
   Notice: ENS v2 is in active development. Registered names on Sepolia and state data may be reset periodically due to routine contract deployments. The most recent deployment was on July 30, 2026.
   Dismiss
   Ownership
   Expires
   Feb 2, 2027 7:21pm
   Registered
   Feb 3, 2024 7:21pm
   Owner
   Address
   Copy
   0x741d…9d08
   Manager
   Address
   Copy
   0x741d…9d08
   Parent
   Name
   Copy
   eth
   HISTORY
   Date
   Transaction
   From
   1
   Feb 21, 2024 8:53am
   Copy
   Etherscan
   0xf8ca…64df
   Address
   Copy
   0x741d…9d08
   1
   Feb 6, 2024 2:19am
   Copy
   Etherscan
   0x82e0…eb22
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 8:59pm
   Copy
   Etherscan
   0xb2a6…f61b
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 8:44pm
   Copy
   Etherscan
   0x88ec…1fbe
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 6:29pm
   Copy
   Etherscan
   0x89cb…3069
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 6:16pm
   Copy
   Etherscan
   0xc7f0…c873
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 6:08pm
   Copy
   Etherscan
   0x56e7…65ee
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 6:03pm
   Copy
   Etherscan
   0x7a44…049d
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 6:01pm
   Copy
   Etherscan
   0xd99b…3f7e
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 5:59pm
   Copy
   Etherscan
   0x58e1…e634
   Address
   Copy
   0x741d…9d08
   1
   Feb 5, 2024 3:06pm
   Copy
   Etherscan
   0x844d…0a4f
   Address
   Copy
   0x741d…9d08
   1
   Feb 4, 2024 2:50pm
   Copy
   Etherscan
   0x32a7…7a44
   Address
   Copy
   0x741d…9d08
   1
   Feb 4, 2024 2:49pm
   Copy
   Etherscan
   0xd5b2…55a1
   Address
   Copy
   0x3cdb…c6de
   3
   Feb 4, 2024 2:48pm
   Copy
   Etherscan
   0x4b8f…30ba
   Address
   Copy
   0x3cdb…c6de
   1
   Feb 3, 2024 8:53pm
   Copy
   Etherscan
   0x5d5d…16a4
   Address
   Copy
   0x3cdb…c6de
   1
   Feb 3, 2024 8:52pm
   Copy
   Etherscan
   0xafbd…da87
   Address
   Copy
   0x3cdb…c6de
   1
   Feb 3, 2024 8:48pm
   Copy
   Etherscan
   0x7e01…7b3c
   Address
   Copy
   0x3cdb…c6de
   1
   Feb 3, 2024 8:41pm
   Copy
   Etherscan
   0x3ffe…65fc
   Address
   Copy
   0x3cdb…c6de
   1
   Feb 3, 2024 8:16pm
   Copy
   Etherscan
   0xf84b…c502
   Address
   Copy
   0x3cdb…c6de
   4
   Feb 3, 2024 7:21pm
   Copy
   Etherscan
   0x0a8b…1d94
   Address
   Copy
   0x3cdb…c6de
  >> shows controller 0x741d…9d08 : true
  >> shows registrant 0x3cDB…c6De : false
```

### The blast-radius leg — `create-subname`, no impact row claimed

A third spec drives the same hook through the create-subname **Owner** field instead of the transfer recipient: it registers a v2 parent to the connected wallet, deploys and sets its subregistry through the app, types `tkn.eth` into the Owner field, and mints the subname. It is the same root cause on a revocable sink, which is why no impact row is claimed for it. Its output, verbatim — screenshots `A1-owner-field-fallback.png` (the Owner field's `Resolved:` status line), `A2-after-mine.png` and `A3-subname-page.png`:

```
[fork] block 11667797
[app] connected as 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[stage] parent v2 name registered to the connected wallet: ownerfb-1788995323.eth
[stage] subregistry deployed and set via the app
[app] Owner field input "tkn.eth" -> status line "Resolved: 0x741d…9d08"
[app] create-subname transaction mined
[chain] 1 ERC-1155 mint(s) since block 11667804
[chain] subname      sub1788995329147.ownerfb-1788995323.eth
[chain] minted by    0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   (the connected wallet)
[chain] minted TO    0x741d627F48C2Db692C59679634e0e3131E489d08
[chain] registry     0x055767e795846b0a0022ee9b5de1cb48bb9e8197
[chain] token id     43329550880948562014714012933180488317037049927278632230051544758758947160064
[chain] tx           0xa10299576c520d85c5948fd6487e5d298b92683c48f5847447f22a345ce490ff

[compare] the user typed              "tkn.eth"
[compare] app displayed               "Resolved: 0x741d…9d08"
[compare] tkn.eth stale controller     0x741d627F48C2Db692C59679634e0e3131E489d08
[compare] tkn.eth actual registrant    0x3cDB9bC080Efe321A797E7286d302B90227cc6De
[compare] connected wallet             0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[compare] name went to the controller: true
[compare] name went to the registrant: false
[chain] balanceOf(controller, tokenId) = 1
[chain] balanceOf(registrant, tokenId) = 0
[app] subname page names the controller 0x741d…9d08: true
[app] subname page names the registrant 0x3cDB…c6De: false
[chain] parent owner 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 calling unregister(43329550880948562014714012933180488317037049927278632230051544758758947160064):
[chain]   SUCCEEDS — the parent owner CAN take the subname back
[chain] substituted owner calling unregister: SUCCEEDS
```
