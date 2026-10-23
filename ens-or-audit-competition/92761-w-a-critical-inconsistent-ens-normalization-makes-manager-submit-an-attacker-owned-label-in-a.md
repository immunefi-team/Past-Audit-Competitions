# 92761 w a critical inconsistent ens normalization makes manager submit an attacker owned label in a victim approved renewal

**Submitted on Sep 12th 2026 at 03:25:07 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92761
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Manager app resolves ownership and expiry for an ENSIP-15-normalized `.eth` name, but its renewal route later preserves the raw route label into `ETHRegistrar.renew`.

An attacker can own a byte-distinct NFD label that normalizes to a victim-owned NFC label.

When the victim opens the attacker's raw profile route, Manager classifies the victim as the owner, suppresses the third-party-renewal warning, displays the victim name's expiry, and then has the victim approve a transaction whose name argument identifies the attacker's different registration.

This is a malicious interaction with an already-connected wallet through modification of a transaction argument: the checked resource and submitted resource differ.

## Vulnerability Details

Affected assets: `apps/manager` and `packages/transaction-manager` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The bug is a time-of-use identity split between the normalized profile services and the raw renewal parser.

`normalizeEthName` applies full ENS normalization and returns the canonical name:

```ts
// apps/manager/src/features/profile/service/profileName.ts:14-38
normalized = normalize(name)
...
return { leafLabel, name: normalized, ... }
```

Both security-relevant reads use that normalized identity. `getOwner` queries V2 ownership using `ethName.name`, and `getExpiry` does the same:

```ts
// apps/manager/src/features/profile/service/profileOwner.ts:24-64
const ethName = normalizeEthName(params.name)
...
ensjsv2_getOwner(client, { name: ethName.name })

// apps/manager/src/features/profile/service/profileExpiry.ts:73-109
const ethName = normalizeEth2LdName(name)
...
ensjsv2_getExpiry(client, { name: ethName.name, registryAddress: ENS_REGISTRY })
```

The profile passes that normalized owner result into `ProfileRenewAction`. If it matches the connected wallet, the component skips `ThirdPartyRenewalDialog` and links directly to `/renew/$name` with the original raw `name`:

```tsx
// apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66-84
if (shouldShowThirdPartyRenewalWarning(isOwner)) {
  return <ThirdPartyRenewalDialog name={name} ... />
}
return <LinkButton params={{ name }} to={getRenewalRoute(resolvedProtocol)} ... />
```

The renewal route again obtains expiry through the normalized profile query, but separately parses the raw route parameter with `parseRenewableName` and returns that raw label:

```ts
// apps/manager/src/routes/renew/$name.tsx:18-63
const parsedName = parseRenewableName(name)
const expiryData = await queryClient.ensureQueryData(profileExpiryQuery(name))
...
return { label: parsedName.value.label, currentExpiry: expiryData.expiry }
```

`parseName` is misleadingly named: it only calls `trim().toLowerCase()` and never performs ENSIP-15 or Unicode normalization:

```ts
// apps/manager/src/features/register-v2/utils/name-parser.ts:44-80
const normalized = name.trim().toLowerCase()
...
return ok({ subLabels: labels, label, tld })
```

That raw label is retained by the renewal state machine and handed to `submitRenewActor`. The transaction package encodes it unchanged as the first argument of `ETHRegistrar.renew`:

```ts
// apps/manager/src/features/renew/state/renewalUi.machine.ts:341-355
return { label: submission.label, ... }

// packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1029,1072-1091
args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS]
...
description: `Renew ${input.label}.eth`
```

For example, `abcde\u0301.eth` (NFD, bytes `6162636465cc81`) normalizes to `abcd\u00e9.eth` (NFC, bytes `61626364c3a9`), but the V2 registrar hashes the supplied label bytes. The two labels therefore have distinct token IDs and can have different owners. Manager checks the NFC token and submits renewal for the NFD token.

The attacker can create this state through Manager itself. `/register/$name` uses the same non-ENS-normalizing `parseName` and returns its raw label (`apps/manager/src/routes/register/$name.tsx:20-58`), while `generateCommitment` and `encodeRegisterCall` preserve those bytes into official registrar calls (`packages/transaction-manager/src/machines/registration/registration.actors.ts:133-165,222-255`). The Foundry control independently proves the identical public commit/reveal setup through the official registrar.

The attack requires no control over the victim's wallet, name, or browser state. The attacker registers the raw NFD sibling and exposes the raw profile URL. The victim follows the ordinary Manager renewal workflow for what the UI treats as their own canonical name.

Once confirmed, the result is deterministic: the victim pays and the attacker's registration is extended.

## Impact Details

Selected impact: Malicious interactions with an already-connected wallet, including modifying transaction arguments or parameters.

Manager establishes that the connected victim owns the canonical NFC name, displays that name's expiry, and suppresses the warning intended for third-party renewals. It then constructs a wallet transaction whose `label` argument identifies the attacker's byte-distinct NFD token instead of the NFC resource that passed those checks.

