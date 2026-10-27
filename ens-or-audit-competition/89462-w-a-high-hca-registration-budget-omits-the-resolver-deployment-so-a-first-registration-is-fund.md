# 89462 w a high hca registration budget omits the resolver deployment so a first registration is funded short and its commitment cannot be recovered

**Submitted on Aug 19th 2026 at 13:28:27 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89462
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Summary

The standalone HCA registration funds itself exactly once. The commit leg carries a single EIP-2612 permit sized by `estimateHcaBudget`, moves that USDC into the HCA, and the reveal leg then pays the registrar and the executor out of that balance. The reveal leg carries no funding pair of its own.

The register leg's share of that budget is priced entirely on one number. `budget.ts:104-118` records the measurement: `/intents/route` prices purely on `destinationGasUnits`, the same request at 450k costs an identical amount whether the batch carries 5 executions or 6, so the quote never sees the calls, only the number. That number is `registerLegGasLimit()`, and it is 450,000.

`buildRevealBatch` prepends `VerifiableFactory.deployProxy(PermissionedResolverImpl, salt, initialize(HCA, ALL_ROLES, []))` whenever the account's PermissionedResolver does not exist yet, which is every account's first registration. `registerLegGasLimit` takes no `resolverDeployed` argument and adds nothing for it, while adding 40,000 for the smaller `setNameWithHCA` write.

Measured on a Sepolia fork against the deployed contracts, that deployment costs 168,851 gas inside the batch. The first-registration batch executes at 455,422 gas, above the 450,000 the permit is sized on, before any ERC-7579 dispatch, IntentExecutor or `HCAOwnerAndSessionValidator` policy work is added on top. The second registration for the same account executes at 286,571.

The reveal then reverts for insufficient USDC. Every reveal failure routes back to `submittingRhinestoneBundle` with the same commitment and the same batch, nothing re-runs the budget or the permit, and the reveal batch has no funding pair to top the account back up, so the retry reproduces the failure on every attempt. The commit is already paid for and the name is never registered.

## Root cause

### 1. The budget is priced on a fixed gas number, not on the batch

`packages/smart-account/src/providers/rhinestone/budget.ts:104-125`

```tsx
/**
 * The `register` leg's gas limit, widened when the batch sets a primary name.
 *
 * This bump is what actually funds the extra call. Verified against the live
 * orchestrator: `/intents/route` prices purely on `destinationGasUnits` [...]
 * So the quote never sees the call; it only ever sees this number. An
 * under-sized limit under-funds the permit and the fill then fails for
 * insufficient USDC.
 */
export function registerLegGasLimit(primaryName?: string): bigint {
  return (
    HCA_LEG_GAS_LIMITS.register +
    (primaryName ? primaryNameGas(primaryName) : 0n)
  )
}
```

`HCA_LEG_GAS_LIMITS.register` is `450_000n`, the same literal as `commit`, and `commit` is the only one with a stated measurement behind it.

### 2. The number ignores the largest conditional call in the batch

`packages/smart-account/src/providers/rhinestone/registration-calls.ts:248-267`

```tsx
if (!params.resolverDeployed) {
  const salt = computeResolverSalt(params.hca)
  calls.push({
    to: c.verifiableFactory,
    value: 0n,
    data: encodeFunctionData({
      abi: verifiableFactoryDeployProxySnippet,
      functionName: 'deployProxy',
      args: [c.permissionedResolverImpl, salt, encodeFunctionData({
        abi: permissionedResolverInitializeAbi,
        functionName: 'initialize',
        args: [params.hca, ROLES_ALL, []],
      })],
    }),
  })
}
```

`registerLegGasLimit` has no parameter that could express this. `HcaBudgetParams` has none either, so neither the quote path nor the fallback model can see it.

### 3. Nothing absorbs the difference

`budget.ts:325-330` removes the buffer the module header still documents as `3% buffer of registerLegCost (gas-spike headroom in the ~60s commit→reveal cooldown)`, on the stated premise that both quotes price the real batches including the conditional resolver `deployProxy`. `estimateHcaBudget` returns `commitCost + registerCost + registrationPrice` exactly, with no headroom over the commit to reveal window.

