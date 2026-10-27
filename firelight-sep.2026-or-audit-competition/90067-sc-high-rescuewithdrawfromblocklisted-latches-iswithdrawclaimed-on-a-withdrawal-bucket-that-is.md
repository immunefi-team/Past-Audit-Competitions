# 90067 sc high rescuewithdrawfromblocklisted latches iswithdrawclaimed on a withdrawal bucket that is still accepting credits permanently freezing every later withdrawal credited into it

**Submitted on Aug 22nd 2026 at 20:19:30 UTC by @tinnohofficial for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90067
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`isWithdrawClaimed[period][account]` encodes "this account has already been paid out for this period". `claimWithdraw` maintains that meaning safely, because it can only ever run against a bucket that is already closed to further credits. `rescueWithdrawFromBlocklisted` writes the same flag with no such restriction, and the bucket a blocklisted user's pending withdrawal most naturally sits in is `currentPeriod() + 1` — the one still being filled.

Once the flag is latched there, any withdrawal credited to that account in that bucket for the rest of the period has its shares burned on the way in and can never be claimed by anyone. No code path in the contract can clear the flag, reassign the credited shares, or return the assets.

## Vulnerability Details

### Two writers of the same flag, two different invariants

`claimWithdraw` refuses to touch a bucket that has not closed (`FirelightVault.sol:788-811`):

```solidity
if (period >= currentPeriod()) revert InvalidPeriod();

address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
...
isWithdrawClaimed[period][sender] = true;
```

Because `_requestWithdraw` always credits `currentPeriod() + 1` (`FirelightVault.sol:1008`), a bucket stops receiving credits the moment its own period begins, and only becomes claimable the period after that. `claimWithdraw` therefore only ever latches a bucket that is already sealed, and the flag's meaning holds.

`rescueWithdrawFromBlocklisted` writes the same flag with no period constraint at all (`FirelightVault.sol:845-876`):

