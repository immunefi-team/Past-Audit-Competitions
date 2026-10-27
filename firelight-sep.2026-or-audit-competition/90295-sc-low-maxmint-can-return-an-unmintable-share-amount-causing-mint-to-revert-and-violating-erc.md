# 90295 sc low maxmint can return an unmintable share amount causing mint to revert and violating erc 4626

**Submitted on Aug 24th 2026 at 09:21:27 UTC by @TrynX for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90295
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

### Summary

`maxMint()` is intended to return the maximum number of shares mintable via `mint()` without exceeding `depositLimit`. However, `maxMint()` converts the full `depositLimit` into shares and then subtracts the existing `totalSupply`, instead of deriving the maximum shares from the remaining asset capacity. Once `totalAssets > totalSupply`, this can cause `maxMint()` to return more shares than the vault can actually accept, causing `mint(maxMint(receiver), receiver)` to revert with `DepositLimitExceeded()`.

### Finding description

`maxMint()` computes:

```solidity
uint256 sharesLimit = convertToShares(depositLimit); // rounds down
return sharesLimit - shares; // shares = totalSupply()
```

`mint()` later converts the returned share amount back into assets using `Math.Rounding.Ceil`, then checks the result against `depositLimit`:

```solidity
if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

**Root cause.** `maxMint()` computes its share limit from the entire `depositLimit`, converted at the vault's current share price, and only then subtracts `totalSupply`. This is not equivalent to computing the share limit directly from the remaining capacity (depositLimit - totalAssets) due to the vault's conversion formula and rounding. This can cause the floored result to exceed the number of shares the vault can actually accept, which the subsequent ceil-rounded `mint()` then exposes as a revert.

### Impact

This violates the EIP-4626 requirement that `maxMint()` return a share amount `mint()` can accept without reverting in the same transaction ([EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)). Impact is limited to a reverted `mint()` call and inaccurate deposit-capacity reporting for integrators.

### Recommendation

Derive `maxMint()` directly from the remaining asset capacity:

```diff
    function maxMint(address receiver) public view override returns (uint256 amount) {
-       uint256 shares = totalSupply();
-       uint256 sharesLimit = convertToShares(depositLimit);
+       uint256 assets = totalAssets();

        if (
            isBlocklisted[receiver] ||
            paused() ||
            _hasActiveIncident() ||
-           shares > sharesLimit
+           assets >= depositLimit
        ) {
            return 0;
        } else {
-           return sharesLimit - shares;
+           return convertToShares(depositLimit - assets);
        }
    }
