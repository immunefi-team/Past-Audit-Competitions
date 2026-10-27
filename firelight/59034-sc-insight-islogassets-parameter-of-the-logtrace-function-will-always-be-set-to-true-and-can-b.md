# 59034 sc insight islogassets parameter of the logtrace function will always be set to true and can be removed&#x20;

**Submitted on Nov 7th 2025 at 22:58:40 UTC by @Tadev for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59034
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The internal `_logTrace` function is called whenever total supply and/or total assets are modified:

* in `_depositFunds` function, called by `deposit` and `mint` functions
* in `redeem` and `withdraw` functions

The `_logTrace` function is defined as follows:

```
    function _logTrace(
        address owner,
        uint256 balance,
        uint256 _totalSupply,
        uint256 _totalAssets,
        // @audit INSIGHT parameter not needed
        bool isLogAssets
    ) private {
        uint48 ts = Time.timestamp();
        _traceBalanceOf[owner].push(ts, balance);
        _traceTotalSupply.push(ts, _totalSupply);

        if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
    }
```

The last parameter, `isLogAssets`, is actually not needed as it will always be set to true during deposits and withdrawals.

## Vulnerability Details

All functions that call `_logTrace` pass `true` for the last parameter:

in `_depositFunds`:

```
        _logTrace(receiver, balanceOf(receiver), _totalSupply, _totalAssets, true);
```

in `redeem`:

```
        _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

in `withdraw`:

```
        _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);

```

Hence, this parameter should be removed and \_logTrace\` could be simplified:

```
    function _logTrace(
        address owner,
        uint256 balance,
        uint256 _totalSupply,
        uint256 _totalAssets,
        // @audit INSIGHT not needed
        bool isLogAssets
    ) private {
        uint48 ts = Time.timestamp();
        _traceBalanceOf[owner].push(ts, balance);
        _traceTotalSupply.push(ts, _totalSupply);
        _traceTotalAssets.push(ts, _totalAssets);
    }
```

## Impact Details

This is an insight highlighting a small logical error with an unneeded parameter.

## Proof of Concept

## Proof of Concept

Please create a *poc.js* file in the *test* folder and copy paste the following code:

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('POC', function() {
  const DECIMALS = 18,
        INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('50000', DECIMALS),
        DEPOSIT_1 = ethers.parseUnits('10000', DECIMALS),
        DEPOSIT_2 = ethers.parseUnits('5000', DECIMALS),
        DEPOSIT_3 = ethers.parseUnits('3000', DECIMALS),
        REDEEM_AMOUNT = ethers.parseUnits('3000', DECIMALS),  // user[1] has 5000, so redeem 3000
        WITHDRAW_AMOUNT = ethers.parseUnits('4000', DECIMALS) // user[0] has 10000, so withdraw 4000

  before(async () => {
    ({ token_contract, firelight_vault, users, utils } = await loadFixture(
      deployVault.bind(null, { decimals: DECIMALS, initial_deposit_limit: INITIAL_DEPOSIT_LIMIT })
    ))

    const FUNDING = ethers.parseUnits('20000', DECIMALS)
    await Promise.all(users.map(account => utils.mintAndApprove(FUNDING, account)))
  })

  it('tracks totalAssetsAt through all possible actions: deposits, redeem, and withdraw', async () => {
    const ts = {}
    
    // Initial state
    ts.start = await time.latest()
    expect(await firelight_vault.totalAssetsAt(ts.start)).to.equal(0)
    
    // Deposit 1
    await firelight_vault.connect(users[0]).deposit(DEPOSIT_1, users[0].address)
    ts.d1 = await time.latest()
    expect(await firelight_vault.totalAssetsAt(ts.d1)).to.equal(DEPOSIT_1)
    
    // Deposit 2
    await firelight_vault.connect(users[1]).deposit(DEPOSIT_2, users[1].address)
    ts.d2 = await time.latest()
    expect(await firelight_vault.totalAssetsAt(ts.d2)).to.equal(DEPOSIT_1 + DEPOSIT_2)
    
    // Deposit 3
    await firelight_vault.connect(users[2]).deposit(DEPOSIT_3, users[2].address)
    ts.d3 = await time.latest()
    expect(await firelight_vault.totalAssetsAt(ts.d3)).to.equal(DEPOSIT_1 + DEPOSIT_2 + DEPOSIT_3)
    
    // Redeem
    await firelight_vault.connect(users[1]).redeem(REDEEM_AMOUNT, users[1].address, users[1].address)
    ts.redeem = await time.latest()
    expect(await firelight_vault.totalAssetsAt(ts.redeem)).to.equal(DEPOSIT_1 + DEPOSIT_2 + DEPOSIT_3 - REDEEM_AMOUNT)
    
    // Withdraw
    await firelight_vault.connect(users[0]).withdraw(WITHDRAW_AMOUNT, users[0].address, users[0].address)
    ts.withdraw = await time.latest()
    expect(await firelight_vault.totalAssetsAt(ts.withdraw)).to.equal(DEPOSIT_1 + DEPOSIT_2 + DEPOSIT_3 - REDEEM_AMOUNT - WITHDRAW_AMOUNT)
  })
})
```

This tests shows a user that deposits a few times, and then redeems and withdraw. Between each step, `totalAssetsAt` is called to see if `_traceTotalAssets` has been updated. No matter what action the user does, `_traceTotalAssets` is updated, because the bool `isLogAssets` is always true.
