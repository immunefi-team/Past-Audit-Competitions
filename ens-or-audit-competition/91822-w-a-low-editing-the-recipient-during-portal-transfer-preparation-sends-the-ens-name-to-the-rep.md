# 91822 w a low editing the recipient during portal transfer preparation sends the ens name to the replaced address

**Submitted on Sep 4th 2026 at 14:59:46 UTC by @manvi for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91822
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief / Intro

I analysed the Portal's ENS V2 transfer flow and found that the recipient shown in the transfer form can diverge from the recipient saved for transaction construction.

The sequence is straightforward:

1. The user enters recipient **A**.
2. The user presses **Transfer name**.
3. `SendNameForm` calls `startTransfer()` with A.
4. `useTransferName` begins an asynchronous `getEnsTokenId()` read and keeps A in the mutation variables.
5. Portal changes the button to **Preparing…**, but leaves the recipient field editable.
6. The user notices A is wrong and replaces it with intended recipient **B**.
7. The form and recipient preview now show B.
8. The earlier `getEnsTokenId()` request completes.
9. `useTransferName.onSuccess` saves the original mutation result containing A.
10. The generated transaction callbacks close over A.
11. Portal opens a generic transaction modal that does not disclose the saved recipient.
12. The user proceeds.
13. `transferToken()` encodes `safeTransferFrom(owner, A, tokenId, 1, "0x")`.
14. The transaction succeeds and A receives the ENS token.

Portal itself warns:

> `Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding.`

The bug defeats that safety check. A user can notice the wrong recipient, correct it before wallet approval, and still transfer the name to the address they replaced.

The final PoC makes the asynchronous interval deterministic by delaying the real `getTokenId(uint256)` RPC response. The response is not altered or replaced; it is released unchanged after the user-visible recipient has been changed to B.

The run then executes the real Portal transaction flow and checks authoritative ERC-1155 balances on Anvil:

```
BEFORE:
victim = 1
A      = 0
B      = 0

AFTER:
victim = 0
A      = 1
B      = 0
```

The test exits successfully:

```
1 passed (26.2s)
```

***

## Vulnerability Details

### 1. `SendNameForm` snapshots the recipient when the user starts the transfer

`apps/portal/src/features/transfer/components/SendNameForm.tsx` derives the current recipient from the form:

```ts
const resolution = useAddressResolution(recipientInput)
const { address: recipient, isResolving } = resolution

const { startTransfer, transactions, isPreparing, prepError } =
  useTransferName({ name, registryAddress, owner })
```

The submit path is:

```ts
const runTransfer = () => {
  if (!recipient || !canStart) return
  startTransfer({ recipient, options: effectiveOptions })
}
```

If A is visible when the user presses the button, A becomes the recipient passed into the asynchronous transfer-preparation mutation.

### 2. `isPreparing` disables the button but not the recipient field

The form prevents a second submit while preparation is running:

```ts
const canStart =
  hasValidRecipient && !isResolving && !isPreparing && detachTargetsSettled
```

The submit button is disabled through `canStart`:

```tsx
<Button
  variant="default"
  onClick={runTransfer}
  disabled={!canStart}
>
  {isPreparing ? 'Preparing…' : 'Transfer name'}
</Button>
```

However the recipient field remains:

```tsx
<AddressNameInput
  value={recipientInput}
  onChange={setRecipientInput}
  resolution={resolution}
  className="h-9"
/>
```

No `disabled={isPreparing}` value is passed.

The reusable `AddressNameInput` component already supports a `disabled` prop. The transfer page simply does not use it here.

The browser PoC proves the production behavior directly:

```
[POC] recipient input remains ENABLED while Preparing = true
```

### 3. `useTransferName` preserves A across the asynchronous token-ID lookup

`apps/portal/src/features/transfer/hooks/useTransferName.ts` prepares the flow with:

```ts
const prepareMutation = useMutation(
  resultMutationOptions({
    mutationFn: ({ recipient, options }: StartTransferParams) =>
      getEnsTokenId({
        label: getLabel(name),
        registryAddress,
      }).map((tokenId) => ({ recipient, tokenId, options })),
    onSuccess: (params) => {
      startedStepsRef.current = new Set()
      setSavedParams(params)
      openModal()
    },
  }),
)
```

The recipient inside the `.map()` callback is the mutation variable supplied when `startTransfer()` was called.

Changing the form from A to B does not mutate that already-started request.

When `getEnsTokenId()` finishes, the value saved by `setSavedParams(params)` is still:

```
recipient = A
```

There is no comparison against the current recipient in the form before opening the transaction modal.

### 4. All later transaction callbacks use `savedParams`

The transaction list is built from the saved snapshot:

```ts
if (!savedParams) return []

const { recipient, tokenId, options } = savedParams
const steps = buildTransferPlan(options)
```

Each runner closes over that saved `recipient`:

```ts
const runners = steps.map((step) => async () => {
  const id = `transfer-${name}-${step}`
  ...
  await runStep(step, id, recipient, tokenId)
})
```

The final transfer step is:

```ts
.with('transfer-token', () =>
  transferToken({
    ...common,
    registryAddress,
    tokenId,
    recipient,
    id,
  }),
)
```

By this point the live form can show B while the transaction runner is permanently bound to A.

