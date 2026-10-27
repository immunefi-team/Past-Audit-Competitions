# 92461 w a high portal registration shows 8 01 for name b charges 160 109598 usdc for name a

## #92461 \[W\&A-High] Portal registration shows $8.01 for name B, charges 160.109598 USDC for name A

**Submitted on Sep 10th 2026 at 00:23:40 UTC by @zaaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92461
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

## Brief/Intro

The Portal registration route updates the displayed name and price without rebinding the registration already held by the transaction machine. A registration started for the four-character name `qzm2.eth` at `$160.11` can continue after the same route displays the long name `low-price-mturyzm2.eth`, `Registering...`, and `Total $8.01`; approving the pending sequence then registers `qzm2.eth` and debits `160.109598 USDC` — `152.099598 USDC` more than the total on screen. A fresh long-name control on the same fork displayed `$8.01`, registered the name it displayed, and debited `8.005501 USDC`. Afterwards `qzm2.eth` is owned by the payer and `low-price-mturyzm2.eth` is still available.

## Vulnerability Details

This path spans the in-scope `apps/portal` application and `packages/transaction-manager`. `/register` reads the current search parameter and passes it to `RegisterName`:

```tsx
const { name } = Route.useSearch()

return (
  <RequireWalletConnected>
    <RegisterName name={name} />
  </RequireWalletConnected>
)
```

`RegisterName` renders its current quote from the current `name` prop:

```tsx
<RegisterNameCheckoutSummary name={name} duration={duration} />
<PaymentTokenSection
  nativePrice={registrationData.nativePrice}
  usdPrice={registrationData.usdPrice}
  isRegistering={isRegistering}
  onRegister={startRegistration}
/>
```

Registration actor creation receives only the chain. Its identity and lifetime are not bound to `name`:

```typescript
const actor: RegistrationMachineActor = useActorRef(registrationMachine, {
  input: { chainId },
})
```

`START_REGISTRATION` copies the name into machine context only when it is first accepted from `idle`:

```typescript
START_REGISTRATION: {
  actions: assign({
    name: ({ event }) => event.name,
    duration: ({ event }) => event.duration,
    selectedToken: ({ event }) => event.token,
    tokenPrice: ({ event }) => event.price,
  }),
}
```

Later registration construction uses that retained context:

```typescript
input: ({ context }) => ({
  name: context.name,
  commitment: context.commitment!,
  signer: context.signer!,
  duration: context.duration,
  selectedToken: context.selectedToken,
  owner: context.ownerAddress ?? context.accountAddress!,
  publicClient: context.publicClient!,
  resolverAddress: context.resolverAddress!,
  id: REGISTRATION_TX_IDS.register,
})
```

Changing only the `name` search parameter therefore produces two simultaneous identities: the route, heading, price and `Registering...` state use B, while the active machine, final calldata and debit still use A.

**Missing control:** The active registration actor is not keyed to the current ENS name, and a same-route name change does not cancel or reinitialize it before current-name UI and pricing are rendered.

**Where:** `apps/portal/src/routes/register/index.tsx` supplies the current search name; `apps/portal/src/features/register/components/RegisterName.tsx` and `RegisterNameSummary.tsx` render that name and quote; `apps/portal/src/features/register/hooks/useRegistrationTransactions.ts` creates the actor without a name identity; `packages/transaction-manager/src/machines/registration/registration.machine.ts` retains and later submits the earlier name.

**Discriminating control:** In the route-change arm, B displayed `$8.01` while the final request decoded to A and the USDC debit was `160.109598`. In the immediately following fresh-name arm, the displayed long name, decoded final label and ownership result aligned, while `$8.01` corresponded to a debit of `8.005501 USDC`. Both arms used the same payer, token, registrar, one-year duration, application revision and fork session.

**Excluded-premise check:** The sequence uses the ordinary registration page, in-application search and a connected account on an isolated Sepolia fork. It does not require phishing, a malicious contract, a modified third-party response, another user's name, an approval bypass or public-chain writes. The navigation warning and all three wallet authorizations remain in the reproduction and limitations below.

Before B opens, the navigation warning says:

```
Your registration is in progress. Leaving may interrupt it and you could lose your commitment. Are you sure you want to leave?
```

That warning does not say that A will continue behind B's registration page or that B's displayed total will not govern the resulting debit.

