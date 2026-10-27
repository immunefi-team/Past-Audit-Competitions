# 90328 sc low period open totalassetsat p start is overwritten by any vault flow in the boundary second mis sizing the payout cap and matching capacity

**Submitted on Aug 24th 2026 at 13:37:17 UTC by @AgentArena for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90328
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

\| **Other Affected Targets:** | `contracts/core/FirelightVault.sol`, `contracts/core/CoverOrderAllocator.sol`, `contracts/core/lib/Checkpoints.sol` |

### Brief / Intro

`totalAssetsAt(P_start)` is supposed to be the capital that was in the vault when period P opened. It is what `payout` uses as the hard exposure cap and what `CoverOrderAllocator` uses to size sellable cover.

There is no dedicated period-open checkpoint. Writers push `_traceTotalAssets` at `block.timestamp`. `currentPeriod()` already assigns the boundary second `P_start` to period P, so a deposit or redeem in a block with `timestamp == P_start` writes the **post-transaction** total at the exact key that later reads treat as “opening assets.”

The vault’s own test (`test/vault_payout.js`, “caps payout to assets at the capture period start”) encodes the intended rule: capital that arrives after the period opens is not in the cap. That rule fails on that one second, in both directions:

* **Direction A.** Bob deposits 5,000 at exactly `P_start` into a vault that held Alice’s 1,000. The snapshot becomes 6,000. A later payout for period P can take all 6,000, including Bob’s fresh capital.
* **Direction B.** Alice fully redeems at exactly `P_start`. The snapshot becomes 0 and stays 0. A later payout pays **0** (the vault treats that as success). Alice’s 1,000 sits in `withdrawAssets[P+1]`, is not slashed, and is fully claimable after the delay.

On Flare (\~1.8s blocks) the attacker cannot force that timestamp, which is why this is Low rather than Medium. When a block does land on `P_start`, the corruption is permanent for that period.

### Vulnerability Details

Period membership uses integer division. `t == P_start` is period P:

```316:319:contracts/core/firelightvault.sol
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        return periodConfiguration.startingPeriod + _timestampSinceEpoch(timestamp, periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

`currentPeriodStart()` is that same aligned timestamp. Deposit/mint/withdraw/redeem all end in `_logTrace`, which keys the asset trace on `Time.timestamp()`:

```1044:1056:contracts/core/firelightvault.sol
    function _logTrace(...) private {
        uint48 ts = Time.timestamp();
        _traceBalanceOf[owner].push(ts, balance);
        _traceTotalSupply.push(ts, _totalSupply);
        if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
    }
```

`Checkpoints.Trace256.push` overwrites on the same key; if there is no prior `P_start` entry it **inserts** one. Either way, `upperLookupRecent(P_start)` returns the post-flow value:

```207:212:contracts/core/lib/checkpoints.sol
        if (exists && key == lastKey) {
            self._values[len - 1] = value;
        } else {
            self._trace.push(key, uint208(len));
            self._values.push(value);
        }
```

Two consumers treat that slot as an opening snapshot:

```909:911:contracts/core/firelightvault.sol
        uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