### 5. `transferToken()` encodes the saved recipient into `safeTransferFrom`

`apps/portal/src/features/transfer/helpers/transferToken.ts` performs the actual ENS V2 ownership transfer:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [
    walletWithAccount.account.address,
    recipient,
    tokenId,
    1n,
    '0x',
  ],
})
```

The transaction manager then submits that calldata to `registryAddress`.

Therefore the stale value controls the real ERC-1155 transfer destination.

### 6. The Portal transaction modal does not reveal which recipient was saved

The transaction descriptor uses:

```ts
title: STEP_LABELS[step],
transactionName: `${STEP_LABELS[step]} - ${name}`,
```

For the final step the user gets generic Portal text such as:

```
Transfer name
```

The successful PoC observed:

```
Transaction overview Transfer name Not Started Est. Cost Not yet Start Close
```

and asserted:

```
modal discloses stale A = false
modal discloses intended B = false
```

So Portal provides no final application-level indication that A is still the prepared destination.

### 7. The same stale snapshot also includes transfer options

`startTransfer()` receives both:

```ts
{
  recipient,
  options: effectiveOptions,
}
```

The same class of stale state can therefore apply to the cleanup switches while preparation is running.

This report does not rely on that secondary effect. The hardened PoC disables the visible detach-resolver option before preparation and demonstrates the Critical impact with only the final `transfer-token` step.

***

## Why this is not merely a UI/UX issue

The UI mismatch is only the entry point. The final effect is a real ownership transfer.

The browser-to-chain PoC executes the actual Portal transaction and proves:

```
Portal field immediately before wallet flow:
B

Transaction:
SUCCESS

ERC-1155 post-state:
victim = 0
A      = 1
B      = 0
```

That is an application transaction-integrity failure, not presentation-only stale state.

***

## Why this is not intended design

Portal explicitly tells the user to check the recipient because the transfer is irreversible.

An interface intentionally allowing edits during preparation would need one of two behaviors:

```
edit during preparation
        |
        +--> cancel / invalidate old preparation
        |
        +--> rebuild using the new values
```

Alternatively, the form can be locked once preparation begins.

The current behavior instead allows:

```
form shows B
prepared transaction uses A
```

There is no reasonable transfer-flow interpretation under which a corrected recipient should be ignored while remaining visibly accepted by the application.

***

## Why this does not require a trusted or privileged actor

The vulnerability does not require:

* Portal administrator access;
* ENS governance access;
* backend credentials;
* a privileged protocol account;
* victim private-key compromise;
* malicious contract deployment;
* malicious resolver behavior;
* a failed transaction;
* a retry;
* transaction-ID reuse.

The victim is the legitimate owner and uses the normal Portal transfer flow.

The only timing condition is that the recipient is edited while the asynchronous token-ID preparation is still pending.

***

## Why the deterministic RPC delay does not fabricate the bug

The PoC intercepts the real browser request for:

```solidity
getTokenId(uint256 labelHash)
```

It does not change:

* the RPC method;
* the contract address;
* the calldata;
* the token ID;
* the RPC response.

The request is paused, the user-visible recipient is changed, and then that exact request is continued unchanged:

```
[POC] 🚧 intercepted production getTokenId(uint256) RPC
...
[POC] ▶ releasing unchanged getTokenId RPC
```

The delay only makes the naturally asynchronous interval reproducible on demand.

Any real RPC latency long enough for the recipient to be edited creates the same state relationship:

```
submitted mutation variables = old recipient
live form state             = new recipient
```

***

## Wallet signing prompt caveat

The browser-to-chain PoC uses the repository's own `@ensdomains/headless-web3-provider` fixture.

That fixture authorizes the wallet request without rendering a MetaMask, Rainbow, hardware-wallet, or other human-facing decoded signing prompt.

For that reason, this report does not claim that every real wallet will hide recipient A.

A wallet that fully decodes ERC-1155 `safeTransferFrom` may show A clearly enough for an attentive user to notice the mismatch and reject the transaction.

That is an independent downstream mitigation, not a fix for the Portal behavior demonstrated here.

Portal itself has already done all of the following before control reaches the wallet:

```
accepted corrected recipient B
displayed corrected recipient B
opened a Portal transaction modal
failed to disclose A
failed to disclose B
submitted a transaction still bound to A
```

Wallet decoding quality also varies. A wallet may show decoded arguments, generic contract-interaction information, truncated values, or raw calldata.

The narrow claim made here is fully demonstrated regardless of wallet presentation:

> Portal accepts and displays B while constructing and successfully submitting the ENS ownership transfer using A.

***

## Why this is not one of the disclosed known issues

### R3-07 has a different root cause and consequence

R3-07 concerns a transaction ID reused after a failed attempt.

Its effect is that a later successful retry skips archive/history/telemetry handling and can remain displayed as pending.

This finding occurs before the transaction actor is started.

It does not require:

```
failed transaction
retry
fixed transaction ID
history
telemetry
archive state
```

The old recipient is already captured during the initial preparation.

### WEB-1257 / QA-01 are separate transfer problems

WEB-1257 concerns registry detachment after a parent transfer.

QA-01 concerns role combinations that lead to errors in later profile / primary-name operations.

Neither issue changes the recipient encoded into the ERC-1155 `safeTransferFrom`.

A fix for those issues would leave this stale-recipient race intact.

***

## Audited Revision Verification

The frozen competition import verified for this finding is:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

That commit records:

```
fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86
```

The relevant frozen blobs include:

```
apps/portal/src/features/transfer/components/SendNameForm.tsx
blob SHA: 0d48b575648a728f7f8d4415e4024b7f6c517258

