# 57970 sc high forcerepay leaves cumulativeearmarked stale&#x20;

**Submitted on Oct 29th 2025 at 17:32:25 UTC by @winnerz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57970
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When liquidation triggers `_forceRepay`, the protocol reduces the user’s `account.earmarked` but doesn't decrease the global `cumulativeEarmarked`. This violates the invariant that the global earmark equals the sum of all account earmarks. Hence, subsequent calculations that depend on `cumulativeEarmarked` (e.g., `liveUnearmarked = totalDebt - cumulativeEarmarked` and weight updates) operate on an inflated value.

## Vulnerability Details

* **Invariant**: Let E = sum over all accounts of `account.earmarked`. The contract maintains a global `cumulativeEarmarked`intended to equal E. For any flow that decreases an account’s earmark by x, `cumulativeEarmarked` must also decrease by x to preserve E = `cumulativeEarmarked`.
* `_forceRepay` decreases the user’s earmark but does not decrease the global tally:
  * `src/AlchemistV3.sol:762` `account.earmarked -= earmarkToRemove;`
  * No corresponding `cumulativeEarmarked -= ...` in `_forceRepay`.
* In contrast, the other flows preserve the invariant:

  * `repay()` decreases both the user earmark and the global tally: `src/AlchemistV3.sol:523` (account) and `src/AlchemistV3.sol:526` (global).

  ```solidity
        account.earmarked -= earmarkToRemove;


        uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
        cumulativeEarmarked -= earmarkPaidGlobal;
  ```

  * `redeem()` decreases the global earmark by the redeemed amount: `src/AlchemistV3.sol:613`.
* After `_forceRepay` removes x from a user, E becomes (E − x) while `cumulativeEarmarked` remains E. The global variable is now overstated by x until a later redemption happens to reduce it. This is observable system-wide and persists across blocks.

Downstream effects (where the overstated global is used)

* `_earmark()` computes new earmarks and weights using global values; see `src/AlchemistV3.sol:1119–1128`, especially:
  * `cumulativeEarmarked += amount;` (`src/AlchemistV3.sol:1128`)
  * `WeightIncrement(amount, liveUnearmarked)` where `liveUnearmarked = totalDebt − cumulativeEarmarked`
* Because `cumulativeEarmarked` is overstated after `_forceRepay`, `liveUnearmarked` is understated, and weight updates can be mis‑scaled. This alters the timing/proportions of subsequent earmarks/redemptions across users.

## Impact Details

Contract fails to deliver promised returns, but doesn't lose value

* The PoC demonstrates a deterministic, system‑wide accounting inconsistency (global earmark > sum of per‑account earmarks) in core logic that persists until later redemptions modify the global tally.
* The overstated `cumulativeEarmarked` reduces `liveUnearmarked = totalDebt − cumulativeEarmarked` and mis‑scales the denominators used by weight updates in `_earmark()`, altering the timing/proportions of subsequent earmarks/redemptions. This does not claim theft/insolvency/freezing in this report; the issue is a correctness failure in how “promised” earmark/redemption accounting is computed over time.

## References

* `_forceRepay` per‑account decrement only: [src/AlchemistV3.sol:762](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L762)
* `repay()` per‑account and global decrement: [src/AlchemistV3.sol:523-526](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L523-L526)
* `redeem()` global decrement: [src/AlchemistV3.sol:613](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L613)

## Mitigation

In `_forceRepay`, after reducing the user’s earmark by `earmarkToRemove`, also reduce the global earmark by the same amount (saturating at zero) to restore consistency:

```solidity
uint256 paidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
cumulativeEarmarked -= paidGlobal;
```

This aligns `_forceRepay` with `repay()` and `redeem()` and preserves the global/per‑account earmark invariant.

## Proof of Concept

## Proof of Concept

**Set-up**

* Foundry, solc 0.8.28, EVM cancun. No RPC/fork required.
* Test: src/test/AlchemistV3\_AuditChecks.t.sol **Commands**
* Both PoCs:

```bash
FOUNDRY_PROFILE=default forge test -vvvv --match-path src/test/AlchemistV3_AuditChecks.t.sol --evm-version cancun
```

* PoC 1: ForceRepay clears user earmark but not global `cumulativeEarmarked`

```bash
FOUNDRY_PROFILE=default forge test -vvvv --match-test testForceRepayLeavesGlobalEarmarkStale --evm-version cancun
```

* PoC 2: After ForceRepay on A, global > sum of account earmarks

```bash
FOUNDRY_PROFILE=default forge test -vvvv --match-test testGlobalEarmarkExceedsAccountsSumAfterForceRepay --evm-version cancun
```

**workflow**

1. Deploy local system: ERC4626 vault (MYT), AlchemistV3, Transmuter, Position NFT.
2. User A: deposit 10k MYT shares, mint \~8k–8.2k debt.
3. Create and mature a Transmuter redemption; call alchemist.poke(A) to apply earmarks.
4. Record pre: A.earmarked, cumulativeEarmarked (cE), Transmuter MYT balance.
5. Tighten collateral limits; call alchemist.liquidate(A) → triggers \_forceRepay.
6. Record post: A.earmarked = 0; cE unchanged; Transmuter MYT balance increased.
7. User B: deposit 10k, mint \~6k; alchemist.poke(B) to apply earmarks.
8. Assert: cE > (A.earmarked + B.earmarked) = B.earmarked.

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {Test} from "../../../lib/forge-std/src/Test.sol";
import {IERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "../../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import {AlchemistV3} from "../../AlchemistV3.sol";
import {Transmuter} from "../../Transmuter.sol";
import {AlchemicTokenV3} from "../mocks/AlchemicTokenV3.sol";
import {AlchemistV3Position} from "../../AlchemistV3Position.sol";
import {AlchemistTokenVault} from "../../AlchemistTokenVault.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "../../../lib/forge-std/src/console.sol";

import {MockMYTVault} from "../mocks/MockMYTVault.sol";
import {MockMYTStrategy} from "../mocks/MockMYTStrategy.sol";
import {MYTTestHelper} from "../libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {TestERC20} from "../mocks/TestERC20.sol";
import {MockYieldToken} from "../mocks/MockYieldToken.sol";
import {AlchemistInitializationParams} from "../../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../../interfaces/ITransmuter.sol";

contract AlchemistV3_AuditChecks is Test {
    // Core
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemicTokenV3 alToken;
    AlchemistV3Position positionNFT;
    AlchemistTokenVault feeVault;

    // Vault/MYT
    MockMYTVault vault;
    MockMYTStrategy strategy;
    address admin = address(0xA11CE);
    address curator = address(0xC0DE);

    // Test actors
    address user = address(0xD00D);
    address user2 = address(0xD002);
    address feeReceiver = address(0xFEE);

    // Config
    uint256 timeToTransmute = 10; // small for tests (blocks)
    uint256 protocolFeeBps = 100; // 1%
    uint256 repayFeeBps = 200; // 2%
    uint256 liquidatorFeeBps = 300; // 3%
    uint256 minColl = 12e17; // 1.2
    uint256 lowerBound = 11e17; // 1.1

    function _dealToken(address token, address to, uint256 amount) internal { deal(token, to, amount); }

    function setUp() public {
        // Underlying + vault
        address underlying = address(new TestERC20(1_000_000 ether, 18));
        vault = new MockMYTVault(admin, underlying);

        // Strategy
        strategy = MYTTestHelper._setupStrategy(address(vault), address(new MockYieldToken(underlying)), admin, "Mock", "MockProtocol", IMYTStrategy.RiskClass.LOW);

        // Curator + adapters/caps
        vm.prank(admin);
        vault.setCurator(curator);
        bytes memory idData = strategy.getIdData();
        vm.prank(curator);
        vault.submit(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        vm.prank(curator);
        vault.addAdapter(address(strategy));
        uint256 largeCap = 1e27;
        vm.prank(curator);
        vault.submit(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, largeCap)));
        vm.prank(curator);
        vault.increaseAbsoluteCap(idData, largeCap);
        vm.prank(curator);
        vault.submit(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, 1e18)));
        vm.prank(curator);
        vault.increaseRelativeCap(idData, 1e18);

        // Seed vault
        address seeder = address(0x5EED);
        _dealToken(vault.asset(), seeder, 1_000_000 ether);
        vm.startPrank(seeder);
        IERC20(vault.asset()).approve(address(vault), type(uint256).max);
        vault.deposit(500_000 ether, seeder);
        vm.stopPrank();

        // Tokens + transmuter
        alToken = new AlchemicTokenV3("alUSD", "alUSD", 0);
        ITransmuter.TransmuterInitializationParams memory tp = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: feeReceiver,
            timeToTransmute: timeToTransmute,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 1_000_000
        });
        transmuter = new Transmuter(tp);

        // Alchemist via proxy
        AlchemistV3 logic = new AlchemistV3();
        AlchemistInitializationParams memory ap = AlchemistInitializationParams({
            admin: admin,
            debtToken: address(alToken),
            underlyingToken: vault.asset(),
            depositCap: type(uint256).max,
            minimumCollateralization: minColl,
            collateralizationLowerBound: lowerBound,
            globalMinimumCollateralization: minColl,
            transmuter: address(transmuter),
            protocolFee: protocolFeeBps,
            protocolFeeReceiver: feeReceiver,
            liquidatorFee: liquidatorFeeBps,
            repaymentFee: repayFeeBps,
            myt: address(vault)
        });
        bytes memory data = abi.encodeWithSelector(AlchemistV3.initialize.selector, ap);
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(logic), address(this), data);
        alchemist = AlchemistV3(address(proxy));

        // Bindings
        alToken.setWhitelist(address(alchemist), true);
        transmuter.setAlchemist(address(alchemist));
        transmuter.setDepositCap(type(uint128).max);

        // Position NFT + fee vault
        positionNFT = new AlchemistV3Position(address(alchemist));
        vm.prank(admin);
        alchemist.setAlchemistPositionNFT(address(positionNFT));
        feeVault = new AlchemistTokenVault(vault.asset(), address(alchemist), admin);
        vm.prank(admin);
        alchemist.setAlchemistFeeVault(address(feeVault));

        // User funding
        _dealToken(vault.asset(), user, 100_000 ether);
        vm.startPrank(user);
        IERC20(vault.asset()).approve(address(vault), type(uint256).max);
        vault.deposit(50_000 ether, user);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        vm.stopPrank();
    }

    function _createUserPosition(uint256 depositShares, uint256 debtToMint) internal returns (uint256 tokenId) {
        vm.startPrank(user);
        alchemist.deposit(depositShares, user, 0);
        tokenId = positionNFT.tokenOfOwnerByIndex(user, 0);
        alchemist.approveMint(tokenId, user, debtToMint);
        alchemist.mintFrom(tokenId, debtToMint, user);
        vm.roll(block.number + 1);
        vm.stopPrank();
    }

    function _createAndMatureRedemption(uint256 amount) internal returns (uint256 id) {
        vm.startPrank(user);
        deal(address(alToken), user, amount, true);
        IERC20(address(alToken)).approve(address(transmuter), type(uint256).max);
        transmuter.createRedemption(amount);
        uint256 idx = transmuter.balanceOf(user) - 1;
        id = transmuter.tokenOfOwnerByIndex(user, idx);
        vm.stopPrank();
        vm.roll(block.number + timeToTransmute + 1);
    }

    function _fundUserWithShares(address who, uint256 assets) internal returns (uint256 shares) {
        _dealToken(vault.asset(), who, assets);
        vm.startPrank(who);
        IERC20(vault.asset()).approve(address(vault), type(uint256).max);
        shares = vault.deposit(assets, who);
        vm.stopPrank();
    }

    // PoC #1: ForceRepay clears user earmark but not global cumulativeEarmarked
    function testForceRepayLeavesGlobalEarmarkStale() public {
        uint256 tokenId = _createUserPosition(10_000 ether, 8_200 ether);
        _createAndMatureRedemption(2_000 ether);
        alchemist.poke(tokenId);

        (, , uint256 earmarkedBefore) = alchemist.getCDP(tokenId);
        uint256 globalEarmarkedBefore = alchemist.cumulativeEarmarked();
        uint256 transmuterBalBefore = IERC20(address(vault)).balanceOf(address(transmuter));
        assertGt(earmarkedBefore, 0);

        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(1_220_000_000_000_000_000);
        alchemist.setCollateralizationLowerBound(1_220_000_000_000_000_000);
        vm.stopPrank();
        (uint256 amountLq,,) = alchemist.liquidate(tokenId);
        assertGt(amountLq, 0);

        (, , uint256 earmarkedAfter) = alchemist.getCDP(tokenId);
        uint256 globalEarmarkedAfter = alchemist.cumulativeEarmarked();
        uint256 transmuterBalAfter = IERC20(address(vault)).balanceOf(address(transmuter));

        assertEq(earmarkedAfter, 0);
        assertEq(globalEarmarkedAfter, globalEarmarkedBefore, "global earmark stale after forceRepay");
        assertGt(transmuterBalAfter, transmuterBalBefore);
    }

    // PoC #2: After ForceRepay on A, global > sum of account earmarks
    function testGlobalEarmarkExceedsAccountsSumAfterForceRepay() public {
        _fundUserWithShares(user, 20_000 ether);
        _fundUserWithShares(user2, 20_000 ether);

        uint256 tokenA = _createUserPosition(10_000 ether, 8_000 ether);
        vm.startPrank(user2);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(10_000 ether, user2, 0);
        uint256 tokenB = positionNFT.tokenOfOwnerByIndex(user2, 0);
        alchemist.approveMint(tokenB, user2, 6_000 ether);
        alchemist.mintFrom(tokenB, 6_000 ether, user2);
        vm.roll(block.number + 1);
        vm.stopPrank();

        deal(address(alToken), user, 3_000 ether, true);
        vm.startPrank(user);
        IERC20(address(alToken)).approve(address(transmuter), type(uint256).max);
        transmuter.createRedemption(3_000 ether);
        vm.stopPrank();

        vm.roll(block.number + 5);
        alchemist.poke(tokenA);
        alchemist.poke(tokenB);

        (, , uint256 aEar1) = alchemist.getCDP(tokenA);
        (, , uint256 bEar1) = alchemist.getCDP(tokenB);
        uint256 cum1 = alchemist.cumulativeEarmarked();
        console.log("Pre-forceRepay earmarks A:%s B:%s Global:%s", aEar1, bEar1, cum1);

        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(1_260_000_000_000_000_000);
        alchemist.setCollateralizationLowerBound(1_260_000_000_000_000_000);
        vm.stopPrank();
        alchemist.liquidate(tokenA);

        (, , uint256 aEar2) = alchemist.getCDP(tokenA);
        (, , uint256 bEar2) = alchemist.getCDP(tokenB);
        uint256 cum2 = alchemist.cumulativeEarmarked();
        console.log("Post-forceRepay earmarks A:%s B:%s Global:%s", aEar2, bEar2, cum2);

        assertEq(aEar2, 0);
        uint256 sumAccounts = aEar2 + bEar2;
        assertGt(cum2, sumAccounts, "global earmark exceeds sum of accounts after forceRepay");
    }
}
```

**Output**

* PoC 1 asserts A’s earmark becomes 0 while `cumulativeEarmarked` remains unchanged.
* PoC 2 prints:

```bash
console::log("Post-forceRepay earmarks A:%s B:%s Global:%s", 0, 642857142857142857143 [6.428e20], 1500000000000000000000 [1.5e21])
```

Additional trace cues

* `_forceRepay` path shows `ForceRepay(accountId, amount, creditToYield, protocolFeeTotal)` emitted, followed by a transfer of `creditToYield` MYT to the Transmuter; afterwards, `A.earmarked = 0` while `cumulativeEarmarked` is unchanged.
