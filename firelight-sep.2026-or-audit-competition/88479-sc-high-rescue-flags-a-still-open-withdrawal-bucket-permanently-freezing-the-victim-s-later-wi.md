# 88479 sc high rescue flags a still open withdrawal bucket permanently freezing the victim s later withdrawals for that period

**Submitted on Aug 13th 2026 at 11:59:41 UTC by @FaithfulLayney for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88479
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`isWithdrawClaimed[period][account]` is a terminal per-address flag: `claimWithdraw` checks it before computing any entitlement and reverts `AlreadyClaimedPeriod` for that (period, account) forever after. `rescueWithdrawFromBlocklisted` reuses the same flag to mean something different — "this address's credit was moved away" — and accepts any period, including `currentPeriod() + 1`, the bucket `_requestWithdraw` is still writing into. Once the flag is set on a live bucket the address is permanently poisoned for that period: after the compliance hold is lifted, any withdrawal credited to it for that period burns the user's shares, increments `pendingWithdrawAssets`, and can never be claimed. No reset path exists anywhere in the contract's own logic — only a new implementation could add one. In production this strands the caller's entire withdrawal — 5,000 fXRP, 100% of the request, in the PoC — with nobody gaining it.

## Vulnerability Details

A withdrawal bucket `p` is OPEN while `currentPeriod() == p-1` (the bucket index is fixed at `:1008` and the credit written at `:1017`), SEALED while `currentPeriod() == p`, and CLAIMABLE once `currentPeriod() > p` (`:791`). The flag has exactly two writers and they disagree on its meaning: `claimWithdraw` sets it at `:806` to mean "paid", and only ever on a CLAIMABLE bucket because `:791` rejects `period >= currentPeriod()`; `rescueWithdrawFromBlocklisted` sets it at `:870` to mean "moved away", on any bucket, with no bound relating `periods[i]` to `currentPeriod()`:

```solidity
// contracts/core/FirelightVault.sol:860-873
        uint256[] memory rescuedShares = new uint256[](len);
        for (uint256 i = 0; i < len; i++) {
            uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

            if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
            if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
            if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

            withdrawSharesOf[periods[i]][to] += _withdrawOf;
            withdrawSharesOf[periods[i]][from] = 0;
            isWithdrawClaimed[periods[i]][from] = true;

            rescuedShares[i] = _withdrawOf;
        }
```

`currentPeriod() + 1` is not an exotic argument: it is the only bucket a fresh pending withdrawal can be in, and the function's own NatSpec ("Rescues pending withdrawals") points at it. Meanwhile the most ordinary operation in the vault keeps crediting that same bucket for that same address, and burns the shares while doing it:

```solidity
// contracts/core/FirelightVault.sol:1008-1025  (_requestWithdraw, reached from withdraw()/redeem())
        uint256 period = currentPeriod() + 1;
        ...
        withdrawSharesOf[period][receiver] += sharesWithdraw;

        pendingWithdrawAssets += assets;
        ...
        _update(owner, address(0), shares);
```

The sequence, as executed in the PoC: Alice requests a withdrawal into the open bucket `p`; Alice is blocklisted and `RESCUER_ROLE` rescues that pending withdrawal to Carol, setting the flag at `:870`; the compliance hold is lifted inside the same period; Alice makes an ordinary `withdraw()`, which credits `p` again and burns her shares; the bucket closes, Carol claims normally, and Alice's `claimWithdraw(p)` reverts `AlreadyClaimedPeriod(p)` forever.

Every recovery path is closed. Three are closed by assertion in the PoC: `claimWithdraw` reverts at `:794`; re-running the rescue reverts `AlreadyClaimedPeriod` at `:864`, because the flag it set is now its own precondition failure — the rescue cannot undo its own damage; and the ABI contains no non-view function matching `/isWithdrawClaimed|resetWithdraw|clearWithdraw/`. The fourth is closed by construction: `rescueSharesFromBlocklisted` moves ERC20 share balance, and the shares behind this request were already burned at `:1025` — the PoC asserts that burn in step 4 via Alice's share balance rather than calling the function.