```

```710:714:contracts/core/coverorderallocator.sol
        uint256 totalAssetsCanonical = Decimals.convert(
            $.vault.totalAssetsAt($.vault.currentPeriodStart()),
```

A flow one second later does **not** change `totalAssetsAt(P_start)` — that is the control in the PoC. Only the boundary second pollutes the key. After that, every payout and every `commitAllocation` in the period reads the corrupted cap.

Direction B is persistent, not a same-second race with `approveCurrentAssessment`. Alice can redeem at `P_start`, then an incident captured in that period can be assessed and paid later in P or P+1. `paidAmount = min(requested, cap, livePayable)` is 0 because the cap is 0. `payout` returning 0 is an intentional success path, so the incident can close unpaid. The `P+1` withdrawal bucket is still “exposed” in `activePayableAmount`, but nothing is drawn from it when `paidAmount` is 0.

## Impact

Selected in-scope impact: **Contract fails to deliver promised returns, but doesn't lose value** (Low).

What actually happens when a block timestamp equals `P_start`:

* **Direction A (deposit).** The exposure cap and matching capacity include capital that arrived in period P. A subsequent vault-leg payout can drain that capital for a claim whose cap was supposed to be the previous period’s closing assets. The depositor cannot opt out of the timestamp.
* **Direction B (redeem/withdraw).** The cap (and that period’s matching capacity) is written down, possibly to zero, for the rest of the period. Cover is underpaid relative to live collateral. An LP who fully exits in that second avoids the haircut that `payout` is supposed to take from `withdrawAssets[P+1]`.

Cover capacity is sized from the same snapshot, so a `P_start` deposit also oversells cover for the period, and a `P_start` exit undersells it.

## Recommendation

Pick one definition of “opening assets” and use it at both call sites.

Preferred: on the first flow of each period, push `_traceTotalAssets` with the **pre-flow** total at key `P_start` (or `P_start - 1`), then apply the flow and push the post-flow value at `block.timestamp` if it is strictly greater than `P_start`.

Alternative: change both readers to `totalAssetsAt(capturePeriodStart - 1)` and `totalAssetsAt(currentPeriodStart() - 1)` so a write at `P_start` cannot sit in the opening lookup. Both `payout` and `_computeAvailableCapacity` must use the same rule or exposure and capacity will diverge.

Do not treat `checkpointTotalAssets()` at `P_start` as a fix by itself — it has the same keying bug.

## Proof of Concept

Runnable Hardhat test against this repository’s `deployVault` fixture.

```bash
npx hardhat test test/poc/periodStartSnapshotOverwrite.js
```

Observed output:

```
  PoC — period-start checkpoint overwrite corrupts payout cap
  control cap at P_start : 1000
  control paid           : 1000 (opening assets only)
    ✔ control: deposit one second after P_start does NOT inflate the exposure cap
  Direction A cap        : 6000 (should have been 1000)
  Direction A paid       : 6000 (Bob's 5000 was exposed)
    ✔ Direction A: deposit at exactly P_start inflates the cap; payout can take the new capital
  Direction B cap        : 0 (should have been 1000)
  Direction B paid       : 0
  Alice claimed unslashed: 1000
    ✔ Direction B: full redeem at exactly P_start zeros the cap; later payout pays 0 and the exit is unslashed

  3 passing
```

`currentPeriodEnd()` is the next period’s `P_start`. The PoC mines the next transaction with `time.setNextBlockTimestamp(await vault.currentPeriodEnd())` so `block.timestamp == P_start`. The control uses `increaseTo(end + 1)` instead.

`payout` is called with `PAYOUT_ROLE`, the same entry `IncidentManager` uses after an approved assessment. No malicious role holder is required.

### PoC source

Full file: `test/poc/periodStartSnapshotOverwrite.js`

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')

const CONFIG = {
  decimals: 0n,
  initial_deposit_limit: 20_000n,
}

async function deploy() {
  return deployVault(CONFIG)
}

async function nextTxAtPeriodStart(vault) {
  const pStart = await vault.currentPeriodEnd()
  await time.setNextBlockTimestamp(pStart)
  return pStart
}

describe('PoC — period-start checkpoint overwrite corrupts payout cap', function () {
  it('control: deposit one second after P_start does NOT inflate the exposure cap', async () => {
    const { firelight_vault: v, payout_signer, payout_receiver, users, utils } = await loadFixture(deploy)
    const alice = users[0]
    const bob = users[1]

    await utils.mintAndApprove(1000n, alice)
    await v.connect(alice).deposit(1000n, alice.address)

    await time.increaseTo((await v.currentPeriodEnd()) + 1n)
    const pStart = await v.currentPeriodStart()

    await utils.mintAndApprove(5000n, bob)
    await v.connect(bob).deposit(5000n, bob.address)

    expect(await v.totalAssetsAt(pStart)).to.equal(1000n)
    await expect(v.connect(payout_signer).payout(payout_receiver.address, 6000n, pStart))
      .to.emit(v, 'PayoutExecuted')
      .withArgs(payout_receiver.address, 6000n, 1000n, pStart)
  })

  it('Direction A: deposit at exactly P_start inflates the cap; payout can take the new capital', async () => {
    const { token_contract, firelight_vault: v, payout_signer, payout_receiver, users, utils } =
      await loadFixture(deploy)
    const alice = users[0]
    const bob = users[1]

    await utils.mintAndApprove(1000n, alice)
    await v.connect(alice).deposit(1000n, alice.address)

    await utils.mintAndApprove(5000n, bob)
    const pStart = await nextTxAtPeriodStart(v)
    await v.connect(bob).deposit(5000n, bob.address)

    expect(await time.latest()).to.equal(pStart)
    expect(await v.totalAssetsAt(pStart)).to.equal(6000n)
    await expect(v.connect(payout_signer).payout(payout_receiver.address, 6000n, pStart))
      .to.emit(v, 'PayoutExecuted')
      .withArgs(payout_receiver.address, 6000n, 6000n, pStart)
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(6000n)
  })

  it('Direction B: full redeem at exactly P_start zeros the cap; later payout pays 0 and the exit is unslashed', async () => {
    const { token_contract, firelight_vault: v, payout_signer, payout_receiver, users, utils } =
      await loadFixture(deploy)
    const alice = users[0]

    await utils.mintAndApprove(1000n, alice)
    await v.connect(alice).deposit(1000n, alice.address)

    const pStart = await nextTxAtPeriodStart(v)
    await v.connect(alice).redeem(await v.balanceOf(alice.address), alice.address, alice.address)

    expect(await v.totalAssetsAt(pStart)).to.equal(0n)
    await expect(v.connect(payout_signer).payout(payout_receiver.address, 1000n, pStart))
      .to.emit(v, 'PayoutExecuted')
      .withArgs(payout_receiver.address, 1000n, 0n, pStart)

    const claimPeriod = (await v.currentPeriod()) + 1n
    await time.increaseTo((await v.currentPeriodEnd()) + 1n)
    await time.increaseTo((await v.currentPeriodEnd()) + 1n)

    const claimed = await v.connect(alice).claimWithdraw.staticCall(claimPeriod)
    await v.connect(alice).claimWithdraw(claimPeriod)
    expect(claimed).to.equal(1000n)
    expect(await token_contract.balanceOf(alice.address)).to.equal(1000n)
  })
})
```
