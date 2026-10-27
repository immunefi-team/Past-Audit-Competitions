# 88231 sc high rescuing a pending withdrawal permanently poisons that bucket

**Submitted on Aug 12th 2026 at 10:16:26 UTC by @shadowHunter for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88231
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Description

`rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal to a safe address and marks the source as having claimed:

`contracts/core/FirelightVault.sol:845-876`

```solidity
function rescueWithdrawFromBlocklisted(
    address from,
    address to,
    uint256[] calldata periods
)
    external
    onlyRole(RESCUER_ROLE)
    onlyBlocklisted(from)
    notBlocklisted(to)
{
    if (to == address(0)) revert InvalidAddress();

    uint256 len = periods.length;
    if(len == 0 ) revert InvalidArrayLength();

    uint256[] memory rescuedShares = new uint256[](len);
    for (uint256 i = 0; i < len; i++) {
        uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

        if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);   // :864
        if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);     // :865
        if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

        withdrawSharesOf[periods[i]][to] += _withdrawOf;
        withdrawSharesOf[periods[i]][from] = 0;
        isWithdrawClaimed[periods[i]][from] = true;                                          // :870
        ...
```

**There is no restriction that `periods[i]` has already elapsed.** Pending withdrawal buckets are inherently *future* periods — `_requestWithdraw` always assigns `currentPeriod() + 1`:

`contracts/core/FirelightVault.sol:1008-1027`

```solidity
uint256 period = currentPeriod() + 1;                                    // :1008
uint256 sharesWithdraw = _convertToSharesTotals(
    assets, withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor
);
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;                    // :1017  no isWithdrawClaimed check
pendingWithdrawAssets += assets;

if (sender != owner) {
    _spendAllowance(owner, sender, shares);
}