apps/portal/src/features/transfer/hooks/useTransferName.ts
blob SHA: d3f124ff0b64fb1e450b3b46259f7b84880cf5f4

apps/portal/src/features/transfer/helpers/transferToken.ts
blob SHA: c76d53ad49f85b76e7d0cffc9acf5dd68e8c0dec

apps/portal/src/features/transfer/utils/buildTransferPlan.ts
blob SHA: 2874722beb2dece5bbef12cc58c52a2be9617f84

apps/portal/src/features/address/components/AddressNameInput.tsx
blob SHA: f3f45bbb9fb17970a6a101fbaec3671eb9afb1c4

apps/portal/src/features/transaction-manager/components/TransactionsOverviewContent.tsx
blob SHA: 837ef9cd17ec94d3c6ae01d8fefc53b36ae1940e
```

The PoC exercises the frozen behavior with:

```
real Portal rendering
real transfer form
real getTokenId(uint256) RPC
real transaction descriptors
real transaction manager
real headless wallet authorization
real ERC-1155 transfer
real Anvil transaction receipt
direct post-transaction ERC-1155 balance reads
```

The final successful run completed with:

```
1 passed (26.2s)
```

The successfully executed PoC file has SHA-256:

```
a540fd562cb90e48f1a14af7232f92b4d8f42cc44c9a71fb08ce63155295045b
```

***

## Impact Details

The demonstrated impact is an irreversible ENS V2 ownership transfer to an address that the user has already replaced in the Portal UI.

Concrete demonstration:

```
Initial Portal recipient:
A = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

User corrects Portal recipient:
B = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

Portal immediately before wallet flow:
B

Portal transaction modal:
A not disclosed
B not disclosed

Real transaction:
SUCCESS

Authoritative ERC-1155 post-state:
victim = 0
A      = 1
B      = 0
```

If A is attacker-controlled, the attacker receives the ENS NFT.

Even when A is merely an unintended third party, the transfer cannot be undone by Portal after it succeeds. Recovery depends on A voluntarily transferring the name back.

I claim only the transfer-recipient mismatch reproduced end-to-end here. I do not claim that every wallet hides A at its signing prompt.

***

## Preconditions

1. The user owns a transferable ENS V2 `.eth` name.
2. Recipient A is present when the user presses **Transfer name**.
3. The `getEnsTokenId()` preparation has not completed yet.
4. The user changes the recipient from A to B during that pending interval.
5. The original preparation completes after the edit.
6. The user proceeds with the transaction.
7. The downstream wallet does not provide a sufficiently clear warning that causes the user to reject the stale-A transaction.

For deterministic reproduction, the PoC pauses the real `getTokenId(uint256)` RPC until after B is visible.

No privileged actor, malicious backend, malicious RPC response, transaction retry, or contract modification is required.

***

## Recommended Remediation

The transfer's displayed parameters and prepared parameters must never diverge.

### Option 1 — freeze prepared fields while `isPreparing`

At minimum:

```tsx
<AddressNameInput
  value={recipientInput}
  onChange={setRecipientInput}
  resolution={resolution}
  disabled={isPreparing}
/>
```

The switches contributing to `effectiveOptions` should also be disabled for the same period.

### Option 2 — invalidate stale preparation results when form values change

If editing during preparation is intended, associate each preparation attempt with a monotonically increasing identifier.

Conceptually:

```ts
const runId = ++currentRun.current

const result = await prepare(...)

if (runId !== currentRun.current) return

setSavedParams(result)
```

Changing the recipient or any transfer option should invalidate the previous run.

An old async response must not be allowed to open the transaction modal.

### Option 3 — re-check the recipient before building transactions

Before committing `savedParams` or before exposing the transaction modal, verify that:

```
prepared recipient
==
current resolved recipient
```

If not, discard the prepared result and require a new preparation.

### Option 4 — show the actual destination in Portal's transaction modal

Portal should show the exact saved recipient used by the final transaction:

```
Transfer name
To: 0x...
```

This should be derived from the same immutable parameters used to generate calldata.

### Recommended invariant

> The recipient displayed by Portal at approval time must be the same recipient encoded in the ENS ownership-transfer transaction. If any prepared field changes, the old preparation must be blocked, cancelled, or discarded.

***

## References

### In-scope Explorer / Portal

* `apps/portal/src/features/transfer/components/SendNameForm.tsx`
  * live `recipientInput`
  * `startTransfer({ recipient, options })`
  * `isPreparing`
  * recipient input remains enabled
  * generic irreversible-transfer warning
* `apps/portal/src/features/transfer/hooks/useTransferName.ts`
  * asynchronous `getEnsTokenId()`
  * original mutation variables preserved
  * `setSavedParams(params)`
  * transaction callbacks close over saved recipient
  * `transferToken(... recipient ...)`
* `apps/portal/src/features/transfer/helpers/transferToken.ts`
  * ERC-1155 `safeTransferFrom`
  * saved recipient used as second argument
  * real transaction-manager submission
* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts`
  * transfer step is always appended
  * cleanup steps execute before transfer
