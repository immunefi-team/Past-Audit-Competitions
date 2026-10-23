# 88960 sc low redeem can burn shares while crediting zero assets

**Submitted on Aug 16th 2026 at 00:36:54 UTC by @Web3Vinay for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88960
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`deposit()` rejects a dust input that would round to zero shares, and says why in a comment. `redeem()` has no matching guard on the *computed* `assets`. Once the share price is below 1 (any `payout()` does this), a small enough `shares` floors `assets` to `0` — the full `shares` still get burned, `WithdrawRequest` still emits as if the request worked, and the later `claimWithdraw` for that period reverts `NoWithdrawalAmount`.

### Vulnerability Details

`deposit()` guards the mirror case:

```solidity
// contracts/core/FirelightVault.sol:662-669
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(assets, true, Math.Rounding.Floor);
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
@> if (shares == 0) revert InvalidAmount();
```

`redeem()` only checks the input, never the computed output:

```solidity
// contracts/core/FirelightVault.sol:731-741
if (shares == 0) revert InvalidAmount();                     // input guard only
(uint256 assets, ...) = _previewTotals(shares, false, Math.Rounding.Floor);
@> // no `if (assets == 0) revert ...` here
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
```

`_requestWithdraw` then burns `shares` unconditionally, whatever `assets` came out to:

```solidity
// contracts/core/FirelightVault.sol:1015-1027
withdrawAssets[period] += assets;                        // += 0
withdrawShares[period] += sharesWithdraw;                // += 0
withdrawSharesOf[period][receiver] += sharesWithdraw;     // += 0
@> _update(owner, address(0), shares);                    // burns the full nonzero shares
emit WithdrawRequest(sender, receiver, owner, period, assets, shares);  // assets == 0
```

The flooring condition (`FirelightVault.sol:1067-1074`, `_decimalsOffset() == 0`): `assets == 0` for every `shares < (totalSupply()+1)/(totalAssets()+1)`, i.e. whenever `totalAssets() < totalSupply()` — the state any incident `payout()` produces.

### Impact Details

* The exact case `deposit()`'s comment says the code refuses to produce ("shares in exchange for nothing") happens anyway on the redeem side.
* `redeem()`'s NatSpec (`:715`) promises the return is the "Amount of assets that will be received" — it silently returns `0` instead.
* `WithdrawRequest` fires on a request that credits nothing; any indexer/dashboard treating the event as "a withdrawal is pending" is wrong until `claimWithdraw` reverts `NoWithdrawalAmount` at claim time.

Any account that can move the shares (owner or an approved spender) can hit this once the share price has dropped — no privileged role needed.

### Mitigation

```diff
         if (shares == 0) revert InvalidAmount();

         (uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
             shares,
             false,
             Math.Rounding.Floor
         );
+        // Mirror deposit()'s dust guard: a small redeem can floor to zero assets once the
+        // share price drops below 1; reject it instead of burning shares for nothing.
+        if (assets == 0) revert InvalidAmount();

         uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
```

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L662-L669>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L731-L741>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1015-L1027>

## Proof of Concept

One deposit, one `payout()` to drop the share price, one `redeem()` at the dust boundary. Save as `test/RedeemDustBurn.t.sol` and run `forge test --match-contract INSIGHT_RedeemDustBurn -vv`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";

contract MockUSD is ERC20 {
    constructor() ERC20("MockUSD", "mUSD") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

contract INSIGHT_RedeemDustBurn is Test {
    address internal constant ADMIN = address(0xA11CE);
    address internal constant PAYOUT_RECEIVER = address(0xBEEF);
    address internal constant VICTIM = address(0xC71CD);

    FirelightVault internal vault;
    MockUSD internal token;
    uint48 internal epoch;

    event WithdrawRequest(
        address indexed sender, address indexed receiver, address indexed owner,
        uint256 period, uint256 assets, uint256 shares
    );

    function setUp() public {
        vm.warp(1_800_000_000);
        token = new MockUSD();

        FirelightVault impl = new FirelightVault();
        IFirelightVault.InitParams memory p = IFirelightVault.InitParams({
            defaultAdmin: ADMIN, limitUpdater: ADMIN, blocklister: ADMIN, pauser: ADMIN,
            periodConfigurationUpdater: ADMIN, rescuer: ADMIN,
            depositLimit: type(uint128).max, periodConfigurationDuration: 1 days
        });
        vault = FirelightVault(address(new ERC1967Proxy(address(impl),
            abi.encodeCall(FirelightVault.initialize,
                (IERC20(address(token)), "Firelight Vault", "fvUSD", abi.encode(p))))));

        vm.startPrank(ADMIN);
        vault.grantRole(vault.PAYOUT_ROLE(), ADMIN);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), ADMIN);
        vault.addToPayoutAllowlist(PAYOUT_RECEIVER);
        vm.stopPrank();

        epoch = uint48(vm.getBlockTimestamp());
    }

    function test_redeemDustBurnsSharesForZeroCredit() public {
        // Normal LP deposit, share price exactly 1.
        token.mint(VICTIM, 100_000e6);
        vm.startPrank(VICTIM);
        token.approve(address(vault), 100_000e6);
        vault.deposit(100_000e6, VICTIM);
        vm.stopPrank();
        assertEq(vault.totalAssets(), vault.totalSupply());

        // A modest incident payout (5% loss) drops the share price below 1.
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 payAmt = 5_000e6;
        vm.prank(ADMIN);
        vault.payout(PAYOUT_RECEIVER, payAmt, epoch);

        uint256 A = vault.totalAssets();
        uint256 S = vault.totalSupply();
        assertLt(A, S, "window did not open");

        // Largest share count that still floors to zero assets.
        uint256 nMax = (S + 1) / (A + 1);
        if (nMax * (A + 1) >= S + 1) nMax -= 1;
        assertGt(nMax, 0, "no dust window");

        uint256 balBefore = vault.balanceOf(VICTIM);
        uint256 supplyBefore = vault.totalSupply();
        uint256 period = vault.currentPeriod() + 1;

        vm.expectEmit(true, true, true, true, address(vault));
        emit WithdrawRequest(VICTIM, VICTIM, VICTIM, period, 0, nMax);
        vm.prank(VICTIM);
        uint256 got = vault.redeem(nMax, VICTIM, VICTIM);

        assertEq(got, 0, "redeem returned nonzero");
        assertEq(vault.balanceOf(VICTIM), balBefore - nMax, "shares NOT burned");
        assertEq(vault.totalSupply(), supplyBefore - nMax, "supply NOT reduced");
        assertEq(vault.withdrawSharesOf(period, VICTIM), 0, "nonzero claim credited");

        console2.log("shares burned  ", nMax);
        console2.log("assets credited", got);
    }
}
```

### Run

```bash
forge test --match-test test_redeemDustBurnsSharesForZeroCredit -vv
```

### Result

```
Ran 1 test for test/RedeemDustBurn.t.sol:INSIGHT_RedeemDustBurn
[PASS] test_redeemDustBurnsSharesForZeroCredit() (gas: 670218)
Logs:
  shares burned   1
  assets credited 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.69ms
```

`redeem()` returned `0`, the shares were still burned, and `WithdrawRequest` fired with `assets = 0` (asserted exactly via `vm.expectEmit`) — the request looks successful on-chain and off-chain, but there is nothing to claim.