_update(owner, address(0), shares);                                      // :1025  shares burned
```

So the rescue at `:870` sets a terminal "already claimed" marker on a bucket that **has not started yet and is still accepting new requests**. `_requestWithdraw` never consults that flag, so a later request lands in the poisoned bucket and burns the user's shares.

The claim path then rejects it forever:

`contracts/core/FirelightVault.sol:788-811`

```solidity
function claimWithdraw(uint256 period) external whenNotPaused notBlocklisted(_msgSender()) nonReentrant returns (uint256 assets) {
    if (period >= currentPeriod()) revert InvalidPeriod();               // :791

    address sender = _msgSender();
    if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // :794
    ...
```

`isWithdrawClaimed` is written in exactly two places — `claimWithdraw:806` and the rescue at `:870` — and is **never cleared**. There is no setter, no administrative override, and no `delete` anywhere in the contract.

### Why re-rescuing cannot recover it

The obvious remediation is self-blocking. Re-blocklisting the user and calling the rescue again reverts at `:864` **because of the flag the first rescue set**. `rescueSharesFromBlocklisted` (`:818`) is no help either: it moves ERC-20 shares, and the user's shares were already burned at `:1025`.

### Relationship to OpenZeppelin M-02 (this is not that finding)

A published OpenZeppelin audit of an earlier revision reported M-02 "Period Duration Cannot Be Changed", which described the same *symptom* — shares burned in `_requestWithdraw`, `claimWithdraw` then blocked by `isWithdrawClaimed`. That finding is distinct and was fixed:

|                   | OZ M-02                                                                                                                      | This finding                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Who sets the flag | A legitimate `claimWithdraw`                                                                                                 | `rescueWithdrawFromBlocklisted` at `:870`                                                       |
| Root cause        | Period numbering could **regress** after a duration change, letting a user re-request into a period they had already claimed | The rescue writes a terminal flag to a **not-yet-elapsed** bucket that is still open for writes |
| Status            | **Fixed** — `newEpoch >= nextPeriodEnd()` at `:1093` makes period numbering strictly monotonic                               | **Unfixed**                                                                                     |

Decisively, **this scenario requires no period-number movement at all**: the entire setup occurs inside a single period `P`. The M-02 fix is structurally orthogonal to it.

Further, commit `a2def47` ("Fix rescuing withdrawal to a user that already claimed them") added the `isWithdrawClaimed[periods[i]][to]` guard at `:865`. The team reviewed **this exact function for this exact class** and hardened only the `to` direction, leaving the `from` side writing a sticky flag to a future bucket.

***

## Steps to reproduce

No attacker. Every actor performs its documented function.

**1. Alice requests a withdrawal during period `P`.** The request is booked into bucket `P+1`, which has not started.

```solidity
withdraw(4_000e6, alice, alice);      // FirelightVault.sol:755-778 -> _requestWithdraw:994-1028
// withdrawSharesOf[P+1][alice] > 0
```

**2. Alice is added to the blocklist** — for example a screening false positive.

```solidity
addToBlocklist(alice);                // FirelightVault.sol:550
```

**3. `RESCUER_ROLE` rescues the pending bucket to a safe address.** This is the intended use of the function.

```solidity
rescueWithdrawFromBlocklisted(alice, treasury, [P + 1]);
// :868  withdrawSharesOf[P+1][treasury] += shares
// :869  withdrawSharesOf[P+1][alice]     = 0
// :870  isWithdrawClaimed[P+1][alice]    = true      <-- on a period that has not happened
```

**4. The screening hit is cleared and Alice is un-blocklisted — still inside period `P`.**

```solidity
removeFromBlocklist(alice);           // FirelightVault.sol:560
```

**5. Alice withdraws again.** `currentPeriod()` is still `P`, so `_requestWithdraw` assigns bucket `P+1` again. There is no `isWithdrawClaimed` check on this path, so the request is accepted and her shares are burned.

```solidity
withdraw(4_000e6, alice, alice);
// :1017  withdrawSharesOf[P+1][alice] += sharesWithdraw   (accepted)
// :1025  _update(alice, address(0), shares)               (shares irreversibly burned)
```

**6. At period `P+2`, Alice claims — and cannot.**

```solidity
claimWithdraw(P + 1);
// :791  P+1 < P+2, passes
// :794  isWithdrawClaimed[P+1][alice] == true  ->  revert AlreadyClaimedPeriod(P+1)
```

**7. There is no recovery.** Re-blocklisting and re-rescuing reverts at `:864` on the same flag. The flag has no clearing function anywhere in the contract.

***

## Impact

**Permanent freezing of funds, from an ordinary operational sequence.**

* Alice's shares are burned at `:1025` and her entitlement is booked in `withdrawSharesOf[P+1][alice]`, but `claimWithdraw` will reject it forever. She has neither shares nor assets.
* The assets are **not merely unclaimable by Alice — they are unclaimable by anyone**. `withdrawSharesOf` is not transferable, and the corresponding assets remain counted in `pendingWithdrawAssets`. Because `totalAssets() = super.totalAssets() - pendingWithdrawAssets` (`:427-429`), they are permanently excluded from every other depositor's NAV as well. The tokens sit in the vault owned by nobody.
* **Only a proxy upgrade can recover them.** No role — including `DEFAULT_ADMIN_ROLE` — can clear `isWithdrawClaimed`, and pausing/unpausing has no effect on it.
* No malice is required. Blocklist → rescue → un-blocklist is exactly the sequence a compliance false positive produces, and Alice re-withdrawing in the same period is normal user behaviour.
* The outcome is **strictly worse than anything the roles can do deliberately**: a malicious rescuer can seize funds, but funds seized still exist and are held by a known address. Here they become unowned and unrecoverable by anyone.

The same code shape appears in `contracts/legacy/FirelightVaultPredeposits.sol:740-766`. That contract is out of scope and no finding is claimed against it; it is mentioned only so the fix is applied in both places.

***

## Recommendation

The simplest correct fix is to delete the flag write, since the balance zeroing already prevents a double claim:

```solidity
// contracts/core/FirelightVault.sol:868-870
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
// isWithdrawClaimed[periods[i]][from] = true;   <-- remove
```

With `withdrawSharesOf[period][from] == 0`, a subsequent `claimWithdraw` computes `assets == 0` and reverts with `NoWithdrawalAmount` — the correct terminal state, and one that is not sticky across future requests.

If the flag must be retained, restrict rescues to already-elapsed buckets so a future period can never be pre-marked:

```solidity
if (periods[i] >= currentPeriod()) revert InvalidPeriod();
```

Additionally, consider adding an administrative function to clear `isWithdrawClaimed[period][account]` when `withdrawSharesOf[period][account] != 0`, so any account already stranded by a deployed version can be recovered without a proxy upgrade. Apply the same fix to `contracts/legacy/FirelightVaultPredeposits.sol`.

***

## Proof of Concept

## Proof of Concept

### Environment

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

### Create `hardhat.poc.config.js` in the repository root

```js
require('@nomicfoundation/hardhat-ethers')
require('@nomicfoundation/hardhat-chai-matchers')
require('@openzeppelin/hardhat-upgrades')

module.exports = {
  defaultNetwork: 'hardhat',
  networks: { hardhat: { allowUnlimitedContractSize: true } },
  solidity: {
    compilers: [
      { version: '0.8.23', settings: { optimizer: { enabled: true, runs: 10000 }, evmVersion: 'london' } },
      { version: '0.8.28', settings: { optimizer: { enabled: true, runs: 10000 }, evmVersion: 'london', viaIR: true } },
      { version: '0.8.29', settings: { optimizer: { enabled: true, runs: 1000 }, evmVersion: 'london' } },
    ],
    overrides: {
      'contracts/core/CoverOrderAllocator.sol': {
        version: '0.8.28',
        settings: { optimizer: { enabled: true, runs: 100 }, evmVersion: 'london', viaIR: true },
      },
      'contracts/core/FirelightVault.sol': {
        version: '0.8.28',
        settings: { optimizer: { enabled: true, runs: 800 }, evmVersion: 'london', viaIR: true },
      },
    },
  },
  paths: { sources: './contracts', tests: './test', cache: './.audit-cache', artifacts: './.audit-artifacts' },
}
```

### Save the PoC as `test/poc_rescue_bucket_poisoning.js`

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const { deployVault } = require('./setup/fixtures.js')

const DECIMALS = 6
const amt = (n) => ethers.parseUnits(String(n), DECIMALS)

describe('PoC: rescuing a pending withdrawal permanently poisons that bucket', function () {
  this.timeout(120000)

  it('leaves a re-requested withdrawal permanently unclaimable after the user is un-blocklisted', async function () {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: amt(100000) }))
    const { firelight_vault: vault, blocklister, rescuer, users, utils, config } = ctx

    const victim = users[0]
    const safeAddress = users[2]

    await utils.mintAndApprove(amt(20000), victim)

    // ---- 1. Victim requests a withdrawal in period P -> bucket P+1 ----------
    await vault.connect(victim).deposit(amt(10000), victim.address)
    const P = await vault.currentPeriod()
    const bucket = P + 1n

    await vault.connect(victim).withdraw(amt(4000), victim.address, victim.address)
    expect(await vault.withdrawSharesOf(bucket, victim.address)).to.be.greaterThan(0n)

    // The bucket is a FUTURE period: it has not started and is still open for
    // new withdrawal requests.
    expect(bucket).to.be.greaterThan(await vault.currentPeriod())

    // ---- 2. Routine compliance action: blocklist, then rescue the bucket ----
    await vault.connect(blocklister).addToBlocklist(victim.address)
    await vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(victim.address, safeAddress.address, [bucket])

    // The rescue marks a period the victim never claimed in — and never can.
    expect(await vault.isWithdrawClaimed(bucket, victim.address)).to.eq(true)
    expect(await vault.withdrawSharesOf(bucket, victim.address)).to.eq(0n)

    // ---- 3. The blocklisting was a false positive and is cleared ------------
    await vault.connect(blocklister).removeFromBlocklist(victim.address)
    expect(await vault.isBlocklisted(victim.address)).to.eq(false)
    // Still inside the same period P, so a new request lands in the same bucket.
    expect(await vault.currentPeriod()).to.equal(P)

    // ---- 4. Victim withdraws again: shares burn, entitlement is booked ------
    const sharesBefore = await vault.balanceOf(victim.address)
    await vault.connect(victim).withdraw(amt(4000), victim.address, victim.address)

    const sharesAfter = await vault.balanceOf(victim.address)
    expect(sharesAfter).to.be.lessThan(sharesBefore) // shares irreversibly burned

    const stranded = await vault.withdrawSharesOf(bucket, victim.address)
    expect(stranded).to.be.greaterThan(0n)
    // _requestWithdraw never consults isWithdrawClaimed, so the flag survives.
    expect(await vault.isWithdrawClaimed(bucket, victim.address)).to.eq(true)

    const pendingBefore = await vault.pendingWithdrawAssets()
    expect(pendingBefore).to.be.greaterThan(0n)

    // ---- 5. Advance past the bucket so the claim window opens ---------------
    await time.increase(config.period_configuration_duration * 2)
    expect(await vault.currentPeriod()).to.be.greaterThan(bucket)

    // ---- 6. The claim is permanently blocked -------------------------------
    await expect(vault.connect(victim).claimWithdraw(bucket)).to.be.revertedWithCustomError(
      vault,
      'AlreadyClaimedPeriod',
    )

    // ---- 7. No recovery path exists ----------------------------------------
    // Re-rescuing self-blocks on the very flag the first rescue set.
    await vault.connect(blocklister).addToBlocklist(victim.address)
    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safeAddress.address, [bucket]),
    ).to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
    await vault.connect(blocklister).removeFromBlocklist(victim.address)

    // The assets remain counted in pendingWithdrawAssets forever, so they are
    // excluded from totalAssets() for every other depositor as well: the tokens
    // sit in the vault owned by nobody.
    expect(await vault.pendingWithdrawAssets()).to.equal(pendingBefore)
    expect(await vault.withdrawSharesOf(bucket, victim.address)).to.equal(stranded)
    expect(await vault.isWithdrawClaimed(bucket, victim.address)).to.eq(true)
  })
})
```

### Run

```bash
node node_modules/hardhat/internal/cli/bootstrap.js --config hardhat.poc.config.js test test/poc_rescue_bucket_poisoning.js
```

### Output

```
  PoC: rescuing a pending withdrawal permanently poisons that bucket
    √ leaves a re-requested withdrawal permanently unclaimable after the user is un-blocklisted (2287ms)


  1 passing (2s)
```

### What the output proves

The test runs against the real `FirelightVault` behind a transparent proxy, using the repository's own roles. Each assertion pins one link in the chain:

* `bucket > currentPeriod()` — the rescued bucket is confirmed to be a **future** period, which is the precondition the rescue function fails to check.
* After the rescue, `isWithdrawClaimed(bucket, victim) == true` while `withdrawSharesOf(bucket, victim) == 0` — the terminal flag is set on a period the victim never claimed in.
* `currentPeriod() == P` after the un-blocklist — proof that the whole setup happens **inside a single period**, so no period-number movement is involved and the OZ M-02 fix (monotonic period numbering) is irrelevant here.
* `sharesAfter < sharesBefore` — the victim's shares are irreversibly burned by the second withdrawal.
* `withdrawSharesOf(bucket, victim) > 0` **and** `isWithdrawClaimed(bucket, victim) == true` simultaneously — the contradictory state at the heart of the bug: a live entitlement in a bucket already marked claimed, because `_requestWithdraw` at `:1017` never checks the flag.
* `claimWithdraw(bucket)` reverts `AlreadyClaimedPeriod` once the window opens — the funds are unreachable.
* The re-rescue attempt reverts with the **same** `AlreadyClaimedPeriod` error, proving the intended remediation path is self-blocking.
* The closing assertions show `pendingWithdrawAssets` unchanged and the entitlement still booked — so the assets stay excluded from `totalAssets()` for every other depositor too, permanently.
