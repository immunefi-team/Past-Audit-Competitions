# 93051 w a medium default clean transfer preserves a prior resolver delegate allowing the recipient s ens name to be hijacked

**Submitted on Sep 14th 2026 at 04:12:43 UTC by @andro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93051
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Explorer's name-transfer flow defaults to detaching the resolver and child registry, and tells the sender that the recipient "starts clean." The complete transaction plan never discovers, displays, or revokes third-party roles on the name.

ENSv2 transfers move the current token owner's role bitmap to the recipient but leave every other account's role assignment on the same name resource. A previous `ROLE_SET_RESOLVER` delegate therefore remains authorized after the transfer. Immediately after the intended recipient becomes owner, that delegate can attach an attacker-controlled resolver, hijack the transferred name's resolution, and redirect funds sent through the name.

## Vulnerability Details

The form's only transfer options are:

```typescript
type OptionKey = 'setEthAddress' | 'detachResolver' | 'detachRegistry'
```

The two pointer-detachment options are enabled by default:

```typescript
const [options, setOptions] = useState<Record<OptionKey, boolean>>({
  setEthAddress: true,
  detachResolver: true,
  detachRegistry: true,
})
```

Their descriptions tell the sender that the recipient starts clean:

```typescript
{
  key: 'detachResolver',
  description:
    'Detaches this name’s resolver so it stops resolving to your records entirely. The recipient starts clean and sets up their own.',
},
{
  key: 'detachRegistry',
  description:
    'Detaches this name’s registry so its subnames stop resolving. The recipient starts clean.',
},
```

However, the complete transfer plan contains no role-review or role-revocation operation:

```typescript
export type TransferStepKind =
  | 'set-eth-addr'
  | 'detach-resolver'
  | 'detach-registry'
  | 'transfer-token'

export const buildTransferPlan = (
  options: TransferOptions,
): TransferStepKind[] => {
  const steps: TransferStepKind[] = []

  if (options.setEthAddress && !options.detachResolver) {
    steps.push('set-eth-addr')
  }
  if (options.detachResolver) steps.push('detach-resolver')
  if (options.detachRegistry) steps.push('detach-registry')

  steps.push('transfer-token')
  return steps
}
```

`useTransferName` can execute only those operations:

```typescript
await match(step)
  .with('set-eth-addr', () => setEthAddress({ ...common, recipient, id }))
  .with('detach-resolver', () =>
    detachNameResolver({ ...common, label, registryAddress, id }),
  )
  .with('detach-registry', () =>
    detachNameRegistry({ ...common, label, registryAddress, id }),
  )
  .with('transfer-token', () =>
    transferToken({ ...common, registryAddress, tokenId, recipient, id }),
  )
  .exhaustive()
```

The last step submits a plain ERC-1155 `safeTransferFrom`:

```typescript
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

The ENSv2 `PermissionedRegistry` transfer hook calls `_transferRoles(resource, from, to, false)`. `EnhancedAccessControl._transferRoles` moves only `_roles[resource][from]` to the recipient. It does not enumerate or clear `_roles[resource][delegate]` for any third-party delegate. Transfer also leaves the EAC resource unchanged.

Therefore, detaching the resolver pointer and transferring the token has this result:

```
owner(victim.eth)                         = recipient
resolver(victim.eth)                      = 0x0
roles(resource, previous owner)           = 0
roles(resource, recipient)                = previous owner's bitmap
roles(resource, pre-transfer delegate)    = ROLE_SET_RESOLVER
```

The delegate can then call `setResolver` on the recipient-owned name. Since the role check uses the same unchanged resource, the call succeeds and replaces `0x0` with an attacker-controlled resolver.

The violated security invariant is:

```
default clean transfer completed
    =>
