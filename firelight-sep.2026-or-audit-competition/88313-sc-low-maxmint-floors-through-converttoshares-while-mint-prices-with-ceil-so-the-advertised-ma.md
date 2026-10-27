# 88313 sc low maxmint floors through converttoshares while mint prices with ceil so the advertised maximum can exceed the deposit limit by the rounding residue

**Submitted on Aug 12th 2026 at 17:49:34 UTC by @ret2basic for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88313
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

ERC-4626 requires `maxMint` to return the largest number of shares `mint` will accept without reverting. `FirelightVault.maxMint` derives that figure by converting the deposit limit into shares, which rounds **down**, while `mint` prices those shares back into assets with `Math.Rounding.Ceil` and then tests the result against the same limit. Whenever the share price is above one — which is the normal state of this vault, and the state every reward distribution moves it further into — the two roundings pull in opposite directions and the advertised maximum is refused by the vault's own limit check.

## Vulnerability Details

`maxMint` works in share units:

```solidity
function maxMint(address receiver) public view override returns (uint256 amount) {
    uint256 shares = totalSupply();
    uint256 sharesLimit = convertToShares(depositLimit);
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || shares > sharesLimit) {
        return 0;
    } else {
        return sharesLimit - shares;
    }
}
```

`convertToShares` floors: `sharesLimit = floor(depositLimit × (S + 1) / (A + 1))`.

`mint` converts back, rounding up, and enforces the limit in **asset** units:

```solidity
(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares,
    false,
    Math.Rounding.Ceil
);
_depositFunds(_msgSender(), receiver, assets, shares, _totalSupply, _totalAssets);
```

```solidity
_totalAssets += assets;
if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

so the call succeeds only if `A + ceil((sharesLimit − S) × (A + 1) / (S + 1)) ≤ depositLimit`.

Round-tripping `depositLimit → shares → assets` through a floor and then a ceil does not return the original when `A > S`, and the residue is on the wrong side of the comparison. The disagreement is structural, not a boundary artefact: it grows out of the share price, and it is `maxDeposit` — expressed in assets, the unit the limit is actually enforced in — that avoids it by never converting at all.

Reaching `A > S` requires nothing unusual. The live vault is already there: at the time of writing it holds 58,284,692.153737 fXRP against 58,280,499.270118 shares, a price of 1.000072, with `depositLimit` set to 60,168,000 fXRP — a limit close enough to bind. Every `VaultRewardDistributor.distributeRewards` and `distributeIncentive` call moves assets in without minting shares, pushing the price further above one.

### Concrete case

From the PoC, with a deposit limit of 10, one deposit of 3 and 4 assets of rewards forwarded in:

```
totalAssets: 7
totalSupply: 3
depositLimit: 10
maxMint(bob): 2
previewMint(maxMint): 4
```

`7 + 4 = 11 > 10`, so `mint(2, bob)` reverts `DepositLimitExceeded` on the amount the vault just advertised. The companion test shows `maxDeposit` in the same state is honoured exactly.

### How often, measured two ways

Enumerating the same two formulas over every state reachable as *deposit, then rewards forwarded in*, for deposit limits 8 through 200, the advertised maximum is unmintable in 590,251 of 989,970 states — 59.6%. That figure describes small-magnitude vaults, where the virtual `+1` offsets and the rounding residues are a large fraction of the numbers involved. **It is not representative of this deployment**, and it would be misleading to quote it alone.

Walking forward from the live vault's actual position instead — `totalAssets` 58,288,993,381,030, `totalSupply` 58,284,800,187,987, `depositLimit` 60,168,000,000,000, read at block `0x4021533` — through twenty thousand realistic deposits and reward distributions:

```
states reached from the live position : 1,044
of which mint(maxMint) would revert    : 1   (0.1%)

  the failing state: totalAssets = 60,167,999,999,999   (one unit below the limit)
                     maxMint = 1, previewMint = 2, overshoot = 1
