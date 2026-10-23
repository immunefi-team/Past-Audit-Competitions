# 91689 w a medium portal direct child route and spoofable resolver authentication substitute an attacker contract as the victim s transaction target

**Submitted on Sep 3rd 2026 at 15:43:23 UTC by @shadowHunter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91689
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Description

Portal rejects a non-normalized ENS name only in the `$name/index` child route. Direct child routes such as `$name/edit-records` do not execute that rejection. This allows an attacker to use an exact raw NFD `.eth` name that is visually identical after NFC normalization to a victim-owned NFC name.

The child route obtains the resolver for the exact raw attacker-owned name. `useIsPermissionedResolver` classifies that resolver by reading storage slot `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc` and comparing the stored word to the configured permissioned-resolver implementation. That storage belongs to the resolver itself. An attacker can therefore deploy a non-proxy contract, write the expected implementation address into that slot in its constructor, and return `true` from `hasRoles(..., victim)`.

Portal then renders Edit controls for the victim and passes the same untrusted resolver address to `prepareSaveRecordsTransaction`. The transaction modal says "Save records" and "Set resolver records" but does not expose that the target is the attacker's contract.

The local proof used these byte-distinct identities:

```
Victim V:                  0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db
Attacker A:                0xCcfE3e898EE111699586B099D6F93F8703bD1FA2
NFC victim name:           caf\u00e9-local-poc-r6.eth
NFC prefix code points:    U+0063 U+0061 U+0066 U+00E9
NFC UTF-8:                 0x636166c3a92d6c6f63616c2d706f632d72362e657468
NFC namehash:              0x59b569000cc49a0b92991135aa3d445b8c6f92870f78fb98fcb5a8ccff35d450
NFD attacker name:         cafe\u0301-local-poc-r6.eth
NFD prefix code points:    U+0063 U+0061 U+0066 U+0065 U+0301
NFD UTF-8:                 0x63616665cc812d6c6f63616c2d706f632d72362e657468
NFD namehash:              0xd5393c2c6c3770262151b1ab7af83174f7f6b184df3cedbad91d126701922957
Attacker resolver M:       0xF1ce623B66d24B8BF746a888156A7194C6627bd3
Mined transaction:         0xb6136e155477d1da5955875ec96c98e3c7ead530f31d46ef8d3e5fcad6c816da
Portal-generated selector: 0x10f13a8c (setText(bytes32,string,string))
```

The victim never owned the exact NFD name. The browser wallet exposed only V. The attacker's key was unavailable to the browser.

## Impact

An attacker can cause Portal to substitute an attacker-controlled contract as the target of a transaction that the UI presents as an ordinary ENS record update. This breaks the connected-wallet transaction trust boundary: the user sees a normalized-looking name and generic record-update steps, while the signed transaction is sent to code that was never deployed or verified by the trusted resolver factory.

The PoC intentionally uses a harmless setter and proves only the minimum attacker-controlled state change. It does not claim theft. The demonstrated consequence matches the program's listed Critical class for malicious interactions with an already-connected wallet, specifically contract-address substitution and modified transaction arguments. The victim must open the crafted child URL, make a record edit, and approve the wallet request; no zero-click claim is made.

## Recommendation

1. Enforce `input === normalize(input)` in the parent `$name` route before any child loader or component runs. Reject the route if normalization throws or changes any byte.
2. Carry a single canonical name value through owner, resolver, record, authorization, display, and transaction code. Never combine canonical ownership with a resolver selected by raw route bytes.
3. Treat an EIP-1967 slot as implementation metadata, not authenticity. Verify that a permissioned resolver was deployed by the configured `VerifiableFactory` and that the verified implementation is the expected one.
4. Immediately before signing, re-resolve the canonical name, repeat the provenance/role checks, and bind the checked resolver, chain ID, namehash, selector, and arguments into the intent.
5. Display the exact target contract, chain, canonical name, and decoded calldata in the confirmation UI.

## Audited source evidence

The following excerpts are copied from the exact competition-fork execution commit so the report is self-contained. Every cited Portal source path is unchanged from the imported program-audited upstream snapshot.