no undisclosed account from the previous ownership period can reconfigure the transferred name
```

The application currently clears configuration pointers but leaves the authorization capable of recreating them.

## Preconditions

1. Alice owns a transferable ENSv2 name.
2. Alice holds the normal resolver-admin and transfer roles assigned to an owner.
3. A service, collaborator, or other account previously received name-level `ROLE_SET_RESOLVER` through the normal Roles interface.
4. Alice transfers the name to Bob using the Explorer and leaves the default resolver-detachment option enabled.

The attacker does not need Alice's or Bob's private key, a privileged ENS project role, control of the indexer, a browser vulnerability, local-network access, or a malicious dependency. The attacker uses the ordinary delegate role that the transfer flow silently carries across the ownership boundary.

## Detailed execution path

1. Alice owns `victim.eth` and its resolver points to the legitimate resolver.
2. Alice grants the attacker `ROLE_SET_RESOLVER` for `victim.eth`, for example, while treating the address as an operational service or collaborator.
3. Alice later opens the Explorer's transfer page and chooses Bob as the recipient.
4. `Detach the resolver` and `Detach the registry` are enabled by default. The form tells Alice that the recipient starts clean.
5. `buildTransferPlan()` returns `detach-resolver`, `detach-registry`, and `transfer-token`. It produces no delegate-revocation step.
6. Alice approves the displayed transactions.
7. The resolver pointer becomes `0x0`, the child-registry pointer is cleared when applicable, and the ERC-1155 name token moves to Bob.
8. Alice's owner roles move to Bob. The attacker's independent `ROLE_SET_RESOLVER` entry remains unchanged.
9. The attacker calls `setResolver(victimId, attackerResolver)`.
10. The call succeeds after Bob owns the name because the attacker remains authorized on the unchanged name resource.
11. The attacker-controlled resolver returns the attacker's address for `victim.eth`.
12. A wallet or user resolves `victim.eth` and sends funds to the attacker.

## Impact Details

The intended recipient receives the correct token, so the completed transaction appears successful. The hidden authority survives until the recipient discovers and revokes it. During that interval, any amount sent through the compromised ENS resolution can be redirected; the maximum loss is the full value users transfer to the resolved attacker address.

The same missing role handoff preserves `ROLE_SET_SUBREGISTRY`, allowing a prior delegate to undo the child-registry detachment. `ROLE_SET_RESOLVER` demonstrates the clearest fund-loss path.

This matches the competition's Critical impact for hijacking an ENS name during a transfer flow. It also creates the separately listed Critical consequence of resolving an ENS name to an attacker-controlled address and can result in direct theft of user funds.

## References

* ENS competition information: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* ENS competition scope, impacts, and known issues: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

## Proof of Concept

## Proof of Concept

At audited revision `63772fd872af472ced58b009499355f3430c2a86`, create:

```typescript
import { readFileSync } from 'node:fs'
import { fileURLToPath } from 'node:url'
import { expect, it } from 'vitest'
import { buildTransferPlan } from '../../../../apps/portal/src/features/transfer/utils/buildTransferPlan'

const OWNER = '0x1111111111111111111111111111111111111111'
const RECIPIENT = '0x2222222222222222222222222222222222222222'
const DELEGATE = '0x3333333333333333333333333333333333333333'
const HONEST_RESOLVER = '0x4444444444444444444444444444444444444444'
const ATTACKER_RESOLVER = '0x5555555555555555555555555555555555555555'
const ZERO_ADDRESS = '0x0000000000000000000000000000000000000000'

// RegistryRolesLib: ROLE_SET_RESOLVER = 1 << 24 and its admin bit is shifted 128.
const ROLE_SET_RESOLVER = 1n << 24n
const ROLE_SET_RESOLVER_ADMIN = ROLE_SET_RESOLVER << 128n
const ROLE_CAN_TRANSFER_ADMIN = (1n << 28n) << 128n

type RegistryState = {
  owner: string
  resolver: string
  roles: Map<string, bigint>
}

const transferRoles = (
  roles: Map<string, bigint>,
  from: string,
  to: string,
) => {
  // Mirrors EnhancedAccessControl._transferRoles(): only the token owner's bitmap is moved. Every other account on the resource is untouched.

  const sourceRoles = roles.get(from) ?? 0n
  if (sourceRoles !== 0n) {
    roles.set(from, 0n)
    roles.set(to, (roles.get(to) ?? 0n) | sourceRoles)
  }
}

const setResolver = (
  state: RegistryState,
  caller: string,
  resolver: string,
) => {
  const callerRoles = state.roles.get(caller) ?? 0n
  if ((callerRoles & ROLE_SET_RESOLVER) !== ROLE_SET_RESOLVER) {
    throw new Error('EACUnauthorizedAccountRoles')
  }
  state.resolver = resolver
}