```

So at production magnitudes the residue is a single unit, and the defect is confined to the last unit or two of capacity: `mint(maxMint(x), x)` reverts when the vault is essentially full. Queried live at that block, `maxMint` returns 1,878,871,447,009 and pricing it costs exactly the remaining room — the current state is one of the working ones.

## Impact Details

No funds are at risk and no attacker is involved. The cost is integration correctness, and it is narrow:

* `mint(maxMint(user), user)` is the standard way for a router, zapper or aggregator to take the largest position a vault will accept. Against this vault it reverts when the vault is within a rounding unit of its limit — which is precisely the moment an integrator is most likely to be probing for the last of the capacity.
* Any integrator that trusts `maxMint` as a bound — for sizing, for UI display, or for a simulation before submitting — is given a figure the vault will not honour, in that state.
* The vault otherwise presents itself as ERC-4626-compatible and documents its deviations precisely (`withdraw` and `redeem` queue rather than transfer, and the `Withdraw` event is not emitted). This deviation is not among them, so an integrator reading the documentation has no warning.

## Recommended fix

Compute the limit in the unit it is enforced in, then convert once, rounding in the direction that keeps the result acceptable:

```solidity
function maxMint(address receiver) public view override returns (uint256 amount) {
    uint256 assets = totalAssets();
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) {
        return 0;
    }
    // Round the remaining asset room DOWN into shares, so pricing those shares
    // back up with Ceil cannot exceed the room that was there.
    return _convertToShares(depositLimit - assets, Math.Rounding.Floor);
}
```

This mirrors `maxDeposit`, which is already correct, and makes `mint(maxMint(x), x)` succeed by construction. Adding an assertion to the test suite that `mint(maxMint(x), x)` never reverts would keep it that way.

## References

* `contracts/core/FirelightVault.sol` — `maxMint`, `maxDeposit`, `mint`, `_depositFunds`, `_previewTotals`, `_convertToSharesTotals`
* EIP-4626, `maxMint`: "MUST return the maximum amount of shares mint would allow to be deposited to receiver and not cause a revert"
* Repository: `https://github.com/immunefi-team/audit-comp-firelight` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

## Proof of Concept

## Proof of Concept

### Prerequisites

* **Toolchain:** `forge 1.7.1` (Foundry), `solc 0.8.28`, Node.js with `npm`.
* **No RPC required.** Runs entirely on a local EVM; no environment variables or forked state.
* **Target:** `https://github.com/immunefi-team/audit-comp-firelight`, branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.
* **Contract under test:** the real `FirelightVault` behind a real `ERC1967Proxy`, initialized through its production `initialize`.

### Attacker capabilities

**None.** There is no attacker. Two ordinary users deposit, and rewards arrive in the vault the way `VaultRewardDistributor` delivers them — as a plain asset transfer that mints no shares. The defect is in what the vault reports about itself.

No `vm.store`, no `vm.etch`, no role impersonation. All capital arrives by mint-then- `deposit`, or as the reward transfer being modelled.

### Environmental doubles, disclosed

* `MockERC20` as the vault asset (6 decimals, matching the live FXRP) — the project's own test double.
* Rewards are modelled as `asset.mint(address(vault), reward)`, which is exactly the state `VaultRewardDistributor.distributeRewards` produces: assets in, no shares out. The distributor itself is not deployed, because the finding is in `maxMint` versus `mint` and needs only the resulting share price.
* Small numbers (limit 10, deposit 3, reward 4) are used so the arithmetic is legible. The bounded fuzz covers limits up to 1,000, and the frequency figure in the report enumerates limits 8–200.

### Reproduction steps