The victim's ERC-20 balance decreases by the quoted renewal price, the attacker's name receives the full renewal duration, and the victim's checked name is unchanged.

The attached official-contract test proves all four postconditions. The payment still goes to the fixed registrar beneficiary, so this report does not claim that the registrar or funds recipient is substituted; the eligible failure is the wrong resource argument in the signed transaction.

This is not the known display-only normalization issue EXP-INPUT-005. No ENS normalization failure or mere homograph rendering is required: both labels are accepted by the V2 registrar, the transaction succeeds, tokens move, and the wrong registration is renewed.

## Existing Safeguards and Triage FAQ

* **"The registrar and payment recipient are correct."** That is why I select this `Malicious interactions` impact instead of theft or unintended-recipient impact. The modified field is the paid renewal's label argument, and the PoC proves that the wrong registration receives the service.
* **"This is only Unicode display confusion."** The labels have distinct token IDs, owners, and expiries. The victim loses tokens, the attacker-owned expiry increases, and the checked victim-owned expiry does not change.
* **"The contract accepts both labels."** Correct contract behavior is execution context. The in-scope flaw is Manager checking NFC ownership and expiry before signing raw NFD calldata.

## Duplicate Check

EXP-INPUT-005 stops at normalization and homograph-style display confusion. This report proves a successful paid state transition against a byte-distinct registration and requires a fix at Manager's profile-to-renewal identity boundary.

## Remediation

Establish one canonical name tuple at route entry and use it for every ownership, expiry, warning, pricing, display, and transaction operation. `parseRenewableName` should use ENS normalization and reject any route value whose raw representation differs from the canonical representation, or the route loader should pass `normalizeEth2LdName(name).label` to `RenewalPage`. Before enabling confirmation, bind the displayed owner and expiry to the exact `labelhash` that will be encoded in `renew` and fail closed on any mismatch. Add regression tests with distinct NFD/NFC registrations and assert that the checked token ID always equals the submitted token ID.

## References

* `apps/manager/src/features/profile/service/profileName.ts`
* `apps/manager/src/features/profile/service/profileOwner.ts`
* `apps/manager/src/features/profile/service/profileExpiry.ts`
* `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx`
* `apps/manager/src/routes/renew/$name.tsx`
* `apps/manager/src/features/register-v2/utils/name-parser.ts`
* `apps/manager/src/features/renew/state/renewalUi.machine.ts`
* `packages/transaction-manager/src/machines/registration/registration.actors.ts`

## Link to Proof of Concept

<https://gist.github.com/niafreu/6bb331fbee265191a8f08e6c2b16de95>

## Proof of Concept

## Proof of Concept

The PoC is available at <https://gist.github.com/niafreu/6bb331fbee265191a8f08e6c2b16de95>

1. Check out `audit-comp-ens` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and install its pinned dependencies.
2. Run the Manager production-flow test:

```bash
export ENS_AUDIT_REPO=/absolute/path/to/audit-comp-ens
cd "$ENS_AUDIT_REPO"
apps/manager/node_modules/.bin/vitest run --config /absolute/path/to/this-poc/vitest.config.ts
```

It imports the in-scope production functions, invokes the actual `/renew/$name` route loader with raw NFD, and proves that:

* owner and expiry queries receive canonical NFC `abcd\u00e9.eth`;
* the connected victim is classified as owner, so the third-party warning predicate is false;
* the route loader combines raw NFD label `abcde\u0301` with the canonical NFC expiry;
* decoded production calldata calls the canonical ETHRegistrar with `renew("abcde\u0301", ...)`, not `renew("abcd\u00e9", ...)`.

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

3. Check out `ensdomains/contracts-v2` at `48b3e2d39513b9dd32ef1850877a29009bc807b9`, install its Foundry dependencies, and run the real-contract consequence test:

```bash
export CONTRACTS_ROOT=/absolute/path/to/contracts-v2/contracts
forge test \
  --root "$CONTRACTS_ROOT" \
  --contracts /absolute/path/to/this-poc/test \
  --out /tmp/manager-nfd-renewal-out \
  --cache-path /tmp/manager-nfd-renewal-cache \
  --match-contract ManagerNFDRenewalTest \
  -vv
```

This deploys the official ENSv2 fixture, registers canonical NFC to the victim and byte-distinct NFD to the attacker through `ETHRegistrar`, and broadcasts the exact NFD `renew` calldata as the victim. It asserts:

* the victim's NFC expiry is unchanged;
* the attacker's NFD expiry increases by 365 days;
* the victim's USDC balance decreases by the quoted amount;
* the beneficiary receives that amount;
* both owners remain unchanged.

The second test is the negative control: using the canonical label renews only the victim's checked token.

Expected result:

```
[PASS] test_managerRawCalldataRenewsAttackersSibling()
[PASS] test_negativeCanonicalBindingRenewsVictimsCheckedToken()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
