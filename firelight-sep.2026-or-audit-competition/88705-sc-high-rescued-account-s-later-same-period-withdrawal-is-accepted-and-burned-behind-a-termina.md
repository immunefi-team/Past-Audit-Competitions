# 88705 sc high rescued account s later same period withdrawal is accepted and burned behind a terminal claim flag permanently freezing user funds

**Submitted on Aug 14th 2026 at 16:17:59 UTC by @Jornason for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88705
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` permanently sets `isWithdrawClaimed[period][from] = true` when it moves a blocklisted account's pending withdrawal to a recovery address. If the account is later removed from the blocklist before the period ends, the restored staker can call the public `withdraw` or `redeem` entrypoints for that same period: the vault accepts the request, burns the staker's real shares, and books a new withdrawal liability — but the already-true terminal flag makes the new right impossible to claim, and the same flag also blocks a second rescue. The staker's funds backing the second request are permanently frozen in the vault with no on-chain recovery path.

## Vulnerability Details

### Affected Asset

* Program: Audit Comp | Firelight
* Asset: `contracts/core/FirelightVault.sol` (with state in `contracts/core/FirelightVaultStorage.sol`, both in scope)
* Repo: <https://github.com/immunefi-team/audit-comp-firelight>
* Branch/commit: `v1_audit_ready` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* Affected functions/state: `rescueWithdrawFromBlocklisted` (lines 845-876), `removeFromBlocklist` (560-563), `withdraw` / `redeem` (717-778), `_requestWithdraw` (994-1028), `claimWithdraw` (788-811), `FirelightVaultStorage.isWithdrawClaimed` (line 71)

### Root Cause

The rescue path uses the same boolean as a permanent terminal marker for the source account:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

Removing the account from the blocklist only clears `isBlocklisted`:

```solidity
function removeFromBlocklist(address account) external onlyRole(BLOCKLIST_ROLE) onlyBlocklisted(account) {
    isBlocklisted[account] = false;
    emit RemovedFromBlocklist(account);
}
```

After removal, both public entrypoints reach `_requestWithdraw`, which creates a new right for `currentPeriod() + 1`, books the liability, and burns the owner's shares — without checking or resetting `isWithdrawClaimed[period][receiver]`:

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);   // shares are burned here
```

The newly created right is born behind a terminal guard shared by both the claim path and the rescue retry path:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

### Exploit Flow

{% stepper %}
{% step %}

#### Request an initial withdrawal

A staker deposits 2,000,000 base units and requests a withdrawal of 1,000,000 for period P+1 during period P.
{% endstep %}

{% step %}

#### Rescue the pending withdrawal

The compliance role blocklists the account, and the rescuer role moves the pending P+1 right to a recovery address. Both are documented, intended lifecycle operations. The source account's `isWithdrawClaimed[P + 1]` becomes `true`.
{% endstep %}

{% step %}

#### Restore the account

The compliance role removes the account from the blocklist, still during period P.
{% endstep %}

{% step %}

#### Submit a second withdrawal

The restored staker — an ordinary unprivileged user — calls `withdraw(1_000_000, staker, staker)` (or `redeem`). The call succeeds: it burns the staker's remaining shares, creates 1,000,000 new withdrawal shares for P+1, and increases `pendingWithdrawAssets` by 1,000,000. The claimed flag remains `true`.
{% endstep %}

{% step %}

#### Claim the rescued right

After maturity, the recovery address claims the old rescued right normally.
{% endstep %}

{% step %}

#### Attempt to claim the new right

The restored staker's `claimWithdraw(P + 1)` reverts with `AlreadyClaimedPeriod`.
{% endstep %}

{% step %}

#### Attempt a second rescue

Re-blocklisting the account and retrying `rescueWithdrawFromBlocklisted` also reverts with `AlreadyClaimedPeriod`.