1. Clone the target at the pinned commit and install its dependencies:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-firelight
   cd audit-comp-firelight
   git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
   npm ci                      # runs patch-package, which restores OpenZeppelin's SafeMath.sol
   git clone --depth 1 https://github.com/foundry-rs/forge-std node_modules/forge-std
   ```
2. Copy every file below into the matching path.
3. Run:

   ```bash
   forge test --match-contract Erc4626LimitsTest -vv
   ```
4. Expected output:

   ```
   Ran 3 tests for fuzz/Erc4626Limits.t.sol:Erc4626LimitsTest
   [PASS] testFuzz_maxMintIsOftenUnmintable(uint256,uint256,uint256) (runs: 256)
   [PASS] test_depositOfMaxDepositSucceeds()
   [PASS] test_mintOfMaxMintReverts()
   Logs:
     totalAssets: 7
     totalSupply: 3
     depositLimit: 10
     maxMint(bob): 2
     previewMint(maxMint): 4
   ```

### What each test establishes

* **`test_mintOfMaxMintReverts`** — the target behaviour. The vault advertises `maxMint(bob) == 2`; pricing those two shares through the vault's own `previewMint` costs 4 assets; `7 + 4 = 11` exceeds the limit of 10; and `mint(2, bob)` reverts `DepositLimitExceeded`. The vault refuses the number it just published.
* **`test_depositOfMaxDepositSucceeds`** — the control, in the identical state. `maxDeposit` is expressed in assets, the unit the limit is enforced in, so no conversion happens and `deposit(maxDeposit(bob), bob)` succeeds with the limit still holding. This isolates the cause as the share/asset round trip in `maxMint` rather than anything about the limit, the pause flags or the incident gate.
* **`testFuzz_maxMintIsOftenUnmintable`** — the same disagreement searched rather than hand-picked, over deposit limits up to 1,000. For each state it asserts the vault is in exactly one of two conditions: the advertised maximum mints, or it reverts `DepositLimitExceeded`. It never lands anywhere else, which is what makes the failure structural rather than incidental.

### Frequency

Two enumerations of the same two formulas (`convertToShares` flooring, `previewMint` ceiling), reported together because quoting either alone misleads.

Over deposit limits 8 through 200 — small-magnitude vaults, where the virtual `+1` offsets are a large share of the numbers — 989,970 states have a non-zero `maxMint` and 590,251 of them (59.6%) refuse it. That is the regime the tests above run in, chosen so the arithmetic is legible.

Walking forward from the **live** vault's position instead (`totalAssets` 58,288,993,381,030, `totalSupply` 58,284,800,187,987, `depositLimit` 60,168,000,000,000 at block `0x4021533`) through twenty thousand realistic deposits and reward distributions, only 1 of 1,044 reachable states refuses it, and that state is `totalAssets` exactly one unit below the limit. At production magnitudes the residue is a single unit, so the defect is confined to the last unit or two of capacity. The report states it this way rather than quoting the 59.6%.

### Complete runnable PoC files

#### `poc-inputs/foundry.toml`

```toml
[profile.default]
src = "contracts"
test = "fuzz"
out = "forge-out"
cache_path = "forge-cache"
libs = ["node_modules"]
# Vendored Flare FAsset sources pin =0.8.23 and are out of the audit scope.
skip = ["contracts/fasset/**"]
solc = "0.8.28"
evm_version = "london"
via_ir = true
optimizer = true
optimizer_runs = 200
remappings = [
  "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/",
  "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/",
  "flare-smart-contracts/=node_modules/flare-smart-contracts/",
  "flare-smart-contracts-v2/=node_modules/flare-smart-contracts-v2/",
  "forge-std/=node_modules/forge-std/src/",
]

[invariant]
runs = 400
depth = 200
fail_on_revert = false
```

#### `poc-inputs/fuzz/Erc4626Limits.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {MockERC20} from "../contracts/test/MockERC20.sol";

