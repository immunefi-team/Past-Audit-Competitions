# 93029 w a high manager writes l2 address records to mainnet derived coin types while the explorer reads sepolia derived ones so all five l2 rows render the owner s coin 60 address and a sender s&#x20;

**Submitted on Sep 14th 2026 at 01:10:35 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93029
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's address page shows five L2 rows (Optimism, Arbitrum, Base, Linea and Scroll) and keys each one on a Sepolia chain id. That is correct for this deployment, and the source comment says so. The Manager offers the owner a slot for each of those five networks and keys each one on the mainnet chain id instead. An L2 address set in the Manager therefore goes into a coin type the Explorer never reads. Each L2 row falls back to the ENSIP-19 default and shows the owner's coin-60 address. Someone reading that page pays it on the L2. When the owner's coin-60 record is a smart account, which is a contract address, there is no code at that address on the L2 and no private key for it, so the payment cannot be recovered.

## Vulnerability Details

All paths are at pin `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### 1. The Explorer keys its five L2 rows on Sepolia chain ids, on purpose

`apps/portal/src/features/forward-resolution/components/AddressResolution/networks.ts:12-17`

```ts
  /**
   * ENSIP-11 / SLIP-44 coin type the address record is keyed on. The explorer
   * runs against Sepolia, so L2 coin types derive from the TESTNET chain ids
   * (e.g. Base → `0x80000000 | 84532`) — that's what the deployed Sepolia
   * UniversalResolver / L2 reverse registrars verify against.
   */
```

The row set is `L2_CHAIN_IDS = [10, 42161, 8453, 59144, 534352]` (`networks.ts:29`). Each row is built with `getCoinTypeForReverseRegistrarChainId(chainId, 'sepolia')` (`:44-46`). For a Sepolia deployment this is the right choice.

### 2. The Manager offers a slot for each of those five, keyed on mainnet

Three of them come from the curated L2 button list at `apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/addresses/addressPickerRecords.ts:21-28`:

```ts
export const evmChainOptions: readonly AddressOption[] = [
  { coinType: 2147483658, label: 'Optimism' },
  { coinType: 2147492101, label: 'Base' },
  { coinType: 2147525809, label: 'Arbitrum' },
  { coinType: 2147483972, label: 'ZKsync' },
  { coinType: 2147483785, label: 'Polygon' },
  { coinType: BSC_COIN_TYPE, label: 'BNB' },
]
```

The other two, Linea and Scroll, are reached through the "Add More" dialog on the same tab. `ChainPickerDialog.tsx:58-59` calls `getPickerRecordGroups` (`addressPickerRecords.ts:54-106`) and renders both groups it returns (`:112-133`). The records it searches are `addressRecords`, built from the ENS address encoder's `coinTypeToNameMap` (`data/records/address.tsx:5-12`), which uses mainnet chain ids. Both networks also sit in the tab's own `EVM_PICKER_POPULAR_COIN_TYPES` (`:37-43`) as `2147542792` (Linea) and `2148018000` (Scroll).

ENSIP-11 encodes `coinType = 0x80000000 | chainId`. The PoC decodes both sides back to the chain each one is keyed on:

| row      | Explorer coin type | Explorer chain          | Manager coin type | Manager chain           | offered via       |
| -------- | ------------------ | ----------------------- | ----------------- | ----------------------- | ----------------- |
| Optimism | 2158639068         | 11155420 (OP Sepolia)   | 2147483658        | 10 (OP mainnet)         | `evmChainOptions` |
| Arbitrum | 2147905262         | 421614 (Arb Sepolia)    | 2147525809        | 42161 (Arb mainnet)     | `evmChainOptions` |
| Base     | 2147568180         | 84532 (Base Sepolia)    | 2147492101        | 8453 (Base mainnet)     | `evmChainOptions` |
| Linea    | 2147542789         | 59141 (Linea Sepolia)   | 2147542792        | 59144 (Linea mainnet)   | picker search     |
| Scroll   | 2148017999         | 534351 (Scroll Sepolia) | 2148018000        | 534352 (Scroll mainnet) | picker search     |

Every pair differs, so no L2 record the Manager writes lands in a slot this deployment reads.

### 3. The Explorer falls back to the coin-60 / default record

`apps/portal/src/routes/$name/address.tsx:58-67`

```ts
function forwardAddress(
  network: ForwardResolutionNetwork,
  addressByCoinType: Map<number, string>,
): string | null {
  const own = addressByCoinType.get(network.coinType) ?? null
  if (own) return own
  return network.l2ChainId != null
    ? (addressByCoinType.get(DEFAULT_EVM_COIN_TYPE) ?? null)
    : null
}
```

The L2 slot the Explorer reads is empty, so each L2 row falls through to the ENSIP-19 default, which is the owner's ordinary EVM address. The page shows it as that network's address and gives no sign that it is a fallback.

The deployed resolver behaves the same way one level down, and the end-to-end rig below shows this on chain. A `PermissionedResolver` returns the default EVM record for any EVM coin type that has no specific record, while a non-EVM coin type returns empty. So the Explorer's miss already resolves to the owner's coin-60 address at the resolver, before `forwardAddress` runs. The displayed result is the same either way, and it comes from the deployed contract as well as from the UI.

### 4. Why this loses money instead of just redirecting it

A plain EOA is controllable on every EVM chain, so a payment to it on the L2 still reaches the owner. A contract address is different. The Manager is built around a smart account (`packages/smart-account`, `useSmartAccountContext`), and an owner only sets a chain-specific record when their addresses on the two chains actually differ.

A contract address comes from `CREATE(deployer, nonce)`. Nobody has a private key for it. If the contract was never deployed at that address on the destination chain, three things hold at once: there is no code there, so nothing can be called; there is no key, because it is not a keypair address; and once the deployer's nonce on that chain has moved past the one that would produce it, the address can never be occupied, since nonces only increase. Funds sent there stay there. The PoC below shows each of the three.

## Impact Details

Selected impact: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient." The Explorer shows, for each L2 row, an address that is not the one the owner set for that network, and a sender pays it.

Measured on the rendered page, with an address record written for every L2 row: 5 of 5 rows show the coin-60 address and 0 of 5 show the address the owner set. In the control arm the same five records are written to the slots the Explorer reads, and the counts are 0 of 5 and 5 of 5. The fallback comes from the slot, not from rows nobody filled in.

The loss was executed: a sender paid 2 ETH on the L2 to the address the page showed, and the owner could not recover it. Each of the three locks was measured separately:

| lock                  | measured                                                                                            |
| --------------------- | --------------------------------------------------------------------------------------------------- |
| nothing to call       | the address has no code on the L2 (`owner_coin60_code_on_l2`)                                       |
| no key                | it is a `CREATE` address, not a keypair                                                             |
| can never be occupied | the deployer's nonce there has passed it, and a rescue deploy provably lands at a different address |

Two preconditions apply, and the rigs measure both.

First, the owner's L2 address has to differ from their coin-60 address, otherwise the fallback shows the right address and nothing is lost. The Manager's own Addresses tab cannot create that difference from a clean profile. The network chip calls `onSetChainToEthAddress` (`EvmAddressSection.tsx:75-79`), which sets the L2 row equal to coin 60, and `applyEthAddressChange` (`AddressesTab.helpers.ts:89-106`) keeps every EVM row in sync when the ETH address changes. A chain-specific L2 record therefore has to exist already, from a migrated name or from another client, and the tab then shows it as an editable row the owner can change. The end-to-end rig runs both journeys. The clean-profile journey is its control.

Second, the coin-60 record has to be a contract that is not deployed at the same address on the destination chain. A plain EOA owner still receives the payment, so for them the wrong address is displayed but no money is lost. The affected population is smart-account owners, which is the account model the Manager is built on, plus anyone whose L2 address differs from their L1 one, which is the only reason to set a chain-specific record in the first place.

One further consequence is recorded here without being claimed: the Manager also offers ZKsync, Polygon and BNB, and the Explorer has no row for any of them, so a record set for those is never shown at all.

No attacker is involved. This is a mismatch between two of ENS's own applications, and the funds end up stranded with nobody receiving them. That is why the report claims impact 6285 only and not "Direct theft of user funds".

## References

Pin `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* `apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/addresses/addressPickerRecords.ts:21-28` (curated L2 list), `:37-43` (popular coin types), `:54-106` (`getPickerRecordGroups`)
* `apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/addresses/ChainPickerDialog.tsx:58-59, 112-133` (the "Add More" dialog renders the search groups)
* `apps/manager/src/features/profile/data/records/address.tsx:5-12` (`addressRecords` from `coinTypeToNameMap`)
* `apps/portal/src/features/forward-resolution/components/AddressResolution/networks.ts:12-17` (the comment), `:29` (`L2_CHAIN_IDS`), `:44-46` (row construction)
* `apps/portal/src/routes/$name/address.tsx:58-67` (`forwardAddress`), `:69-88` (`buildRow`)
* `apps/portal/src/lib/coinType.ts` (`DEFAULT_EVM_COIN_TYPE`, `MAINNET_COIN_TYPE`)
* `apps/manager/src/features/profile/service/profileRecordTransactions.ts:372` (`buildRecordsUpdateCalls`, the real save encoder)
* `apps/manager/.../tabs/addresses/EvmAddressSection.tsx:75-79` (the chip sets the L2 row equal to coin 60)
* `apps/manager/.../tabs/addresses/AddressesTab.helpers.ts:89-106` (`applyEthAddressChange` keeps EVM rows in sync)
* ENSIP-11 EVM coin types: <https://docs.ens.domains/ensip/11>
* ENSIP-19 reverse resolution / default EVM address: <https://docs.ens.domains/ensip/19>

## Proof of Concept

## Proof of Concept