* `apps/portal/src/features/address/components/AddressNameInput.tsx`
  * supports `disabled`
  * transfer form does not set it during preparation
* `apps/portal/src/features/transaction-manager/components/TransactionsOverviewContent.tsx`
  * generic transaction overview
  * no recipient disclosure

### E2E infrastructure used by the PoC

* `e2e/fixtures/playwright.portal.fixture.ts`
  * Portal browser fixture
  * headless wallet
  * default Anvil accounts
  * real local Sepolia fork
* `e2e/fixtures/makeName.ts`
  * real V2 `.eth` registration on Anvil
* `e2e/helpers/portal-auth.ts`
  * transaction authorization helper

***

## Proof of Concept

## PoC Classification

This is a repo-native browser-to-chain E2E PoC against the frozen competition code.

It uses the real:

* Portal transfer page;
* `SendNameForm`;
* `useTransferName`;
* `buildTransferPlan`;
* `transferToken`;
* `getTokenId(uint256)` RPC request;
* transaction modal;
* transaction manager;
* headless wallet authorization path;
* V2 ETH Registry;
* ERC-1155 ownership transfer;
* Anvil Sepolia fork;
* post-transaction `balanceOf` reads.

The only deterministic intervention is that the exact real `getTokenId(uint256)` browser RPC is paused and later continued unchanged.

No fake token ID, fake RPC response, mocked transfer helper, or mocked ownership result is used for the demonstrated impact.

The PoC deliberately disables the visible detach-resolver option before preparation so that the demonstrated transaction plan contains only the final irreversible `transfer-token` step.

## PoC File

```
e2e/projects/portal/tests/stale-recipient-transfer.poc.spec.ts
```

SHA-256 of the successfully executed file:

```
a540fd562cb90e48f1a14af7232f92b4d8f42cc44c9a71fb08ce63155295045b
```

## Complete Hardened E2E PoC

The complete successfully executed PoC source is included inline below.

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import type { Web3ProviderBackend } from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  erc1155Abi,
  labelhash,
  parseAbi,
  toFunctionSelector,
  type Address,
} from 'viem'

import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import {
  publicClient,
} from '../../../helpers/anvil-client.js'
import {
  authorizeTransaction,
} from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL =
  process.env.PORTAL_APP_URL ??
  'http://127.0.0.1:3001'

const ensjsSepolia =
  ensL1Contracts[
    supportedL1Chains.sepolia
  ]

const ETH_REGISTRY =
  ensjsSepolia.ensRegistry.address

/*
 * ENSJS getTokenId() calls:
 *
 *   registry.getTokenId(labelhash(label))
 *
 * Detect exactly this read at the browser RPC boundary.
 */
const GET_TOKEN_ID_SELECTOR =
  toFunctionSelector(
    'getTokenId(uint256)',
  ).toLowerCase()

const GET_TOKEN_ID_ABI =
  parseAbi([
    'function getTokenId(uint256 labelHash) view returns (uint256)',
  ])

type RpcCall = {
  method?: string
  params?: unknown[]
}

function containsTargetGetTokenIdCall(
  payload: unknown,
): boolean {
  const calls =
    Array.isArray(payload)
      ? payload
      : [payload]

  return calls.some(
    (entry) => {
      const call =
        entry as RpcCall

      if (
        call.method !==
        'eth_call'
      ) {
        return false
      }

      const tx =
        (
          call.params?.[0] ??
          {}
        ) as {
          to?: string
          data?: string
        }

      if (
        typeof tx.to !==
          'string' ||
        typeof tx.data !==
          'string'
      ) {
        return false
      }

      return (
        tx.to.toLowerCase() ===
          ETH_REGISTRY.toLowerCase() &&
        tx.data
          .toLowerCase()
          .startsWith(
            GET_TOKEN_ID_SELECTOR,
          )
      )
    },
  )
}

async function readTokenBalance(
  account: Address,
  tokenId: bigint,
): Promise<bigint> {
  return publicClient.readContract({
    address:
      ETH_REGISTRY,
    abi:
      erc1155Abi,
    functionName:
      'balanceOf',
    args: [
      account,
      tokenId,
    ],
  })
}