### 4. Nothing tops the account back up and the retry is a fixed point

`buildRevealBatch` emits no `permit` and no `transferFrom` on any path, so the reveal can only spend the HCA's standing balance. `planHcaIntentFunding`, the helper written to size a permit for a short HCA, has one caller in the repository, `apps/manager/src/features/profile/service/setPrimaryName.ts:191`, and registration never reaches it. Reveal submission failure sets `retryTarget: 'submittingRhinestoneBundle'` at `packages/transaction-manager/src/machines/registration/registration.machine.ts:1209`, the on-chain verification fallback sets the same target at `:1479` and `:1497`, and that state re-submits with `secret: context.commitment!.secret` at `:1190`. `MAX_COMMITMENT_AGE` is read nowhere: `readCommitmentAges` in this same package is the only function that reads it and has no call sites, so the commitment silently ages out of its 86,400 second window while the retry loop runs.

## Impact

Every account's first registration is funded against a gas limit that omits 168,851 gas of work the batch performs. The user signs the funding permit, the commit lands and is paid for, and the reveal then reverts for insufficient USDC. The retry offered by the UI rebuilds the identical batch against the identical short balance, so it cannot succeed, and no path in the flow re-runs `computingHcaBudget` or `signingFundingPermit` once a commitment has landed. The user is left with a burned commitment, an HCA holding a budget that is short, and a name that is never registered.

The second registration for the same account runs at 286,571 gas with 163,429 of headroom under the same limit, which is why the shortfall does not appear once an account already has a resolver.

The primary name option does not cover it. It raises the limit to 490,000, leaving 34,578 over the measured 455,422, while adding `setNameWithHCA` whose cost `budget.ts:73` puts at 34,624 for a name of 31 bytes or fewer.

## Recommendation

Thread the resolver state into the gas limit so the budget tracks the batch, `budget.ts`:

```tsx
/** Measured on Sepolia: VerifiableFactory.deployProxy + PermissionedResolver.initialize. */
export const HCA_RESOLVER_DEPLOY_GAS = 175_000n

export function registerLegGasLimit(params: {
  readonly primaryName?: string
  readonly resolverDeployed: boolean
}): bigint {
  return (
    HCA_LEG_GAS_LIMITS.register +
    (params.resolverDeployed ? 0n : HCA_RESOLVER_DEPLOY_GAS) +
    (params.primaryName ? primaryNameGas(params.primaryName) : 0n)
  )
}
```

Add `resolverDeployed` to `HcaBudgetParams` so the fallback model uses the same value. `estimateHcaBudgetActor` already reads `getCode(resolver)` when it builds the quote batch, so the value is in hand at the call site and only needs passing through.

`HCA_LEG_GAS_LIMITS.register` should also be raised above the measured 455,422 for the batch itself, since the current value is a copy of the commit leg's number and the reveal batch is the larger of the two.

Give the reveal leg a recovery path. `planHcaIntentFunding` already sizes a permit and `transferFrom` pair for a one-off HCA intent and is used by `setPrimaryName`. Routing a reveal retry through it, or re-entering `computingHcaBudget` and `signingFundingPermit` when the reveal fails on balance, converts a permanent failure into one further prompt.

Enforce the reveal window. `readCommitmentAges` already reads `MAX_COMMITMENT_AGE` and has no callers, so a commitment that has aged out is currently retried indefinitely instead of being reported.

## Known issues

R3-01 through R3-08 are the Round 3 transaction-manager defects and none covers this. R3-04 is the commitment retry resuming at submission after commitment generation fails, before any commit has landed and with an undefined value submitted; this occurs after the commit has landed and been paid for, the retry target is correct for that state, and the batch is simply unaffordable. R3-02 and R3-03 are actors that never settle and leave a permanent spinner; here every attempt settles with an on-chain revert. R3-05 through R3-08 are transaction history, archiving and error-record defects.