There are two rigs. The first shows the rendering defect across all five rows. The second executes the whole path from the Manager's UI to a payment on the L2, with nothing seeded. Each is gated separately with `verify_poc.py --strict` and each passes 13 of 13 checks. Neither uses `vi.mock`, a stub, or `anvil_setCode`.

### Rig 1: what the Explorer renders (`verification/pocs/H-C7-62/`)

Four steps. The coin types come from the Manager's own picker code, and the displayed addresses are read out of the DOM of the real route.

1. `apps/manager`. For each of the five networks the Explorer renders, resolve the slot the user gets when they pick that network: the curated `evmChainOptions` entry matching the label they click, or else the record the tab's own `getPickerRecordGroups` search returns for that name. If the Manager offers the network by neither path, the test fails, since an unofferable slot would make the mismatch unreachable for that row. Each resolved coin type is decoded back to its chain id, and all five come out mainnet (`manager_slot_chainids`).
2. Two chains. Build the owner's coin-60 record as a real smart account: deploy it on L1, and on the L2 burn the deployer's nonce so that `CREATE` address can never be produced there again.
3. `apps/portal`. Mount the real `/$name/address` route, write an address record for every L2 row, and read the rendered table out of the DOM, so `forwardAddress` and `buildRow` execute. An earlier version of this rig re-implemented the ENSIP-19 fallback ternary inside the test, which only showed that the test agreed with itself. This version does not.
4. Two chains. A sender pays whatever the page rendered, on the L2, and the rig then tries to recover it.

The arms differ in one thing only: which coin-type slots those five records were written to, the ones the Manager writes or the ones the Explorer reads.

### Prerequisites

Two local chains. The Explorer and Manager steps need no network.

```bash
pnpm install                                   # from the repo root, once
anvil --fork-url <a Sepolia RPC> --port 8946   # stands in for L1
# the L2 chain is started by the rig if absent:
anvil --port 8947 --chain-id 8453              # stands in for the destination L2
```

### Steps to reproduce

```bash
bash verification/pocs/H-C7-62/run.sh exploit
bash verification/pocs/H-C7-62/run.sh control
```

### Measured result

Exploit, with the owner's five L2 addresses in the slots the Manager writes:

```
arm: exploit
manager_base_cointype: 2147492101
manager_base_chainid: 8453
manager_slots_for_portal_l2_rows: {"Optimism":2147483658,"Arbitrum":2147525809,"Base":2147492101,"Linea":2147542792,"Scroll":2148018000}
manager_slot_offered_via: {"Optimism":"evmChainOptions","Arbitrum":"evmChainOptions","Base":"evmChainOptions","Linea":"pickerSearch","Scroll":"pickerSearch"}
manager_slot_chainids: {"Optimism":10,"Arbitrum":42161,"Base":8453,"Linea":59144,"Scroll":534352}
portal_base_cointype: 2147568180
portal_base_chainid: 84532
write_slot: 2147492101
l2_rows_total: 5
l2_rows_rendering_eth_address: 5
l2_rows_rendering_intended_l2_address: 0
impact_row: 6285
SENDER_PAYS_ETH_NOT_L2: 1
owner_coin60_record: 0xB09297A56A54E37261cb68853d57CE99332707c1
owner_coin60_code_on_l1: 4743
owner_coin60_code_on_l2: 3
owner_intended_l2_address: 0xf672fbdf38D9d1f83E73a3530f2456B916F5462B
address_the_page_rendered: 0xb09297a56a54e37261cb68853d57ce99332707c1
sender_paid_on_l2_wei: 2000000000000000000
owner_recovered_the_funds: 0
deployer_nonce_on_l2: 1
rescue_deploy_would_land_at: 0x476f7303C0cEC50B794fF61e2262aEdA452Af3Ac
FUNDS_PERMANENTLY_UNRECOVERABLE: 1
```

Control, with the same five addresses in the slots the Explorer reads:

```
arm: control
manager_slots_for_portal_l2_rows: {"Optimism":2147483658,"Arbitrum":2147525809,"Base":2147492101,"Linea":2147542792,"Scroll":2148018000}
manager_slot_chainids: {"Optimism":10,"Arbitrum":42161,"Base":8453,"Linea":59144,"Scroll":534352}
portal_base_cointype: 2147568180
portal_base_chainid: 84532
write_slot: 2147568180
l2_rows_total: 5
l2_rows_rendering_eth_address: 0
l2_rows_rendering_intended_l2_address: 5
impact_row: 6285
SENDER_PAYS_ETH_NOT_L2: 0
owner_coin60_record: 0xA29a1b7D8Be80A179B372EFC4DE98e223e42246E
owner_coin60_code_on_l1: 4743
owner_coin60_code_on_l2: 3
owner_intended_l2_address: 0xd36C06a14b7c5d2585d9df3726ebEf137AC40537
address_the_page_rendered: 0xd36c06a14b7c5d2585d9df3726ebef137ac40537
sender_paid_on_l2_wei: 3000000000000000000
owner_recovered_the_funds: 1
deployer_nonce_on_l2: 1
rescue_deploy_would_land_at: 0x5b9f8f13b6446d545674B5FFEc141569C9113F11
FUNDS_PERMANENTLY_UNRECOVERABLE: 0
```

Read across the arms:

| measurement                                 | exploit                 | control                  |
| ------------------------------------------- | ----------------------- | ------------------------ |
| slots the five L2 records were written to   | the Manager's (mainnet) | the Explorer's (Sepolia) |
| L2 rows rendering the coin-60 address       | 5 of 5                  | 0 of 5                   |
| L2 rows rendering the address the owner set | 0 of 5                  | 5 of 5                   |
| owner recovered the sender's funds          | no                      | yes                      |
| funds permanently unrecoverable             | 1                       | 0                        |

`sender_paid_on_l2_wei` is 3 ETH in the control and 2 in the exploit because the balance is read before recovery. The control's rendered address is the owner's own L2 EOA, which step 2 had already seeded with 1 ETH. The payment is 2 ETH in both arms.

The control arm matters as much as the exploit arm. Same code path, same sender, same amount, and because the page rendered an address the owner holds a key for, the rig moves the money out again. The loss in the exploit arm comes from which slot was written, not from anything in the harness.

`FUNDS_PERMANENTLY_UNRECOVERABLE` is set only when all three of these hold: the recovery attempt failed, funds are sitting at the address, and the address has no code on the L2.

### Rig 2: the write path, executed end to end (`verification/pocs/H-C7-62-E2E/`)

Rig 1 seeds the address record into the Explorer's query cache, so on its own the "Manager writes it there" step is inferred. Rig 2 executes it.

1. A real `PermissionedResolver` is deployed through the Sepolia `VerifiableFactory`, using the same `deployProxy(impl, salt, initialize(...))` call the Manager uses.
2. The real `AddressesTab` is mounted behind the real `editProfileDialogMachine`, opened the way `EditProfileDialog.tsx` opens it, and driven with DOM events. The coin type is whatever the tab puts in the form. The pre-existing Base row is keyed on `evmChainOptions.find(o => o.label === 'Base').coinType`, so it is never hardcoded.
3. Those records go to the real `buildRecordsUpdateCalls` (`profileRecordTransactions.ts:372`), the function `saveRecords` calls on the real save path, which encodes through ensjs `setRecordsWriteParameters`. The rig prints the calldata and does not rewrite it.
4. That calldata is broadcast verbatim, and the rig then reads `addr(node, coinType)` back off the resolver. The test never decodes the calldata; the chain reports which slot was written.
5. The real address route renders over records read off that resolver, and a sender pays whatever it displayed, on the L2.

Steps to reproduce:

```bash
bash verification/pocs/H-C7-62-E2E/run.sh exploit
bash verification/pocs/H-C7-62-E2E/run.sh control
```

Exploit, where the name already carries a chain-specific Base record and the owner types a different Base address (`portal_coins_read_from_chain` omitted for width; it is the twelve `addr(node, coinType)` reads listed in `run.sh`):

```
arm: exploit
journey: custom
resolver: 0xcF84DA5B1b0440ED2c1eA6709a6059049E29c1D8
name: alice.eth
node: 0x787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec
ui_base_cointype: 2147492101
ui_base_value: 0x14f1e4f4Bbfc8403cA4568c2CBF06f55d81a6cbB
ui_eth_value: 0x9640B928367fe6365e19D695Aa4619A4eA590680
ui_base_differs_from_eth: 1
ui_emitted_sepolia_base_slot: 0
write_calldata: 0x8b95dd71787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec00000000000000000000000000000000000000000000000000000000800021050000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000001414f1e4f4bbfc8403ca4568c2cbf06f55d81a6cbb000000000000000000000000
write_tx_status: true
onchain_at_manager_slot_2147492101: 0x14f1e4f4bbfc8403ca4568c2cbf06f55d81a6cbb
onchain_at_portal_slot_2147568180: 0x
resolver_probe_default_2147483648: 0x9640b928367fe6365e19d695aa4619a4ea590680
resolver_probe_unset_evm_2147999999: 0x9640b928367fe6365e19d695aa4619a4ea590680
resolver_probe_non_evm_999999999: 0x
resolver_probe_portal_slot_after_default_2147568180: 0x9640b928367fe6365e19d695aa4619a4ea590680
l2_chain_id: 8453
owner_coin60_record: 0x9640B928367fe6365e19D695Aa4619A4eA590680
owner_coin60_code_on_l1: 4743
owner_coin60_code_on_l2: 3
owner_intended_l2_address: 0x14f1e4f4Bbfc8403cA4568c2CBF06f55d81a6cbB
address_the_page_rendered: 0x9640b928367fe6365e19d695aa4619a4ea590680
owner_set_on_base: 0x14f1e4f4Bbfc8403cA4568c2CBF06f55d81a6cbB
rendered_equals_coin60: 1
RENDERED_IS_COIN60_NOT_BASE: 1
sender_paid_on_l2_wei: 2000000000000000000
owner_recovered_the_funds: 0
FUNDS_PERMANENTLY_UNRECOVERABLE: 1
```

