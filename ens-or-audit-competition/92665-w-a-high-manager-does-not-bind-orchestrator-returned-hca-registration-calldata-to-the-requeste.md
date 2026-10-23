# 92665 w a high manager does not bind orchestrator returned hca registration calldata to the requested calldata allowing an attacker controlled subregistry on a victim owned root name

**Submitted on Sep 11th 2026 at 11:29:49 UTC by @Pig46940 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92665
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The frozen Manager registration builder requests `ETHRegistrar.register` with `subregistry = address(0)`. In the HCA path, Manager passes those calls to the repository-pinned `@rhinestone/sdk@1.8.0`, but it does not verify that the destination operation returned by the Orchestrator is identical to the registration it requested before the active session signs it.

The two complementary runnable PoCs demonstrate the gap. The Node PoC shows the pinned SDK signing an Orchestrator-returned registration for the same label and owner but with a different secret and an attacker-controlled subregistry. The Foundry PoC shows that the corresponding separately precommitted tuple passes the vulnerable HCA policy, leaves the victim as root owner, and installs the attacker-controlled subregistry. Manager's post-registration verification also passes because it does not inspect that field.

The Critical impact is reached by the registration itself: the victim-owned root name is registered with its subregistry pointing to a registry controlled by the attacker. The later `pay.<registered-name>` operation only proves control of that registry.

Exploitation requires a Byzantine or faulty Orchestrator response. Rhinestone documents the Orchestrator as “trustless from the user's perspective.” This report does not claim that an ordinary visitor can override the configured endpoint, intercept HTTPS, or compromise the deployed Rhinestone service.

## Vulnerability Details

Manager deliberately constructs the reveal with a zero subregistry:

```typescript
args: [
  params.label,
  params.wallet,
  params.secret,
  ZERO_ADDRESS,
  params.resolver,
  params.duration,
  c.usdc,
  REFERER,
],
```

The owner-authorized destination-session salt binds the session nonce, expiry, resolver, refund token, and refund limits. The on-chain registration policy separately constrains the owner and resolver. Neither layer binds the registration label, commitment secret, or subregistry.

The transaction manager enables SDK execution verification and sends the requested calls:

```typescript
const sessionSigners = signer.session
  ? ({
      type: 'experimental_session' as const,
      session: signer.session.session,
      ...(sessionEnableData ? { enableData: sessionEnableData } : {}),
      verifyExecutions: true,
    } satisfies NonNullable<Transaction['signers']>)
  : undefined

const sdkParams = {
  sourceChains: [chain],
  targetChain: chain,
  calls: [...calls],
  // ...
  ...(sessionSigners ? { signers: sessionSigners } : {}),
} satisfies Transaction

const transaction = await account.sendTransaction(sdkParams)
```

The pinned SDK obtains a route and signs the destination operation returned by the Orchestrator. Here, `verifyExecutions: true` verifies the returned executions when producing the session authorization; it does not establish that they equal Manager's original `calls`. Manager does not independently compare the returned chain, target, selector, arguments, value, and call order with its request.

The attacker does not need Manager's random secret. `ETHRegistrar.makeCommitment` binds the registration tuple but not `msg.sender`, so a second commitment can be submitted for the same label and victim owner using an attacker-controlled secret and subregistry.

Before the real commit/reveal, HCA budget estimation sends the same Orchestrator a reveal-shaped quote containing the intended label, duration, deterministic resolver, and HCA context. The quote uses placeholder owner and secret values, but the HCA owner is publicly readable from the HCA contract. A Byzantine Orchestrator can therefore prepare its second commitment in time to mature alongside Manager's commitment. The attack still requires that commitment to be mined and satisfy the registrar's minimum commitment age.

The exploit sequence is:

```
The victim starts a normal Manager HCA registration for label L
→ the pre-commit quote reveals L and the HCA context to the Orchestrator
→ the attacker commits the same L and victim owner with its own secret
  and attacker-controlled subregistry
→ Manager commits L with its own secret and zero subregistry
→ after both commitments mature, Manager requests its normal reveal
→ the Orchestrator returns the attacker's separately committed reveal for L
→ the HCA session signs it, and the policy accepts the unchanged owner and resolver
→ registration completes with the victim as root owner and the attacker's
  registry installed as the name's subregistry
```

