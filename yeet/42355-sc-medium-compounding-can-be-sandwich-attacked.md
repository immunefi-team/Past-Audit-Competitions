# #42355 \[SC-Medium] Compounding can be sandwich attacked

**Submitted on Mar 23rd 2025 at 08:52:44 UTC by @Oxl33 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42355
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

**Description:**

Attackers (MEV bots) can sandwich attack manager's transaction, when he calls `MoneyBrinter::compound`, which leads to part of the rewards going to the attacker, instead of to regular users.

Example scenario:

Reminder - vault has no max-deposit/max-mint restrictions (`ERC4626::maxDeposit` and `ERC4626::maxMint` functions not overridden), so attackers can take a flashloan and deposit as much as they want, to end up with the majority of vault shares after depositing.

State before frontrun (normal state):

* assets in vault: `100 000e18`
* shares in vault: `100 000e23`

State before compound (attacker frontran and deposited `100 000e18` assets):

* assets in vault: `200 000e18`
* shares in vault: `200 000e23`

State after compound (before backrun):

* assets in vault: `210 000e18` (`10 000e18` assets added as rewards)
* shares in vault: `200 000e23` (no shares minted)

State after backrun (Attacker withdrew all his shares):

* assets in vault: `105 000e18` (only `5 000e18` of rewards left)
* shares in vault: `100 000e23`

If fee is `1%`, fee = `1 039.6e18`, profit for attacker = `5 000e18` - `1 039.6e18` = `3 960.4e18`

One way to fight this attack is for owner to increase exit fee before calling `compound` to make the attack unprofitable, then decrease the fee after. But this is not a convenient solution, because of these reasons:

1. It requires owner AND manager to be involved for every compound action (only owner can adjust exit fee, so manager alone cannot complete the process)
2. If there will be a time where manager calls `compound` without increasing the fee, especially if current fee is `0%`, then the sandwich attacker will steal up to `99%` of the newly added rewards from regular users

Easier solution to fight this type of attack would be to prevent withdrawals in the same block as deposits, or give the manager a function to pause the ability to deposit, so he can call it before calling `compound`.

**Impact:**

I believe the best fitting impact for this issue is `Theft of unclaimed royalties`, because attackers can steal newly added rewards, before regular users have the chance to claim them.

**Recommended Mitigation:**

Consider implementing withdrawal/redeem prevention in the same block as a deposit/mint, this will protect against attackers who need to take out a flashloan for the attack.

Additionally, consider implementing a function that pauses the ability to deposit/mint, and give the rights to call it only to manager, so he can call it before calling `compound`.

## Proof of Concept

**Proof of Concept:**

Consider this scenario:

1. Manager calls `MoneyBrinter::harvestBeradromeRewards`
2. Manager calls `MoneyBrinter::compound`
3. MEV bot sees this transaction in the mempool, and frontruns it with a big deposit into the vault
4. Manager's `compound` transaction goes through after the bot already deposited
5. MEV bot backruns the `compound` and redeems all his shares, receiving more assets than he deposited, due to increased `totalAssets`, but the same `totalSupply`
6. Regular users are left with less rewards, and the MEV bot profits from it