Control, a clean profile where the Base chip can only set Base equal to coin 60 (`write_calldata` and `portal_coins_read_from_chain` omitted for width):

```
arm: control
journey: chip
resolver: 0xB23Ca8D1A1B6aD38aD26e5537906CC199251B216
name: alice.eth
node: 0x787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec
ui_base_cointype: 2147492101
ui_base_value: 0xe3Aa8320f073A64dc7036a253EDD1F07def81270
ui_eth_value: 0xe3Aa8320f073A64dc7036a253EDD1F07def81270
ui_base_differs_from_eth: 0
ui_emitted_sepolia_base_slot: 0
write_tx_status: true
onchain_at_manager_slot_2147492101: 0xe3aa8320f073a64dc7036a253edd1f07def81270
onchain_at_portal_slot_2147568180: 0x
resolver_probe_default_2147483648: 0xe3aa8320f073a64dc7036a253edd1f07def81270
resolver_probe_unset_evm_2147999999: 0xe3aa8320f073a64dc7036a253edd1f07def81270
resolver_probe_non_evm_999999999: 0x
resolver_probe_portal_slot_after_default_2147568180: 0xe3aa8320f073a64dc7036a253edd1f07def81270
l2_chain_id: 8453
owner_coin60_record: 0xe3Aa8320f073A64dc7036a253EDD1F07def81270
owner_coin60_code_on_l1: 4743
owner_coin60_code_on_l2: 3
owner_intended_l2_address: 0x7bcA34dC37A39f7C016b3571376b61E2F0d0116d
address_the_page_rendered: 0xe3aa8320f073a64dc7036a253edd1f07def81270
owner_set_on_base: 0xe3Aa8320f073A64dc7036a253EDD1F07def81270
rendered_equals_coin60: 1
RENDERED_IS_COIN60_NOT_BASE: 0
sender_paid_on_l2_wei: 2000000000000000000
owner_recovered_the_funds: 0
FUNDS_PERMANENTLY_UNRECOVERABLE: 0
```

The write lands at 2147492101. The slot the Explorer reads, 2147568180, is empty at that point. The resolver's own fallback is probed in the same run: `resolver_probe_unset_evm_2147999999`, a coin type belonging to no chain either app knows, returns the default EVM record, and `resolver_probe_non_evm_999999999` returns empty. The page then displays the owner's coin-60 smart account in place of the Base address the owner set, the payment goes through, and recovery fails. `RENDERED_IS_COIN60_NOT_BASE` and `FUNDS_PERMANENTLY_UNRECOVERABLE` are both 1 in the exploit arm and both 0 in the control.

The two arms are the two journeys described under Impact Details. Exploit: the name already carries a chain-specific Base record, the tab shows it as an editable row, and the owner types a different Base address. Control: a clean profile, where the Base chip can only set Base equal to coin 60, so the page shows exactly what the owner chose and nothing is lost. The comparison is between the rendered address and what the owner actually set, which is what makes the clean-profile arm a control.

### The declared fix

The patch applied by the differential check is the smallest one that flips the measured keys: the Base entry in `evmChainOptions`, `2147492101` to `2147568180`. Both keys are read off the Base row, so with that patch the record lands in the slot the Explorer reads, the Base row shows the owner's intended address, and the sender's funds become recoverable.

That one-row patch is not a sufficient fix, and the rig showed why. An earlier revision of step 1 resolved Base's slot by searching for the mainnet coin type instead of matching the label the user clicks. Under that resolution the patched build still produced `2147492101` and both keys stayed at 1, because the mainnet Base record is also reachable through the tab's search over `addressRecords`, which comes from the address encoder's `coinTypeToNameMap`. Patching the button does not remove the mainnet slot from the picker.

A complete fix has to cover both paths, and the per-row table above shows the size of the job: Linea and Scroll are only offered through that encoder-derived search, so no constant swap corrects them. The Manager needs the same deployment-aware derivation the Explorer already uses, `getCoinTypeForReverseRegistrarChainId(chainId, <network>)`, applied to every EVM network it offers by either path, so that both apps agree on the coin type for whichever deployment they point at.

### Full PoC source, rig 1 (`H-C7-62`)

#### `verification/pocs/H-C7-62/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-62 — the Manager writes L2 address records to MAINNET-derived coin types
# while the Explorer reads SEPOLIA-derived ones, so every L2 row on the address
# page renders the owner's coin-60 address instead. When that coin-60 record is a
# CONTRACT address, funds a sender pays on the L2 are permanently unrecoverable.
#
# Four steps. Step 1 needs no chain; steps 2 and 4 are the loss arm.
#
#   step 1  apps/manager — resolve, through the app's OWN picker code, the coin
#           type the Manager writes for each of the five L2 networks the Explorer
#           renders a row for, and decode each back to its chain id.
#   step 2  two chains    — deploy the owner's smart account on "Ethereum", and on
#           the L2 burn the deployer's nonce 0 so that CREATE address can never be
#           produced there again.
#   step 3  apps/portal   — MOUNT THE REAL `/$name/address` ROUTE with those two
#           addresses and read the rendered table out of the DOM, so
#           `forwardAddress` (:58-67) and `buildRow` (:69-88) execute. EVERY L2
#           row gets a record written for it, so a row that falls back does so
#           because of the SLOT and not because nobody set it.
#   step 4  two chains    — a sender pays WHATEVER THE PAGE RENDERED, on the L2,
#           and the rig then tries to recover it.
#
#   exploit : the owner's L2 addresses sit in the slots the MANAGER writes
#             (mainnet-derived). Every row renders the coin-60 CONTRACT address.
#   control : they sit in the slots the EXPLORER reads (Sepolia-derived). Every
#             row renders the L2 address, an EOA whose key the owner holds.
#
# Prerequisites: an anvil Sepolia fork on 127.0.0.1:8946 (or $POC_RPC_URL) standing
# in for L1, and a plain anvil on 127.0.0.1:8947 standing in for the L2 — started
# here if absent.
#
# Usage: run.sh [exploit|control]
set -euo pipefail
export PATH="$HOME/.foundry/bin:$PATH"

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
ROOT="$HERE/../../../repos/audit-comp-ens"
MANAGER="$ROOT/apps/manager"; PORTAL="$ROOT/apps/portal"
ETH="${POC_RPC_URL:-http://127.0.0.1:8946}"
L2=http://127.0.0.1:8947
MC_COPY="$MANAGER/src/hc762-manager.test.ts"
PR_COPY="$PORTAL/src/hc762-portal.test.tsx"
SHIM_COPY="$PORTAL/src/shim.ts"

[ -d "$MANAGER/src" ] || { echo "FATAL: manager app not found at $MANAGER" >&2; exit 1; }
[ -d "$PORTAL/src" ]  || { echo "FATAL: portal app not found at $PORTAL" >&2; exit 1; }
chainid () { curl -s -m 3 -X POST -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' "$1" 2>/dev/null; }
chainid "$ETH" | grep -q 0xaa36a7 || { echo "FATAL: no Sepolia anvil fork at $ETH" >&2; exit 1; }
if ! chainid "$L2" | grep -q 0x; then
  nohup anvil --port 8947 --chain-id 8453 --silent >/tmp/hc762-l2-anvil.log 2>&1 &
  for _ in $(seq 1 30); do chainid "$L2" | grep -q 0x && break; sleep 1; done
fi
chainid "$L2" | grep -q 0x || { echo "FATAL: could not start the L2 chain on $L2" >&2; exit 1; }

cleanup() { rm -f "$MC_COPY" "$PR_COPY" "$SHIM_COPY"; }
trap cleanup EXIT
trap 'cleanup; exit 143' INT TERM HUP
cp "$HERE/manager-coin.test.ts" "$MC_COPY"
cp "$HERE/portal-render.test.tsx" "$PR_COPY"
cp "$HERE/shim.ts" "$SHIM_COPY"

pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '"\r'; }

