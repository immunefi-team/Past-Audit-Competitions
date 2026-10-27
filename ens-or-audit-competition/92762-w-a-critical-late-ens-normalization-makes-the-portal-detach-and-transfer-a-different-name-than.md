# 92762 w a critical late ens normalization makes the portal detach and transfer a different name than the transfer page authorizes

**Submitted on Sep 12th 2026 at 03:49:29 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92762
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

I found that the Portal checks ownership and configuration for the exact raw route name, but ENSIP-15-normalizes that name before building its transfer calls. An attacker can gift the victim a raw NFD label `R` that normalizes to the victim's valuable NFC label `N`. When the victim transfers displayed name `R`, Portal instead detaches and transfers `N`.

## Vulnerability Details

Affected asset: `apps/portal` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The transfer route passes its raw `$name` parameter into owner discovery and trusts the returned owner:

```tsx
// apps/portal/src/routes/$name/ownership/transfer.tsx
const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
...
if (!isAddressEqual(data.owner, address)) return <Unauthorized />
return <AuthorizedTransfer name={name} registryAddress={data.registryAddress} owner={data.owner} />
```

`resolveEnsOwner` passes the supplied name unchanged to ENSJS `getOwnerV2` and `getNameRegistries`. ENSJS uses DNS packet encoding, so byte-distinct NFD and NFC labels select distinct ENSv2 resources.

The cleanup discovery hook then splits its identity model. It reads the resolver and registry from the raw name, but checks the owner's roles using `getLabel(name)`, which ENSIP-15-normalizes NFD into NFC:

```ts
// apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts
label = getLabel(name)
const resolverQuery = useNameResolverAddress({ name })
const registriesQuery = useQuery(getNameRegistriesQueryOptions({ name }))
...
getNameRolesForAccountQueryOptions({ registryAddress, label: label ?? '', account: owner })
```

`SendNameForm` initializes both cleanup switches to `true`. When raw `R` has a resolver and subregistry and the victim has the relevant roles on canonical `N`, both options are visible and included by default:

```ts
const [options, setOptions] = useState({
  setEthAddress: true,
  detachResolver: true,
  detachRegistry: true,
})
```

`useTransferName` normalizes the name again before all three writes:

```ts
const tokenId = await getEnsTokenId({ label: getLabel(name), registryAddress })
...
const label = getLabel(name)
detachNameResolver({ ...common, label, registryAddress, id })
detachNameRegistry({ ...common, label, registryAddress, id })
transferToken({ ...common, registryAddress, tokenId, recipient, id })
```

The helper descriptions still use raw `name`, while the calldata uses the normalized label or its token ID. Thus the Portal presents cleanup and transfer for `R`, but signs:

```
setResolver(labelhash(N), 0)
setSubregistry(labelhash(N), 0)
safeTransferFrom(victim, recipient, tokenId(N), 1, 0x)
```

My PoC uses `R = cafe\u0301.eth` and `N = caf\u00e9.eth`. Both are registered through the official ETHRegistrar as separate resources. The attacker gifts `R` to the victim, who already owns `N`. Production Portal functions discover `R` but generate and broadcast the three calls above for `N`.

The resulting state is:

```
R resolver = attacker-configured resolver
R subregistry = attacker-configured subregistry
R owner = victim
N resolver = 0
N subregistry = 0
N owner = recipient selected for R
```

The negative control binds the same helpers to the exact raw label; only `R` is detached and transferred, while `N` remains unchanged.

The vulnerable page is directly reachable as `/cafe%CC%81.eth/ownership/transfer`. The router percent-decodes the route without NFC-normalizing it. The victim connects their wallet, enters any legitimate recipient, leaves the two visible cleanup defaults enabled, selects `Transfer name`, and approves the three prompted transactions. The Portal's own transaction descriptions name raw `R`; an external wallet's presentation may vary and is not relied upon by the PoC.

## Impact Details

It seems to be again the impact "Malicious interactions with an already-connected wallet," specifically "Modifying transaction arguments or parameters."

The victim sees raw gifted name `R` throughout a normal transfer flow, but Portal changes the resource argument of two cleanup transactions and the token ID of the final transfer. The wrong cleanup calls damage valuable sibling `N` even when the recipient is unrelated to the attacker and is exactly whom the victim intended to receive `R`.

The attacker creates the entire precondition with ordinary calls and no privilege: register byte-distinct `R`, set its nonzero pointers, and gift it to the victim. No third-party bug, leaked credential, production probing, browser defect, or unprompted background action is required. The victim only follows the Portal's ordinary transfer workflow and approves transactions whose descriptions name `R`.