**The programme's own published known issue draws this exact line.** QA-03, "Minor price mismatch in registration and extension modals", reads in full: "The per-year discounted price and the displayed total do not always agree (for example, a 7-year discount of $7.00 per year showing a total of $14.02). Affects both the registration and extension modals. Display-level only; **a mismatch between the displayed total and the amount actually charged on-chain would be a new finding**." That is the boundary this reproduction crosses and measures: the page displays `Total $8.01` for the name it is rendering, and the approved sequence debits `160.109598 USDC`. The discrepancy is not between two displayed numbers; it is between the displayed total and the amount actually taken on-chain.

WEB-1233 concerns a stale browser-tab title. Here the URL and body are current for B, while the active registration and on-chain result remain A. R3-07 concerns completed-transaction bookkeeping under a reused transaction identifier. Clearing that completion marker did not change this result; recreating the registration actor on the name boundary changed the machine context from A to B. This path does not depend on terminal bookkeeping.

## Impact Details

**Attacker:** A connected name registrant can reach this state using ordinary controls after starting A and navigating to B while the registration remains pending. No external attacker, other-user account or wallet-approval bypass was measured.

**Victim:** The same connected registrant. Long name B and `$8.01` appear on the live page, but the approved sequence charges for and registers the retained four-character name A.

**Gain (measured):** No external party gain is claimed. Measured monetary effect on the registrant was a `160.109598 USDC` debit while the current page displayed `Total $8.01`, an excess of `152.099598 USDC` over that displayed total. By comparison, the aligned control debited `8.005501 USDC`.

**Executed end-to-end:** yes

**Verified out of band:** The payer's USDC balance changed from `19663769802` to `19503660204` atomic units. Its successful final receipt independently emitted a USDC `Transfer` of `0x98b141e`, or `160109598` atomic units, from the payer. Direct registrar and ownership reads showed A changing from available/unowned to unavailable/owned by the payer, while B remained available and unowned. In the control, the balance changed from `19503660204` to `19495654703`, and the final receipt emitted `0x7a277d`, or `8005501` atomic units.

Exact in-scope impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

The measured impact matches the second example: an incorrect amount shown to the user, which the user then authorizes. The registrar address is the same in both arms, so nothing here turns on an unintended recipient; what is wrong is the amount. The page states the total is `$8.01` and the approved sequence takes `160.109598 USDC` — twenty times that figure, and `152.099598 USDC` more than the number the user was shown.

The conditions the reproduction requires, stated in full: the registrant began A deliberately and ends up owning it, the registrar address is unchanged, B stays available, no direct theft is claimed, the generic navigation warning is accepted, and three wallet authorizations are given. The final registrar calldata carries plaintext `qzm2`, so an ABI-aware wallet can expose the retained label; no production wallet confirmation display was measured. None of these change what was measured on the page and on the chain: `$8.01` displayed, `160.109598 USDC` debited.

## References

* Registration route: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/register/index.tsx>
* Current-name rendering and navigation warning: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/register/components/RegisterName.tsx>
* Current-name price summary: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/register/components/RegisterNameSummary.tsx>
* Registration actor creation and start event: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/register/hooks/useRegistrationTransactions.ts>
* Registration machine context and submission: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/transaction-manager/src/machines/registration/registration.machine.ts>
* Registration transaction construction: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/transaction-manager/src/machines/registration/registration.actors.ts>
* Competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Public known issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>

### Proof of Concept

## Proof of Concept

This reproduction uses the audited revision and a Sepolia fork pinned at block `11671311` (hash `0x0f7e24b54d145a4ca55348ab07ede122eb77161cd9db355aaeca0e2a32d2058b`). The registration flow advances the local chain while waiting through the commitment period, so the recorded receipts land later, at `11671343`–`11671345` for the test arm and `11671347`–`11671349` for the control. Its standard local development account is public test material:

```
User A:      0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Private key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
Registrar:   0xa88553F454b77203B0D036A05c894d555EAAa2Cc
USDC:        0x768f42455a2d082e23ceef7d51e5787c82d67a39
Name A:      qzm2.eth
Name B:      low-price-mturyzm2.eth
Control:     fresh-low-mtus0hlo.eth
Duration:    one year
```