# ---- step 1: the Manager's own coin types -----------------------------------
MC=$(cd "$MANAGER" && pnpm exec vitest run src/hc762-manager.test.ts \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$MC" | grep -q 'Test Files  1 passed' || { echo "FATAL: manager step failed" >&2; echo "$MC" >&2; exit 1; }
MANAGER_BASE=$(pick manager_base_cointype "$MC")
MANAGER_CHAIN=$(pick manager_base_chainid "$MC")
MANAGER_TABLE=$(printf '%s\n' "$MC" | sed -n 's/^manager_curated_l2_table: *//p' | head -1)
MANAGER_SLOTS=$(printf '%s\n' "$MC" | sed -n 's/^manager_slots_for_portal_l2_rows: *//p' | head -1)
MANAGER_VIA=$(printf '%s\n' "$MC" | sed -n 's/^manager_slot_offered_via: *//p' | head -1)
MANAGER_CHAINIDS=$(printf '%s\n' "$MC" | sed -n 's/^manager_slot_chainids: *//p' | head -1)
[ -n "$MANAGER_BASE" ] || { echo "FATAL: step 1 published no Manager coin type" >&2; exit 1; }
[ -n "$MANAGER_SLOTS" ] || { echo "FATAL: step 1 published no Manager slot map" >&2; exit 1; }

# ---- step 2: build the owner's two addresses on two chains ------------------
FUND=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
DK=0x$(python3 -c "import secrets;print(secrets.token_hex(32))")
D=$(cast wallet address --private-key "$DK")
L2K=0x$(python3 -c "import secrets;print(secrets.token_hex(32))")
OWNER_L2=$(cast wallet address --private-key "$L2K")
for R in "$ETH" "$L2"; do
  cast send "$D"        --value 5ether --private-key "$FUND" --rpc-url "$R" >/dev/null
  cast send "$OWNER_L2" --value 1ether --private-key "$FUND" --rpc-url "$R" >/dev/null
done
# The owner's coin-60 record is a SMART ACCOUNT: a CREATE address, so no key exists.
OWNER_ETH=$(cast compute-address "$D" --nonce 0 --rpc-url "$ETH" | awk '{print $NF}')
cast send --private-key "$DK" --rpc-url "$ETH" --create "$(cat "$HERE/../H-C7-32-CHAIN/wallet.hex")" >/dev/null
CODE_ETH=$(cast code "$OWNER_ETH" --rpc-url "$ETH" | wc -c | tr -d ' ')
# On the L2 the same deployer burns nonce 0, so CREATE(D,0) can never be produced
# there again — nonces only increase. This is what makes the address unreachable.
cast send "$D" --value 0 --private-key "$DK" --rpc-url "$L2" >/dev/null
CODE_L2=$(cast code "$OWNER_ETH" --rpc-url "$L2" | wc -c | tr -d ' ')
NONCE_L2=$(cast nonce "$D" --rpc-url "$L2")

# ---- step 3: what the real address page renders -----------------------------
PR=$(cd "$PORTAL" && ARM="$ARM" MANAGER_SLOTS="$MANAGER_SLOTS" \
      OWNER_ETH="$OWNER_ETH" OWNER_L2="$OWNER_L2" \
      pnpm exec vitest run src/hc762-portal.test.tsx \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$PR" | grep -q 'Test Files  1 passed' || { echo "FATAL: portal step failed" >&2; echo "$PR" >&2; exit 1; }
cleanup; trap - EXIT
RENDERED=$(pick base_row_rendered_address "$PR")

# ---- step 4: a sender pays what the page rendered, on the L2 ----------------
cast send "$RENDERED" --value 2ether --private-key "$FUND" --rpc-url "$L2" >/dev/null
PAID=$(cast balance "$RENDERED" --rpc-url "$L2")
# Recovery: the owner can only move it if they hold a key for the rendered address.
RECOVERED=0
if [ "$(echo "$RENDERED" | tr 'A-Z' 'a-z')" = "$(echo "$OWNER_L2" | tr 'A-Z' 'a-z')" ]; then
  cast send "$D" --value 1ether --private-key "$L2K" --rpc-url "$L2" >/dev/null 2>&1 && RECOVERED=1
fi
RESCUE_ADDR=$(cast compute-address "$D" --nonce "$(cast nonce "$D" --rpc-url "$L2")" --rpc-url "$L2" | awk '{print $NF}')
UNRECOVERABLE=0
if [ "$RECOVERED" = 0 ] && [ "$PAID" != 0 ] && [ "$CODE_L2" -le 4 ]; then UNRECOVERABLE=1; fi

echo "arm: $ARM"
echo "manager_base_cointype: $MANAGER_BASE"
echo "manager_base_chainid: $MANAGER_CHAIN"
echo "manager_curated_l2_table: $MANAGER_TABLE"
echo "manager_slots_for_portal_l2_rows: $MANAGER_SLOTS"
echo "manager_slot_offered_via: $MANAGER_VIA"
echo "manager_slot_chainids: $MANAGER_CHAINIDS"
for k in portal_base_cointype portal_base_chainid write_slot l2_rows l2_rows_total \
         l2_rows_rendering_eth_address l2_rows_rendering_intended_l2_address \
         impact_row SENDER_PAYS_ETH_NOT_L2; do
  echo "$k: $(pick $k "$PR")"
done
echo "owner_coin60_record: $OWNER_ETH"
echo "owner_coin60_code_on_l1: $CODE_ETH"
echo "owner_coin60_code_on_l2: $CODE_L2"
echo "owner_intended_l2_address: $OWNER_L2"
echo "address_the_page_rendered: $RENDERED"
echo "sender_paid_on_l2_wei: $PAID"
echo "owner_recovered_the_funds: $RECOVERED"
echo "deployer_nonce_on_l2: $NONCE_L2"
echo "rescue_deploy_would_land_at: $RESCUE_ADDR"
echo "FUNDS_PERMANENTLY_UNRECOVERABLE: $UNRECOVERABLE"
```

#### `verification/pocs/H-C7-62/manager-coin.test.ts` (step 1)

```ts
/**
 * H-C7-62 step 1 — the coin type the MANAGER writes for each of the five L2
 * networks the Explorer renders a row for, resolved through the Manager's OWN
 * picker code. Nothing is hardcoded: `evmChainOptions` is the curated L2 list
 * the Edit-Profile address tab renders, and `getPickerRecordGroups` is the
 * search the tab runs when the user types a network name — the two paths by
 * which an owner can set an L2 address at all.
 *
 * The five chain ids are the Explorer's own `L2_CHAIN_IDS`, so the two apps are
 * compared on exactly the networks they both have.
 */
import { expect, test } from 'vitest'
import {
  evmChainOptions,
  getPickerRecordGroups,
} from '@/features/profile/components/dialogs/edit-profile/tabs/addresses/addressPickerRecords'

/** apps/portal/.../AddressResolution/networks.ts:29 — the Explorer's L2 rows. */
const PORTAL_L2 = [
  { chainId: 10, label: 'Optimism' },
  { chainId: 42161, label: 'Arbitrum' },
  { chainId: 8453, label: 'Base' },
  { chainId: 59144, label: 'Linea' },
  { chainId: 534352, label: 'Scroll' },
] as const

test('H-C7-62: the slot the Manager gives the user is MAINNET-keyed, for every Explorer L2 row', () => {
  // The curated list, decoded back to the chain each row is keyed on.
  const curated = evmChainOptions.map((o) => ({
    label: o.label,
    coinType: o.coinType,
    chainId: o.coinType & ~0x80000000,
  }))

  const slots: Record<string, number> = {}
  const via: Record<string, string> = {}
  const decodedPick: Record<string, number> = {}

  for (const net of PORTAL_L2) {
    // WHICH SLOT DOES THE USER ACTUALLY GET when they pick this network?
    // Path A: the curated L2 button list, matched on the LABEL the user clicks.
    const fromCurated = evmChainOptions.find((o) => o.label === net.label)

    // Path B: the picker search the tab runs when the user types the name.
    // Both groups are rendered by the dialog, so both count as "offered".
    const groups = getPickerRecordGroups({
      mode: 'evm',
      normalizedSearchValue: net.label.toLowerCase(),
      unavailableCoinTypes: new Set<number>(),
    })
    const candidates = [...groups.popularRecords, ...groups.otherRecords]
    const fromSearch =
      candidates.find((r) => r.name.toLowerCase() === net.label.toLowerCase()) ??
      candidates[0]

    const picked = fromCurated ?? fromSearch
    // The Manager must actually offer this network by one path or the other,
    // otherwise the mismatch would be unreachable for that row.
    expect(picked, `Manager offers no slot for ${net.label}`).toBeTruthy()

    slots[net.label] = picked!.coinType
    via[net.label] = fromCurated ? 'evmChainOptions' : 'pickerSearch'
    // Decode it back so the record shows WHICH chain the user's pick is keyed on.
    decodedPick[net.label] = picked!.coinType & ~0x80000000
  }

  const base = evmChainOptions.find((o) => o.label === 'Base')
  expect(base).toBeDefined()

  console.log(`manager_base_cointype: ${base!.coinType}`)
  console.log(`manager_base_chainid: ${base!.coinType & ~0x80000000}`)
  console.log(`manager_curated_l2_table: ${JSON.stringify(curated)}`)
  console.log(`manager_slots_for_portal_l2_rows: ${JSON.stringify(slots)}`)
  console.log(`manager_slot_offered_via: ${JSON.stringify(via)}`)
  console.log(`manager_slot_chainids: ${JSON.stringify(decodedPick)}`)
})
```

#### `verification/pocs/H-C7-62/portal-render.test.tsx` (step 3, the real route)

```tsx
/**
 * H-C7-62 step 3 — what the Explorer's address page ACTUALLY RENDERS.
 *
 * The real `/$name/address` route is mounted and the answer is read out of the
 * DOM. `forwardAddress` (routes/$name/address.tsx:58-67) and `buildRow` (:69-88)
 * therefore EXECUTE — the ENSIP-19 default fallback is not restated by this
 * test, which is the whole point: an earlier version of this rig re-implemented
 * that ternary and so proved only that the test agreed with itself.
 *
 * Nothing under test is mocked. The two query caches are SEEDED rather than
 * fetched, because the indexer is not a listed asset and safe harbor is FALSE,
 * so no request may reach an ENS-operated system. The seeded shape is the
 * profile record set the route consumes; every derivation from it is the app's.
 *
 * EVERY L2 row the Explorer renders gets an address record written for it, so
 * "5 of 5 rows fall back" is a measurement and not an artefact of rows nobody
 * set. The arms differ in ONE thing: which coin-type slot those five records
 * were written to.
 *   exploit : the slots the MANAGER writes  (mainnet-derived, published by step 1)
 *   control : the slots the EXPLORER reads  (Sepolia-derived, the app's own)
 */
import './shim'
import { TransactionManagerProvider } from '@ens-apps/transaction-manager'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import {
  createMemoryHistory, createRootRoute, createRoute, createRouter, Outlet, RouterProvider,
} from '@tanstack/react-router'
import { render, waitFor } from '@testing-library/react'
import { createClient, createPublicClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { createConfig, WagmiProvider } from 'wagmi'
import { expect, test } from 'vitest'
import { FORWARD_RESOLUTION_NETWORKS } from '@/features/forward-resolution/components/AddressResolution/networks'
import { DEFAULT_EVM_COIN_TYPE, MAINNET_COIN_TYPE } from '@/lib/coinType'
import { Route as AddressRoute } from '@/routes/$name/address'

const RPC = process.env.POC_RPC_URL ?? 'http://127.0.0.1:8946'
const NAME = 'alice.eth'
/**
 * Both addresses are published by step 2 of run.sh, which deploys them:
 *  - OWNER_L2  an EOA the owner controls on the L2 (what they set as their L2 record)
 *  - OWNER_ETH the owner's coin-60 record. In the chain arm this is a CONTRACT
 *              address (a smart account), which is what makes the fallback lossy.
 */
const OWNER_L2 = process.env.OWNER_L2 ?? '0x1111111111111111111111111111111111111111'
const OWNER_ETH = process.env.OWNER_ETH ?? '0x2222222222222222222222222222222222222222'

const ARM = process.env.ARM ?? 'exploit'
/** Published by step 1, resolved through the Manager's own picker code. */
const MANAGER_SLOTS: Record<string, number> = JSON.parse(
  process.env.MANAGER_SLOTS ?? '{}',
)

test('H-C7-62: every L2 row renders the coin-60 address', async () => {
  const l2Networks = FORWARD_RESOLUTION_NETWORKS.filter((n) => n.l2ChainId != null)
  expect(l2Networks.length).toBeGreaterThan(0)
  // Step 1 must have published a Manager slot for every row we are about to set.
  for (const n of l2Networks) expect(typeof MANAGER_SLOTS[n.label]).toBe('number')

  // Write the owner's L2 address for EVERY L2 row — into the slot the Manager
  // writes (exploit) or the slot the Explorer reads (control).
  const writeSlots = Object.fromEntries(
    l2Networks.map((n) => [
      n.label,
      ARM === 'exploit' ? MANAGER_SLOTS[n.label] : n.coinType,
    ]),
  )

  const qc = new QueryClient({
    defaultOptions: { queries: { retry: false, staleTime: Number.POSITIVE_INFINITY } },
  })
  qc.setQueryData(['get-ens-owner', { name: NAME }], {
    protocolVersion: 'ENSv2',
    owner: OWNER_ETH,
    registryAddress: '0x0000000000000000000000000000000000000001',
  })
  qc.setQueryData(['profile', { name: NAME, protocolVersion: 'ENSv2' }], {
    records: {
      coins: [
        { coinType: MAINNET_COIN_TYPE, value: OWNER_ETH },
        { coinType: DEFAULT_EVM_COIN_TYPE, value: OWNER_ETH },
        ...l2Networks.map((n) => ({ coinType: writeSlots[n.label], value: OWNER_L2 })),
      ],
      texts: [],
    },
  })

  const cfg = createConfig({
    chains: [sepolia], connectors: [],
    client: () => createClient({ chain: sepolia, transport: http(RPC) }),
  })
  const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
  const root = createRootRoute({ component: () => <Outlet /> })
  const child = createRoute({
    getParentRoute: () => root,
    path: '/$name/address',
    component: AddressRoute.options.component as never,
  })
  const router = createRouter({
    routeTree: root.addChildren([child]),
    history: createMemoryHistory({ initialEntries: [`/${NAME}/address`] }),
  })

  render(
    <WagmiProvider config={cfg}>
      <QueryClientProvider client={qc}>
        <TransactionManagerProvider publicClient={pub as never}>
          <RouterProvider router={router as never} />
        </TransactionManagerProvider>
      </QueryClientProvider>
    </WagmiProvider>,
  )

  await waitFor(
    () => expect(document.body.textContent ?? '').toMatch(/Base|went wrong/i),
    { timeout: 20_000 },
  )
  const body = document.body.textContent ?? ''
  // A rendered error boundary would otherwise pass silently as "no L2 address".
  expect(body).not.toMatch(/went wrong/i)

  const rowAddress = (label: string): string => {
    const i = body.indexOf(label)
    if (i < 0) return ''
    const m = body.slice(i, i + 90).match(/0x[a-fA-F0-9]{40}/)
    return m ? m[0].toLowerCase() : ''
  }

  const eth = OWNER_ETH.toLowerCase()
  const l2 = OWNER_L2.toLowerCase()
  const rows = l2Networks.map((n) => ({
    label: n.label,
    portalCoinType: n.coinType,
    portalChainId: n.coinType & ~0x80000000,
    managerCoinType: MANAGER_SLOTS[n.label],
    wroteTo: writeSlots[n.label],
    rendered: rowAddress(n.label),
  }))

  const showingEth = rows.filter((r) => r.rendered === eth)
  const showingIntended = rows.filter((r) => r.rendered === l2)
  const baseShown = rowAddress('Base')
  const paysWrong = baseShown === eth && baseShown !== l2 ? 1 : 0

  const portalBase = l2Networks.find((n) => n.label === 'Base')!.coinType

  console.log(`arm: ${ARM}`)
  console.log(`portal_base_cointype: ${portalBase}`)
  // Decode the Explorer's coin type back to its chain id, the mirror of step 1:
  // 84532 is Base Sepolia, against 8453 (Base mainnet) on the Manager side.
  console.log(`portal_base_chainid: ${portalBase & ~0x80000000}`)
  console.log(`write_slot: ${writeSlots.Base}`)
  console.log(`l2_rows: ${JSON.stringify(rows)}`)
  console.log(`owner_intended_l2_address: ${OWNER_L2}`)
  console.log(`owner_eth_address: ${OWNER_ETH}`)
  console.log(`base_row_rendered_address: ${baseShown}`)
  console.log(`l2_rows_total: ${rows.length}`)
  console.log(`l2_rows_rendering_eth_address: ${showingEth.length}`)
  console.log(`l2_rows_rendering_intended_l2_address: ${showingIntended.length}`)
  console.log(`impact_row: 6285`)
  console.log(`SENDER_PAYS_ETH_NOT_L2: ${paysWrong}`)

  if (ARM === 'control') {
    // Every row was written to the slot the Explorer reads, so every row must
    // show the owner's L2 address. This is what makes the exploit arm's 5/5 a
    // consequence of the slot, not of unset records.
    expect(showingIntended.length).toBe(rows.length)
  }
}, 60_000)
```

#### `verification/pocs/H-C7-62/shim.ts` (environment scaffolding)

```ts
/**
 * Environment scaffolding only — nothing under test is touched.
 *
 * 1. happy-dom installs no `localStorage`, and `@/lib/wagmi` reads one at module
 *    scope (lib/wagmi.ts -> hooks/useTheme.ts), so importing the app's module
 *    graph throws without it.
 * 2. TanStack Router serialises location state, and ENS profile data carries
 *    BigInt expiries. Without a `toJSON` the route renders its error boundary
 *    ("Something went wrong!") instead of the table — a silent false negative.
 */
const store = new Map<string, string>()
const ls = {
  getItem: (k: string) => (store.has(k) ? store.get(k)! : null),
  setItem: (k: string, v: string) => void store.set(k, String(v)),
  removeItem: (k: string) => void store.delete(k),
  clear: () => store.clear(),
  key: (i: number) => [...store.keys()][i] ?? null,
  get length() {
    return store.size
  },
}
for (const t of [globalThis, typeof window !== 'undefined' ? window : null]) {
  if (t) Object.defineProperty(t, 'localStorage', { configurable: true, value: ls })
}
if (typeof window !== 'undefined' && typeof window.matchMedia !== 'function') {
  window.matchMedia = ((q: string) => ({
    matches: false, media: q, onchange: null,
    addListener: () => {}, removeListener: () => {},
    addEventListener: () => {}, removeEventListener: () => {},
    dispatchEvent: () => false,
  })) as never
}
;(BigInt.prototype as unknown as { toJSON(): string }).toJSON = function () {
  return this.toString()
}
```

### Full PoC source, rig 2 (`H-C7-62-E2E`)

#### `verification/pocs/H-C7-62-E2E/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-62-E2E — the missing link, executed end to end:
#
#   Manager UI selection -> Manager's real save encoding -> on-chain ENS record
#   -> Portal lookup -> wrong displayed address -> unrecoverable payment.
#
# Nothing is seeded. The coin type comes from the app's own Addresses tab, the
# calldata from the app's own buildRecordsUpdateCalls, the record from a real
# PermissionedResolver deployed through the genuine Sepolia VerifiableFactory,
# and the rendered address from the real /$name/address route reading the
# records this rig READ BACK OFF THAT RESOLVER.
#
#   step 1  two chains   — deploy a PermissionedResolver; build the owner's
#                          coin-60 record as a SMART ACCOUNT (a CREATE address,
#                          no key) and burn the deployer's nonce on the L2 so it
#                          can never be produced there again.
#   step 2  apps/manager — drive the REAL Edit-Profile Addresses tab, pick Base,
#                          and encode the save with the REAL builder.
#   step 3  this script  — broadcast that calldata verbatim, then ask the CHAIN
#                          which coin-type slot it wrote.
#   step 4  apps/portal  — render the REAL address route over the records read
#                          back off the resolver.
#   step 5  two chains   — a sender pays what the page rendered, on the L2, and
#                          the rig tries to recover it.
#
#   exploit : journey 'custom' — the name already carries a chain-specific Base
#             record, so the tab lets the owner type a DIFFERENT Base address.
#   control : journey 'chip'   — a clean profile. The Base chip can only set Base
#             EQUAL to coin 60, so the Explorer's fallback renders the same
#             address and nothing is lost. This is the honest boundary of the
#             finding, measured rather than asserted.
#
# Usage: run.sh [exploit|control]
set -euo pipefail
export PATH="$HOME/.foundry/bin:$PATH"

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac
JOURNEY=$([ "$ARM" = exploit ] && echo custom || echo chip)

HERE="$(cd -P "$(dirname "$0")" && pwd)"
ROOT="$HERE/../../../repos/audit-comp-ens"
MANAGER="$ROOT/apps/manager"; PORTAL="$ROOT/apps/portal"
ETH="${POC_RPC_URL:-http://127.0.0.1:8946}"
L2=http://127.0.0.1:8947
UI_COPY="$MANAGER/src/hc762e2e-ui.test.tsx";  UI_SHIM="$MANAGER/src/shim.ts"
PR_COPY="$PORTAL/src/hc762e2e-portal.test.tsx"; PR_SHIM="$PORTAL/src/shim.ts"

[ -d "$MANAGER/src" ] || { echo "FATAL: manager app not found at $MANAGER" >&2; exit 1; }
[ -d "$PORTAL/src" ]  || { echo "FATAL: portal app not found at $PORTAL" >&2; exit 1; }
chainid () { curl -s -m 3 -X POST -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' "$1" 2>/dev/null; }
chainid "$ETH" | grep -q 0xaa36a7 || { echo "FATAL: no Sepolia anvil fork at $ETH" >&2; exit 1; }
if ! chainid "$L2" | grep -q 0x; then
  nohup anvil --port 8947 --chain-id 8453 --silent >/tmp/hc762-l2-anvil.log 2>&1 &
  for _ in $(seq 1 30); do chainid "$L2" | grep -q 0x && break; sleep 1; done
fi
chainid "$L2" | grep -q 0x || { echo "FATAL: could not start the L2 chain on $L2" >&2; exit 1; }

cleanup() { rm -f "$UI_COPY" "$UI_SHIM" "$PR_COPY" "$PR_SHIM"; }
trap cleanup EXIT
trap 'cleanup; exit 143' INT TERM HUP
pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '"\r'; }

# ---- step 1: a real resolver, and the owner's two addresses -----------------
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
ADMIN=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
ADMINK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
ALL_ROLES=0x1111111111111111111111111111111111111111111111111111111111111111

OID=$(cast keccak "$(cast from-utf8 'OwnedResolver')")
VERSION=$(python3 -c "import secrets;print(secrets.randbelow(2**64))")
SALT=$(cast keccak "$(cast abi-encode 'f(bytes32,address,uint256)' "$OID" $ADMIN "$VERSION")")
INIT=$(cast calldata 'initialize(address,uint256,bytes[])' $ADMIN $ALL_ROLES '[]')
RESOLVER=$(cast call $FACTORY 'deployProxy(address,uint256,bytes)(address)' $IMPL "$SALT" "$INIT" --from $ADMIN --rpc-url "$ETH" | head -1)
cast send $FACTORY 'deployProxy(address,uint256,bytes)' $IMPL "$SALT" "$INIT" \
  --private-key "$ADMINK" --rpc-url "$ETH" >/dev/null
[ "$(cast code "$RESOLVER" --rpc-url "$ETH" | wc -c)" -gt 4 ] || { echo "FATAL: resolver did not deploy" >&2; exit 1; }

# The owner's coin-60 record is a SMART ACCOUNT: a CREATE address, no key exists.
DK=0x$(python3 -c "import secrets;print(secrets.token_hex(32))")
D=$(cast wallet address --private-key "$DK")
L2K=0x$(python3 -c "import secrets;print(secrets.token_hex(32))")
OWNER_L2=$(cast wallet address --private-key "$L2K")
for R in "$ETH" "$L2"; do
  cast send "$D"        --value 5ether --private-key "$ADMINK" --rpc-url "$R" >/dev/null
  cast send "$OWNER_L2" --value 1ether --private-key "$ADMINK" --rpc-url "$R" >/dev/null
done
OWNER_ETH=$(cast compute-address "$D" --nonce 0 --rpc-url "$ETH" | awk '{print $NF}')
cast send --private-key "$DK" --rpc-url "$ETH" --create "$(cat "$HERE/../H-C7-32-CHAIN/wallet.hex")" >/dev/null
CODE_ETH=$(cast code "$OWNER_ETH" --rpc-url "$ETH" | wc -c | tr -d ' ')
cast send "$D" --value 0 --private-key "$DK" --rpc-url "$L2" >/dev/null
CODE_L2=$(cast code "$OWNER_ETH" --rpc-url "$L2" | wc -c | tr -d ' ')

NAME=alice.eth
NODE=$(cast namehash "$NAME")

# ---- step 2: the REAL Addresses tab, and the REAL encoder -------------------
cp "$HERE/ui-write.test.tsx" "$UI_COPY"; cp "$HERE/shim.ts" "$UI_SHIM"
UI=$(cd "$MANAGER" && JOURNEY="$JOURNEY" RESOLVER="$RESOLVER" E2E_NAME="$NAME" \
      OWNER_ETH="$OWNER_ETH" OWNER_L2="$OWNER_L2" POC_RPC_URL="$ETH" \
      pnpm exec vitest run src/hc762e2e-ui.test.tsx \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$UI" | grep -q 'Test Files  1 passed' || { echo "FATAL: UI step failed" >&2; echo "$UI" >&2; exit 1; }
rm -f "$UI_COPY" "$UI_SHIM"

UI_COINTYPE=$(pick ui_base_cointype "$UI")
UI_VALUE=$(pick ui_base_value "$UI")
UI_ETH=$(pick ui_eth_value "$UI")
UI_DIFFERS=$(pick ui_base_differs_from_eth "$UI")
UI_SEPOLIA_SLOT=$(pick ui_emitted_sepolia_base_slot "$UI")
CALLDATA=$(pick write_calldata "$UI")
[ -n "$CALLDATA" ] || { echo "FATAL: the UI step emitted no calldata" >&2; exit 1; }

# ---- step 3: broadcast it VERBATIM, then ask the chain what it wrote --------
TX=$(cast send "$RESOLVER" "$CALLDATA" --private-key "$ADMINK" --rpc-url "$ETH" \
      | grep -E '^transactionHash' | awk '{print $2}')
STATUS=$(cast receipt "$TX" --rpc-url "$ETH" status 2>/dev/null || echo "")

# Which slot did it land in? The CHAIN answers, not the test.
MANAGER_SLOT=2147492101      # 0x80000000 | 8453   (Base mainnet)  — what the Manager used
PORTAL_SLOT=2147568180       # 0x80000000 | 84532  (Base Sepolia)  — what the Explorer reads
readback () { cast call "$RESOLVER" 'addr(bytes32,uint256)(bytes)' "$NODE" "$1" --rpc-url "$ETH" 2>/dev/null | head -1; }
ONCHAIN_AT_MANAGER_SLOT=$(readback $MANAGER_SLOT)
ONCHAIN_AT_PORTAL_SLOT=$(readback $PORTAL_SLOT)

# Also write the coin-60 record the owner holds, the same way, so the page has
# the default to fall back to.
C60=$(cast calldata 'setAddr(bytes32,uint256,bytes)' "$NODE" 60 "$OWNER_ETH")
cast send "$RESOLVER" "$C60" --private-key "$ADMINK" --rpc-url "$ETH" >/dev/null
DEFAULT_CT=2147483648         # 0x80000000, the ENSIP-19 default EVM coin type
CDEF=$(cast calldata 'setAddr(bytes32,uint256,bytes)' "$NODE" $DEFAULT_CT "$OWNER_ETH")
cast send "$RESOLVER" "$CDEF" --private-key "$ADMINK" --rpc-url "$ETH" >/dev/null

# The resolver's OWN fallback, probed rather than assumed: any EVM coin type with
# no specific record returns the ENSIP-19 default EVM address, while a NON-EVM
# coin type returns empty. This is why the Explorer's miss resolves to the
# owner's coin-60 address at the RESOLVER, before the app's own ternary.
PROBE_UNSET_EVM=$(readback 2147999999)
PROBE_NON_EVM=$(readback 999999999)
PROBE_DEFAULT=$(readback $DEFAULT_CT)
PROBE_PORTAL_SLOT_AFTER_DEFAULT=$(readback $PORTAL_SLOT)
L2_CHAIN_ID=$(printf '%d' "$(chainid "$L2" | sed -n 's/.*"result":"\(0x[0-9a-fA-F]*\)".*/\1/p')")

# ---- step 4: render the real route over the records READ OFF THE CHAIN ------
PORTAL_COINS=$(python3 - "$RESOLVER" "$NODE" "$ETH" <<'PY'
import json, subprocess, sys
resolver, node, rpc = sys.argv[1], sys.argv[2], sys.argv[3]
# Every coin type either app keys on, read straight off the resolver.
cts = [60, 2147483648,
       2147483658, 2147525809, 2147492101, 2147542792, 2148018000,   # Manager (mainnet)
       2158639068, 2147905262, 2147568180, 2147542789, 2148017999]   # Explorer (Sepolia)
out = []
for ct in cts:
    r = subprocess.run(['cast','call',resolver,'addr(bytes32,uint256)(bytes)',node,str(ct),
                        '--rpc-url',rpc], capture_output=True, text=True)
    v = r.stdout.strip().split('\n')[0].strip() if r.returncode == 0 else ''
    if v.startswith('0x') and len(v) == 42:
        out.append({'coinType': ct, 'value': v})
print(json.dumps(out))
PY
)

cp "$HERE/portal-chain.test.tsx" "$PR_COPY"; cp "$HERE/shim.ts" "$PR_SHIM"
PR=$(cd "$PORTAL" && PORTAL_COINS="$PORTAL_COINS" E2E_NAME="$NAME" \
      OWNER_ETH="$OWNER_ETH" OWNER_SET_ON_BASE="$UI_VALUE" POC_RPC_URL="$ETH" \
      pnpm exec vitest run src/hc762e2e-portal.test.tsx \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$PR" | grep -q 'Test Files  1 passed' || { echo "FATAL: portal step failed" >&2; echo "$PR" >&2; exit 1; }
cleanup; trap - EXIT
RENDERED=$(pick base_row_rendered_address "$PR")
RENDERED_WRONG=$(pick RENDERED_IS_COIN60_NOT_BASE "$PR")
RENDERED_EQ_C60=$(pick rendered_equals_coin60 "$PR")

# ---- step 5: a sender pays what the page rendered, on the L2 ----------------
cast send "$RENDERED" --value 2ether --private-key "$ADMINK" --rpc-url "$L2" >/dev/null
PAID=$(cast balance "$RENDERED" --rpc-url "$L2")
RECOVERED=0
if [ "$(echo "$RENDERED" | tr 'A-Z' 'a-z')" = "$(echo "$OWNER_L2" | tr 'A-Z' 'a-z')" ]; then
  cast send "$D" --value 1ether --private-key "$L2K" --rpc-url "$L2" >/dev/null 2>&1 && RECOVERED=1
fi
LOST=0
if [ "$RENDERED_WRONG" = 1 ] && [ "$RECOVERED" = 0 ] && [ "$PAID" != 0 ] && [ "$CODE_L2" -le 4 ]; then LOST=1; fi

echo "arm: $ARM"
echo "journey: $JOURNEY"
echo "resolver: $RESOLVER"
echo "name: $NAME"
echo "node: $NODE"
echo "ui_base_cointype: $UI_COINTYPE"
echo "ui_base_value: $UI_VALUE"
echo "ui_eth_value: $UI_ETH"
echo "ui_base_differs_from_eth: $UI_DIFFERS"
echo "ui_emitted_sepolia_base_slot: $UI_SEPOLIA_SLOT"
echo "write_calldata: $CALLDATA"
echo "write_tx_status: $STATUS"
echo "onchain_at_manager_slot_2147492101: $ONCHAIN_AT_MANAGER_SLOT"
echo "onchain_at_portal_slot_2147568180: $ONCHAIN_AT_PORTAL_SLOT"
echo "resolver_probe_default_2147483648: $PROBE_DEFAULT"
echo "resolver_probe_unset_evm_2147999999: $PROBE_UNSET_EVM"
echo "resolver_probe_non_evm_999999999: $PROBE_NON_EVM"
echo "resolver_probe_portal_slot_after_default_2147568180: $PROBE_PORTAL_SLOT_AFTER_DEFAULT"
echo "l2_chain_id: $L2_CHAIN_ID"
echo "portal_coins_read_from_chain: $PORTAL_COINS"
echo "owner_coin60_record: $OWNER_ETH"
echo "owner_coin60_code_on_l1: $CODE_ETH"
echo "owner_coin60_code_on_l2: $CODE_L2"
echo "owner_intended_l2_address: $OWNER_L2"
echo "address_the_page_rendered: $RENDERED"
echo "owner_set_on_base: $UI_VALUE"
echo "rendered_equals_coin60: $RENDERED_EQ_C60"
echo "RENDERED_IS_COIN60_NOT_BASE: $RENDERED_WRONG"
echo "sender_paid_on_l2_wei: $PAID"
echo "owner_recovered_the_funds: $RECOVERED"
echo "FUNDS_PERMANENTLY_UNRECOVERABLE: $LOST"
```

#### `verification/pocs/H-C7-62-E2E/ui-write.test.tsx`

```tsx
/**
 * H-C7-62-E2E step 1 — drive the REAL Edit-Profile → Addresses UI, then hand the
 * records it produces to the REAL transaction builder.
 *
 * Nothing here is seeded into a record store. The component under test is the
 * app's own `AddressesTab`, mounted behind the app's own
 * `EditProfileDialogProvider` with a REAL `editProfileDialogMachine` actor. The
 * coin type is whatever the UI puts in the form when the user picks Base, and
 * the calldata is whatever `buildRecordsUpdateCalls`
 * (profileRecordTransactions.ts:372) encodes from it — the same function
 * `saveRecords` calls on the real save path.
 *
 * TWO JOURNEYS, because the UI constrains what a user can express:
 *
 *   journey 'chip'   — a clean profile. The user types their ETH address and
 *                      clicks the Base chip. `onSetChainToEthAddress`
 *                      (EvmAddressSection.tsx:75-79) sets Base to the SAME value
 *                      as coin 60. The coin type is still mainnet-derived, so
 *                      the record still lands in a slot the Explorer never
 *                      reads — but the fallback renders the same address, so
 *                      this journey loses no money.
 *
 *   journey 'custom' — the name ALREADY carries a chain-specific Base record (a
 *                      migrated name, or one set by another client). The tab
 *                      renders it as an editable row (`customEvmOptions`) and
 *                      the user types a DIFFERENT Base address. This is the
 *                      journey that loses money, and the one the report claims.
 */
import './shim'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import { useState } from 'react'
import { createPublicClient, http, type PublicClient } from 'viem'
import { sepolia } from 'viem/chains'
import { expect, test } from 'vitest'
import { createActor } from 'xstate'
import { EditProfileDialogProvider } from '@/features/profile/components/dialogs/edit-profile/EditProfileDialog.context'
import { editProfileDialogMachine } from '@/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine'
import { AddressesTab } from '@/features/profile/components/dialogs/edit-profile/tabs/addresses/AddressesTab'
import { evmChainOptions } from '@/features/profile/components/dialogs/edit-profile/tabs/addresses/addressPickerRecords'
import { buildRecordsUpdateCalls } from '@/features/profile/service/profileRecordTransactions'
import type { ProfileRecords } from '@/features/profile/types'
import { defaultProfileRecords } from '@/features/profile/utils/transformRecords'

const RPC = process.env.POC_RPC_URL ?? 'http://127.0.0.1:8946'
const NAME = process.env.E2E_NAME ?? 'alice.eth'
const RESOLVER = (process.env.RESOLVER ??
  '0x0000000000000000000000000000000000000001') as `0x${string}`
const OWNER_ETH = (process.env.OWNER_ETH ??
  '0x1111111111111111111111111111111111111111') as `0x${string}`
const OWNER_L2 = (process.env.OWNER_L2 ??
  '0x2222222222222222222222222222222222222222') as `0x${string}`
const JOURNEY = process.env.JOURNEY ?? 'custom'

const PRE_EXISTING_BASE = '0x000000000000000000000000000000000000dEaD'

/**
 * The coin type the MANAGER itself uses for Base — read from its own picker
 * list, never hardcoded, so that correcting `evmChainOptions` moves this record
 * too. The pre-existing row stands for a name that already carries a
 * chain-specific Base address (migrated, or set in another client).
 */
const MANAGER_BASE_COINTYPE = evmChainOptions.find((o) => o.label === 'Base')!
  .coinType

const initialAddresses = () =>
  JOURNEY === 'custom'
    ? [
        { coinType: 60, value: OWNER_ETH },
        { coinType: MANAGER_BASE_COINTYPE, value: PRE_EXISTING_BASE },
      ]
    : []

let emitted: { coinType: number; value: string }[] = initialAddresses()

/** The app's own default record shape, with the addresses under test. */
const recordsFor = (addresses: { coinType: number; value: string }[]) =>
  ({
    ...defaultProfileRecords,
    addresses,
    resolverAddress: RESOLVER,
  }) as unknown as ProfileRecords

// ONE real actor for the whole mount — the app's own machine, constructed and
// opened exactly as EditProfileDialog does it.
const dialogActor = createActor(editProfileDialogMachine, {
  input: { records: recordsFor(initialAddresses()) },
})
dialogActor.start()
dialogActor.send({ type: 'OPEN', records: recordsFor(initialAddresses()) })

const Harness = () => {
  const [addresses, setAddresses] = useState(initialAddresses())
  return (
    <EditProfileDialogProvider actor={dialogActor}>
      <AddressesTab
        onAddressesChange={(next) => {
          emitted = [...next] as typeof emitted
          setAddresses(emitted)
        }}
        values={recordsFor(addresses)}
      />
    </EditProfileDialogProvider>
  )
}

test('H-C7-62-E2E: the UI picks the coin type and the app encodes the write', async () => {
  const before = {
    texts: [] as { key: string; value: string }[],
    coins: initialAddresses().map((a) => ({ coinType: a.coinType, value: a.value })),
  }

  render(<Harness />)

  if (JOURNEY === 'chip') {
    const eth = await screen.findByLabelText(/Your Ethereum Address/i)
    fireEvent.change(eth, { target: { value: OWNER_ETH } })
    await waitFor(() => expect(emitted.some((a) => a.coinType === 60)).toBe(true))
    const chip = await screen.findByText('Base')
    fireEvent.click(chip.closest('button') ?? chip)
  } else {
    const row = await screen.findByLabelText(/^Base$/i)
    fireEvent.change(row, { target: { value: OWNER_L2 } })
  }

  await waitFor(() =>
    expect(emitted.some((a) => a.coinType === MANAGER_BASE_COINTYPE)).toBe(true),
  )

  const baseRow = emitted.find((a) => a.coinType === MANAGER_BASE_COINTYPE)!
  const ethRow = emitted.find((a) => a.coinType === 60)
  const ethValue = ethRow?.value ?? ''

  console.log(`journey: ${JOURNEY}`)
  console.log(`ui_emitted_rows: ${JSON.stringify(emitted)}`)
  console.log(`ui_base_cointype: ${baseRow.coinType}`)
  console.log(`ui_base_value: ${baseRow.value}`)
  console.log(`ui_eth_value: ${ethValue}`)
  console.log(
    `ui_base_differs_from_eth: ${
      baseRow.value.toLowerCase() !== ethValue.toLowerCase() ? 1 : 0
    }`,
  )
  // The UI must NOT have produced the Sepolia-derived slot.
  console.log(
    `ui_emitted_sepolia_base_slot: ${emitted.some((a) => a.coinType === 2147568180) ? 1 : 0}`,
  )

  // --- the REAL encoder, over the records the UI just produced ---------------
  const publicClient = createPublicClient({
    chain: sepolia,
    transport: http(RPC),
  }) as unknown as PublicClient

  const { calls } = await buildRecordsUpdateCalls({
    name: NAME,
    before,
    after: {
      texts: [],
      coins: emitted.map((a) => ({ coinType: a.coinType, value: a.value })),
    },
    publicClient,
    resolverAddress: RESOLVER,
  })

  expect(calls.length).toBe(1)
  console.log(`write_to: ${calls[0].to}`)
  console.log(`write_calldata: ${calls[0].data}`)
  // The CHAIN, not this test, will say which coin-type slot that calldata wrote.
}, 90_000)
```

#### `verification/pocs/H-C7-62-E2E/portal-chain.test.tsx`

```tsx
/**
 * H-C7-62-E2E step 4 — render the REAL `/$name/address` route over records that
 * were READ OFF THE CHAIN.
 *
 * This is the difference from the original rig: the coins handed to the route's
 * query cache are not invented here. `run.sh` reads `addr(node, coinType)` from
 * the deployed resolver for every coin type the Explorer and the Manager each
 * key on, and passes the result in as PORTAL_COINS. The cache stands in for
 * packages/indexer — not a listed asset, and unreachable because safe harbor is
 * FALSE — but the VALUES are the chain's, so nothing about the record set is
 * assumed by the test.
 */
import './shim'
import { TransactionManagerProvider } from '@ens-apps/transaction-manager'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import {
  createMemoryHistory, createRootRoute, createRoute, createRouter, Outlet, RouterProvider,
} from '@tanstack/react-router'
import { render, waitFor } from '@testing-library/react'
import { createClient, createPublicClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { createConfig, WagmiProvider } from 'wagmi'
import { expect, test } from 'vitest'
import { FORWARD_RESOLUTION_NETWORKS } from '@/features/forward-resolution/components/AddressResolution/networks'
import { Route as AddressRoute } from '@/routes/$name/address'

const RPC = process.env.POC_RPC_URL ?? 'http://127.0.0.1:8946'
const NAME = process.env.E2E_NAME ?? 'alice.eth'
const OWNER_ETH = process.env.OWNER_ETH ?? '0x1111111111111111111111111111111111111111'
/** What the owner ACTUALLY set for Base in the UI, published by step 2. In the
 *  'chip' journey this equals the coin-60 address, which is why that arm is the
 *  control: the fallback then renders exactly what the owner chose. */
const OWNER_SET_ON_BASE = (
  process.env.OWNER_SET_ON_BASE ?? '0x2222222222222222222222222222222222222222'
).toLowerCase()

/** Read off the chain by run.sh: [{coinType, value}, ...] */
const PORTAL_COINS: { coinType: number; value: string }[] = JSON.parse(
  process.env.PORTAL_COINS ?? '[]',
)

test('H-C7-62-E2E: the address page renders over on-chain records', async () => {
  expect(PORTAL_COINS.length).toBeGreaterThan(0)

  const qc = new QueryClient({
    defaultOptions: { queries: { retry: false, staleTime: Number.POSITIVE_INFINITY } },
  })
  qc.setQueryData(['get-ens-owner', { name: NAME }], {
    protocolVersion: 'ENSv2',
    owner: OWNER_ETH,
    registryAddress: '0x0000000000000000000000000000000000000001',
  })
  qc.setQueryData(['profile', { name: NAME, protocolVersion: 'ENSv2' }], {
    records: { coins: PORTAL_COINS, texts: [] },
  })

  const cfg = createConfig({
    chains: [sepolia], connectors: [],
    client: () => createClient({ chain: sepolia, transport: http(RPC) }),
  })
  const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
  const root = createRootRoute({ component: () => <Outlet /> })
  const child = createRoute({
    getParentRoute: () => root,
    path: '/$name/address',
    component: AddressRoute.options.component as never,
  })
  const router = createRouter({
    routeTree: root.addChildren([child]),
    history: createMemoryHistory({ initialEntries: [`/${NAME}/address`] }),
  })

  render(
    <WagmiProvider config={cfg}>
      <QueryClientProvider client={qc}>
        <TransactionManagerProvider publicClient={pub as never}>
          <RouterProvider router={router as never} />
        </TransactionManagerProvider>
      </QueryClientProvider>
    </WagmiProvider>,
  )

  await waitFor(
    () => expect(document.body.textContent ?? '').toMatch(/Base|went wrong/i),
    { timeout: 20_000 },
  )
  const body = document.body.textContent ?? ''
  expect(body).not.toMatch(/went wrong/i)

  const rowAddress = (label: string): string => {
    const i = body.indexOf(label)
    if (i < 0) return ''
    const m = body.slice(i, i + 90).match(/0x[a-fA-F0-9]{40}/)
    return m ? m[0].toLowerCase() : ''
  }

  const baseShown = rowAddress('Base')
  const eth = OWNER_ETH.toLowerCase()

  // The defect is a DISPLAY defect: the page shows an address that is not the
  // one the owner set for this network. Comparing against what the owner set —
  // not against a fixed address — is what makes the 'chip' arm a real control.
  const wrong = baseShown !== '' && baseShown !== OWNER_SET_ON_BASE ? 1 : 0

  console.log(`portal_coins_from_chain: ${JSON.stringify(PORTAL_COINS)}`)
  console.log(`portal_base_cointype: ${
    FORWARD_RESOLUTION_NETWORKS.find((n) => n.label === 'Base')!.coinType
  }`)
  console.log(`base_row_rendered_address: ${baseShown}`)
  console.log(`owner_set_on_base: ${OWNER_SET_ON_BASE}`)
  console.log(`owner_coin60: ${eth}`)
  console.log(`rendered_equals_coin60: ${baseShown === eth ? 1 : 0}`)
  console.log(`RENDERED_IS_COIN60_NOT_BASE: ${wrong}`)
}, 90_000)
```

#### `verification/pocs/H-C7-62-E2E/shim.ts`

```ts
/**
 * Environment scaffolding only — nothing under test is touched.
 *
 * 1. happy-dom installs no `localStorage`, and `@/lib/wagmi` reads one at module
 *    scope (lib/wagmi.ts -> hooks/useTheme.ts), so importing the app's module
 *    graph throws without it.
 * 2. TanStack Router serialises location state, and ENS profile data carries
 *    BigInt expiries. Without a `toJSON` the route renders its error boundary
 *    ("Something went wrong!") instead of the table — a silent false negative.
 */
const store = new Map<string, string>()
const ls = {
  getItem: (k: string) => (store.has(k) ? store.get(k)! : null),
  setItem: (k: string, v: string) => void store.set(k, String(v)),
  removeItem: (k: string) => void store.delete(k),
  clear: () => store.clear(),
  key: (i: number) => [...store.keys()][i] ?? null,
  get length() {
    return store.size
  },
}
for (const t of [globalThis, typeof window !== 'undefined' ? window : null]) {
  if (t) Object.defineProperty(t, 'localStorage', { configurable: true, value: ls })
}
if (typeof window !== 'undefined' && typeof window.matchMedia !== 'function') {
  window.matchMedia = ((q: string) => ({
    matches: false, media: q, onchange: null,
    addListener: () => {}, removeListener: () => {},
    addEventListener: () => {}, removeEventListener: () => {},
    dispatchEvent: () => false,
  })) as never
}
;(BigInt.prototype as unknown as { toJSON(): string }).toJSON = function () {
  return this.toString()
}
```

### What is and is not substituted

Nothing under test is mocked. There is no `vi.mock` in either rig. `evmChainOptions`, `getPickerRecordGroups`, `FORWARD_RESOLUTION_NETWORKS`, the real address route, `forwardAddress`, `buildRow`, the real `AddressesTab`, the real `editProfileDialogMachine` and the real `buildRecordsUpdateCalls` all execute from the pinned tree.

The displayed addresses are read from the DOM, never computed by the test. That is the correction to an earlier version of rig 1, which re-implemented the fallback ternary and so measured only its own agreement.

Rig 1 writes a record for every L2 row, which is what makes 5 of 5 a measurement. An earlier version wrote only the Base slot, so the other four rows fell back simply because nobody had set them. That is normal behaviour, and it is not the defect under report. The control arm asserts all five rows, so the exploit arm's 5 of 5 cannot be an artefact.

A rendered error boundary fails the run explicitly. Without the `BigInt.toJSON` in `shim.ts` the route renders "Something went wrong!" in place of the table, which would otherwise pass silently as "no L2 address shown". The test asserts the body does not match `/went wrong/`.

The Explorer's query cache is seeded in both rigs, because it stands in for `packages/indexer`, which is not a listed asset and cannot be reached: safe harbor is FALSE, so no request may go to an ENS-operated system. In rig 1 the seeded value is the profile record set the route consumes, and every derivation from it is the application's. In rig 2 every value in the cache is read off the deployed resolver with `cast call`, so the record set is chain state.

In rig 2 the resolver write is broadcast by the resolver's own `initialize` admin. It does not go through the wallet and transaction-manager stack, because the smart-account signer path needs third-party infrastructure. The calldata is the application's, unmodified. Only the broadcaster differs.

The L2 is a plain anvil with chain id 8453. It cannot be the real Base for the same safe-harbor reason. Nothing chain-specific is relied on. The demonstration turns only on EVM `CREATE` address derivation and monotonic nonces, which behave the same on every EVM chain.

The smart account is a minimal contract, used only so that the coin-60 record is a `CREATE` address with no keypair behind it. What it does is irrelevant. What matters is that no private key exists for it.