async function driveTransactionsToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  successTxIds: string[],
  timeoutMs = 180_000,
): Promise<void> {
  const succeeded =
    new Set<string>()

  const onConsole =
    (msg: {
      text(): string
    }) => {
      const text =
        msg.text()

      if (
        text.includes(
          '[TRANSACTION MANAGER]',
        )
      ) {
        console.log(
          '[BROWSER]',
          text,
        )
      }

      for (
        const id of
        successTxIds
      ) {
        if (
          text.includes(
            `Transaction ${id} state: success`,
          )
        ) {
          succeeded.add(
            id,
          )
        }
      }
    }

  page.on(
    'console',
    onConsole,
  )

  try {
    const deadline =
      Date.now() +
      timeoutMs

    while (
      Date.now() <
      deadline
    ) {
      const dialog =
        page.locator(
          '[data-slot="dialog-content"][data-state="open"]',
        )

      const retry =
        dialog.getByRole(
          'button',
          {
            name:
              /^(Retry|Try again)$/i,
          },
        )

      if (
        await retry
          .isVisible()
          .catch(
            () => false,
          )
      ) {
        const text =
          (
            await dialog
              .innerText()
              .catch(
                () => '',
              )
          )
            .replace(
              /\s+/g,
              ' ',
            )
            .trim()

        throw new Error(
          `Portal transaction entered error state: ${text}`,
        )
      }

      const openWallet =
        dialog.getByRole(
          'button',
          {
            name:
              /open wallet/i,
          },
        )

      if (
        await openWallet
          .isVisible()
          .catch(
            () => false,
          )
      ) {
        console.log(
          '[POC] clicking Open wallet',
        )

        await openWallet.click()

        await authorizeTransaction(
          wallet,
          60_000,
        )

        await page.waitForTimeout(
          500,
        )

        continue
      }

      const waitingButton =
        dialog.getByRole(
          'button',
          {
            name:
              /^Waiting\.\.\.$/i,
          },
        )

      if (
        await waitingButton
          .isVisible()
          .catch(
            () => false,
          )
      ) {
        const iconWalletButton =
          waitingButton.locator(
            'xpath=preceding-sibling::button[1]',
          )

        if (
          await iconWalletButton
            .isVisible()
            .catch(
              () => false,
            )
        ) {
          await iconWalletButton.click()

          await authorizeTransaction(
            wallet,
            60_000,
          )

          await page.waitForTimeout(
            500,
          )

          continue
        }
      }

      const primary =
        dialog.getByRole(
          'button',
          {
            name:
              /^(Start|Next|Done)$/i,
          },
        )

      if (
        await primary
          .isVisible()
          .catch(
            () => false,
          ) &&
        await primary
          .isEnabled()
          .catch(
            () => false,
          )
      ) {
        const label =
          await primary
            .innerText()

        console.log(
          '[POC] clicking modal primary =',
          label,
        )

        await primary.click()

        await page.waitForTimeout(
          500,
        )

        continue
      }

      if (
        succeeded.size ===
        successTxIds.length
      ) {
        break
      }

      await page.waitForTimeout(
        500,
      )
    }
  } finally {
    page.off(
      'console',
      onConsole,
    )
  }

  expect(
    succeeded.size,
  ).toBe(
    successTxIds.length,
  )
}