The intended protocol constraint that `:870` bypasses is that `isWithdrawClaimed` is terminal by design: `claimWithdraw` may only ever set it on a bucket that can no longer receive credit, because `:791` rejects `period >= currentPeriod()`. The rescue path is bound by no such rule, and neither role can see the damage — the call does not revert, no argument validation rejects the open bucket, and no documented procedure compares `periods[i]` against `currentPeriod()`. This is the exception the scope page's "Privileged actors" rule carves out; that rule reads, in full: *"Any bug report that requires a privileged role holder to act within the permissions explicitly granted to that role is out of scope, unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint."* Neither role needs to misbehave for the report to land on the second half — the constraint is bypassed by the code, not by the operator. And the transaction that actually loses the money is an unprivileged `withdraw()` by the victim herself.

The flag is not needed for safety, which makes the fix cheap: the sibling `rescueSharesFromBlocklisted` (`:818-837`) moves value and sets no terminal flag, so it never poisons the address it touches — a repeat call simply reverts `InsufficientShares` at `:828` once the balance is zero. Zeroing `withdrawSharesOf[periods[i]][from]` at `:869` already makes a second rescue revert `NoWithdrawalAmount` at `:866` — the only protection `:870` adds on the `from` side. Either bound the rescue to buckets that can no longer receive credit (`if (periods[i] > currentPeriod()) revert InvalidPeriod();` at the top of the loop) or drop the `:870` write entirely.

## Impact Details

Permanent freezing of funds: 100% of the affected withdrawal request, with no path inside the contract that can ever release it and no beneficiary. The assets are not stolen — they are removed from the system, excluded from the user's claim and, because `totalAssets() = super.totalAssets() - pendingWithdrawAssets` (`:427-429`), from every LP's share value as well. The per-incident bound is the victim's entire remaining position: one ordinary `withdraw()` at the wrong moment strands whatever it requests. The poison is confined to bucket `p` and to that one address, so it bites only while `p` is still open — the remainder of one period, 604800 seconds in the sponsor's own fixture — and the victim's un-requested shares survive, still withdrawable to any other `receiver` or to herself once the period rolls. What no path inside the contract can recover, at any later time, is every asset credited to that address in that bucket in the meantime.

Measured in the PoC on the real `FirelightVault` proxy using the sponsor's own `deployVault` fixture (fXRP, 6 decimals):

|                                 |                                                                                                    |
| ------------------------------- | -------------------------------------------------------------------------------------------------- |
| Frozen forever                  | 5,000.0 fXRP — 100.00% of the second request                                                       |
| Alice's remaining share balance | 2,000,000,000 — unaffected, but her request is gone                                                |
| `pendingWithdrawAssets`         | 5,000,000,000 — permanently overstated by exactly the frozen amount                                |
| Vault ERC20 balance             | 7,000,000,000                                                                                      |
| `totalAssets()`                 | 2,000,000,000                                                                                      |
| Asserted invariant              | `vaultBalance - pendingWithdrawAssets == totalAssets()` — the stranded tokens back nobody's shares |

No external precondition is involved: no oracle, no price move, no other protocol, no MEV. The in-scope V2 implementation is not deployed yet, so no in-scope contract holds a live position at this moment.

## References

Repository `immunefi-team/audit-comp-firelight` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`:

* [`FirelightVault.sol#L845-L876`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L876) — `rescueWithdrawFromBlocklisted`; the defect is `:870`
* [`FirelightVault.sol#L788-L811`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L811) — `claimWithdraw`; flag check `:794`, the only other writer `:806`
* [`FirelightVault.sol#L994-L1028`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028) — `_requestWithdraw`; `period = currentPeriod() + 1` at `:1008`, share burn at `:1025`
* [`FirelightVault.sol#L818-L837`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L818-L837) — `rescueSharesFromBlocklisted`, the idempotent sibling that sets no flag
* [`FirelightVault.sol#L427-L429`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L427-L429) — `totalAssets()` excludes `pendingWithdrawAssets`

## Proof of Concept

### Step-by-step

Roles (fixture signers): `users[0]` = Alice (an ordinary LP), `users[2]` = `carol` (the safe address the rescue moves value to), `RESCUER_ROLE`, `BLOCKLIST_ROLE`. Bucket `p = currentPeriod() + 1` is the OPEN bucket — the only one a fresh pending withdrawal can land in.