```

### References

* `FirelightVault::maxMint`
* `FirelightVault::mint`
* [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)

## Link to Proof of Concept

<https://gist.github.com/Tryn0x0/7222434ecef9cfff638c4aa30630db5e>

## Proof of Concept

## Proof of Concept

> The test demonstrates that `maxMint()` can return a share amount that `mint()` subsequently rejects with `DepositLimitExceeded()`.

Add the following function to `audit-comp-firelight/test/deposit_and_withdraw.js`:

> (Also provided in the Gist)

```javascript
describe.only('L-03 PoC — maxMint() can return more shares than the vault can actually accept', function () {
  const DECIMALS = 6

  function randBig(min, max) {
    const range = max - min + 1n
    const r = BigInt(Math.floor(Math.random() * Number(range)))
    return min + r
  }

  it('proves the issue with a deterministic case and validates it across randomized post-yield states', async () => {
    {
      const { token_contract, firelight_vault, limit_updater, users, utils } =
        await loadFixture(
          deployVault.bind(null, {
            decimals: DECIMALS,
            initial_deposit_limit: 10_000_000n
          })
        )

      const depositor = users[0]

      await utils.mintAndApprove(900_000n, depositor)

      await firelight_vault
        .connect(depositor)
        .deposit(900_000n, depositor.address)

      await token_contract.mintTo(
        firelight_vault.target,
        90_000n
      )

      expect(await firelight_vault.totalSupply()).to.equal(900_000n)
      expect(await firelight_vault.totalAssets()).to.equal(990_000n)

      await firelight_vault
        .connect(limit_updater)
        .updateDepositLimit(1_000_000n)

      const totalSupply = await firelight_vault.totalSupply()
      const totalAssets = await firelight_vault.totalAssets()
      const depositLimit = await firelight_vault.depositLimit()
      const remainingCapacity = depositLimit - totalAssets

      expect(remainingCapacity).to.equal(10_000n)

      console.log('\n--- Deterministic reproduction ---')
      console.log(`totalSupply         = ${totalSupply}`)
      console.log(`totalAssets         = ${totalAssets}`)
      console.log(`depositLimit        = ${depositLimit}`)
      console.log(`remainingCapacity   = ${remainingCapacity}`)

      const maxMint =
        await firelight_vault.maxMint(depositor.address)

      const assetsRequired =
        await firelight_vault.previewMint(maxMint)

      console.log('\n--- maxMint() result ---')
      console.log(`maxMint()           = ${maxMint} shares`)
      console.log(`assets required     = ${assetsRequired} assets`)
      console.log(`remaining capacity  = ${remainingCapacity} assets`)
      console.log(
        `overshoot           = ${assetsRequired - remainingCapacity} assets`
      )

      expect(assetsRequired).to.be.gt(remainingCapacity)

      await utils.mintAndApprove(
        assetsRequired,
        depositor
      )

      console.log('\n--- Calling mint(maxMint()) ---')

      await expect(
        firelight_vault
          .connect(depositor)
          .mint(maxMint, depositor.address)
      ).to.be.revertedWithCustomError(
        firelight_vault,
        'DepositLimitExceeded'
      )

      console.log(
        'mint(maxMint())       = reverted with DepositLimitExceeded'
      )
    }

    {
      const N = 60

      const { token_contract, firelight_vault, limit_updater, users, utils } =
        await loadFixture(
          deployVault.bind(null, {
            decimals: DECIMALS,
            initial_deposit_limit: 10_000_000_000n
          })
        )

      const depositor = users[0]

      let tested = 0
      let hits = 0
      let controlTested = 0
      let controlHits = 0
      let maxOvershoot = 0n

      const outerSnapshot =
        await network.provider.send('evm_snapshot')

      for (let i = 0; i < N; i++) {
        const snap =
          await network.provider.send('evm_snapshot')

        const totalSupplyTarget =
          randBig(1_000n, 2_000_000n)

        const isControl = i % 10 === 0

        const donation = isControl
          ? 0n
          : randBig(1n, totalSupplyTarget)

        const depositLimit =
          totalSupplyTarget +
          donation +
          randBig(1n, 1_000_000n)

        await utils.mintAndApprove(
          totalSupplyTarget,
          depositor
        )

        await firelight_vault
          .connect(depositor)
          .deposit(
            totalSupplyTarget,
            depositor.address
          )

        if (donation > 0n) {
          await token_contract.mintTo(
            firelight_vault.target,
            donation
          )
        }

        await firelight_vault
          .connect(limit_updater)
          .updateDepositLimit(depositLimit)

        const maxMint =
          await firelight_vault.maxMint(
            depositor.address
          )

        if (maxMint > 0n) {
          const remainingCapacity =
            depositLimit -
            (await firelight_vault.totalAssets())

          const assetsRequired =
            await firelight_vault.previewMint(maxMint)

          tested++

          if (isControl) {
            controlTested++
          }

          if (assetsRequired > remainingCapacity) {
            hits++

            if (isControl) {
              controlHits++
            }

            const overshoot =
              assetsRequired - remainingCapacity

            if (overshoot > maxOvershoot) {
              maxOvershoot = overshoot
            }

            await utils.mintAndApprove(
              assetsRequired,
              depositor
            )

            await expect(
              firelight_vault
                .connect(depositor)
                .mint(
                  maxMint,
                  depositor.address
                )
            ).to.be.revertedWithCustomError(
              firelight_vault,
              'DepositLimitExceeded'
            )
          }
        }

        await network.provider.send(
          'evm_revert',
          [snap]
        )
      }

      await network.provider.send(
        'evm_revert',
        [outerSnapshot]
      )

      console.log('\n--- Randomized fuzz validation ---')
      console.log(
        `tested states          = ${tested}`
      )
      console.log(
        `mint(maxMint()) reverts = ${hits}/${tested} ` +
        `= ${((100 * hits) / tested).toFixed(2)}%`
      )
      console.log(
        `largest overshoot      = ${maxOvershoot} asset units`
      )
      console.log(
        `1:1 control group      = ${controlHits}/${controlTested} hits`
      )
    }
  })
})
```

And run:

> ```bash
> npx hardhat test deposit_and_withdraw.js
> ```

The output i obtain:

```
  L-03 PoC — maxMint() can return more shares than the vault can actually accept

--- Deterministic reproduction ---
totalSupply         = 900000
totalAssets         = 990000
depositLimit        = 1000000
remainingCapacity   = 10000

--- maxMint() result ---
maxMint()           = 9091 shares
assets required     = 10001 assets
remaining capacity  = 10000 assets
overshoot           = 1 assets

--- Calling mint(maxMint()) ---
mint(maxMint())       = reverted with DepositLimitExceeded

--- Randomized fuzz validation ---
tested states          = 60
mint(maxMint()) reverts = 20/60 = 33.33%
largest overshoot      = 1 asset units
1:1 control group      = 0/6 hits
```

> **Note:** The randomized fuzzing results may vary between runs because the test uses randomly generated states. The deterministic reproduction should produce the same result.