test.describe(
  'CRITICAL stale transfer recipient',
  () => {
    test(
      'Portal displays corrected B but transfers ENS ERC1155 to stale A',
      async ({
        portalPage: page,
        wallet,
        accounts,
        makeName,
      }) => {
        test.setTimeout(
          240_000,
        )

        console.log('')
        console.log(
          '============================================================',
        )
        console.log(
          'CRITICAL STALE RECIPIENT — REAL BROWSER / REAL CHAIN',
        )
        console.log(
          '============================================================',
        )

        await connectWithHeadlessWallet(
          page,
          wallet,
        )

        /*
         * user  = victim / current ENS owner
         * user2 = stale recipient A
         * user3 = corrected intended recipient B
         */
        const victim =
          accounts.getAddress(
            'user',
          )

        const staleRecipient =
          accounts.getAddress(
            'user2',
          )

        const intendedRecipient =
          accounts.getAddress(
            'user3',
          )

        expect(
          staleRecipient
            .toLowerCase(),
        ).not.toBe(
          intendedRecipient
            .toLowerCase(),
        )

        const name =
          await makeName({
            label:
              'critical-stale-recipient',
            owner:
              'user',
          })

        const label =
          name.replace(
            /\.eth$/,
            '',
          )

        const tokenId =
          await publicClient
            .readContract({
              address:
                ETH_REGISTRY,

              abi:
                GET_TOKEN_ID_ABI,

              functionName:
                'getTokenId',

              args: [
                BigInt(
                  labelhash(
                    label,
                  ),
                ),
              ],
            })

        console.log(
          '[POC] name =',
          name,
        )

        console.log(
          '[POC] victim =',
          victim,
        )

        console.log(
          '[POC] stale recipient A =',
          staleRecipient,
        )

        console.log(
          '[POC] intended recipient B =',
          intendedRecipient,
        )

        console.log(
          '[POC] tokenId =',
          tokenId.toString(),
        )

        const victimBefore =
          await readTokenBalance(
            victim,
            tokenId,
          )

        const staleBefore =
          await readTokenBalance(
            staleRecipient,
            tokenId,
          )

        const intendedBefore =
          await readTokenBalance(
            intendedRecipient,
            tokenId,
          )

        console.log('')
        console.log(
          '=== ON-CHAIN PRE-STATE ===',
        )

        console.log(
          '[POC] victim balance BEFORE =',
          victimBefore.toString(),
        )

        console.log(
          '[POC] A balance BEFORE =',
          staleBefore.toString(),
        )

        console.log(
          '[POC] B balance BEFORE =',
          intendedBefore.toString(),
        )

        expect(
          victimBefore,
        ).toBe(
          1n,
        )

        expect(
          staleBefore,
        ).toBe(
          0n,
        )

        expect(
          intendedBefore,
        ).toBe(
          0n,
        )

        /*
         * Open the real transfer page and let all normal owner,
         * transfer-role and detach-target reads settle BEFORE
         * installing the RPC interception.
         */
        await page.goto(
          `${PORTAL_APP_URL}/${name}/ownership/transfer`,
        )

        await expect(
          page.getByRole(
            'heading',
            {
              name:
                'Transfer ownership',
            },
          ),
        ).toBeVisible({
          timeout:
            30_000,
        })

        const recipientInput =
          page.getByPlaceholder(
            'ENS name or address',
          )

        await expect(
          recipientInput,
        ).toBeVisible({
          timeout:
            30_000,
        })

        /*
         * Start with stale A.
         */
        await recipientInput.fill(
          staleRecipient,
        )

        await expect(
          page
            .getByText(
              staleRecipient,
              {
                exact:
                  true,
              },
            )
            .first(),
        ).toBeVisible({
          timeout:
            15_000,
        })

        const transferButton =
          page.getByRole(
            'button',
            {
              name:
                'Transfer name',
            },
          )

        await expect(
          transferButton,
        ).toBeEnabled({
          timeout:
            30_000,
        })

        /*
         * Keep the test to ONE irreversible transfer transaction.
         *
         * A newly registered name can expose "Detach the resolver"
         * because the resolver pointer is populated. The switch defaults
         * to true. Disable it before starting preparation so the saved
         * plan contains only transfer-token.
         */
        const detachResolver =
          page.getByRole(
            'switch',
            {
              name:
                /Detach the resolver/i,
            },
          )

        if (
          await detachResolver
            .isVisible()
            .catch(
              () => false,
            )
        ) {
          const checked =
            await detachResolver
              .isChecked()

          console.log(
            '[POC] detach-resolver visible = true',
          )

          console.log(
            '[POC] detach-resolver initially checked =',
            checked,
          )

          if (
            checked
          ) {
            await detachResolver.click()
          }

          await expect(
            detachResolver,
          ).not.toBeChecked()
        } else {
          console.log(
            '[POC] detach-resolver visible = false',
          )
        }

        await expect(
          transferButton,
        ).toBeEnabled({
          timeout:
            15_000,
        })

        console.log('')
        console.log(
          '=== PORTAL INITIAL STATE ===',
        )

        console.log(
          '[POC] Portal visible recipient BEFORE click =',
          await recipientInput.inputValue(),
        )

        expect(
          await recipientInput
            .inputValue(),
        ).toBe(
          staleRecipient,
        )

        /*
         * Deterministically hold only the production getTokenId(uint256)
         * RPC used inside useTransferName.prepareMutation.
         */
        let tokenCallSeenResolve!:
          () => void

        const tokenCallSeen =
          new Promise<void>(
            (resolve) => {
              tokenCallSeenResolve =
                resolve
            },
          )

        let releaseTokenCall!:
          () => void

        const tokenCallRelease =
          new Promise<void>(
            (resolve) => {
              releaseTokenCall =
                resolve
            },
          )

        let interceptedCalls =
          0

        await page.route(
          '**/rpc',
          async (route) => {
            let payload:
              unknown

            try {
              payload =
                route
                  .request()
                  .postDataJSON()
            } catch {
              await route.continue()
              return
            }

            if (
              interceptedCalls ===
                0 &&
              containsTargetGetTokenIdCall(
                payload,
              )
            ) {
              interceptedCalls++

              console.log(
                '[POC] 🚧 intercepted production getTokenId(uint256) RPC',
              )

              tokenCallSeenResolve()

              /*
               * Deliberately pause the real response. No response data is
               * changed and no fake token ID is supplied.
               */
              await tokenCallRelease

              console.log(
                '[POC] ▶ releasing unchanged getTokenId RPC',
              )
            }

            await route.continue()
          },
        )

        /*
         * Captures recipient A in:
         *
         * startTransfer({
         *   recipient: A,
         *   options
         * })
         */
        await transferButton.click()

        await Promise.race([
          tokenCallSeen,

          page
            .waitForTimeout(
              20_000,
            )
            .then(() => {
              throw new Error(
                'Timed out waiting for real getTokenId RPC interception',
              )
            }),
        ])

        await expect(
          page.getByRole(
            'button',
            {
              name:
                'Preparing…',
            },
          ),
        ).toBeVisible({
          timeout:
            10_000,
        })

        /*
         * THE UI BUG:
         * preparation is locked, but recipient input is not.
         */
        await expect(
          recipientInput,
        ).toBeEnabled()

        console.log(
          '[POC] recipient input remains ENABLED while Preparing = true',
        )

        /*
         * Victim corrects A -> B while the OLD preparation is pending.
         */
        await recipientInput.fill(
          intendedRecipient,
        )

        await expect(
          recipientInput,
        ).toHaveValue(
          intendedRecipient,
        )

        await expect(
          page
            .getByText(
              intendedRecipient,
              {
                exact:
                  true,
              },
            )
            .first(),
        ).toBeVisible({
          timeout:
            15_000,
        })

        console.log(
          '[POC] Portal corrected visible recipient B =',
          await recipientInput.inputValue(),
        )

        expect(
          await recipientInput
            .inputValue(),
        ).toBe(
          intendedRecipient,
        )

        /*
         * Now complete the original A preparation request with the
         * untouched real Anvil response.
         */
        releaseTokenCall()

        const transactionDialog =
          page.locator(
            '[data-slot="dialog-content"][data-state="open"]',
          )

        await expect(
          transactionDialog,
        ).toHaveCount(
          1,
          {
            timeout:
              30_000,
          },
        )

        await expect(
          transactionDialog,
        ).toBeVisible({
          timeout:
            30_000,
        })

        /*
         * The held RPC has completed now; remove the route interception.
         */
        await page.unroute(
          '**/rpc',
        )

        const dialogText =
          (
            await transactionDialog
              .innerText()
          )
            .replace(
              /\s+/g,
              ' ',
            )
            .trim()

        console.log('')
        console.log(
          '=== TRANSACTION MODAL ===',
        )

        console.log(
          '[POC] modal text =',
          dialogText,
        )

        console.log(
          '[POC] modal discloses stale A =',
          dialogText
            .toLowerCase()
            .includes(
              staleRecipient
                .toLowerCase(),
            ),
        )

        console.log(
          '[POC] modal discloses intended B =',
          dialogText
            .toLowerCase()
            .includes(
              intendedRecipient
                .toLowerCase(),
            ),
        )

        expect(
          dialogText
            .toLowerCase(),
        ).not.toContain(
          staleRecipient
            .toLowerCase(),
        )

        expect(
          dialogText
            .toLowerCase(),
        ).not.toContain(
          intendedRecipient
            .toLowerCase(),
        )

        /*
         * Even after the stale preparation completes, the underlying
         * Portal form still holds the victim's corrected B.
         */
        expect(
          await recipientInput
            .inputValue(),
        ).toBe(
          intendedRecipient,
        )

        console.log(
          '[POC] Portal field immediately before wallet flow =',
          await recipientInput.inputValue(),
        )

        /*
         * Run the REAL Portal transaction flow and authorize the
         * headless wallet.
         */
        await driveTransactionsToSuccess(
          page,
          wallet,
          [
            `transfer-${name}-transfer-token`,
          ],
        )

        /*
         * Authoritative post-condition: ERC-1155 balances on Anvil.
         */
        const victimAfter =
          await readTokenBalance(
            victim,
            tokenId,
          )

        const staleAfter =
          await readTokenBalance(
            staleRecipient,
            tokenId,
          )

        const intendedAfter =
          await readTokenBalance(
            intendedRecipient,
            tokenId,
          )

        console.log('')
        console.log(
          '=== AUTHORITATIVE ON-CHAIN POST-STATE ===',
        )

        console.log(
          '[POC] victim balance AFTER =',
          victimAfter.toString(),
        )

        console.log(
          '[POC] stale A balance AFTER =',
          staleAfter.toString(),
        )

        console.log(
          '[POC] intended B balance AFTER =',
          intendedAfter.toString(),
        )

        /*
         * This is the Critical impact.
         *
         * The user-visible corrected recipient was B.
         *
         * But ownership moved to the old replaced address A.
         */
        expect(
          victimAfter,
        ).toBe(
          0n,
        )

        expect(
          staleAfter,
        ).toBe(
          1n,
        )

        expect(
          intendedAfter,
        ).toBe(
          0n,
        )

        console.log('')
        console.log(
          '============================================================',
        )

        console.log(
          '🚨 CRITICAL BROWSER-TO-CHAIN VULNERABILITY CONFIRMED',
        )

        console.log(
          '============================================================',
        )

        console.log(
          'Victim initially entered recipient A.',
        )

        console.log(
          'Portal began real asynchronous getTokenId preparation for A.',
        )

        console.log(
          'Recipient input remained editable while Portal showed Preparing…',
        )

        console.log(
          'Victim replaced A with intended recipient B.',
        )

        console.log(
          'Portal visibly displayed B before wallet approval.',
        )

        console.log(
          'The transaction modal did not disclose the snapshotted recipient.',
        )

        console.log(
          'The real Portal transaction succeeded on Anvil.',
        )

        console.log(
          'ERC1155 balance proves the ENS name moved to replaced address A.',
        )

        console.log(
          'Corrected intended recipient B received nothing.',
        )

        console.log(
          '============================================================',
        )
      },
    )
  },
)