The substitution preserves the requested label, owner, resolver, duration, payment token, and referrer while changing the secret and subregistry. The negative control changes only the subregistry while relying on Manager's original commitment; it reverts because the reveal tuple no longer matches that commitment. This confirms why the attacker needs a separately matching commitment.

Manager's post-registration verifier checks only status, owner, and resolver:

```typescript
const verified =
  Number(state.status) === STATUS_REGISTERED &&
  isAddressEqual(state.latestOwner, input.wallet) &&
  isAddressEqual(registryResolver, expectedResolver)
```

The Foundry PoC satisfies all three conditions while the subregistry remains attacker-controlled, so the same-label substitution is not detected after execution.

### PoC coverage and scope

The Node PoC supplies Manager-equivalent calldata to the pinned SDK. Its deterministic Orchestrator adapter receives the requested zero-subregistry registration, returns the same label and owner with a different secret and nonzero subregistry, and confirms that the packed `targetExecutionSignature` authorizes the returned operation.

The Foundry PoC independently tests the matching on-chain authorization and state transition. Its `MockRegistrationIntentExecutor` reproduces the production `SingleChainOps` digest and invokes the real HCA validator/account execution path, but it does not use the production Intent Executor contract. It does not replace the HCA authorization policy, registrar, registry, or resulting state transitions.

The PoCs are complementary, not one continuous end-to-end execution against the deployed service or a public chain: Node establishes route-to-signature behavior, while Foundry establishes on-chain acceptance and impact.

The report does not claim a vulnerability in the out-of-scope Rhinestone service. The in-scope defect is that Manager requests one registration tuple but does not enforce equality with the operation receiving its active HCA session signature. Contracts-v2 PR #443 independently corroborates the missing subregistry constraint and implements the protocol-side restriction; it is cited as technical evidence, not as an in-scope asset or automatic basis for Code Freeze treatment.

This differs from `R2-03`, which concerns the authority and lifetime of the browser-held session key, and from `WEB-1249`, which concerns replacing an existing child registry during later subname creation. This report concerns substituted arguments during initial registration.

## Impact Details

Selected impact, quoted from the Immunefi interface:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

The demonstrated registration leaves the victim as root owner but installs an attacker-controlled registry in the name's subregistry field. This matches the listed example of a registration assigning the name's registry to an attacker; the example lists ownership, registry, and resolver as alternatives and does not require theft of the root NFT.

The impact is complete inside the Manager registration flow: Manager requested `subregistry = address(0)`, but the session authorized and executed the same-label registration with the attacker-controlled subregistry. The Foundry PoC then has the attacker create `pay`, own it, and set its resolver pointer without another victim authorization. That later operation demonstrates control of the installed registry and is not needed to create the qualifying state.

The victim can later repair the subregistry pointer. The PoCs do not establish permanent bricking, root-NFT theft, actual fund theft, or a complete name-resolution query. Those stronger outcomes are not claimed.

## Remediation

1. Require `subregistry == address(0)` for HCA session registrations, as implemented by contracts-v2 PR #443. A nonzero subregistry should require explicit owner authorization.
2. Before session signing, decode every Orchestrator-returned destination execution and compare its chain, target, selector, arguments, value, and call order with Manager's requested executions. Reject any mismatch.

## References