```solidity
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

`periods` is caller-supplied and unconstrained.

### Exactly one bucket is dangerous, and it is the one a rescue normally targets

Rescuing a bucket that has already closed is harmless: no further credit can ever land there, so the flag is inert. I verified this rather than assuming it — a victim holding unclaimed balances in two closed buckets can have both rescued and still withdraw and claim normally afterwards (PoC case A).

The hazard is confined to `currentPeriod() + 1`. That is also the only bucket a user blocklisted shortly after requesting a withdrawal can have a balance in, so it is the bucket a rescuer will routinely be asked to operate on, not an exotic input. Buckets beyond `currentPeriod() + 1` can never hold a balance, because `_requestWithdraw` has no way to write them, so `_withdrawOf == 0` and the rescue reverts `NoWithdrawalAmount`.

### What breaks after the latch

For the remainder of the current period, any credit into `withdrawSharesOf[currentPeriod()+1][from]` is destroyed on arrival:

* `_requestWithdraw` credits the shares and adds the assets to `pendingWithdrawAssets` (`FirelightVault.sol:1015-1019`),
* it burns the caller's vault shares (`FirelightVault.sol:1025`, `_update(owner, address(0), shares)`),
* and `claimWithdraw` then reverts `AlreadyClaimedPeriod` forever on the check at `FirelightVault.sol:794`.

### No recovery path exists

`isWithdrawClaimed` is written in exactly two places in the entire repository — `FirelightVault.sol:806` and `:870` — and **both write only `true`**. There is no setter, no admin override, and no branch that writes `false`.

* `removeFromBlocklist` returns the account to service but cannot undo the latch; its documented contract is simply to remove the address from the blocklist.
* Re-blocklisting and rescuing again does not help: the rescue path hits its own `isWithdrawClaimed[periods[i]][from]` guard at `FirelightVault.sol:864` and reverts. Verified in the PoC.
* Nothing reassigns `withdrawSharesOf` or mints replacement shares, and the burn in `_requestWithdraw` is already final.

Recovery would require a proxy upgrade carrying a bespoke migration written specifically to rewrite `isWithdrawClaimed` or `withdrawSharesOf`.

### The two reachable sequences

Both require the blocklisting to be reversed inside the same period as the rescue, because `withdraw` and `redeem` carry `notBlocklisted(owner)` and `notBlocklisted(receiver)` modifiers. I confirmed that a `redeem` naming a still-blocklisted receiver reverts `BlocklistedAddress`, so the reversal is a hard prerequisite rather than an incidental step (PoC case D). I state this up front because it is the report's main precondition.

**Sequence 1 — the victim's own funds.** A user is blocklisted shortly after requesting a withdrawal; the rescuer recovers the pending amount from the open bucket; the blocklisting is determined to have been a mistake and is reversed; still inside the same period, the user withdraws the remainder of their position. Those assets are gone.

**Sequence 2 — an unrelated third party's funds.** After the rescue and the reversal, any user calling the ordinary ERC-4626 `redeem(shares, receiver, owner)` with `receiver` set to the rescued account has their own shares burned and the resulting assets stranded. The victim never acts, and the third party was never blocklisted. Redeeming to another address is a first-class feature the vault supports: `WithdrawRequest` indexes `receiver` separately from `owner`, and `_requestWithdraw` credits `withdrawSharesOf[period][receiver]`.

### Distinction from the known OpenZeppelin finding

The OpenZeppelin audit of this vault (21 July 2025) described the same destructive mechanism under **M-02 "Period Duration Cannot Be Changed"**:

> "their shares would be burned during the call to `_requestWithdraw`, but the subsequent call to `claimWithdraw` would revert due to the `isWithdrawClaimed` check."

That report reached the mechanism through period renumbering after a `periodDuration` change, and it was closed as resolved by the period-configuration rework. The underlying hazard — `isWithdrawClaimed` being latchable on a bucket that can still be credited — was never addressed, and the rescue path is a second, still-live route to it that OpenZeppelin did not review: their scope was `Checkpoints.sol`, `FirelightVault.sol` and `FirelightVaultStorage.sol` at a commit whose privileged-role inventory lists `MINTER_ROLE` and `BURNER_ROLE` and **does not include `RESCUER_ROLE` at all**.

The project has also already patched the mirror-image collision on the `to` side (`if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod`, commit `a2def47`, "Fix rescuing withdrawal to a user that already claimed them"), which shows the flag/rescue interaction was recognised as dangerous. The `from` side and the still-open-bucket case were simply not covered.

### On the privileged-role rules

The program excludes reports that need a privileged role to act within its permissions, "unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint". Both limbs apply.

`RESCUER_ROLE`'s documented authority is to **move** a blocklisted account's pending withdrawal — the NatSpec is "Rescues pending withdrawals from a blocklisted address". Nothing grants it the power to permanently destroy assets deposited later by an un-blocklisted account, still less assets belonging to an unrelated third party who was never blocklisted. That is the role reaching past its intended effect.

The protocol constraint `isWithdrawClaimed[p][a] == true` implies `a was paid for period p` is broken: the flag is set with no payment, on a bucket that is still accepting credits.

Note also that the rescuer performs no malicious or incorrect act. Zeroing `withdrawSharesOf[periods[i]][from]` is already sufficient to stop `from` claiming the rescued amount, because `claimWithdraw` computes the payout from `withdrawSharesOf` and reverts `NoWithdrawalAmount` on zero. The `isWithdrawClaimed` write is redundant for its stated purpose and is the sole cause of the freeze.

## Impact Details

**Direct, permanent, unrecoverable loss of principal.** In the demonstrated sequence a victim with a 1,000,000-unit position loses **600,000 units — 60% of their deposit**. The shares were burned by `_requestWithdraw`, the flag cannot be cleared, and the assets sit in the vault owned by nobody.

**Bounds, stated honestly.** The damage is scoped to one bucket. If the victim waits until the next period before withdrawing again, the credit lands in a fresh bucket and claims normally with zero loss (PoC case B). The exposure window is therefore the remainder of the period in which the rescue occurred. Within that window the amount at risk is unbounded: the victim's entire remaining position, plus anything a third party credits to them.

**Secondary effect, also stated honestly.** The stranded amount remains inside `pendingWithdrawAssets` permanently, because that counter is only decremented by a successful `claimWithdraw` or by `payout`. Since `totalAssets() = balanceOf(vault) - pendingWithdrawAssets`, the vault permanently holds real tokens that are excluded from its own accounting and belong to no one, which also understates the collateral basis every capacity and payout calculation reads. I measured this: after the freeze the vault's token balance exceeds `totalAssets() + claimable` by exactly the stranded amount. It does **not** depress the surviving LPs' share price — a control LP's position measured exactly unchanged before and after (PoC case C) — so this is orphaned value rather than a second loss, and I do not claim otherwise.

This matches **Permanent freezing of funds**: user principal that the vault holds, that the accounting acknowledges, and that no actor in the system can ever release.

## Recommended Fix

The minimal and complete fix is to stop writing the flag at all:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
// isWithdrawClaimed[periods[i]][from] = true;   // remove
```