1. **Alice deposits.** The open bucket is `currentPeriod() + 1`.
2. **Alice requests a withdrawal.** `_requestWithdraw` fixes the bucket at `currentPeriod() + 1` (`FirelightVault.sol:1008`), credits `withdrawSharesOf[p][alice]` (`:1017`), increases `pendingWithdrawAssets` (`:1019`) and burns her shares (`:1025`).
3. **Alice is blocklisted**, and `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, carol, [p])` — `p` is where her pending withdrawal is. Carol receives the credit and `isWithdrawClaimed[p][alice] = true` is written at `:870`. The call does not revert and nothing warns that `p` is still accepting credit.
4. **The compliance hold is lifted** — `removeFromBlocklist(alice)`, inside the same period.
5. **Alice makes an ordinary `withdraw()`.** `_requestWithdraw` credits the *same* bucket `p` again, burns her shares again, and increases `pendingWithdrawAssets` again.
6. **The bucket closes.** Carol claims normally. Alice's `claimWithdraw(p)` reverts `AlreadyClaimedPeriod(p)` — permanently, because `:794` tests the flag before the entitlement is even computed at `:796`, and nothing anywhere clears it.
7. **No recovery exists** — two paths closed by assertion, one by construction: re-running the rescue reverts `AlreadyClaimedPeriod` at `:864` (asserted; the flag it set is now its own precondition failure); the ABI has no non-view function matching `/isWithdrawClaimed|resetWithdraw|clearWithdraw/` (asserted); and `rescueSharesFromBlocklisted` cannot help because it moves ERC20 share balance, while the shares behind this request were burned at `:1025` — the test asserts that burn in step 4 via Alice's share balance rather than calling the function.
8. **The assets leave the system**, asserted as `vaultBalance - pendingWithdrawAssets == totalAssets()`: they are neither claimable by Alice nor counted in any LP's share value.

Net result: **5,000 fXRP — 100% of the second request — frozen forever, with nobody gaining it.** No oracle, no price move, no other protocol, no MEV.

### Runnable test