```

## Command

The successful run used the following local services and test command.

### Terminal 1 — Anvil

```bash
cd /workspaces/audit-comp-ens
nvm use 22

docker compose -f e2e/infra/docker-compose.yml down --remove-orphans
docker compose -f e2e/infra/docker-compose.yml up -d anvil

curl -s   -H 'content-type: application/json'   --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'   http://127.0.0.1:8545
```

Expected chain:

```
0xaa36a7
```

### Terminal 2 — Portal

```bash
cd /workspaces/audit-comp-ens
nvm use 22

cp apps/portal/.env.ci apps/portal/.env.local

pnpm --filter portal dev --host 0.0.0.0
```

Portal readiness check:

```bash
curl -I http://127.0.0.1:3001
```

Observed:

```
HTTP/1.1 200 OK
```

### Terminal 3 — PoC

```bash
cd /workspaces/audit-comp-ens
nvm use 22

export ANVIL_RPC_URL=http://127.0.0.1:8545
export PORTAL_APP_URL=http://127.0.0.1:3001

pnpm --filter @ens-apps/e2e exec playwright test   --config=projects/portal/playwright.config.ts   stale-recipient-transfer.poc.spec.ts   --workers=1   --retries=0   --reporter=line
```

## Observed Output

```
Running 1 test using 1 worker
[portal-e2e] › projects/portal/tests/stale-recipient-transfer.poc.spec.ts:368:5 › CRITICAL stale transfer recipient › Portal displays corrected B but transfers ENS ERC1155 to stale A