Known Issue `EXP-INPUT-005` covers normalization transformations whose impact is homograph-style display confusion rather than incorrect resolution. My report crosses that boundary because the decoded, broadcast calldata and authoritative post-state target a different labelhash and token.

## Existing Safeguards and Counterarguments

* **"The recipient is legitimate and the wallet transaction succeeds."** The corrupted field is the resource identity: Portal describes raw `R` but detaches and transfers distinct `N`. The impact I selected is about modified transaction arguments and does not require attacker profit.
* **"The victim owns both names."** Ownership makes the calls executable; it does not authorize Portal to substitute one valuable token for another. The wrong cleanup calls damage `N` even when the recipient is unrelated to the attacker.
* **"This is only normalization display confusion."** My PoC decodes and broadcasts different labelhash and token-ID arguments and reads the resulting authoritative state.

## Duplicate Check

EXP-INPUT-005 has no successful wrong-resource transaction. My report discovers raw state and submits normalized cleanup and transfer calls. Resolver detachment, subregistry detachment, and transfer share one identity-boundary root.

## Remediation

I recommend resolving the route name once to an immutable tuple containing its exact label bytes, labelhash or resource, token ID, registry, owner, resolver, and subregistry. The same tuple should drive discovery, role checks, descriptions, cleanup, and transfer.

Before dispatch, decode every prepared call and require its labelhash or token ID to match the resource that passed owner authorization. Rejecting new non-canonical registrations is useful defense in depth, but it does not protect existing byte-distinct resources.

## References

* `apps/portal/src/routes/$name/ownership/transfer.tsx`
* `apps/portal/src/utils/ens/resolveEnsOwner.ts`
* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts`
* `apps/portal/src/features/transfer/components/SendNameForm.tsx`
* `apps/portal/src/features/transfer/hooks/useTransferName.ts`
* `apps/portal/src/utils/token/getLabel.ts`
* `apps/portal/src/features/transfer/helpers/detachNameResolver.ts`
* `apps/portal/src/features/transfer/helpers/detachNameRegistry.ts`
* `apps/portal/src/features/transfer/helpers/transferToken.ts`

## Link to Proof of Concept

<https://gist.github.com/niafreu/90623083a74f71369c0858a978a29f63>

## Proof of Concept

## Proof of Concept

My PoC is available at <https://gist.github.com/niafreu/90623083a74f71369c0858a978a29f63> and contains:

* `raw-cleanup-integrated.test.ts`: starts loopback Anvil, deploys official ENSv2 contracts, invokes the production Portal read/build helpers, broadcasts the resulting calls, and asserts exact post-state.
* `NFDCleanup.t.sol`: independent official `ensdomains/contracts-v2` commit/reveal proof, including exact-raw negative control. - `vitest.config.ts`: resolves imports to the audited monorepo.
* `README.md`: setup and run instructions.

Run the integrated Portal proof from the audited repository after installing its pinned dependencies and building `ensdomains/contracts-v2` at commit `48b3e2d39513b9dd32ef1850877a29009bc807b9`:

```bash
export ENS_AUDIT_REPO=/absolute/path/to/audit-comp-ens
export CONTRACTS_ROOT=/absolute/path/to/contracts-v2/contracts
cd "$ENS_AUDIT_REPO"
apps/portal/node_modules/.bin/vitest run --config /absolute/path/to/poc/vitest.config.ts
```

Expected result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

Run the independent contract proof:

```bash
forge test --root "$CONTRACTS_ROOT" --contracts /absolute/path/to/poc/test --out /tmp/nfd-cleanup-out --cache-path /tmp/nfd-cleanup-cache --match-contract NFDCleanupTest -vv
```

Expected result:

```
[PASS] test_negativeControlExactRawCleanupDoesNotMutateCanonicalSibling()                                             [PASS] test_portalCleanupAndTransferMutateCanonicalSibling()                                                          Suite result: ok. 2 passed; 0 failed; 0 skipped
```

My positive test decodes both cleanup calls and the ERC-1155 transfer, broadcasts them from the victim, and asserts that only `N` was detached and transferred. My negative control sends otherwise identical calls using the exact raw label and asserts that only `R` changes.

For speed, my integrated Portal test creates the two registry entries directly before exercising the production reads and transaction builders. My independent Foundry test establishes the same raw-NFD/NFC setup through the official ETHRegistrar commit/reveal flow, including the attacker's configuration and gift of `R` to the victim.