Reproduced from a clean checkout of the in-scope repository at the competition commit with this single file added and nothing else changed. **All execution is local Hardhat; no transaction was sent to any live or public network.**

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62      # branch v1_audit_ready HEAD
npm install                                                # postinstall runs patch-package
mkdir -p test/poc && cp 020-rescue-marks-open-bucket.js test/poc/
npx hardhat test test/poc/020-rescue-marks-open-bucket.js
```

Result: **8 passing**. The test uses the repository's own `test/setup/fixtures.js` and the production `FirelightVault` behind the sponsor's own `upgrades.deployProxy`; the asset is the repository's vendored Flare `FAsset` token (`contracts/fasset/fassetToken/implementation/FAsset.sol`), and the only mock in the fixture is `contracts/test/MockContract.sol`, installed as that token's asset manager so it can mint. The sponsor's suite is unaffected: **448 passing** on its own, **456 passing** with this file added.

`test/poc/020-rescue-marks-open-bucket.js`:

```javascript
/**
 * PoC 020 — rescueWithdrawFromBlocklisted marks isWithdrawClaimed on a STILL-OPEN
 *           withdrawal bucket, permanently freezing every later withdrawal that the
 *           same address is credited with for that period.
 *
 * FirelightVault.sol:845-876  (esp. :870  `isWithdrawClaimed[periods[i]][from] = true`)
 * vs FirelightVault.sol:788-811 (`claimWithdraw` -> :794 `AlreadyClaimedPeriod`)
 * vs FirelightVault.sol:994-1028 (`_requestWithdraw` -> :1008 `period = currentPeriod() + 1`)
 *
 * Root cause: `isWithdrawClaimed[p][a]` has exactly one meaning in `claimWithdraw`
 * ("a has already been paid out for bucket p, never pay again"), but the rescue path
 * reuses it as an "a's balance for bucket p has been moved away" marker. The rescue
 * accepts ANY period index, including `currentPeriod() + 1`, which is the bucket
 * `_requestWithdraw` is still writing into. Once flagged, the bucket is a black hole
 * for that address: shares are burned, `pendingWithdrawAssets` is incremented, and
 * `claimWithdraw` can never be called again for that period.
 *
 * Run:  npx hardhat test test/poc/020-rescue-marks-open-bucket.js
 */
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('PoC 020 — rescue flags a still-open withdrawal bucket (permanent freeze)', function () {
  const DECIMALS = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)
  const DEPOSIT = ethers.parseUnits('10000', DECIMALS)
  const FIRST_REQUEST = ethers.parseUnits('3000', DECIMALS)   // rescued away
  const SECOND_REQUEST = ethers.parseUnits('5000', DECIMALS)  // permanently frozen

  let env

  before(async () => {
    env = await loadFixture(deployVault.bind(null, { decimals: DECIMALS, initial_deposit_limit: DEPOSIT_LIMIT }))
    const { utils, users } = env
    for (const u of users) await utils.mintAndApprove(DEPOSIT, u)
  })

  it('step 0 — Alice deposits; the open bucket is currentPeriod()+1', async () => {
    const { firelight_vault, users } = env
    const alice = users[0]
    await firelight_vault.connect(alice).deposit(DEPOSIT, alice.address)
    expect(await firelight_vault.balanceOf(alice.address)).to.eq(DEPOSIT)

    const p = await firelight_vault.currentPeriod()
    // `_requestWithdraw` (FirelightVault.sol:1008) always credits currentPeriod()+1
    env.openBucket = p + 1n
    // `claimWithdraw` (FirelightVault.sol:791) refuses period >= currentPeriod(),
    // so the bucket only closes when currentPeriod() reaches openBucket+1.
    expect(env.openBucket).to.be.gt(p)
  })

  it('step 1 — Alice requests a withdrawal into the OPEN bucket', async () => {
    const { firelight_vault, users } = env
    const alice = users[0]
    await firelight_vault.connect(alice).withdraw(FIRST_REQUEST, alice.address, alice.address)
    expect(await firelight_vault.withdrawSharesOf(env.openBucket, alice.address)).to.be.gt(0n)
    expect(await firelight_vault.withdrawAssets(env.openBucket)).to.eq(FIRST_REQUEST)
  })

  it('step 2 — Alice is blocklisted and the rescuer moves the pending withdrawal to Carol', async () => {
    const { firelight_vault, blocklister, rescuer, users } = env
    const alice = users[0], carol = users[2]

    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)

    // The rescue is accepted for the OPEN bucket: nothing in
    // rescueWithdrawFromBlocklisted (FirelightVault.sol:845-876) bounds `periods[i]`
    // to a closed period.
    const currentPeriod = await firelight_vault.currentPeriod()
    expect(env.openBucket).to.eq(currentPeriod + 1n) // still the live accrual bucket

    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, carol.address, [env.openBucket])
    ).to.emit(firelight_vault, 'WithdrawRescuedFromBlocklisted')

    expect(await firelight_vault.withdrawSharesOf(env.openBucket, alice.address)).to.eq(0n)
    expect(await firelight_vault.withdrawSharesOf(env.openBucket, carol.address)).to.be.gt(0n)

    // THE DEFECT: the "already paid out" flag is now set on a bucket that is still
    // accepting new deposits of withdrawal shares.
    expect(await firelight_vault.isWithdrawClaimed(env.openBucket, alice.address)).to.eq(true)
  })

  it('step 3 — Alice is un-blocklisted (compliance hold lifted) inside the same period', async () => {
    const { firelight_vault, blocklister, users } = env
    const alice = users[0]
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await firelight_vault.isBlocklisted(alice.address)).to.eq(false)

    // Still the same period => the next request lands in the SAME bucket.
    expect(await firelight_vault.currentPeriod()).to.eq(env.openBucket - 1n)
  })

  it('step 4 — Alice makes an ordinary withdrawal request; it lands in the flagged bucket', async () => {
    const { firelight_vault, users } = env
    const alice = users[0]

    env.pendingBefore = await firelight_vault.pendingWithdrawAssets()

    await expect(firelight_vault.connect(alice).withdraw(SECOND_REQUEST, alice.address, alice.address))
      .to.emit(firelight_vault, 'WithdrawRequest')
      .withArgs(alice.address, alice.address, alice.address, env.openBucket, SECOND_REQUEST, SECOND_REQUEST)

    // Shares are burned and the assets are booked as pending — the user has paid.
    expect(await firelight_vault.withdrawSharesOf(env.openBucket, alice.address)).to.be.gt(0n)
    expect(await firelight_vault.pendingWithdrawAssets()).to.eq(env.pendingBefore + SECOND_REQUEST)
    expect(await firelight_vault.balanceOf(alice.address)).to.eq(DEPOSIT - FIRST_REQUEST - SECOND_REQUEST)
  })

  it('step 5 — bucket closes; Carol claims normally, Alice can NEVER claim', async () => {
    const { firelight_vault, token_contract, config, users } = env
    const alice = users[0], carol = users[2]

    // Advance until the bucket is claimable: currentPeriod() > openBucket.
    await time.increase(config.period_configuration_duration * 2)
    expect(await firelight_vault.currentPeriod()).to.be.gt(env.openBucket)

    // Carol's leg works.
    const carolBefore = await token_contract.balanceOf(carol.address)
    await firelight_vault.connect(carol).claimWithdraw(env.openBucket)
    const carolGot = (await token_contract.balanceOf(carol.address)) - carolBefore
    expect(carolGot).to.be.gt(0n)

    // Alice's own, later, legitimate request is unreachable forever.
    await expect(firelight_vault.connect(alice).claimWithdraw(env.openBucket))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(env.openBucket)

    env.aliceEntitlement = await firelight_vault.withdrawalsOf(env.openBucket, alice.address)
    expect(env.aliceEntitlement).to.be.gt(0n)
  })

  it('step 6 — no recovery path exists in the whole contract', async () => {
    const { firelight_vault, blocklister, rescuer, users } = env
    const alice = users[0], carol = users[2]

    // (a) claimWithdraw: blocked by the flag (proved above).
    // (b) re-running the rescue for Alice requires Alice blocklisted AND the flag clear.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, carol.address, [env.openBucket])
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(env.openBucket)
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)

    // (c) There is no setter for isWithdrawClaimed anywhere.
    expect(firelight_vault.interface.fragments.filter(
      f => f.type === 'function' && /isWithdrawClaimed|resetWithdraw|clearWithdraw/i.test(f.name) && f.stateMutability !== 'view'
    ).length).to.eq(0)

    // (d) rescueSharesFromBlocklisted moves ERC20 shares, not withdrawal credit —
    //     Alice's shares for this amount were already burned at request time.
  })

  it('step 7 — the frozen assets leave the system permanently (quantified)', async () => {
    const { firelight_vault, token_contract, users } = env
    const alice = users[0]

    const vaultBalance = await token_contract.balanceOf(await firelight_vault.getAddress())
    const pending = await firelight_vault.pendingWithdrawAssets()
    const totalAssets = await firelight_vault.totalAssets()

    // `pendingWithdrawAssets` still carries Alice's second request forever:
    // `claimWithdraw` is the only decrementer besides `payout`, and Alice can never call it.
    expect(pending).to.be.gte(env.aliceEntitlement)

    // totalAssets() = balance - pendingWithdrawAssets (FirelightVault.sol:427-429), so the
    // stranded tokens are excluded from LP share value too: not Alice's, not the LPs'.
    expect(vaultBalance - pending).to.eq(totalAssets)

    // eslint-disable-next-line no-console
    console.log(
      `\n      frozen forever: ${ethers.formatUnits(env.aliceEntitlement, DECIMALS)} fXRP` +
      ` (${((Number(env.aliceEntitlement) / Number(SECOND_REQUEST)) * 100).toFixed(2)}% of the second request)` +
      `\n      alice share balance: ${await firelight_vault.balanceOf(alice.address)}` +
      `\n      pendingWithdrawAssets: ${pending}  vaultBalance: ${vaultBalance}  totalAssets: ${totalAssets}\n`
    )
  })
})
```

### Output (one run; the total runtime is machine-dependent, every other value is deterministic)

```
  PoC 020 — rescue flags a still-open withdrawal bucket (permanent freeze)
    ✔ step 0 — Alice deposits; the open bucket is currentPeriod()+1
    ✔ step 1 — Alice requests a withdrawal into the OPEN bucket
    ✔ step 2 — Alice is blocklisted and the rescuer moves the pending withdrawal to Carol
    ✔ step 3 — Alice is un-blocklisted (compliance hold lifted) inside the same period
    ✔ step 4 — Alice makes an ordinary withdrawal request; it lands in the flagged bucket
    ✔ step 5 — bucket closes; Carol claims normally, Alice can NEVER claim
    ✔ step 6 — no recovery path exists in the whole contract

      frozen forever: 5000.0 fXRP (100.00% of the second request)
      alice share balance: 2000000000
      pendingWithdrawAssets: 5000000000  vaultBalance: 7000000000  totalAssets: 2000000000

    ✔ step 7 — the frozen assets leave the system permanently (quantified)


  8 passing (630ms)
```