============================================================
CRITICAL STALE RECIPIENT — REAL BROWSER / REAL CHAIN
============================================================
[makeName] registering critical-stale-recipient-1788348652.eth (duration=2419200s, desiredGapPastExpiry=0s)
[makeName] waiting 61s for MIN_COMMITMENT_AGE
[makeName] ✅ registered critical-stale-recipient-1788348652.eth
[time] sync — browser clock set to 2026-09-02T11:32:56.000Z
[makeName] ready: critical-stale-recipient-1788348652.eth
[POC] name = critical-stale-recipient-1788348652.eth
[POC] victim = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[POC] stale recipient A = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[POC] intended recipient B = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
[POC] tokenId = 26100276157303549828673643272941724725817132847856319908496485817464408506368

=== ON-CHAIN PRE-STATE ===
[POC] victim balance BEFORE = 1
[POC] A balance BEFORE = 0
[POC] B balance BEFORE = 0
[POC] detach-resolver visible = true
[POC] detach-resolver initially checked = true

=== PORTAL INITIAL STATE ===
[POC] Portal visible recipient BEFORE click = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[POC] 🚧 intercepted production getTokenId(uint256) RPC
[POC] recipient input remains ENABLED while Preparing = true
[POC] Portal corrected visible recipient B = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
[POC] ▶ releasing unchanged getTokenId RPC

=== TRANSACTION MODAL ===
[POC] modal text = Transaction overview Transfer name Not Started Est. Cost Not yet Start Close
[POC] modal discloses stale A = false
[POC] modal discloses intended B = false
[POC] Portal field immediately before wallet flow = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
[POC] clicking modal primary = Start
[POC] clicking Open wallet
[BROWSER] 📊 [TRANSACTION MANAGER] Transaction transfer-critical-stale-recipient-1788348652.eth-transfer-token state: submitting
[BROWSER] 📊 [TRANSACTION MANAGER] Transaction transfer-critical-stale-recipient-1788348652.eth-transfer-token state: pending
[BROWSER] 📊 [TRANSACTION MANAGER] Transaction transfer-critical-stale-recipient-1788348652.eth-transfer-token state: success
[POC] clicking modal primary = Done

=== AUTHORITATIVE ON-CHAIN POST-STATE ===
[POC] victim balance AFTER = 0
[POC] stale A balance AFTER = 1
[POC] intended B balance AFTER = 0

============================================================
🚨 CRITICAL BROWSER-TO-CHAIN VULNERABILITY CONFIRMED
============================================================
Victim initially entered recipient A.
Portal began real asynchronous getTokenId preparation for A.
Recipient input remained editable while Portal showed Preparing…
Victim replaced A with intended recipient B.
Portal visibly displayed B before wallet approval.
The transaction modal did not disclose the snapshotted recipient.
The real Portal transaction succeeded on Anvil.
ERC1155 balance proves the ENS name moved to replaced address A.
Corrected intended recipient B received nothing.
============================================================
  1 passed (26.2s)

=== POC SHA256 ===
a540fd562cb90e48f1a14af7232f92b4d8f42cc44c9a71fb08ce63155295045b  e2e/projects/portal/tests/stale-recipient-transfer.poc.spec.ts

============================================================
✅ CRITICAL STALE-RECIPIENT BROWSER-TO-CHAIN POC PASSED
============================================================
```

## What the PoC Proves

1. A real V2 `.eth` name is registered on the repository's local Sepolia/Anvil environment.
2. The victim is the authoritative ERC-1155 holder before transfer.
3. Stale recipient A and intended recipient B both have zero token balance initially.
4. Production Portal initially displays A as the transfer recipient.
5. The user starts the normal transfer flow.
6. The PoC intercepts the real production `getTokenId(uint256)` RPC used during transfer preparation.
7. The RPC response is delayed but not modified.
8. Portal enters **Preparing…**.
9. The recipient input remains enabled while that preparation is pending.
10. The user changes A to B.
11. Production Portal visibly displays B.
12. The unchanged `getTokenId` request is released.
13. The Portal transaction modal opens.
14. Portal's modal discloses neither A nor B.
15. The Portal field immediately before the wallet flow still contains B.
16. The real Portal transaction flow is started.
17. The real transaction manager reaches `submitting`.
18. The transaction reaches `pending`.
19. The transaction reaches `success`.
20. A direct post-transaction ERC-1155 read shows the victim balance is `0`.
21. A direct post-transaction ERC-1155 read shows stale recipient A balance is `1`.
22. A direct post-transaction ERC-1155 read shows intended recipient B balance is `0`.
23. The Playwright test exits successfully with `1 passed (26.2s)`.

The demonstrated invariant violation is:

```
Portal-visible transfer recipient
!=
recipient receiving the ENS ERC-1155 token

Portal immediately before wallet flow:
B

Portal transaction modal:
A not disclosed
B not disclosed

Chain after successful transfer:
A owns token
B owns nothing
```

This finding is therefore a **Portal transaction-construction vulnerability where asynchronous preparation preserves an earlier recipient after the application has visibly accepted a corrected destination, causing the real ENS ownership transfer to go to the replaced address**.
