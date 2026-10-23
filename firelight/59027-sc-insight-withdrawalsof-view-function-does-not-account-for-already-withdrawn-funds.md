# 59027 sc insight withdrawalsof view function does not account for already withdrawn funds

**Submitted on Nov 7th 2025 at 20:33:12 UTC by @a16 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59027
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The external view function withdrawalsOf() is supposed to return the amount an account can withdraw for a given period, but it will still return the same value even after the funds were already withdrawn.

## Vulnerability Details

withdrawalsOf() does not take isWithdrawClaimed into account, meaning that even if funds were already withdrawn, withdrawalsOf() will still return a non zero value for that period and account.

## Impact Details

If withdrawalsOf() is supposed to be a helper function that allows users to know how much they can still claim for each period, this will return the wrong value. If it is supposed to return just historical data, this behavior is correct and there's no bug.

## Proof of Concept

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers') const { deployVault } = require('./setup/fixtures.js') const { expect } = require('chai') const { ethers } = require('hardhat')

function () { const DECIMALS = 6 const INITIAL\_DEPOSIT\_LIMIT = ethers.parseUnits('1000000', DECIMALS) const DEPOSIT\_AMOUNT = ethers.parseUnits('10000', DECIMALS) const WITHDRAW\_AMOUNT = ethers.parseUnits('4000', DECIMALS)

let token\_contract, firelight\_vault, users, utils, config let withdraw\_period

const advancePastClaimable = async (period) => { const currentEnd = await firelight\_vault.currentPeriodEnd() const currentStart = await firelight\_vault.currentPeriodStart() const duration = Number(currentEnd - currentStart)

```
await time.increaseTo(Number(currentEnd))
await time.increase(duration)
await time.increase(duration)
await time.increase(1) 
```

}

before(async () => { ({ token\_contract, firelight\_vault, users, utils, config } = await loadFixture( deployVault.bind(null, { decimals: DECIMALS, initial\_deposit\_limit: INITIAL\_DEPOSIT\_LIMIT }) ))

```
await utils.mintAndApprove(DEPOSIT_AMOUNT, users[0])
await firelight_vault.connect(users[0]).deposit(DEPOSIT_AMOUNT, users[0].address)

const tx = await firelight_vault
  .connect(users[0])
  .withdraw(WITHDRAW_AMOUNT, users[0].address, users[0].address)

const rc = await tx.wait()
withdraw_period = rc.logs[1].args[3]
```

})

it('view shows a non-zero pending amount before claim', async () => { const pending\_before = await firelight\_vault.withdrawalsOf(withdraw\_period, users\[0].address) expect(pending\_before).to.equal(WITHDRAW\_AMOUNT) })

it('after claim, withdrawalsOf still returns non-zero (BUG)', async () => { await advancePastClaimable(withdraw\_period)

```
const pre = await firelight_vault.withdrawalsOf(withdraw_period, users[0].address)
expect(pre).to.equal(WITHDRAW_AMOUNT)

const claim = await firelight_vault.connect(users[0]).claimWithdraw(withdraw_period)
const receipt = await claim.wait()

const ev = receipt.logs.find(l => l.fragment && l.fragment.name === 'CompleteWithdraw')
expect(ev, 'CompleteWithdraw not emitted').to.not.equal(undefined)
expect(ev.args[0]).to.equal(users[0].address)
expect(ev.args[1]).to.equal(WITHDRAW_AMOUNT)
expect(ev.args[2]).to.equal(withdraw_period)

const post = await firelight_vault.withdrawalsOf(withdraw_period, users[0].address)
expect(post).to.equal(WITHDRAW_AMOUNT) // should be 0 if view reflected claim state
```

}) })