QA-03 is a display-only price mismatch in the registration and extension modals and states that a mismatch between the displayed total and the amount charged on-chain would be new. This is neither: the quoted budget is what is charged, and the defect is that the gas limit the quote is derived from omits a call in the batch.

WEB-1188 and WEB-1187 concern the checkout flow not warning that sETH is needed for gas on the EOA path. This is USDC funding of the HCA on the Rhinestone path and involves no ETH balance.

WEB-1148 is two users racing the same name, resolved by a page refresh. This is a single user with no race, and a refresh does not help because the shortfall is on-chain in the HCA's balance.

The `packages/smart-account` shared-resolver root grant finding concerns authorization on a resolver that deployed successfully. This concerns the deployment never being funded. R2-03 and R2-06 concern session key scope and revocation; the session validates correctly throughout this failure.

## Proof of Concept

Runs on a local Anvil fork. Nothing is broadcast to a public network.

{% stepper %}
{% step %}

## Structural assertions with no chain access

From `packages/smart-account`:

```bash
pnpm vitest run src/audit-hca-underfunding.test.ts
```

```
 Test Files  1 passed (1)
      Tests  9 passed (9)
```

`packages/smart-account/src/audit-hca-underfunding.test.ts` asserts that `buildRevealBatch({resolverDeployed:false})` returns 5 calls whose first is a `VerifiableFactory` `deployProxy` and whose remaining 4 are byte-identical to the `resolverDeployed:true` batch, that `registerLegGasLimit()` is `450_000n` for both, that the same function adds `40_000n` for a primary name, that `estimateHcaBudget` returns `total === commitCost + registerCost + registrationPrice`, and that no reveal batch on any path contains `permit` `0xd505accf` or `transferFrom` `0x23b872dd`.
{% endstep %}

{% step %}

## Fork Sepolia at a pinned block

```bash
anvil --fork-url <https://ethereum-sepolia-rpc.publicnode.com> --fork-block-number 11522249 --port 8547
```

{% endstep %}

{% step %}

## Create `src/Batcher.sol`

A minimal stand-in for the HCA's batched execution so that inter-call warm and cold access is accounted for in a single transaction:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Batcher {
    struct Call { address to; uint256 value; bytes data; }
    event BatchGas(uint256 gasUsed);

    function run(Call[] calldata cs) external {
        uint256 start = gasleft();
        for (uint256 i; i < cs.length; i++) {
            (bool ok, bytes memory r) = cs[i].to.call{value: cs[i].value}(cs[i].data);
            if (!ok) { assembly { revert(add(r, 32), mload(r)) } }
        }
        emit BatchGas(start - gasleft());
    }
}
```

{% endstep %}

{% step %}

## Run `measure.sh`

Run from the same foundry project:

```bash
#!/usr/bin/env bash
set -e
R=http://127.0.0.1:8547
# anvil's canned accounts carry EIP-7702 delegations on real Sepolia, so use a fresh key
PK=0x70a72ffa9f211cb57cdd956df35ddc74742dd6cfc215c916b8fa58d1e21bbd96
OWNER=0x000000000000000000000000000000000c0ffee1
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
REG=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
VF=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
Z=0x0000000000000000000000000000000000000000
B32=0x0000000000000000000000000000000000000000000000000000000000000000
SECRET=0x7777777777777777777777777777777777777777777777777777777777777777
ALL=0x1111111111111111111111111111111111111111111111111111111111111111
DUR=31536000

cast rpc anvil_setBalance $(cast wallet address $PK) 0xde0b6b3a7640000 --rpc-url $R > /dev/null
B=$(forge create src/Batcher.sol:Batcher --private-key $PK --rpc-url $R --broadcast --json \
    | python3 -c "import sys,json;print(json.load(sys.stdin)['deployedTo'])")
cast send $USDC "mint(address,uint256)" $B 1000000000 --private-key $PK --rpc-url $R --json > /dev/null

