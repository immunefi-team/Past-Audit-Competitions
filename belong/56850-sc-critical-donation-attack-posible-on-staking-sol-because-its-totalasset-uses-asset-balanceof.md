# 56850 sc critical donation attack posible on staking sol because its totalasset uses asset balanceof&#x20;

**Submitted on Oct 21st 2025 at 07:41:15 UTC by @kaysoft for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56850
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The `Staking.sol` inherits ERC4626 from solady. This ERC4626 contract uses the ERC20 `balanceOf(address(this))` function to keep record of the total assets in the contract. The `totalAssets()` function of `Staking.sol` can be increased by anyone by direct donation.

This `totalAssets()` function is used in shares minting during `deposit` and the attacker's aim will be to cause the victim to mint zero shares losing their deposits.

### Vulnerability Details

The main attack is to cause victims to mint zero shares during deposits. Zero shares means the victim cannot withdraw their deposit.

The `totalAssets()` is the denominator in the `convertToShares` function of ERC4626. An attacker can frontrun a victim's deposit with a direct transfer of Long token to the `Staking.sol` contract increasing `totalAssets()` without changing `totalSupply()`.

Due to solidity rounding, a victim deposit can round down to zero if the `totalAssets()` is made large enough with the donation. shares = userAsset × totalSupply / totalAssets.

This issue compounds if the next depositor deposits asset amount less than the current total amount. The Staking Contract takes in the asset and mints zero shares again. Now `totalAssets` has increased again while `totalSupply` remains the same; subsequent users who deposit less than the new `totalAssets` also mint zero shares.

Example snippet from solady ERC4626:

```solidity
function totalAssets() public view virtual returns (uint256 assets) {
    assets = SafeTransferLib.balanceOf(asset(), address(this));//@audit donation attack
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

## Issue Scenario

{% stepper %}
{% step %}

### Step

Staking.sol is deployed
{% endstep %}

{% step %}

### Step

Bob (victim) tries to deposit 100 Long tokens
{% endstep %}

{% step %}

### Step

Alice (attacker) frontruns Bob to directly transfer 101 Long tokens to Staking.sol
{% endstep %}

{% step %}

### Step

Bob's deposit is executed with `100 × 1 / 101 = 0`
{% endstep %}

{% step %}

### Step

Zero shares are minted to Bob making Bob lose his 100 Long tokens
{% endstep %}

{% step %}

### Step

John (another victim) deposits 150 Long tokens and gets `150 × 1 / 201 = 0`
{% endstep %}
{% endstepper %}

As total assets keep growing while totalSupply remains the same, repeated small donations or repeated deposits smaller than the current `totalAssets` can cause many users to mint zero shares and lose funds.

## Impact Details

* Causing loss of funds to the first depositors by making them mint zero shares.

{% hint style="info" %}
Recommendation: Consider overriding the `totalAssets(...)` function of the ERC4626 then keep record of the total assets with a storage variable which will be updated during `deposit()`, `withdraw()` and `distributeRewards()`.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Test setup

1. Copy and paste the test below into the `staking.test.sol` file in the `'Staking features'` test suite
2. Run `yarn test`
   {% endstep %}

{% step %}

### Observed effect

This demonstrates a donation attack that causes the first depositor to lose their Long token deposit amount.
{% endstep %}
{% endstepper %}

```solidity
// File: staking.test.ts
it('Donation to cause Zero shares minting', async () => {
      const { staking, long, admin, minter, user1, user2 } = await loadFixture(fixture);

      const amount = ethers.utils.parseEther('100');
      const amountUser2 = ethers.utils.parseEther('101');

      //1. Fund user1 and user2
      await long.connect(admin).transfer(user1.address, amount);
      await long.connect(admin).transfer(user2.address, amountUser2);

      //2. user1 getting ready to deposit.
      await long.connect(user1).approve(staking.address, amount);

      //3. user2 frontruns user1 to increase totalAssets while totalSupply remain same.
      await long.connect(user2).transfer(staking.address, amountUser2);

      //4. user1 deposit is executed after user2's donation attack
      const tx = await staking.connect(user1).deposit(amount, user1.address);

      // Zero shares minted to user1 causing loss of user1's 100 Long token.
      expect(await staking.balanceOf(user1.address)).to.eq(0);
      
    });
```