#### Normalization rejection exists only in the index child

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\routes\$name\index.tsx:438`

```tsx
  438: function App() {
  439:   const { name } = useParams({ from: '/$name/' })
  440: 
  441:   // Validate name format - must be a valid ENS name (normalized + ends with .eth)
  442:   const isValidName = isValidEnsName(name)
  443: 
  444:   const {
  445:     data: resolverAddress,
  446:     isLoading,
  447:     error,
  448:   } = useEnsResolver({
  449:     name,
  450:     universalResolverAddress,
  451:     query: {
  452:       // Don't fetch resolver for invalid names
  453:       enabled: isValidName,
  454:     },
  455:   })
  456: 
  457:   // Show 404 for invalid/malformed names
  458:   if (!isValidName) {
  459:     return <InvalidNameMessage title="Invalid name" />
  460:   }
```

#### Parent route lacks the same invariant

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\routes\$name.tsx:25`

```tsx
   25:   beforeLoad: async ({ params }) => {
   26:     const { name } = params
   27: 
   28:     // Redirect valid TLDs to the dedicated /tld/$tld route
   29:     if (isValidEnsName(name) && isTLD(name)) {
   30:       const shouldRedirect =
   31:         name === 'eth' || (await getDnsSecEnabled(name)) === true
   32:       if (shouldRedirect) {
   33:         throw redirect({ to: '/tld/$tld', params: { tld: name } })
   34:       }
   35:     }
   36: 
   37:     const ownerData = await queryClient.fetchQuery(
   38:       getEnsOwnerQueryOptions({ name }),
   39:     )
   40:     return {
   41:       protocolVersion: ownerData?.protocolVersion as
   42:         | ProtocolVersion
   43:         | undefined,
   44:     }
   45:   },
   46:   loader: ({ params }) =>
   47:     Promise.all([
   48:       queryClient.prefetchQuery(
   49:         getNameRegistriesQueryOptions({ name: params.name }),
   50:       ),
   51:       ...(isRegistrable(params.name)
   52:         ? [
   53:             queryClient.prefetchQuery(
   54:               getNameAvailabilityQueryOptions({ name: params.name }),
   55:             ),
   56:           ]
   57:         : []),
   58:     ]),
   59: })
   60: 
   61: function RouteComponent() {
   62:   const { name } = useParams({ from: '/$name' })
   63:   return (
   64:     <SidebarProvider>
   65:       <ProfileSidebar name={name} />
   66:       <SidebarInset className="w-full min-w-0">
   67:         <MobileHeader />
   68:         <SepoliaNoticeBanner />
   69:         <PageContainer>
   70:           <Outlet />
```

#### Direct child consumes the raw resolver and authorization result

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\routes\$name\edit-records.tsx:61`

```tsx
   61:   const { name } = Route.useParams()
   62:   const { address: connectedAddress } = useConnection()
   63: 
   64:   const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
   65:   const profileQuery = useQuery(
   66:     getProfileQueryOptions({
   67:       name,
   68:       protocolVersion: ownerQuery.data?.protocolVersion,
   69:     }),
   70:   )
   71: 
   72:   // Get resolver address from the correct registry (V1 or V2)
   73:   const { data: resolverAddress, isLoading: isResolverLoading } =
   74:     useNameResolverAddress({ name })
   75:   const { canEdit, isLoading: isCanEditLoading } = useCanEditRecords({ name })
```

#### Save flow prepares the transaction using that resolver

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\routes\$name\edit-records.tsx:540`

```tsx
  540:       <TransactionModal
  541:         transactions={[
  542:           {
  543:             id: SAVE_RECORDS_TRANSACTION_ID,
  544:             title: 'Save records',
  545:             transactionName: 'Set resolver records',
  546:             intent: {
  547:               prepare: () => saveRecordsIntentQuery.data,
  548:               isPending: saveRecordsIntentQuery.isLoading,
  549:               isError: saveRecordsIntentQuery.isError,
  550:             },
  551:             onStart: handleStartSaveRecordsTransaction,
```

#### Resolver classification trusts only attacker-owned storage

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\features\resolver\hooks\useIsPermissionedResolver.ts:35`

```typescript
   35: export const getIsPermissionedResolver = ResultFn(async function* (
   36:   params: GetIsPermissionedResolverParams,
   37: ) {
   38:   const normalizedResolverAddress = params.resolverAddress.toLowerCase()
   39:   if (
   40:     knownPermissionedResolverImplementations.includes(normalizedResolverAddress)
   41:   )
   42:     return ok(true)
   43: 
   44:   const client = yield* safeGetClient()
   45: 
   46:   const implementationSlotValue = yield* fromPromise(
   47:     getStorageAt(client, {
   48:       address: params.resolverAddress,
   49:       slot: EIP1967_IMPLEMENTATION_SLOT,
   50:     }),
   51:     (error) =>
   52:       new IsPermissionedResolverError({
   53:         cause: error,
   54:       }),
   55:   )
   56: 
   57:   const implementationAddress = decodeImplementationAddress(
   58:     implementationSlotValue,
   59:   )
   60:   if (!implementationAddress) return ok(false)
   61: 
   62:   return ok(
   63:     knownPermissionedResolverImplementations.includes(
   64:       implementationAddress.toLowerCase(),
   65:     ),
   66:   )
```

#### Attacker resolver supplies the victim role answer

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\features\records\hooks\useCanEditRecords.ts:75`

```typescript
   75:   const roleQueries = useQueries({
   76:     queries: roles.map((role) => ({
   77:       ...getHasRolesQueryOptions({
   78:         resolverAddress: (resolverAddress ??
   79:           '0x0000000000000000000000000000000000000000') as Address,
   80:         resource: nameResource,
   81:         roles: [role],
   82:         account: (connectedAddress ??
   83:           '0x0000000000000000000000000000000000000000') as Address,
   84:       }),
   85:       enabled:
   86:         enabled && !!connectedAddress && !!resolverAddress && isPermissioned,
   87:     })),
   88:   })
   89: 
   90:   const isOwner =
   91:     !!connectedAddress &&
   92:     !!ownerQuery.data?.owner &&
   93:     isAddressEqual(connectedAddress, ownerQuery.data.owner)
   94: 
   95:   const rolesLoading =
   96:     isPermissioned && roleQueries.some((query) => query.isLoading)
   97: 
   98:   const isLoading =
   99:     (enabled &&
  100:       (ownerQuery.isLoading ||
  101:         resolverQuery.isLoading ||
  102:         (!!resolverAddress && isPermissionedQuery.isLoading) ||
  103:         rolesLoading)) ||
  104:     false
  105: 
  106:   const hasResolverRole = roleQueries.some((query) => query.data === true)
  107: 
  108:   const canEdit = (() => {
  109:     if (!enabled || !connectedAddress || isLoading || !resolverAddress) {
  110:       return false
  111:     }
  112: 
  113:     if (isPermissioned) return hasResolverRole
  114: 
  115:     // Public / non-permissioned resolvers: token owner can edit.
  116:     return isOwner
```

#### Untrusted resolver becomes the transaction target

Execution-commit excerpt, unchanged from the imported audited snapshot: `apps\portal\src\features\records\helpers\saveRecords.ts:92`

```typescript
   92:  * resolver) on chain. Shared by {@link saveRecords} and the modal's pre-start
   93:  * gas estimate, so the estimated call matches what's submitted.
   94:  *
   95:  * @throws if the wallet isn't ready or there are no record changes.
   96:  */
   97: export async function prepareSaveRecordsTransaction({
   98:   name,
   99:   resolverAddress,
  100:   originalRecords,
  101:   pendingChanges,
  102:   walletClient,
  103:   chainId,
  104: }: SaveRecordsTransactionParameters): Promise<CustomTransactionIntent> {
  105:   if (!walletClient.account || !walletClient.chain) {
  106:     throw new Error('Wallet client must have account and chain configured')
  107:   }
  108: 
  109:   // Transform pending changes to ensjs-compatible format
  110:   const recordsInput = transformPendingChangesToSetRecords(
  111:     originalRecords,
  112:     pendingChanges,
  113:   )
  114: 
  115:   const hasChanges =
  116:     (recordsInput.texts?.length ?? 0) > 0 ||
  117:     (recordsInput.coins?.length ?? 0) > 0 ||
  118:     recordsInput.contentHash !== undefined ||
  119:     recordsInput.abi !== undefined
  120: 
  121:   if (!hasChanges) {
  122:     throw new Error('No record changes to save')
  123:   }
  124: 
  125:   // ensjs handles both Public Resolver and Dedicated Resolver patterns.
  126:   // Type assertion is safe since we validated account and chain above.
  127:   const client = walletClient as Parameters<typeof setRecordsWriteParameters>[0]
  128: 
  129:   const writeParams = await setRecordsWriteParameters(client, {
  130:     name,
  131:     resolverAddress,
  132:     texts: recordsInput.texts,
  133:     coins: recordsInput.coins,
  134:     contentHash: recordsInput.contentHash,
  135:     abi: recordsInput.abi,
  136:   })
  137: 
  138:   const data = encodeFunctionData({
  139:     abi: writeParams.abi,
  140:     functionName: writeParams.functionName,
  141:     args: writeParams.args,
  142:   } as Parameters<typeof encodeFunctionData>[0])
  143: 
  144:   return toEoaCustomIntent({
  145:     from: walletClient.account.address,
  146:     to: resolverAddress,
  147:     data,
  148:     chainId,
  149:   })
```

## Proof of Concept

## Proof of Concept

## Actual user flow

1. Victim V owns the canonical NFC name `caf\u00e9-local-poc-r6.eth`. Attacker A separately owns the exact raw NFD name `cafe\u0301-local-poc-r6.eth`. The strings render alike, but their UTF-8 bytes, label hashes, namehashes, owners, and resolvers differ.
2. A assigns the NFD name to attacker-deployed resolver M. M is not a trusted-factory proxy. Its constructor merely writes the known implementation address into its own EIP-1967 implementation slot. M returns `true` when Portal asks whether V has resolver roles.
3. With V's wallet already connected, open the ordinary NFD profile URL as a control:

   ```
   /cafe%CC%81-local-poc-r6.eth
   ```

   Portal renders `Invalid name` because the index route notices that normalization changes the exact input.
4. Open the direct child URL instead:

   ```
   /cafe%CC%81-local-poc-r6.eth/edit-records
   ```

   The parent route does not enforce the index invariant. Portal renders `Edit records`, displays the NFC-looking victim name, obtains attacker-controlled records from M, and enables editing for V based on M's forged role response.
5. Add the harmless text record `com.ens07.poc = local-only victim-authorized proof`, click `Save records`, and advance through the normal Portal transaction dialog. Before wallet authorization, confirm that the local block number is unchanged.
6. Authorize the queued `eth_sendTransaction` request as V. The application-generated transaction is mined with status 1, `from == V`, `to == M`, and selector `0x10f13a8c`.
7. Verify M's `ResolverWriteCaptured` event and storage. `writeCount` changes from 0 to 1; `lastCaller` is V; `lastNode` is the exact NFD namehash; and the stored selector matches the mined calldata.

## Complete runnable Coded PoC bundle

Google Drive: [**DOWNLOAD `ens-07.rar`**](https://drive.google.com/file/d/153JrVL7Y3bhw_8dWIH0hamCN4JcJxx-2/view?usp=sharing)

```
Archive: ens-07.rar
Format: RAR 5
Primary successful evidence run: qualification-20260903-ens07-r6
Independent successful reproduction: qualification-20260903-ens07-r5
```

The linked archive is the complete, untruncated `audit-pocs/ens-07` workspace--not a source excerpt. It contains the attacker-resolver contract, setup code, Docker/Anvil/Panoptes configuration, real Portal Playwright test, one-command runner, generated report, raw transactions and receipts, decoded calldata, event and storage evidence, screenshots, video, Playwright trace, browser evidence, post-seal actor-key scan, and recursively verifiable successful-run manifests. No source file required by the runner is omitted.

The RAR is valid and contains every current ENS-07 workspace file with matching SHA-256 content. Some retained Playwright paths are up to 273 characters, so extract it with WSL/Linux `unrar` as shown below.

The form's image uploads are supplementary UI evidence only. The runnable code and machine-readable transaction/state evidence are in the linked archive.

### Which retained runs are evidence

`qualification-20260903-ens07-r6` is the primary report evidence. It has a passed Playwright result, wrapper exit status `0`, a status-1 exploit receipt, confirmed attacker-contract post-state, an immutable manifest, and a passed independent post-seal actor-key scan. `qualification-20260903-ens07-r5` is a second independently sealed exit-0 reproduction.

The complete workspace also retains three unsealed development captures and sealed failed run `qualification-20260903-ens07-r4` with exit status `141`. Those are preserved for auditability and are not presented as successful evidence.

The r6 post-seal scan inspected every retained r6 file and recursively inspected ZIP members, derived addresses from isolated 32-byte candidates, and found no candidate matching the disposable victim or attacker. This is not a generic claim that the later outer Drive RAR was scanned for every possible credential class.

## Safety and revision

This PoC does not send any transaction to public Sepolia or mainnet. Anvil reads historical Sepolia state at fixed block `11625819`, but every setup and exploit transaction is signed and mined only by local Anvil. Portal, RPC, and the Panoptes relay bind to loopback; the browser blocks non-loopback HTTP/WebSocket requests and receives only the disposable victim key.

```
Program-listed upstream audited revision: 63772fd872af472ced58b009499355f3430c2a86
Competition import commit: cda79acaad59711b943fc68207ebb3f1d0ff8596
Exact competition-fork execution commit: 1c9b47f18fcddd2e864dfe385c4171061c9811ae
Branch: audit-comp-ready
Fork block: 11625819
Local chain ID: 11155111
RPC: http://127.0.0.1:8545
Portal: http://127.0.0.1:5173
Panoptes: http://127.0.0.1:5655/graphql
```

The cited Portal source is unchanged between the imported program revision and execution commit; the later commit changes only dependency manifests/lockfile. Provenance is recorded in `report.md` and r6 `environment/source-revision.txt`.

## Archive map

The principal entry points are:

```
ens-07/
  README.md
  report.md
  contracts/MaliciousResolver.sol
  src/evidence.ts
  src/setup.ts
  tests/ens-07.portal.spec.ts
  tests/playwright.config.ts
  panoptes/contracts.fork-11625819.json
  scripts/run.sh
  scripts/docker-compose.fork-11625819.yml
  scripts/docker-compose.loopback.yml
  scripts/assert-local-only.sh
  scripts/verify-panoptes-labels.sh
  qa/scan-retained-actor-keys.py
  qa/qualification-20260903-ens07-r6-secret-scan.json
  artifacts/runs/qualification-20260903-ens07-r6/
    environment/run-status.txt
    environment/setup.json
    indexer/label-preservation-proof.json
    ui/wallet-account-proof.json
    ui/state-before.json
    ui/transaction.json
    ui/receipt.json
    ui/browser-evidence.json
    ui/state-after.json
    ui/evm-call-trace.json
    ui/*.png
    ui/ens-07-trace.zip
    manifest/SHA256SUMS
```

`src/setup.ts` performs normal local registrations and deploys the attacker resolver through an attacker-signed creation transaction. `tests/ens-07.portal.spec.ts` drives the unmodified Portal UI with only the victim account, proves the normal raw-name route rejects the input, reaches the direct child route, pauses before authorization, authorizes the actual wallet request, decodes the mined transaction, and checks its event and attacker-controlled storage effect. Panoptes responses are not mocked or intercepted.

## Prerequisites

* WSL2 Ubuntu on x86-64
* Git, Node.js 22, Corepack/pnpm, Docker Engine with Compose, Python 3, `curl`, `jq`, `unrar`, `openssl`, `realpath`, `sha256sum`, and `timeout`
* Enough space for a Linux-native checkout, dependencies, and the pinned Docker images
* A read-only Sepolia archive RPC capable of serving block `11625819`; the credentialless endpoint below was used by the retained run

The r6 environment used Ubuntu 24.04.1, Node `v22.23.2`, pnpm `10.27.0`, cast `1.7.1`, solc `0.8.30`, Docker `29.1.3`, and Docker Compose `2.40.3`.

## Exact reproduction commands

Run in WSL Ubuntu. Set `POC_ARCHIVE` to the downloaded RAR. The victim and attacker keys generated below are disposable and used only on local Anvil.

```bash
set -Eeuo pipefail

export POC_ARCHIVE='/path/to/ens-07.rar'
export REPO_ROOT='/root/ens5-poc'
export EXTRACT_ROOT="$(mktemp -d)"

test "$(sha256sum "$POC_ARCHIVE" | cut -d ' ' -f 1)" = \
  '38463de793aba9fd10de55b3ad8ab52caa9dcf144aeafab52a25b78cb966b9f2'
unrar x -idq "$POC_ARCHIVE" "$EXTRACT_ROOT/"

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git "$REPO_ROOT"
git -C "$REPO_ROOT" checkout --detach \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
test "$(git -C "$REPO_ROOT" rev-parse HEAD)" = \
  '1c9b47f18fcddd2e864dfe385c4171061c9811ae'

mkdir -p "$REPO_ROOT/audit-pocs"
cp -a "$EXTRACT_ROOT/ens-07" "$REPO_ROOT/audit-pocs/"

cd "$REPO_ROOT"
corepack pnpm install --frozen-lockfile --ignore-scripts \
  --filter portal... --filter @ens-apps/e2e...
corepack pnpm --filter @ens-apps/e2e exec playwright install --with-deps chromium

docker pull \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
docker pull \
  ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
docker pull \
  ethereum/solc@sha256:b116bf835554d40c501feab0b2c943a8c5eec003b804bcc5b326b85c93da00c2

mkdir -p /root/ens5-poc-tools
docker create --name ens07-foundry-tools \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd true
docker cp ens07-foundry-tools:/usr/local/bin/cast /root/ens5-poc-tools/cast
docker rm ens07-foundry-tools
docker create --name ens07-solc-tools \
  ethereum/solc@sha256:b116bf835554d40c501feab0b2c943a8c5eec003b804bcc5b326b85c93da00c2 --version
docker cp ens07-solc-tools:/usr/bin/solc /root/ens5-poc-tools/solc-0.8.30
docker rm ens07-solc-tools
chmod 0755 /root/ens5-poc-tools/cast /root/ens5-poc-tools/solc-0.8.30

export PATH="/root/ens5-poc-tools:/usr/local/bin:/usr/bin:/bin"
export ENS07_SOLC_BIN='/root/ens5-poc-tools/solc-0.8.30'
export SEPOLIA_FORK_URL='https://sepolia.gateway.tenderly.co'
export RPC_URL='http://127.0.0.1:8545'
export PORTAL_URL='http://127.0.0.1:5173'
export INDEXER_URL='http://127.0.0.1:5655/graphql'
export VICTIM_PRIVATE_KEY="0x$(openssl rand -hex 32)"
export ATTACKER_PRIVATE_KEY="0x$(openssl rand -hex 32)"
test "$VICTIM_PRIVATE_KEY" != "$ATTACKER_PRIVATE_KEY"
export ENS07_RUN_ID='triager-ens07-01'
export ENS07_LABEL_SUFFIX='triager-ens07-01'

cd "$REPO_ROOT/audit-pocs/ens-07"
bash ./scripts/run.sh
```

`SEPOLIA_FORK_URL` supplies read-only historical state to local Anvil. It is never passed to Portal/the browser and never receives a signed transaction. A different read-only archive endpoint can be substituted. Never use funded or production keys.

The runner fails closed unless application endpoints are loopback, `web3_clientVersion` identifies Anvil, chain ID is `11155111`, the checkout is at the expected commit, and required files are present. It performs normal registrations, waits for unmodified Panoptes to preserve both exact names, starts Portal, pauses at the actual wallet request, verifies no block was mined before authorization, mines the victim transaction, checks receipt/event/storage, captures evidence, and seals its manifest.

## Expected terminal result

Fresh account, resolver, suffix, and transaction values will differ. Success ends with:

```
[ENS-07] Loopback URL policy passed.
[ENS-07] Safety gate passed: loopback-only Anvil RPC, chain ID 11155111.
[ENS-07] Deploying the malicious resolver and distinct NFC/NFD local names.
[ENS-07] Panoptes preserved the exact NFD/NFC pair.
[ENS-07] Portal is ready: http://127.0.0.1:5173
[ENS-07] Running the normal connected-wallet Portal flow and negative control.

Running 1 test using 1 worker
  1 passed

[ENS-07] Captured after chain state.
[ENS-07] ENS-07 browser flow completed successfully.
```

## Decisive output from primary successful run r6

All hashes below are local-Anvil transaction hashes and are verified from the included raw transactions, receipts, call trace, event, and state--not from a public block explorer. Unicode is written with escapes and UTF-8 hex so form normalization cannot change the identities being proved.

```
Victim V:   0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db
Attacker A: 0xCcfE3e898EE111699586B099D6F93F8703bD1FA2

Victim NFC name:       caf\u00e9-local-poc-r6.eth
Attacker raw NFD name: cafe\u0301-local-poc-r6.eth
Canonical UTF-8: 0x636166c3a92d6c6f63616c2d706f632d72362e657468
Raw UTF-8:       0x63616665cc812d6c6f63616c2d706f632d72362e657468
Canonical namehash: 0x59b569000cc49a0b92991135aa3d445b8c6f92870f78fb98fcb5a8ccff35d450
Raw namehash/node:  0xd5393c2c6c3770262151b1ab7af83174f7f6b184df3cedbad91d126701922957
Exact names and namehashes distinct: true
Victim owns exact raw name: false

Raw main route: /cafe%CC%81-local-poc-r6.eth
Main-route result: Invalid name rendered
Direct child route: /cafe%CC%81-local-poc-r6.eth/edit-records
Direct child edit controls rendered: true

Attacker resolver M: 0xF1ce623B66d24B8BF746a888156A7194C6627bd3
Trusted implementation value: 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
Configured verifiable factory: 0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
M is a trusted-factory proxy: false
M's self-controlled EIP-1967 slot decodes to trusted implementation: true
M returns victim role: true
No anvil_setCode/anvil_setStorageAt or impersonation used: true
M deployment tx: 0xc230becb8460854e5833d050c089af1f350a8d8b78c57cba0e127ce7ed7e156f

Browser-exposed accounts: [V]
Exposed account count: 1
Attacker key available to browser: false

Wallet request kind: eth_sendTransaction
Pending request count: 1
Block before request: 11625832
Block while pending: 11625832
Transaction mined before victim authorization: false

Transaction: 0xb6136e155477d1da5955875ec96c98e3c7ead530f31d46ef8d3e5fcad6c816da
From: V
To: attacker resolver M
Chain ID: 11155111
Receipt status: 1
EVM trace: direct CALL from V to M

Decoded function: setText
Selector: 0x10f13a8c
Decoded node: 0xd5393c2c6c3770262151b1ab7af83174f7f6b184df3cedbad91d126701922957
Decoded key: com.ens07.poc
Decoded value: local-only victim-authorized proof

Emitted event: ResolverWriteCaptured
Event caller: V
Event node: exact attacker-owned raw namehash
writeCount: 0 -> 1
lastCaller: V
lastSelector: 0x10f13a8c
lastNode: exact attacker-owned raw namehash

Playwright: 1 passed (22.6s)
Browser evidence result: confirmed
Runner exit status: 0
Post-seal actor-key scan: passed; no matching victim/attacker key candidate
```

The raw-Unicode delivery route overlaps public issue `EXP-INPUT-005`, but the independent resolver-authentication defect does not depend on visual confusion alone: Portal treats attacker contract M as a trusted permissioned resolver solely because M populates its own EIP-1967 slot with the expected implementation address. No factory/provenance check binds M to an ENS deployment. Portal then uses M's forged role answer and sends the victim-approved transaction directly to M.

The PoC intentionally uses a harmless setter and does not claim theft or zero-click compromise. It proves the listed connected-wallet consequence: the UI presents an ENS record update, while the victim signs a mined transaction whose target is attacker-deployed code.

## Verify the retained evidence

```bash
cd "$REPO_ROOT/audit-pocs/ens-07"
bash ./scripts/generate-manifest.sh --verify \
  ./artifacts/runs/qualification-20260903-ens07-r6
```

The expected result is one `OK` line per r6 evidence file and exit status `0`. The recorded primary-run results are:

```
Playwright status: passed
Playwright failed tests: []
Runner exit status: 0
Post-seal actor-key scan result: passed
```

## Uploaded image evidence

1. `ENS-07-01-main-route-negative-control.png` -- the ordinary raw-NFD profile route rejects the name as invalid.
2. `ENS-07-02-direct-child-false-authorization.png` -- the direct child route for the same raw-NFD name renders **Edit records** to the connected victim and displays the attacker's address record.
3. `ENS-07-03-status-1-transaction-success.png` -- Portal's **Set resolver records** step completes after the separately authorized mined transaction.

The images establish the visible route bypass and UI sequence. The included pending-request evidence, status-1 transaction/receipt, decoded attacker-owned node, direct EVM call trace, emitted event, and attacker-contract storage delta establish the connected-wallet impact.