* [HCA reveal builder passes zero as subregistry](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/registration-calls.ts#L280-L298)
* [HCA destination-session salt fields](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/session.ts#L101-L137)
* [Transaction manager enables execution verification and calls the SDK](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/warp-transport.actor.ts#L107-L183)
* [Pre-commit budget estimation submits a reveal-shaped quote](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L308-L431)
* [Manager post-registration verification omits subregistry](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L906-L951)
* [ETHRegistrar commitments are not sender-bound](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registrar/ETHRegistrar.sol#L209-L220)
* [The HCA owner is publicly readable](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/hca/StandaloneSingleOwnerHCA.sol#L249-L258)
* [Rhinestone trust assumptions](https://docs.rhinestone.dev/home/concepts/intents-and-erc4337)
* [PR #443: require a zero subregistry for HCA session registrations](https://github.com/ensdomains/contracts-v2/pull/443)

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/86f10adaf67be4131fc6f653b049b0dc>

## Proof of Concept

## Proof of Concept

The PoC is provided through the following unlisted Secret Gist:

<https://gist.github.com/HiroyukiNaito/86f10adaf67be4131fc6f653b049b0dc>

Pinned Gist revision:

```
8c98e934748cbea2f59e5bbc205bb5961c4480ac
```

The Gist contains two complementary runnable PoCs:

* `hca-session-subregistry-route-substitution.poc.mjs` isolates the signing boundary. It supplies calldata matching Manager's cited zero-subregistry builder to the repository-pinned SDK and proves that the packed `targetExecutionSignature` authorizes the same-label operation returned by the controlled Orchestrator adapter, including its different secret and nonzero subregistry.
* `HCARegistrationSessionSubregistryImpact.poc.t.sol` independently proves that Manager's and the attacker's commitments can coexist for the same label with distinct secrets, that the attacker-subregistry tuple passes the vulnerable HCA policy, and that Manager's status/owner/resolver post-check still passes. It then proves attacker child ownership and resolver-pointer assignment. Its negative control proves that changing only the subregistry while retaining Manager's secret cannot consume Manager's commitment.

### Environment

```
Linux or WSL
Node.js 22.16.0
pnpm 10.27.0
Foundry forge 1.6.0 or compatible
Solidity 0.8.27
```

### Tested source revisions

```
Application repository:
https://github.com/immunefi-team/audit-comp-ens.git
1c9b47f18fcddd2e864dfe385c4171061c9811ae

Pinned vulnerable contracts-v2 repository:
https://github.com/ensdomains/contracts-v2.git
8d1c89350729f87b3968cf41ef3d683951444a44
```

### Setup

Create a reproduction directory, clone the pinned Secret Gist into it, and obtain the two fixed source revisions:

```bash
mkdir ens-hca-subregistry-poc
cd ens-hca-subregistry-poc

git clone \
  https://gist.github.com/86f10adaf67be4131fc6f653b049b0dc.git \
  poc

git -C poc checkout \
  8c98e934748cbea2f59e5bbc205bb5961c4480ac

test -f poc/hca-session-subregistry-route-substitution.poc.mjs
test -f poc/HCARegistrationSessionSubregistryImpact.poc.t.sol

git clone https://github.com/immunefi-team/audit-comp-ens.git
git -C audit-comp-ens checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
(cd audit-comp-ens && pnpm install --frozen-lockfile)

git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
git -C contracts-v2 checkout 8d1c89350729f87b3968cf41ef3d683951444a44
git -C contracts-v2 submodule update --init --recursive
```

### Run the signing-boundary PoC

From `ens-hca-subregistry-poc`:

```bash
ENS_HCA_POC_TARGET="./audit-comp-ens" \
  node "./poc/hca-session-subregistry-route-substitution.poc.mjs"
```

Expected decisive output:

```
"classification": "LOCKED SDK SIGNED SAME-LABEL ORCHESTRATOR-SUBSTITUTED PRECOMMITTED REGISTRATION"
"requestedLabel": "victim-session-name"
"returnedAndSignedLabel": "victim-session-name"
"requestedSecret": "0x2222222222222222222222222222222222222222222222222222222222222222"
"returnedAndSignedSecret": "0x3333333333333333333333333333333333333333333333333333333333333333"
"requestedSubregistry": "0x0000000000000000000000000000000000000000"
"returnedAndSignedSubregistry": "0x2000000000000000000000000000000000000001"
"sessionSignatureContainsSubstitutedOperation": true
"unexpectedExternalRequests": 0
```

### Run the authorization and impact PoC

Continue from the same shell:

```bash
mkdir -p "./contracts-v2/contracts/test/audit"
cp "./poc/HCARegistrationSessionSubregistryImpact.poc.t.sol" \
  "./contracts-v2/contracts/test/audit/HCARegistrationSessionSubregistryImpact.t.sol"

cd "./contracts-v2/contracts"
forge test \
  --match-path test/audit/HCARegistrationSessionSubregistryImpact.t.sol \
  -vv
```

Expected decisive output includes:

```
[PASS] test_attackerPrecommitThenSessionAssignsAttackerRegistry()
[PASS] test_managerZeroSubregistryCommitRejectsAttackerRegistryReveal()
```

The deterministic adapters replace network transport only. They do not replace the HCA authorization policy, registrar, ownership transition, subregistry selection, or child-creation logic.