ID=$(cast keccak "OwnedResolver")
SALT=$(cast to-dec $(cast keccak $(cast abi-encode "f(bytes32,address,uint256)" $ID $B 0)))
INIT=$(cast calldata "initialize(address,uint256,bytes[])" $B $ALL "[]")
RESOLVER=$(cast call $VF "deployProxy(address,uint256,bytes)(address)" $IMPL $SALT $INIT --from $B --rpc-url $R)
echo "batcher  $B"
echo "resolver $RESOLVER"

reveal () {
  LABEL=$1
  PRICE=$(cast call $REG "getRegisterPrice(string,uint64,address)(uint256,uint256)" $LABEL $DUR $USDC --rpc-url $R | head -1 | awk '{print $1}')
  C=$(cast call $REG "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" $LABEL $OWNER $SECRET $Z $RESOLVER $DUR $B32 --rpc-url $R)
  cast send $REG "commit(bytes32)" $C --private-key $PK --rpc-url $R --json > /dev/null
  cast rpc anvil_increaseTime 120 --rpc-url $R > /dev/null
  cast rpc anvil_mine --rpc-url $R > /dev/null
  NODE=$(cast namehash "$LABEL.eth")
  D1=$(cast calldata "deployProxy(address,uint256,bytes)" $IMPL $SALT $INIT)
  D2=$(cast calldata "approve(address,uint256)" $REG $PRICE)
  D3=$(cast calldata "register(string,address,bytes32,address,address,uint64,address,bytes32)" $LABEL $OWNER $SECRET $Z $RESOLVER $DUR $USDC $B32)
  D4=$(cast calldata "setAddr(bytes32,uint256,bytes)" $NODE 60 $OWNER)
  D6=$(cast calldata "authorizeNameRoles(bytes,uint256,address,bool)" 0x00 $ALL $OWNER true)
  if [ "$2" = "deploy" ]; then
    ARG="[($VF,0,$D1),($USDC,0,$D2),($REG,0,$D3),($RESOLVER,0,$D4),($RESOLVER,0,$D6)]"
  else
    ARG="[($USDC,0,$D2),($REG,0,$D3),($RESOLVER,0,$D4),($RESOLVER,0,$D6)]"
  fi
  cast send $B "run((address,uint256,bytes)[])" "$ARG" --private-key $PK --rpc-url $R --json \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  inner batch gas =', f\"{int(d['logs'][-1]['data'],16):,}\", '  tx gasUsed =', f\"{int(d['gasUsed'],16):,}\")"
}

echo "first registration, deploys the resolver:"; reveal firstreg20260819 deploy
echo "second registration, resolver exists:";     reveal secondreg20260819 skip
```

{% endstep %}

{% step %}

## Review the output

```
batcher  0x78EFe4542670164EB83d3762Eb1a6a8202eDa0f6
resolver 0x8aDd209AB57FdbD8Ac1832Db4304894c9D54FC30
first registration, deploys the resolver:
  inner batch gas = 455,422   tx gasUsed = 467,265
second registration, resolver exists:
  inner batch gas = 286,571   tx gasUsed = 296,982
```

455,422 against the 450,000 that `registerLegGasLimit()` funds. The 168,851 difference between the two runs is the resolver deployment, and `registerLegGasLimit` budgets zero for it. The figure excludes the HCA proxy delegatecall per call, the ERC-7579 execution-mode dispatch, the IntentExecutor, and the `HCAOwnerAndSessionValidator` policy loop that keccak-reconstructs the whole `deployProxy` calldata for `_checkResolverDeployment`, all of which the real fill adds on top of it.

Individual calls measured standalone on the same fork, for reference: `deployProxy` 176,703, `register` 218,615, `setAddr` 64,530, `approve` 46,353, `authorizeNameRoles` 34,857, `commit` 45,433, `StandaloneHCAFactory.deploy` 191,624, `USDC.transferFrom` 57,385. Registration price at that block was 8.000021 USDC for one year, `MIN_COMMITMENT_AGE` 60, `MAX_COMMITMENT_AGE` 86,400.
{% endstep %}
{% endstepper %}