Final state for the new right: staker vault shares `0`; withdrawal shares for P+1 `1,000,000`; vault balance attributable to the right `1,000,000`; claimed flag `true`; claim path unreachable; rescue path unreachable.
{% endstep %}
{% endstepper %}

### Triggerability

The value-destroying step is a public, unprivileged call by an ordinary restored staker through the documented delayed-withdrawal flow. The privileged calls (blocklist, rescue, unblock) are benign compliance lifecycle operations explicitly supported by the contract; no privileged actor must behave maliciously, choose false data, exceed a role, or collude with the staker. The missing protocol constraint — "a right whose terminal flag is already set must not accept new liability" — is enforced nowhere: `_requestWithdraw` accepts and burns behind the flag, and neither the claim path nor the rescue path can ever release the funds. No published rule warns that a restored account must avoid an otherwise accepted request until the period rolls over.

### Why Existing Checks Do Not Prevent It

* The `notBlocklisted` checks on `withdraw`/`redeem` pass because the account was legitimately restored.
* `_requestWithdraw` has no check on `isWithdrawClaimed[period][receiver]` at all, so the new liability is booked and the shares are burned.
* `claimWithdraw` reverts on the already-true flag — it protects against double-claiming but also permanently bricks the new right.
* The rescue retry contains the same `AlreadyClaimedPeriod` guard, so the protocol's own compliance recovery path cannot reach the stranded right either.

## Impact Details

Selected impact:

```
Permanent freezing of funds
```

Impact:

* Affected funds/assets: the vault's staked collateral (FXRP on Flare) belonging to restored stakers.
* Per occurrence: the full amount of the second accepted request, bounded only by the staker's remaining balance. In the executable PoC, 1,000,000 base units are permanently stranded with the user's share balance at zero; both the `withdraw` and `redeem` variants reproduce.
* Cumulative exposure: repeatable for every restored account that re-enters the withdrawal flow in the same period; the vault this code upgrades held approximately 60.16M FXRP of predeposit capital on Flare mainnet at the time of the previous audit (July 2026).
* Attacker cost / preconditions: no attacker is required; the loss arises from the ordinary compliance lifecycle (blocklist → rescue → restore) followed by an ordinary user action.
* Recovery status: none on-chain. Both delivery (`claimWithdraw`) and the protocol's own recovery path (`rescueWithdrawFromBlocklisted`) permanently revert. Waiting additional periods does not change the stored right or the terminal flag. Only a contract upgrade or off-chain reimbursement could recover the funds, and the program states that emergency actions are not an automatic reason to downgrade an otherwise valid report.

Why this is not merely theoretical:

* The executable PoC (attached) demonstrates the full sequence against the in-scope contracts at the audit commit, using the repository's normal upgradeable vault fixture and a standard zero-fee token: no storage injection, malicious token, callback, or unusual configuration.
* A control test proves two ordinary same-period withdrawals aggregate and pay out in full when no rescue poisons the flag, isolating the defect to the rescue terminal-flag interaction.

### Distinction from Known Issues

* The Macro report describes `BLOCKLIST_ROLE` + `RESCUER_ROLE` as a standard compliance rescue and targeted-confiscation capability — that is a description of intended privilege, not this state-machine bug. No Macro finding identifies the persistent `isWithdrawClaimed` flag on the rescued source account, the restore-then-re-request sequence, the accept-and-burn behind a terminal flag, or the simultaneous failure of both claim and re-rescue.
* This is distinct from the withdrawal-rounding family (Macro L-1): the entire second right is lost with exact equal integer amounts and a standard token; no rounding, price, or transfer-fee behavior participates.
* The privileged-actor exclusion does not apply: the roles stay within their documented compliance powers, and the bypass is performed by the public withdrawal surface accepting a liability that the protocol's own terminal flag has already made unreachable.

## Recommendation

At minimum, prevent `_requestWithdraw` from accepting a right that is already terminal for its receiver, before any accounting update or share burn:

```solidity
uint256 period = currentPeriod() + 1;
if (isWithdrawClaimed[period][receiver]) revert AlreadyClaimedPeriod(period);
```

This safely forces the restored account to wait until its next request maps to a different period. Alternatively, redesign rescue state so moving an old right does not permanently pre-consume a future right for the same account and period; if the claimed flag is cleared or separated from rescue state, ensure the moved right cannot be claimed twice.

Add regression tests for both `withdraw` and `redeem` covering: request → blocklist → rescue → unblock → second same-period request; claim and rescue reachability after maturity; and ordinary same-period aggregation without rescue.

## References

* Official scope: <https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/>
* Affected source (audit commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`):
  * `contracts/core/FirelightVault.sol` — `rescueWithdrawFromBlocklisted`, `removeFromBlocklist`, `withdraw`/`redeem`, `_requestWithdraw`, `claimWithdraw`
  * `contracts/core/FirelightVaultStorage.sol` — `isWithdrawClaimed` terminal flag
* PoC package and raw runtime output: attached in the Proof of Concept **field**

## Link to Proof of Concept

<https://gist.github.com/Jornason/84658b4b8a3e66419c837357dc7ee2c8>

## Proof of Concept

### Summary

This is an executable runtime PoC, not static analysis. It runs on the project's own Hardhat setup against the in-scope contracts at the audit commit and demonstrates the full permanent-freeze path: a staker's pending withdrawal is rescued while the account is blocklisted, the account is then restored, and the restored staker's new same-period `withdraw` (or `redeem`) is accepted — shares are burned and the liability is booked — even though the terminal `isWithdrawClaimed` flag makes the new right permanently unclaimable and unrescuable. A control test proves that two ordinary same-period withdrawals pay out in full when no rescue poisons the flag.

### Environment

```
Repository: https://github.com/immunefi-team/audit-comp-firelight
Commit/tag: v1_audit_ready @ 42f9ea5e43d88b35197b901acc2a8c24b314fa62
Framework: Hardhat (the repository's own toolchain; no .env required)
Network: local Hardhat runtime
Mainnet transactions: none
Read-only live checks: not required (no live dependency)
```

The PoC uses only the repository's own test fixture (`test/setup/fixtures.js`) and its standard zero-fee test token. No storage injection, malicious token, callback, public deployment, or unusual configuration is involved. The only external helper is the repo's own `deployVault` fixture.

### Files

```
test/rescue_withdrawal_claim_flag_poisoning.js   (supplied below; drop into the repo's test/ directory)
```

### Reproduction Steps

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install

# Copy the supplied PoC file into the checkout:
cp rescue_withdrawal_claim_flag_poisoning.js test/

npx hardhat test test/rescue_withdrawal_claim_flag_poisoning.js
```

### Expected Output

```
  FirelightVault rescue poisons a later same-period withdrawal
    ✔ burns shares for a new withdrawal that can never be claimed or rescued
    ✔ also accepts an unrecoverable same-period redeem after the account is unblocked
    ✔ pays two same-period withdrawals when rescue does not poison the flag

  3 passing
```

Key invariants (test 1, `withdraw` route):

```
isWithdrawClaimed[P+1][staker]   == true        // set permanently by the rescue
withdraw(P+1) after restore      succeeds       // emits WithdrawRequest, burns all shares
balanceOf(staker)                == 0           // real shares burned
withdrawSharesOf[P+1][staker]    == 1_000_000   // new liability booked
claimWithdraw(P+1) by staker     reverts AlreadyClaimedPeriod
rescueWithdrawFromBlocklisted    reverts AlreadyClaimedPeriod   // recovery path also dead
vault token balance              == 1_000_000   // funds stranded in the vault
```

### What the PoC Proves

{% stepper %}
{% step %}

#### Rescue marks the source account's period terminal

The rescue moves the pending right to the recovery address and permanently sets `isWithdrawClaimed[P+1][staker] = true` (asserted on-chain).
{% endstep %}

{% step %}

#### A restored account can create a stranded withdrawal

After `removeFromBlocklist`, the public `withdraw` accepts a new same-period request: it emits `WithdrawRequest`, burns the staker's remaining 1,000,000 shares, books 1,000,000 new withdrawal shares, and increases `pendingWithdrawAssets` — all behind the already-terminal flag.
{% endstep %}

{% step %}

#### The rescued right is claimable, but the new right is not

After maturity, the rescued right is claimed normally by the recovery address, while the staker's `claimWithdraw` reverts with `AlreadyClaimedPeriod`.
{% endstep %}

{% step %}

#### The stranded withdrawal cannot be rescued

Re-blocklisting and retrying `rescueWithdrawFromBlocklisted` also reverts with `AlreadyClaimedPeriod`: neither the delivery path nor the protocol's own compliance recovery path can ever reach the stranded 1,000,000 units.
{% endstep %}

{% step %}

#### The `redeem` path has the same result

The `redeem` sibling entrypoint produces the same terminal state.
{% endstep %}

{% step %}

#### Ordinary same-period withdrawals work without rescue

The no-rescue control pays two ordinary same-period withdrawals in full (2,000,000), proving the freeze is caused specifically by the rescue terminal-flag interaction.
{% endstep %}
{% endstepper %}

### PoC Source

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

const AMOUNT = 1_000_000n
const TOTAL_DEPOSIT = AMOUNT * 2n

async function depositForTwoWithdrawals(ctx, user) {
  await ctx.utils.mintAndApprove(TOTAL_DEPOSIT, user)
  await ctx.firelight_vault.connect(user).deposit(TOTAL_DEPOSIT, user.address)
}

describe('FirelightVault rescue poisons a later same-period withdrawal', function () {
  it('burns shares for a new withdrawal that can never be claimed or rescued', async () => {
    const ctx = await loadFixture(deployVault)
    const user = ctx.users[0]
    const recovery = ctx.users[1]
    await depositForTwoWithdrawals(ctx, user)

    const withdrawalPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n
    await ctx.firelight_vault.connect(user).withdraw(AMOUNT, user.address, user.address)

    await ctx.firelight_vault.connect(ctx.blocklister).addToBlocklist(user.address)
    await ctx.firelight_vault
      .connect(ctx.rescuer)
      .rescueWithdrawFromBlocklisted(user.address, ctx.rescuer.address, [withdrawalPeriod])

    expect(await ctx.firelight_vault.withdrawSharesOf(withdrawalPeriod, user.address)).to.equal(0)
    expect(await ctx.firelight_vault.withdrawSharesOf(withdrawalPeriod, ctx.rescuer.address)).to.equal(AMOUNT)
    expect(await ctx.firelight_vault.isWithdrawClaimed(withdrawalPeriod, user.address)).to.equal(true)

    await ctx.firelight_vault.connect(ctx.blocklister).removeFromBlocklist(user.address)

    const pendingBeforeSecondRequest = await ctx.firelight_vault.pendingWithdrawAssets()
    const secondRequest = await ctx.firelight_vault
      .connect(user)
      .withdraw(AMOUNT, user.address, user.address)

    await expect(secondRequest)
      .to.emit(ctx.firelight_vault, 'WithdrawRequest')
      .withArgs(user.address, user.address, user.address, withdrawalPeriod, AMOUNT, AMOUNT)
    expect(await ctx.firelight_vault.balanceOf(user.address)).to.equal(0)
    expect(await ctx.firelight_vault.withdrawSharesOf(withdrawalPeriod, user.address)).to.equal(AMOUNT)
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(pendingBeforeSecondRequest + AMOUNT)
    expect(await ctx.firelight_vault.isWithdrawClaimed(withdrawalPeriod, user.address)).to.equal(true)

    await time.increase(ctx.config.period_configuration_duration * 2)

    const rescuerBefore = await ctx.token_contract.balanceOf(ctx.rescuer.address)
    await ctx.firelight_vault.connect(ctx.rescuer).claimWithdraw(withdrawalPeriod)
    expect((await ctx.token_contract.balanceOf(ctx.rescuer.address)) - rescuerBefore).to.equal(AMOUNT)

    await expect(ctx.firelight_vault.connect(user).claimWithdraw(withdrawalPeriod))
      .to.be.revertedWithCustomError(ctx.firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawalPeriod)

    await ctx.firelight_vault.connect(ctx.blocklister).addToBlocklist(user.address)
    await expect(
      ctx.firelight_vault
        .connect(ctx.rescuer)
        .rescueWithdrawFromBlocklisted(user.address, recovery.address, [withdrawalPeriod]),
    )
      .to.be.revertedWithCustomError(ctx.firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawalPeriod)

    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(AMOUNT)
    expect(await ctx.token_contract.balanceOf(ctx.firelight_vault.target)).to.equal(AMOUNT)
  })

  it('also accepts an unrecoverable same-period redeem after the account is unblocked', async () => {
    const ctx = await loadFixture(deployVault)
    const user = ctx.users[0]
    const recovery = ctx.users[1]
    await depositForTwoWithdrawals(ctx, user)

    const withdrawalPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n
    await ctx.firelight_vault.connect(user).withdraw(AMOUNT, user.address, user.address)
    await ctx.firelight_vault.connect(ctx.blocklister).addToBlocklist(user.address)
    await ctx.firelight_vault
      .connect(ctx.rescuer)
      .rescueWithdrawFromBlocklisted(user.address, ctx.rescuer.address, [withdrawalPeriod])
    await ctx.firelight_vault.connect(ctx.blocklister).removeFromBlocklist(user.address)

    await ctx.firelight_vault.connect(user).redeem(AMOUNT, user.address, user.address)
    expect(await ctx.firelight_vault.balanceOf(user.address)).to.equal(0)
    expect(await ctx.firelight_vault.withdrawSharesOf(withdrawalPeriod, user.address)).to.equal(AMOUNT)

    await time.increase(ctx.config.period_configuration_duration * 2)
    await expect(ctx.firelight_vault.connect(user).claimWithdraw(withdrawalPeriod))
      .to.be.revertedWithCustomError(ctx.firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawalPeriod)

    await ctx.firelight_vault.connect(ctx.blocklister).addToBlocklist(user.address)
    await expect(
      ctx.firelight_vault
        .connect(ctx.rescuer)
        .rescueWithdrawFromBlocklisted(user.address, recovery.address, [withdrawalPeriod]),
    )
      .to.be.revertedWithCustomError(ctx.firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawalPeriod)
  })

  it('pays two same-period withdrawals when rescue does not poison the flag', async () => {
    const ctx = await loadFixture(deployVault)
    const user = ctx.users[0]
    await depositForTwoWithdrawals(ctx, user)

    const withdrawalPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n
    await ctx.firelight_vault.connect(user).withdraw(AMOUNT, user.address, user.address)
    await ctx.firelight_vault.connect(user).withdraw(AMOUNT, user.address, user.address)
    await time.increase(ctx.config.period_configuration_duration * 2)

    const userBefore = await ctx.token_contract.balanceOf(user.address)
    await ctx.firelight_vault.connect(user).claimWithdraw(withdrawalPeriod)

    expect((await ctx.token_contract.balanceOf(user.address)) - userBefore).to.equal(TOTAL_DEPOSIT)
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(0)
  })
})
```

### Raw Runtime Evidence

The raw passing output is included with this report's local package (`evidence/find001_full_run.raw.txt`), with `EXIT_CODE=0` and `3 passing`. The command above reproduces this output from a clean checkout.