Zeroing `withdrawSharesOf` already prevents `from` from claiming the rescued amount, so the flag write adds nothing and causes the freeze.

If the write is to be kept for auditability, restrict it to buckets that can no longer be credited:

```solidity
if (periods[i] < currentPeriod()) {
    isWithdrawClaimed[periods[i]][from] = true;
}
```

Independently, consider adding a guarded recovery path — a `RESCUER_ROLE` or admin function that clears `isWithdrawClaimed[period][account]` when `withdrawSharesOf[period][account] > 0` and no transfer has occurred — because as the contract stands today a single mistaken rescue is unrecoverable by design.

## References

* `contracts/core/FirelightVault.sol:845-876` — `rescueWithdrawFromBlocklisted`, unconstrained `periods`
* `contracts/core/FirelightVault.sol:864-865` — the `AlreadyClaimedPeriod` guards that also block re-rescue
* `contracts/core/FirelightVault.sol:870` — the latch
* `contracts/core/FirelightVault.sol:806` — the only other write to the flag, also `true`
* `contracts/core/FirelightVault.sol:788-811` — `claimWithdraw`; the `period >= currentPeriod()` guard at `:791` and the revert at `:794`
* `contracts/core/FirelightVault.sol:994-1028` — `_requestWithdraw`; bucket assignment at `:1008`, share burn at `:1025`
* `contracts/core/FirelightVaultStorage.sol:71` — `isWithdrawClaimed` declaration
* Commit `a2def47` — "Fix rescuing withdrawal to a user that already claimed them" (the `to`-side patch)
* OpenZeppelin, *Sentora Firelight Vault Audit*, 21 July 2025, M-02 — same mechanism via a different trigger, closed as resolved

## Proof of Concept

## Proof of Concept

### Step-by-step

**Sequence 1 — victim's own funds.**