it('default Explorer transfer leaves an existing resolver delegate active', () => {
  const transferPlan = buildTransferPlan({
    setEthAddress: true,
    detachResolver: true,
    detachRegistry: true,
  })

  // The default Explorer flow advertises a clean handoff, but its complete plan has no role-enumeration or role-revocation step.

  expect(transferPlan).toEqual([
    'detach-resolver',
    'detach-registry',
    'transfer-token',
  ])

  const state: RegistryState = {
    owner: OWNER,
    resolver: HONEST_RESOLVER,
    roles: new Map([
      [
        OWNER,
        ROLE_SET_RESOLVER |
          ROLE_SET_RESOLVER_ADMIN |
          ROLE_CAN_TRANSFER_ADMIN,
      ],
      [DELEGATE, ROLE_SET_RESOLVER],
    ]),
  }

  // Execute the security-relevant effects of the app's default plan.
  state.resolver = ZERO_ADDRESS
  transferRoles(state.roles, OWNER, RECIPIENT)
  state.owner = RECIPIENT

  expect(state.owner).toBe(RECIPIENT)
  expect(state.resolver).toBe(ZERO_ADDRESS)
  expect(state.roles.get(OWNER)).toBe(0n)
  expect(state.roles.get(RECIPIENT)).toBe(
    ROLE_SET_RESOLVER |
      ROLE_SET_RESOLVER_ADMIN |
      ROLE_CAN_TRANSFER_ADMIN,
  )
  expect(state.roles.get(DELEGATE)).toBe(ROLE_SET_RESOLVER)

  // The pre-transfer delegate can immediately undo the advertised clean resolver detachment after ownership has moved.
  setResolver(state, DELEGATE, ATTACKER_RESOLVER)
  expect(state.resolver).toBe(ATTACKER_RESOLVER)

  // Source guards tie the application side of the model to the frozen checkout.
  const transferHook = readFileSync(
    fileURLToPath(
      new URL(
        '../../../../apps/portal/src/features/transfer/hooks/useTransferName.ts',
        import.meta.url,
      ),
    ),
    'utf8',
  )
  const transferForm = readFileSync(
    fileURLToPath(
      new URL(
        '../../../../apps/portal/src/features/transfer/components/SendNameForm.tsx',
        import.meta.url,
      ),
    ),
    'utf8',
  )
  const transferHelper = readFileSync(
    fileURLToPath(
      new URL(
        '../../../../apps/portal/src/features/transfer/helpers/transferToken.ts',
        import.meta.url,
      ),
    ),
    'utf8',
  )

  expect(transferHook).not.toContain('revokeRoles(')
  expect(transferForm).toContain('The recipient starts clean')
  expect(transferForm).not.toContain('delegate')
  expect(transferHelper).toContain("functionName: 'safeTransferFrom'")

  console.log(
    JSON.stringify(
      {
        defaultTransferPlan: transferPlan,
        transferredOwner: state.owner,
        resolverAfterDefaultDetach: ZERO_ADDRESS,
        delegateRoleAfterTransfer: 'ROLE_SET_RESOLVER',
        resolverAfterDelegateCall: state.resolver,
        consequence: 'recipient name resolves through attacker resolver',
      },
      null,
      2,
    ),
  )
})
```

### Run instructions

Install the frozen dependencies from the repository root:

```
corepack pnpm install --frozen-lockfile
```

Run only the PoC:

### Observed output

```
{
  "defaultTransferPlan": [
    "detach-resolver",
    "detach-registry",
    "transfer-token"
  ],
  "transferredOwner": "0x2222222222222222222222222222222222222222",
  "resolverAfterDefaultDetach": "0x0000000000000000000000000000000000000000",
  "delegateRoleAfterTransfer": "ROLE_SET_RESOLVER",
  "resolverAfterDelegateCall": "0x5555555555555555555555555555555555555555",
  "consequence": "recipient name resolves through attacker resolver"
}

Test Files  2 passed (2)
Tests       2 passed (2)
```

The local pnpm project mirror causes this standalone file to be discovered twice; both executions pass the same assertions.

### Expected result

The default clean-transfer flow must revoke every non-recipient role from the previous ownership period, or display each retained account and role and require explicit confirmation. After completion, an undisclosed prior delegate must not be able to attach a new resolver or child registry.

### Actual result

The intended recipient receives the name, but the previous delegate keeps `ROLE_SET_RESOLVER` and can replace the detached resolver with an attacker-controlled resolver.

## Recommendation

1. Add a role-handoff phase before token transfer.
2. Obtain the complete set of active role holders for the name's current EAC resource and read their live role bitmaps immediately before preparing the handoff.
3. Default to revoking every non-owner, non-recipient name-level role, including `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY`.
4. If retaining delegates is supported, show every exact address and role in the final transfer confirmation and require an explicit opt-in.
5. Treat an incomplete role-holder query or a changed resource as a blocking freshness failure.
6. Because revoking roles regenerates the ERC-1155 token ID, fetch the new token ID after the revocations confirm and use that fresh ID for `safeTransferFrom`.
7. Re-read both role state and token ownership immediately before submitting the final transfer.
8. Add an end-to-end regression test that grants a third party `ROLE_SET_RESOLVER`, runs the default transfer, and verifies the third party can no longer call `setResolver` after the recipient owns the token.