1. As User A, obtain the audited branch and verify the exact revision:

   ```bash
   git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git ens-audit
   git -C ens-audit rev-parse HEAD
   ```

   The tested competition snapshot prints `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
2. As User A, install the repository dependencies from `ens-audit`:

   ```bash
   cd ens-audit && pnpm install --frozen-lockfile
   ```
3. As User A, start a local Sepolia fork with `anvil` in one terminal:

   ```bash
   anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number 11671311 --chain-id 11155111
   ```
4. On the fork, User A already holds test USDC. Approve the registrar before recording the debit window:

   ```bash
   cast send 0x768f42455a2d082e23ceef7d51e5787c82d67a39 'approve(address,uint256)' 0xa88553F454b77203B0D036A05c894d555EAAa2Cc 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
   ```
5. As User A, start Portal with `pnpm --filter portal dev` from a second terminal and use the URL printed by the server:

   ```bash
   VITE_SEPOLIA_RPC_URL=/rpc pnpm --filter portal dev
   ```
6. As User A, import the public development key listed above into a wallet connected to the local fork, connect it to Portal, and record the starting USDC balance with `cast call`:

   ```bash
   cast call 0x768f42455a2d082e23ceef7d51e5787c82d67a39 'balanceOf(address)(uint256)' 0xf39Fd6e51aad88F6F4ce6ab8827279cffFb92266
   ```

   In that connected Portal session, open `/register?name=qzm2.eth`, select USDC, confirm the one-year `$160.11` quote, click `Register`, then `Start`, then open the wallet. Leave the first request pending. The browser interaction cannot be reduced to a single request; the complete request bodies and state checks are included below.
7. As User A, close only Portal's transaction dialog. Use Portal's visible search field to select `low-price-mturyzm2.eth` and accept the warning. Confirm that the URL and heading identify B, the button says `Registering...`, and the summary says `Total $8.01`. The attached `01-b-shows-8-01-while-retained-registration-runs.png` records this exact state.
8. As User A, authorize the pending wallet request and then each of the two requests that follow. Wait for the registration to finish, then read the balance and availability results with `cast call`:

   ```bash
   cast call 0x768f42455a2d082e23ceef7d51e5787c82d67a39 'balanceOf(address)(uint256)' 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
   cast call 0xa88553F454b77203B0D036A05c894d555EAAa2Cc 'isAvailable(string)(bool)' qzm2
   cast call 0xa88553F454b77203B0D036A05c894d555EAAa2Cc 'isAvailable(string)(bool)' low-price-mturyzm2
   ```
9. As User A, decode the successful final request. The string below is the `input` of the mined final transaction `0x6a008f2a03da18ef060f1ea7eab6107b29c96366ae07fd98d464e2632b185db2` listed in the summary further down, copied byte for byte; this check prints the dynamic string it carries:

   ```bash
   python3 - <<'PY'
   calldata='0xcff3e7c20000000000000000000000000000000000000000000000000000000000000100000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266d08e2a8c971d0bcc9784b29b393066efd377afaeea4840331fa3ebc57fe686eb0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a98fa658bf057493f3e1e5a971ff544c18cde1b10000000000000000000000000000000000000000000000000000000001e187e0000000000000000000000000768f42455a2d082e23ceef7d51e5787c82d67a3900000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004717a6d3200000000000000000000000000000000000000000000000000000000'
   raw=bytes.fromhex(calldata[2:]); args=raw[4:]; offset=int.from_bytes(args[:32],'big'); length=int.from_bytes(args[offset:offset+32],'big')
   print(args[offset+32:offset+32+length].decode())
   PY
   ```

   Observed output:

   ```
   qzm2
   ```
10. As User A, repeat the registration from a fresh Portal page with `fresh-low-mtus0hlo.eth`, without changing the route during the flow. The fresh page displays `$8.01`; the displayed name is also the final decoded name, it becomes owned, and the measured debit is `8.005501 USDC`. Record the before and after balance using the same `balanceOf` command from step 8.

Compact paired measurement:

```json
{
  "test": {
    "nameA": "qzm2.eth",
    "nameBDisplayed": "low-price-mturyzm2.eth",
    "displayedAQuoteUsd": 160.11,
    "displayedBQuoteUsd": 8.01,
    "displayedBRegistering": true,
    "usdcBeforeAtomic": "19663769802",
    "usdcAfterAtomic": "19503660204",
    "actualDebitAtomic": "160109598",
    "actualDebitUsd": 160.109598,
    "finalDecodedLabel": "qzm2",
    "aAvailableAfter": false,
    "aOwnerAfter": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "bAvailableAfter": true,
    "bOwnerAfter": "0x0000000000000000000000000000000000000000"
  },
  "freshControl": {
    "displayedName": "fresh-low-mtus0hlo.eth",
    "displayedQuoteUsd": 8.01,
    "usdcBeforeAtomic": "19503660204",
    "usdcAfterAtomic": "19495654703",
    "actualDebitAtomic": "8005501",
    "actualDebitUsd": 8.005501,
    "finalDecodedLabel": "fresh-low-mtus0hlo",
    "availableAfter": false,
    "ownerAfter": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
  },
  "excessDebitOverDisplayedBQuoteUsd": 152.099598,
  "testToControlDebitRatio": 19.999947286247295
}
```

All three test-arm request bodies and the receipt fields used to verify execution were:

```json
[
  {
    "hash": "0x19b607202ec8e7d47e77a25124f8b1dcc81a3b79a7e01a4903d8f0ad0121e99c",
    "selector": "0x5d84121a",
    "receipt": {
      "status": "success",
      "blockNumber": "11671343",
      "gasUsed": "176703"
    }
  },
  {
    "hash": "0x727c6642dede82cb2b6c3fd971e12f34d41a1c666ad579ca9533f41894d115da",
    "selector": "0xf14fcbc8",
    "receipt": {
      "status": "success",
      "blockNumber": "11671344",
      "gasUsed": "45433"
    }
  },
  {
    "hash": "0x6a008f2a03da18ef060f1ea7eab6107b29c96366ae07fd98d464e2632b185db2",
    "selector": "0xcff3e7c2",
    "receipt": {
      "status": "success",
      "blockNumber": "11671345",
      "gasUsed": "216004"
    },
    "finalDecodedLabel": "qzm2",
    "usdcTransfer": {
      "address": "0x768f42455a2d082e23ceef7d51e5787c82d67a39",
      "data": "0x00000000000000000000000000000000000000000000000000000000098b141e",
      "atomic": "160109598"
    }
  }
]
```

Aligned-control request and receipt summary:

```json
[
  {
    "hash": "0xdc6d969ee3132a719393b5d5635da8633052ba555378a9252edd3841be2b7c84",
    "selector": "0x5d84121a",
    "receipt": {
      "status": "success",
      "blockNumber": "11671347",
      "gasUsed": "176703"
    }
  },
  {
    "hash": "0x89d559c1553a82a97d5ad3ab52760965a538e30bf669f231419f0a46b5b6dadd",
    "selector": "0xf14fcbc8",
    "receipt": {
      "status": "success",
      "blockNumber": "11671348",
      "gasUsed": "45433"
    }
  },
  {
    "hash": "0xc359cd3b4a5a9fabfd1e49a7b87be5a59faeefeba6576972ee7e0c8927c88512",
    "selector": "0xcff3e7c2",
    "receipt": {
      "status": "success",
      "blockNumber": "11671349",
      "gasUsed": "220491"
    },
    "finalDecodedLabel": "fresh-low-mtus0hlo",
    "usdcTransfer": {
      "address": "0x768f42455a2d082e23ceef7d51e5787c82d67a39",
      "data": "0x00000000000000000000000000000000000000000000000000000000007a277d",
      "atomic": "8005501"
    }
  }
]
```

The attachments are:

* `01-b-shows-8-01-while-retained-registration-runs.png` — the load-bearing browser state: the heading identifies `low-price-mturyzm2.eth`, the summary reads `Total $8.01`, the registration status reads `Registering...`, and the USDC balance shown is `19,663.77`, matching the pre-transaction reading below.
* `02-displayed-8-01-charged-160-usdc.png` — the same run's saved measurement read back: the displayed totals for both names, the USDC balance before and after, the `160.109598` debit, the decoded final label `qzm2`, the ownership readback for both names, and the USDC `Transfer` log carried by the mined registration.

Bind the actor instance to the canonical name. A name change should either recreate the registration actor or require explicit completion/cancellation of the existing registration before rendering a new name as active. A regression test should start A, hold its first request, change the current route to B, and assert that B cannot display an active registration backed by A's context. Keep the fresh-name path binding display, quote, calldata and debit to the same name.