1. Victim deposits 1,000,000 and calls `redeem(400_000, …)` during period 0. The request is credited to bucket 1 (`currentPeriod() + 1`).
2. Compliance blocklists the victim.
3. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(victim, rescuer, [1])`. Bucket 1 is the open bucket. `isWithdrawClaimed[1][victim]` becomes `true`.
4. The blocklisting is reversed with `removeFromBlocklist(victim)`.
5. Still in period 0, the victim redeems their remaining 600,000. It is credited to bucket 1 again. `withdrawalsOf(1, victim)` reports 600,000 owed.
6. Two periods later the victim calls `claimWithdraw(1)`. It reverts `AlreadyClaimedPeriod(1)`. Their shares are already burned, so there is nothing left to redeem.
7. Re-blocklisting the victim and rescuing again also reverts on the `isWithdrawClaimed[periods[i]][from]` guard — there is no recovery path.

**Sequence 2 — third party's funds.** After steps 1–4, any other user calls `redeem(500_000, victim, self)`. Their shares are burned, 500,000 is credited to the victim in bucket 1, and nobody can ever claim it.

**Boundary checks** (these establish what the bug is *not*):

* Rescuing already-closed buckets is inert. A victim holding unclaimed balances in buckets 1 and 2 while in period 2 can have both rescued, then withdraw and claim normally from bucket 3 — no future credit can reach a closed bucket.
* Waiting one period before re-withdrawing avoids the loss entirely; the credit lands in the next bucket and claims in full.
* The stranded value is orphaned, not redistributed: a control LP's position is exactly unchanged.
* The third-party sequence cannot run while the victim is still blocklisted; `redeem` reverts `BlocklistedAddress`.

### Run

Save the two files under `### PoC Code` as `test/poc_H01_rescue_latches_claim_flag.js` and `test/verify_h01_bounds.js`, then run:

```bash
npx hardhat test test/poc_H01_rescue_latches_claim_flag.js
npx hardhat test test/verify_h01_bounds.js
```

### Observed Output

Tested against `v1_audit_ready` (`42f9ea5`):

```
  PROBE-H01: rescue latches isWithdrawClaimed on a still-open bucket
period 0: victim requested -> withdrawSharesOf[1] = 400000
rescued -> isWithdrawClaimed[1][victim] = true
victim re-requested -> withdrawSharesOf[1] = 600000, withdrawalsOf = 600000
claimWithdraw REVERTS AlreadyClaimedPeriod -> assets permanently unreachable
re-blocklist + re-rescue also REVERTS -> no recovery path at all
stranded assets = 600000; pendingWithdrawAssets still counts them = 1000000
    ✔ un-blocklisted user's fresh withdrawal is permanently unclaimable
third party credited victim: withdrawalsOf = 500000
OTHER's shares are burned, the assets are credited to the victim, and nobody can ever claim them
    ✔ same latch strands a third-party credit after the blocklist reversal

  2 passing
```

```
  VERIFY-H01: boundaries of the isWithdrawClaimed latch
currentPeriod=2; unclaimed balances in bucket 1=300000, bucket 2=300000
rescued CLOSED buckets 1,2 -> latch set but no future credit can ever land there
victim claims fresh bucket 3 normally -> 400000 (latch on closed buckets is harmless)
    ✔ A: closed buckets are harmless; only the open bucket admits future credits
waited one period before re-withdrawing -> claim succeeds for 600000; zero loss
    ✔ B: damage is scoped to ONE bucket - victim withdraws normally in the very next period
vault token balance = 5600000
pendingWithdrawAssets = 600000  (600000 of this is unclaimable forever)
totalAssets() = 5000000   balance - pending = 5000000
surviving LP deposited 5000000, position now worth 5000000
    ✔ C: stranded value remains pending but does not reduce another LP position
redeem(receiver=blocklisted victim) reverts -> un-blocklisting is a hard prerequisite
    ✔ D: third-party case requires the victim to be un-blocklisted first

  4 passing
```

### PoC Code

`test/poc_H01_rescue_latches_claim_flag.js`:

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