/// @dev ERC-4626 requires `maxMint` to return the largest share amount `mint`
///      will accept without reverting. The vault derives it from
///      `convertToShares(depositLimit)`, which floors, while `mint` prices those
///      shares with `Math.Rounding.Ceil` and then tests the result against the
///      same limit. When the share price is above one, the two roundings pull in
///      opposite directions and the advertised maximum is refused.
contract Erc4626LimitsTest is Test {
    FirelightVault internal vault;
    MockERC20 internal asset;
    address internal alice = address(0xA11CE);
    address internal bob = address(0xB0B);

    function _deploy(uint256 depositLimit) internal {
        vm.warp(1_800_000_000);
        asset = new MockERC20("fXRP", "fXRP", 6);
        FirelightVault impl = new FirelightVault();
        FirelightVault.InitParams memory p = FirelightVault.InitParams({
            defaultAdmin: address(this), limitUpdater: address(this), blocklister: address(this),
            pauser: address(this), periodConfigurationUpdater: address(this), rescuer: address(this),
            depositLimit: depositLimit, periodConfigurationDuration: 7 days
        });
        vault = FirelightVault(address(new ERC1967Proxy(address(impl),
            abi.encodeCall(FirelightVault.initialize,
                (IERC20(address(asset)), "s", "s", abi.encode(p))))));
        for (uint256 i; i < 2; i++) {
            address who = i == 0 ? alice : bob;
            asset.mint(who, 1e30);
            vm.prank(who);
            asset.approve(address(vault), type(uint256).max);
        }
    }

    /// A share price above one is reached the ordinary way: someone donates, or
    /// rewards are forwarded into the vault, which is exactly what
    /// `VaultRewardDistributor` does on every distribution.
    function test_mintOfMaxMintReverts() public {
        _deploy(10);

        vm.prank(alice);
        vault.deposit(3, alice);
        // Rewards arrive: 3 assets against 3 shares becomes 7 against 3.
        asset.mint(address(vault), 4);

        uint256 advertised = vault.maxMint(bob);
        console.log("totalAssets:", vault.totalAssets());
        console.log("totalSupply:", vault.totalSupply());
        console.log("depositLimit:", vault.depositLimit());
        console.log("maxMint(bob):", advertised);
        console.log("previewMint(maxMint):", vault.previewMint(advertised));

        assertGt(advertised, 0, "the vault advertises room to mint");
        assertGt(
            vault.totalAssets() + vault.previewMint(advertised),
            vault.depositLimit(),
            "pricing the advertised shares already exceeds the limit"
        );

        vm.expectRevert(FirelightVault.DepositLimitExceeded.selector);
        vm.prank(bob);
        vault.mint(advertised, bob);
    }

    /// `maxDeposit` does not have the problem: it is expressed in assets, which is
    /// the unit the limit is enforced in, so no conversion is involved.
    function test_depositOfMaxDepositSucceeds() public {
        _deploy(10);

        vm.prank(alice);
        vault.deposit(3, alice);
        asset.mint(address(vault), 4);

        uint256 room = vault.maxDeposit(bob);
        assertGt(room, 0, "the vault advertises room to deposit");
        vm.prank(bob);
        vault.deposit(room, bob);
        assertLe(vault.totalAssets(), vault.depositLimit(), "the limit still holds");
    }

    /// The same disagreement, searched rather than hand-picked, to show it is not
    /// a single contrived point.
    function testFuzz_maxMintIsOftenUnmintable(uint256 limitSeed, uint256 depositSeed, uint256 rewardSeed)
        public
    {
        uint256 limit = bound(limitSeed, 4, 1_000);
        _deploy(limit);

        uint256 deposited = bound(depositSeed, 1, limit / 2);
        vm.prank(alice);
        vault.deposit(deposited, alice);

        uint256 reward = bound(rewardSeed, 1, limit - deposited);
        asset.mint(address(vault), reward);

        uint256 advertised = vault.maxMint(bob);
        if (advertised == 0) return;

        // Either the advertised maximum is mintable, or the vault is advertising a
        // figure its own limit check will refuse. Only the first is ERC-4626.
        uint256 priced = vault.previewMint(advertised);
        if (vault.totalAssets() + priced <= vault.depositLimit()) {
            vm.prank(bob);
            vault.mint(advertised, bob);
        } else {
            vm.expectRevert(FirelightVault.DepositLimitExceeded.selector);
            vm.prank(bob);
            vault.mint(advertised, bob);
        }
    }
}
```