// rescueWithdrawFromBlocklisted writes isWithdrawClaimed[period][from] = true for periods it
// rescues. Nothing restricts `periods` to already-closed buckets, and the natural bucket to
// rescue is the one currently being filled (currentPeriod()+1). If the blocklisting is later
// reversed inside the same period, the un-blocklisted user can still create a NEW withdrawal
// request into that same bucket -- but the claimed flag is already latched, so claimWithdraw
// reverts forever and the assets are unreachable.
describe("PROBE-H01: rescue latches isWithdrawClaimed on a still-open bucket", function () {
  it("un-blocklisted user's fresh withdrawal is permanently unclaimable", async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: 6, initial_deposit_limit: 10n ** 18n }),
    );
    const { firelight_vault: v, token_contract, blocklister, rescuer, users, config } = ctx;
    for (const u of users) await ctx.utils.mintAndApprove(10n ** 14n, u);

    const VICTIM = users[0];
    await v.connect(VICTIM).deposit(1_000_000n, VICTIM.address);

    const P = await v.currentPeriod();
    const bucket = P + 1n;

    // 1. victim requests a partial withdrawal (normal user action)
    await v.connect(VICTIM).redeem(400_000n, VICTIM.address, VICTIM.address);
    console.log(`period ${P}: victim requested -> withdrawSharesOf[${bucket}] = ${await v.withdrawSharesOf(bucket, VICTIM.address)}`);

    // 2. compliance blocklists the victim (later determined to be a mistake)
    await v.connect(blocklister).addToBlocklist(VICTIM.address);

    // 3. rescuer recovers the victim's pending withdrawal -- the only bucket with a balance is
    //    the one still being filled, so `periods` must contain it.
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(VICTIM.address, rescuer.address, [bucket]);
    console.log(`rescued -> isWithdrawClaimed[${bucket}][victim] = ${await v.isWithdrawClaimed(bucket, VICTIM.address)}`);

    // 4. the blocklisting is reversed in the same period
    await v.connect(blocklister).removeFromBlocklist(VICTIM.address);
    expect(await v.isBlocklisted(VICTIM.address)).to.equal(false);

    // 5. victim withdraws the rest of their position -- still period P, so the SAME bucket
    expect(await v.currentPeriod()).to.equal(P);
    await v.connect(VICTIM).redeem(600_000n, VICTIM.address, VICTIM.address);
    const credited = await v.withdrawSharesOf(bucket, VICTIM.address);
    const owed = await v.withdrawalsOf(bucket, VICTIM.address);
    console.log(`victim re-requested -> withdrawSharesOf[${bucket}] = ${credited}, withdrawalsOf = ${owed}`);
    expect(credited).to.be.gt(0n);
    expect(owed).to.be.gt(0n);

    // 6. wait out the delay and try to claim
    await time.increase(config.period_configuration_duration);
    await time.increase(config.period_configuration_duration);
    expect(await v.currentPeriod()).to.be.gt(bucket);

    const before = await token_contract.balanceOf(VICTIM.address);
    await expect(v.connect(VICTIM).claimWithdraw(bucket))
      .to.be.revertedWithCustomError(v, "AlreadyClaimedPeriod")
      .withArgs(bucket);
    console.log("claimWithdraw REVERTS AlreadyClaimedPeriod -> assets permanently unreachable");
    expect(await token_contract.balanceOf(VICTIM.address)).to.equal(before);

    // there is no recovery path: shares are already burned, the flag cannot be reset,
    // and a second rescue requires re-blocklisting (which then also reverts).
    await v.connect(blocklister).addToBlocklist(VICTIM.address);
    await expect(
      v.connect(rescuer).rescueWithdrawFromBlocklisted(VICTIM.address, rescuer.address, [bucket]),
    ).to.be.revertedWithCustomError(v, "AlreadyClaimedPeriod");
    console.log("re-blocklist + re-rescue also REVERTS -> no recovery path at all");

    console.log(`stranded assets = ${owed}; pendingWithdrawAssets still counts them = ${await v.pendingWithdrawAssets()}`);
  });

  it("same latch strands a third-party credit after the blocklist reversal", async () => {
    // Variant: once the victim has been un-blocklisted, an unrelated owner can burn
    // their own shares while naming the poisoned account as the withdrawal receiver.
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: 6, initial_deposit_limit: 10n ** 18n }),
    );
    const { firelight_vault: v, blocklister, rescuer, users, config } = ctx;
    for (const u of users) await ctx.utils.mintAndApprove(10n ** 14n, u);
    const VICTIM = users[0];
    const OTHER = users[1];

    await v.connect(VICTIM).deposit(1_000_000n, VICTIM.address);
    await v.connect(OTHER).deposit(1_000_000n, OTHER.address);
    const P = await v.currentPeriod();
    const bucket = P + 1n;

    await v.connect(VICTIM).redeem(400_000n, VICTIM.address, VICTIM.address);
    await v.connect(blocklister).addToBlocklist(VICTIM.address);
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(VICTIM.address, rescuer.address, [bucket]);
    await v.connect(blocklister).removeFromBlocklist(VICTIM.address);

    // a third party redeems with receiver = victim (a perfectly ordinary ERC4626 call)
    await v.connect(OTHER).redeem(500_000n, VICTIM.address, OTHER.address);
    console.log(`third party credited victim: withdrawalsOf = ${await v.withdrawalsOf(bucket, VICTIM.address)}`);

    await time.increase(config.period_configuration_duration);
    await time.increase(config.period_configuration_duration);
    await expect(v.connect(VICTIM).claimWithdraw(bucket)).to.be.revertedWithCustomError(v, "AlreadyClaimedPeriod");
    console.log("OTHER's shares are burned, the assets are credited to the victim, and nobody can ever claim them");
  });
});
```

`test/verify_h01_bounds.js`:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

// Independent probe of H-01's boundaries, not its core claim.
// The core latch is already proven by poc_H01. These checks bound its trigger window,
// affected bucket, accounting effect, and blocklist precondition.
describe('VERIFY-H01: boundaries of the isWithdrawClaimed latch', function () {
  const setup = async () => {
    const ctx = await loadFixture(deployVault.bind(null, { decimals: 6, initial_deposit_limit: 10n ** 18n }))
    for (const u of ctx.users) await ctx.utils.mintAndApprove(10n ** 14n, u)
    return ctx
  }

  it('A: closed buckets are harmless; only the open bucket admits future credits', async () => {
    const { firelight_vault: v, blocklister, rescuer, users, config } = await setup()
    const victim = users[0]
    await v.connect(victim).deposit(1_000_000n, victim.address)

    // request in period 0 -> bucket 1
    const p0 = await v.currentPeriod()
    await v.connect(victim).redeem(300_000n, victim.address, victim.address)
    // roll to period 1, request again -> bucket 2
    await time.increase(config.period_configuration_duration)
    const p1 = await v.currentPeriod()
    await v.connect(victim).redeem(300_000n, victim.address, victim.address)
    // roll to period 2 without ever claiming
    await time.increase(config.period_configuration_duration)
    const p2 = await v.currentPeriod()

    const b1 = p0 + 1n, b2 = p1 + 1n
    console.log(`currentPeriod=${p2}; unclaimed balances in bucket ${b1}=${await v.withdrawSharesOf(b1, victim.address)}, bucket ${b2}=${await v.withdrawSharesOf(b2, victim.address)}`)
    expect(await v.withdrawSharesOf(b1, victim.address)).to.be.gt(0n)
    expect(await v.withdrawSharesOf(b2, victim.address)).to.be.gt(0n)

    // These candidate buckets are closed; the only dangerous bucket is currentPeriod()+1.
    await v.connect(blocklister).addToBlocklist(victim.address)
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, rescuer.address, [b1, b2])
    console.log(`rescued CLOSED buckets ${b1},${b2} -> latch set but no future credit can ever land there`)

    // Prove the latch on a closed bucket is inert: unblocklist, withdraw again, land in a fresh bucket.
    await v.connect(blocklister).removeFromBlocklist(victim.address)
    await v.connect(victim).redeem(400_000n, victim.address, victim.address)
    const fresh = p2 + 1n
    expect(await v.withdrawSharesOf(fresh, victim.address)).to.be.gt(0n)
    await time.increase(config.period_configuration_duration * 2)
    const got = await v.connect(victim).claimWithdraw.staticCall(fresh)
    console.log(`victim claims fresh bucket ${fresh} normally -> ${got} (latch on closed buckets is harmless)`)
    expect(got).to.be.gt(0n)
  })

  it('B: damage is scoped to ONE bucket - victim withdraws normally in the very next period', async () => {
    const { firelight_vault: v, blocklister, rescuer, users, config } = await setup()
    const victim = users[0]
    await v.connect(victim).deposit(1_000_000n, victim.address)
    const P = await v.currentPeriod()
    const poisoned = P + 1n

    await v.connect(victim).redeem(400_000n, victim.address, victim.address)
    await v.connect(blocklister).addToBlocklist(victim.address)
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, rescuer.address, [poisoned])
    await v.connect(blocklister).removeFromBlocklist(victim.address)

    // roll ONE period: new requests now land in bucket P+2, untouched
    await time.increase(config.period_configuration_duration)
    await v.connect(victim).redeem(600_000n, victim.address, victim.address)
    const nextBucket = (await v.currentPeriod()) + 1n
    expect(nextBucket).to.equal(poisoned + 1n)
    await time.increase(config.period_configuration_duration * 2)
    const got = await v.connect(victim).claimWithdraw.staticCall(nextBucket)
    console.log(`waited one period before re-withdrawing -> claim succeeds for ${got}; zero loss`)
    expect(got).to.equal(600_000n)
  })

  it('C: stranded value remains pending but does not reduce another LP position', async () => {
    const { firelight_vault: v, token_contract, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], lp = users[1]
    await v.connect(lp).deposit(5_000_000n, lp.address)
    await v.connect(victim).deposit(1_000_000n, victim.address)
    const P = await v.currentPeriod()
    const poisoned = P + 1n

    await v.connect(victim).redeem(400_000n, victim.address, victim.address)
    await v.connect(blocklister).addToBlocklist(victim.address)
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, rescuer.address, [poisoned])
    await v.connect(blocklister).removeFromBlocklist(victim.address)
    await v.connect(victim).redeem(600_000n, victim.address, victim.address)

    await time.increase(config.period_configuration_duration * 2)
    await v.connect(rescuer).claimWithdraw(poisoned) // rescuer takes the 400k it legitimately holds

    const bal = await token_contract.balanceOf(v.target)
    const pending = await v.pendingWithdrawAssets()
    const ta = await v.totalAssets()
    console.log(`vault token balance = ${bal}`)
    console.log(`pendingWithdrawAssets = ${pending}  (600000 of this is unclaimable forever)`)
    console.log(`totalAssets() = ${ta}   balance - pending = ${bal - pending}`)
    expect(ta).to.equal(bal - pending)

    // The stranded pending amount is orphaned, but does not depress this LP's share price.
    const lpWorth = await v.previewRedeem(await v.balanceOf(lp.address))
    console.log(`surviving LP deposited 5000000, position now worth ${lpWorth}`)
    expect(lpWorth).to.equal(5_000_000n)
  })

  it('D: third-party case requires the victim to be un-blocklisted first', async () => {
    const { firelight_vault: v, blocklister, rescuer, users } = await setup()
    const victim = users[0], other = users[1]
    await v.connect(victim).deposit(1_000_000n, victim.address)
    await v.connect(other).deposit(1_000_000n, other.address)
    const poisoned = (await v.currentPeriod()) + 1n

    await v.connect(victim).redeem(400_000n, victim.address, victim.address)
    await v.connect(blocklister).addToBlocklist(victim.address)
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, rescuer.address, [poisoned])

    await expect(v.connect(other).redeem(500_000n, victim.address, other.address))
      .to.be.revertedWithCustomError(v, 'BlocklistedAddress')
    console.log('redeem(receiver=blocklisted victim) reverts -> un-blocklisting is a hard prerequisite')
  })
})
```
