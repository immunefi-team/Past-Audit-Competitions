# \#99000 [SC-High] HousePool keeps the whole owner recapitalization queued as a reservation after a tranche wipe although only part of it becomes LP equity: the stale remainder blocks senior exits and each later settlement lowers the senior high-water mark

**Submitted on Nov 11th 2026 at 11:52:37 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99000
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Smart contract unable to operate due to lack of token funds

## Description

## Summary

After a full tranche wipe with trader claims outstanding, the owner's documented recovery (`CfdEngine.recapitalizePool(R)` then `clearDegradedMode()`) is queued by `HousePool` as a pending recapitalization bucket of the full amount R. Only R minus the terminal deficit X of that cash can ever become LP equity, yet nothing writes the remaining X off. The stale remainder (a) remains fully reserved without ordinary settlement progress and is not backed by the original recapitalization cash; only later unrelated value or owner-only recovery releases it, so a senior LP's exit cannot be funded and, with that exit as the pending work, every permissionless `router.settleLpEpoch` reverts with `HousePool__NoLpEpochProgress` while the pool holds the cash; and (b) is re-applied at every later reconcile, which wipes the tranches again and re-runs the bootstrap, lowering the senior high-water mark by the equity slice each time although no loss occurred. One accounting root, two manifestations, and no malicious or compromised actor: the trusted owner is the trigger actor, acting honestly with the documented recovery; keepers are permissionless; the harmed party is the senior LP.

## Programme text this report relies on (Scope tab, quoted)

- Trusted role clause: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol." and "an in-scope impact arises from a logic error that a privileged address can trigger, which remains valid."
- Owner capabilities: "Immediately, can pause and unpause, set the treasury, recapitalize the pool, clear degraded mode ..."
- PoC rule: "PoCs must run either in the repository's Foundry test suite at the commit in scope, or on a local Anvil fork of the Arbitrum Sepolia v2.0.0 deployment (see Build Commands), without modifying the contracts in scope. Mocks may only replace components outside the scope, such as Pyth price updates and USDG."
- Feasibility: "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."
- Out-of-scope clauses answered here: "Attacks that require a malicious or compromised privileged role (admin, operator, sequencer, prover, guardian, governance) ... are out of scope" and "attacks that require a trusted role to act maliciously, to be compromised, or to act outside its documented operational procedures": the owner acts honestly and recapitalization is a documented procedure (SECURITY.md 155-161, 1509-1511; ACCOUNTING_SPEC.md 639-643). "Impacts involving centralization risks": no owner discretion is exercised against anyone. "Lack of liquidity impacts": the pool holds the cash, see the table below. "Incorrect data supplied by third party oracles": the stand-in feed values are the correct values of the hypothetical market move, not incorrect data. "Setting a parameter to a value the code allows is a trusted action, not a bug": no parameter is set.

## Impact selected: High, "Temporary freezing of funds for at least 24 hours"

Impact selected: High, "Temporary freezing of funds for at least 24 hours". The frozen funds are a third-party senior LP's redemption claim (2,267,675.253119 USDG funded at once on the patched pool); the LP, not the owner, owns them; the actor who triggers the state is the trusted owner performing the documented recovery, acting with its real capabilities and honestly; the state lasts at least 30 hours by hourly settlement attempts and about 14 days until the owner's fourth timelocked lever; ordinary retries do not end it.

Why this is not excluded as "a protocol-only liveness failure without an attacker" (the detailed definition's own exclusion list, which also excludes sender-controlled self-harm, voluntary self-lock, and ordinary by-design pending periods): this is not an infrastructure or environmental outage with no identifiable trigger. It is a genuine accounting bug, reached through a real, named, documented action (`recapitalizePool` then `clearDegradedMode`), harming a specific identified third party (the senior LP, not the owner and not the sender) for a specific measured amount and duration. The programme's own scope text directly addresses this exact shape: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol." That sentence sits beside, but textually separate from, "A bug that grants an address privileges it did not originally hold is also in scope, and may be downgraded by one severity level" — the downgrade is attached only to the privilege-escalation sentence, not to this one; if the programme meant every trusted-role-triggered bug to be capped below attacker-required impacts, this is exactly where it would say so, and it does not.

Critical, "Permanent freezing of funds" was considered and is not supported: the mechanism itself documents a working recovery path (the owner's fourth timelocked lever, 14 days after the recapitalization, or a further large owner recapitalization releases it sooner; see Release, below). A freeze with a known, working, if slow, recovery path is temporary by definition, not permanent.

Medium, "Smart contract unable to operate due to lack of token funds" is independently selected alongside High on these same facts, not as a conditional fallback: the contract is HousePool (the LP redemption and settlement contract); it is unable to operate (it cannot fund a queued senior redemption and every permissionless `settleLpEpoch` reverts); the required token funds are the free USDG the redemption needs, which the stale reservation drives to zero (`free` 0 in the table below) although 9,970,303.365360 USDG sit above all liabilities; the cause is an accounting defect reached through ordinary flows (trader orders, the owner's documented recovery, an LP exit), with no attacker; the state lasts at least 30 hours and about 14 days to the owner's release, and never ends by retry. Both impacts are selected together per the competition's rule that every applying impact may be selected and severity is set to the highest selected (High).

## Root cause (frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, `packages/perps/src`)

1. `CfdEngine.recapitalizePool` (CfdEngine.sol:632) pulls R from the owner into the pool and calls `HousePool.recordClaimantInflow(Recapitalization, R)` (HousePool.sol:624), which adds R to `pendingRecapitalizationAssets` (`_recordPendingClaimantInflow`, HousePool.sol:1794-1801).
2. At reconcile the bucket is capped by distributable value (`_settleablePendingClaimantBuckets`, HousePool.sol:1829-1834; `capClaimantBuckets`, HousePoolPendingPreviewLib.sol:72) and the capped part is applied by `applyClaimantRecapitalizationIntent` (HousePoolPendingPreviewLib.sol:111-137). When both principals are zero the whole applied amount becomes senior principal and the high-water mark becomes `max(target, amount)` (lines 117-123). Only the capped part is removed from the bucket (`_decreasePendingClaimantBuckets`, HousePool.sol:1772 and 1787-1791). The uncapped remainder, equal to the terminal deficit X that the cash had to fill, stays in `pendingRecapitalizationAssets` and nothing ever clears it.
3. The remainder is an ownerless reservation: `_buildWithdrawalSnapshot` (HousePool.sol:1727-1735) subtracts `_pendingClaimantBucketAssets()` from withdrawal capacity, and the reconcile plan treats it as pending claimant value that is removed from distributable value before the waterfall (HousePoolReconcilePlanLib.sol:68-71). With no cash behind it, every reconcile sees distributable value below the bucket, the tranches are wiped again, and the bootstrap branch of step 2 runs again with the remainder as the new target.

## Mechanism with numbers (deployed v2.0.0 state, all USDG, produced by the PoC below)

State after ordinary trader activity wipes both tranches: pool cash P = 43,428,135.992567; trader claims C = 75,731,545.598389; maximum liability of positions that remain open L = 11,878,653.855744; terminal deficit X = 43,513,122.755958; minimum recovery Rmin = C + L - P = 44,182,063.461566; `getPendingTrancheState` reports senior and junior principal 0 and 0 and the degraded latch is set. A senior LP holds 2,987,848,707,829,685 share units (from a 3,000,000 USDG deposit made before the shock) and queues a full exit.

The owner recapitalizes R = Rmin + Y with Y = 10,000,000 (54,182,063.461566) and clears degraded mode. Then:

| quantity | value |
|---|---|
| pool raw assets | 97,610,199.454133 (= P + R) |
| C + L + settlement buffer B (B = 29,696.634640) | 87,639,896.088773 |
| cash above those liabilities (= Y - B) | 9,970,303.365360 |
| reserved assets reported by the pool | 141,821,959.550339 = 87,639,896.088773 + the whole bucket 54,182,063.461566 |
| free assets | 0 |
| the first `settleLpEpoch` and 30 further hourly calls | all revert `HousePool__NoLpEpochProgress` (0x86cca6b8) |
| the LP's exit | never funded; `cancelRedeemRequest` reverts `RedeemCancellationUnavailable` (0x3260a8c1) |

The same exit on a pool with the stale remainder written off is funded 2,267,675.253119 at the first settlement (control below). The block is deterministic: the first settlement reverts if and only if Y <= X + B (sweep: Y = X + 29,000 blocked; Y = X + 31,000 funded 1,303.365360 = Y - X - B). No third party can enter around it: after the wipe both vaults refuse deposits (senior `DepositsUnavailable` 0x15fb1002, junior `TerminallyWiped` 0x5616d460).

Release: the stale remainder drains only one equity slice (R - X = 10,668,940.705608) per owner lever, each lever being `proposePoolConfig` then `finalizePoolConfig` after the 72 h timelock with a fresh mark from the permissionless `router.updateMarkPrice`; the exit is funded after the fourth lever, 14 days after the recapitalization (finalize times Fri 2026-11-20 08:00:01Z, Mon 11-23 09:00:01Z, Thu 11-26 10:00:01Z, Sun 11-29 23:00:01Z; the recapitalization is at Sun 11-15 23:00Z). A further 20,000,000 owner recapitalization also releases it. HousePool is not upgradeable.

Second manifestation (no exit queued; the junior zero-NAV exits only supply settlement progress): the first settlement bootstraps senior principal 10,668,940.705608 with high-water mark R = 54,182,063.461566 (the recorded bootstrap) and leaves bucket 43,513,122.755958. Each later settlement lowers the mark by exactly 10,668,940.705608 with no loss: 54,182,063.461566, 43,513,122.755958, 32,844,182.050350, 22,175,241.344742, 11,506,300.639134, while senior principal stays 10,668,940.705608. The documented rules say the opposite: "losses do not reduce [the high-water mark]" (HousePoolWaterfallAccountingLib header), ACCOUNTING_SPEC.md 337-338 ("Losses do not reduce the Senior high-water mark." / "Route surplus first to restore Senior principal to its high-water mark, then to Junior."). Junior participates in revenue before senior is restored to the mark the code itself set; the re-routing is capped by R minus the final mark and no principal is lost.

## Actor model

No malicious or compromised attacker. The trusted owner is the identifiable trigger actor for High; Medium requires no attacker. Owner: honest trusted role performing the documented recovery (SECURITY.md 155-161, 1509-1511; ACCOUNTING_SPEC.md 639-643), choosing R inside the range the code allows (no document fixes R); the PoC impersonates the real owner address for the four owner calls only. Keepers: permissionless (`router.updateMarkPrice`, `router.settleLpEpoch`). LP: queues an ordinary exit. Harmed party: a third-party senior LP (not the owner; the funding call mints the owner no shares). Beneficiary of the re-routing: junior LPs. Bug: the pool keeps counting the part of R that only filled the deficit as a live reservation.

## Feasibility chain, disclosed exactly

- The wipe with claims above cash needs a large basket move. The PoC uses a stand-in price path (two basket moves, about +98 percent then -95 percent) through a mock of the Pyth contract at the deployed Pyth address, which the programme permits ("Mocks may only replace components outside the scope, such as Pyth price updates and USDG"). The feed values are the correct values for that hypothetical market move, not incorrect oracle data. Genuine signed Pyth history for the six feeds could not be retrieved (the public Hermes and benchmarks price endpoints answer HTTP 401); against public ECB daily history 2010-2026 the best single hindsight round trip reaches 0.74 of the pool where a wipe needs 1.0838 (round-trip payout x + 1.0811 y of entry, x the up leg and y the down leg).
- The pinned deployment still carries unidentified open positions (long open interest 122,982 lots; only 8 accounts identified), so L stays 11,878,653.855744 and the exact zero-open-interest case cannot be built there; the mechanism and the arithmetic hold for any L (the stale part is exactly the terminal deficit X).
- Amounts are test liquidity (mock USDG, 102.17M USDG pool at the pin); the arithmetic is ratio-based. The programme says severity uses the stated parameters; the release deployment (Arbitrum One) is not yet live.

## Knownness distinction (minimum detail)

The first post-wipe recapitalization resetting the senior mark to the queued target is documented in code (HousePoolPendingPreviewLib.sol:103-110) and asserted by the retained suite (TrancheRecoveryBehavior.t.sol:13-79); this report does not rely on that reset. The defect here is that the part of R that only fills the terminal deficit is never written off: it stays reserved with no cash behind it and is re-applied at every later reconcile. The pre-audit finding SC-L1 was fixed by plether-core PR #29 (closed unmerged; its branch commit 31a38bb7 is contained in master), whose response says "Anything that can't be settled yet stays pending, stays reserved everywhere, and gets applied later once it's backed." SC-L1 deliberately preserved amounts that could not yet settle. M175 shows incomplete handling of that design in terminal-deficit recovery: the part of the recapitalization consumed by pre-existing claims cannot be backed by the original recap cash, yet it remains fully reserved; this newly deadlocks an otherwise payable senior exit, and later reconciliation reuses the residual as a bootstrap target and lowers HWM without a new loss. Later unrelated value can release the reservation, as the 20M recap control demonstrates, so the novelty claim is these two new consequences/incomplete-fix behaviors, not that the residual can never receive future backing. The documented-action support for the owner's recovery is the commit that added `recapitalizePool` (plether-core PR #121). Cecuro MED-12 states that partial-settlement residuals stay reserved; that is a design for partial settlement of real value, whereas here the reserved amount exceeds the cash above liabilities (141,821,959.550339 reserved against 97,610,199.454133 raw) and the PoC proves no cash backs it. MED-1, MED-11 and LOW-54 concern freshness gating of finalization, withdrawal scaling and sub-atom dust. The programme's open-issue list (MED-7, MED-13, MED-16, LOW-57, MED-22, MED-24, MED-29, MED-30) does not contain it.

## Why the obvious objections do not apply

- "The owner chose R": `recapitalizePool` accepts any nonzero amount (CfdEngine.sol:632-653) and clearing degraded mode needs the pool to cover its liabilities (CfdEngine.sol, `clearDegradedMode` NatSpec), so R of at least Rmin is the documented recovery; no document fixes R, and the same exit on the patched pool is funded at the first settlement (shown for R = Rmin + 10,000,000). The defect is what the pool does with the R it was given.
- "Lack of liquidity": the pool holds 9,970,303.365360 of cash above all liabilities and the block exists only because the stale remainder is reserved against it.
- "Centralization": no owner discretion is used; the owner does not withhold or redirect funds.
- "Setting a parameter to a value the code allows": no parameter is set; the funding amount is an input, the accounting is the bug.
- "Temporary, the owner can release": correct, and that is exactly why the selected impact is "temporary freezing" rather than a permanent-freezing claim, not a reason the finding fails; the release still takes four 72 h levers (about 14 days) or an extra 20,000,000 owner recapitalization, and ordinary retries (30 of 30) fail in the meantime.
- "No attacker, so this cannot be a High freeze": the honest, documented trigger is exactly the scope text's named in-scope case ("a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol"), stated beside, but without, the downgrade the scope text gives the adjacent privilege-escalation case; High is supported directly on this text, and Medium SC-17 (no attacker required) is selected alongside it as a second, independently-fitting impact on the same facts, not a condition on High being rejected.

## Limits

Stand-in price path (disclosed); only 8 of the open positions are identifiable on the pinned state; contracts are not upgradeable so a fix needs redeployment; patched HousePool runtimes exceed the EIP-170 limit by 57 bytes (patch A) so a deployable fix also needs byte savings elsewhere.


## Proof of Concept

## Proof of Concept: fork test of the deployed Plether perps v2.0.0 contracts (Arbitrum Sepolia) at a pinned block

Rule text this PoC follows. Immunefi Web3 PoC Guidelines: "The smart contract PoC should always be made by forking the mainnet using tools like Hardhat or Foundry." and "No unit test PoCs will be accepted." Programme: "PoCs must run either in the repository's Foundry test suite at the commit in scope, or on a local Anvil fork of the Arbitrum Sepolia v2.0.0 deployment (see Build Commands), without modifying the contracts in scope. Mocks may only replace components outside the scope, such as Pyth price updates and USDG." This PoC is a Foundry test that forks the deployed contracts at block 326928451 from a local Anvil fork and calls them only through their production entry points; no contract in scope is modified in the vulnerable run.

### What the test does that is not a production call (every device, line by line)
1. `vm.createSelectFork(url, 326928451)` and `vm.chainId(421614)`: the fork of the deployment. `vm.warp`/`vm.roll`: time moves to Sunday 2026-11-15 23:00:00Z (market open) and then forward hour by hour; the deployment has 72 h timelocks and hourly LP epochs.
2. `vm.etch` of a mock `ArbSys` at 0x64 (the local EVM does not simulate that precompile).
3. `vm.etch` of the repository's `MockPyth` at the deployed Pyth address, so the six-feed basket takes the price path of the test (a Pyth price-update mock, which the programme permits). Real signed Pyth history for the six feeds is not publicly retrievable and the wipe needs a basket path of about +98 percent then -95 percent.
4. Mock USDG `mint(address,uint256)`: the deployed permissionless mint, used to fund the actors.
5. `vm.prank` of the real owner address 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B for exactly four calls: `CfdEngine.recapitalizePool`, `CfdEngine.clearDegradedMode`, `HousePool.proposePoolConfig`, `HousePool.finalizePoolConfig` (the trusted role the impact names, acting as itself). Every other call is made by the actor's own address: traders (`commitOrder`), a permissionless keeper (`executeOrder`, `router.updateMarkPrice`, `router.settleLpEpoch`), LPs (`requestDeposit`, `requestRedeem`, `claimDeposit`, `cancelRedeemRequest`).
6. Only in the fix-check builds (step 5): `vm.etch` puts the patched `HousePool` runtime over the deployed one. The vulnerable run does not.
`vm.deal` is used only to give native ETH for gas and Pyth update fees (keeper 1000 ether, each trader 10 ether, each LP and the probe account 1 ether), in setup and actor creation. No USDG is minted by cheatcode: all USDG comes from the deployed permissionless mint of item 4. There is no `vm.store` and no `vm.mockCall`, and no cheatcode grants any party a capability in the exploit phase.

### Step 1. Setup
```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether && git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
# Foundry 1.5.1-stable (solc 0.8.35, via-IR, optimizer 200, Prague; from packages/perps/foundry.toml). Make three separate copies of the checkout
# (one per build) so the patch of step 5 never touches the vulnerable copy:
cp -r . ../plether-frozen && cp -r . ../plether-fixa && cp -r . ../plether-fixh
# Local Anvil fork of Arbitrum Sepolia pinned at the block (needs an archive-capable JSON-RPC URL for chain 421614 in ARB_SEPOLIA_RPC; only state reads are sent):
anvil --fork-url "$ARB_SEPOLIA_RPC" --fork-block-number 326928451 --port 8545 --host 127.0.0.1 --no-storage-caching &
```
State is fetched lazily from the RPC on first use (the first run took about 9 minutes against a rate-limited free tier; later runs in the same Anvil session take under 10 seconds).

### Step 2. The code (save the files exactly as shown; the run script copies them into the checkout)
Layout: `poc/run_fork.sh`, `poc/fixA_writeoff.diff`, `poc/fixH_intent_mark.diff`, `poc/test-m175/LiveBase.sol`, `poc/test-m175/PoCBase.sol`, `poc/test-m175/M175Fork.t.sol`.

`poc/run_fork.sh`
```bash
#!/bin/bash
# Usage: run_fork.sh <checkout-root> <build> <fork-url>      build = frozen | fixa | fixh
# <checkout-root>: a copy of the competition checkout at a82d4fefb02768e5b3bddcc97dee371eee24fea8 (use a separate copy per build).
# <fork-url>: JSON-RPC URL of a local Anvil fork of Arbitrum Sepolia pinned at block 326928451 (see step 3), or an archive RPC for chain 421614.
# The test itself selects block 326928451 (vm.createSelectFork(url, 326928451)) and sets the chain id; no contract in scope is modified for "frozen".
set -eu
ROOT="$1"; BUILD="${2:-frozen}"; export M175_FORK_URL="$3"; HERE="$(cd "$(dirname "$0")" && pwd)"
PERPS="$ROOT/packages/perps"
mkdir -p "$PERPS/test-m175"
cp "$HERE"/test-m175/*.sol "$PERPS/test-m175/"
case "$BUILD" in
  fixa) (cd "$PERPS" && patch -p0 src/HousePool.sol < "$HERE/fixA_writeoff.diff") ;;
  fixh) (cd "$PERPS" && patch -p0 src/HousePool.sol < "$HERE/fixH_intent_mark.diff") ;;
esac
cd "$PERPS"
export FOUNDRY_TEST=test-m175
if [ "$BUILD" = frozen ]; then export M175_EXPECT=frozen; else export M175_EXPECT="$BUILD"; export M175_ETCH=1; fi
forge test --match-contract M175Fork -vv
```

`poc/test-m175/LiveBase.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";

interface IUSDG {
    function mint(address to, uint256 amount) external;
    function approve(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
    function totalSupply() external view returns (uint256);
}

contract MockArbSys {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}

/// @notice Addresses of the Arbitrum Sepolia v2.0.0 deployment (state forked at block 326928451, Sat 2026-11-07 12:00:00Z).
/// Setup-only cheats: chainId, warp/roll, ArbSys code at 0x64 (the precompile is not simulated by the local EVM), Pyth mock at the deployed
/// Pyth address (the real Pyth verifies signed VAAs for a price path that does not exist in market history).
abstract contract LiveBase is Test {
    uint256 internal constant PIN_TS = 1794052800;
    uint256 internal constant PIN_BLOCK = 326928451;
    address internal constant OWNER = address(uint160(0x005a71a4094ec81165ada48aa4c27da48ec27e0d6b));

    IUSDG internal token = IUSDG(address(uint160(0x003d051dbbf5ff427a4ae8ad09746902666af8fb9d)));
    MarginClearinghouse internal ch = MarginClearinghouse(address(uint160(0x00b5b851f627d6df3cf3f096e23e20a58884d7b89a)));
    CfdEngine internal engine = CfdEngine(address(uint160(0x00ca18a6783d7542f9d2f28f4f259772b74c4d6baa)));
    HousePool internal pool = HousePool(address(uint160(0x00eb5604c393843b2f3c347ec24fa9ddad6ab2131e)));
    TrancheVault internal senior = TrancheVault(address(uint160(0x005dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d)));
    TrancheVault internal junior = TrancheVault(address(uint160(0x004a8b67184a1e3224c11a191cc69f22c50eef734e)));
    OrderRouter internal router = OrderRouter(payable(address(uint160(0x00918663d6da8cb33b325a04c4a5e8c51d42f3b444))));
    PletherOracle internal oracle = PletherOracle(address(uint160(0x004f2c62a0e03657f52c633c19c11f88d8f0897bb2)));
    PositionProtectionBook internal prot = PositionProtectionBook(address(uint160(0x0081e3b03cf29865b6c0f85925839567e2bc61073d)));
    MockPyth internal pyth;

    function _fund(address a, uint256 amt) internal {
        token.mint(a, amt); // permissionless mint on the testnet Mock USDG (deployed function)
        vm.startPrank(a);
        token.approve(address(ch), type(uint256).max);
        token.approve(address(senior), type(uint256).max);
        token.approve(address(junior), type(uint256).max);
        vm.stopPrank();
    }
}
```

`poc/test-m175/PoCBase.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {LiveBase, MockArbSys} from "./LiveBase.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// Helpers. Price device: MockPyth code placed at the deployed Pyth address; every feed is set so the six-feed basket equals the
/// requested basket price exactly (component 0, the 57.6 percent weight, carries the move; the other five stay at their base).
abstract contract PoCBase is LiveBase {
    uint256 constant LOT = 1e20;
    address internal keeper = address(0xBEEF0001);
    uint256 internal lastTick;
    uint256 internal constant W0 = 57_600_000; // component 0 weight, 8 decimals (0.576)

    function _setUpLive(uint256 startTs) internal {
        // state source: the deployed contracts at the pinned block, through the fork RPC given in M175_FORK_URL
        vm.createSelectFork(vm.envString("M175_FORK_URL"), PIN_BLOCK);
        vm.chainId(421614);
        vm.warp(startTs);
        vm.roll(PIN_BLOCK + (startTs - PIN_TS) / 1); // arbitrum block numbers are not used for timing here; monotone is enough
        MockArbSys m = new MockArbSys();
        vm.etch(address(100), address(m).code);
        address pythAddr = address(oracle.pyth());
        vm.etch(pythAddr, address(new MockPyth()).code);
        pyth = MockPyth(pythAddr);
        vm.deal(keeper, 1000 ether);
    }

    // ------------------------------------------------------------------ price device
    function _feedPx(uint256 i) internal view returns (uint256) {
        uint256 b = oracle.basePrices(i);
        return oracle.inversions(i) ? 1e16 / b : b;
    }
    /// feed vector whose basket equals `tau` (8 decimals; 1e8 = neutral): every component scaled by the same multiplier.
    function _payload(uint256 tau, uint256 t, uint256 prev) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            // uniform multiplier m = tau/1e8 on every component's normalised price: non-inverted raw = base*m, inverted raw = 1e16/(base*m)
            uint256 b = oracle.basePrices(i);
            uint256 px = oracle.inversions(i) ? (1e16 * 1e8) / (b * tau) : (b * tau) / 1e8;
            u[i] = MockPyth.FeedUpdate({id: oracle.pythFeedIds(i), price: MockPyth.MockPrice({price: int64(int256(px)), conf: 0, expo: -8, publishTime: t, prevPublishTime: prev})});
        }
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }
    function _now() internal view returns (uint256) { return vm.getBlockTimestamp(); }
    function _adv(uint256 dt) internal { vm.warp(_now() + dt); vm.roll(vm.getBlockNumber() + 1 + dt / 4); }
    /// permissionless production mark refresh (router.updateMarkPrice), keeper address, payload at `tau`
    function _mark(uint256 tau) internal returns (bool ok, bytes memory err) {
        uint256 pub = _now(); uint256 prev = lastTick; lastTick = pub;
        vm.prank(keeper);
        (ok, err) = address(router).call(abi.encodeWithSignature("updateMarkPrice(bytes[])", _payload(tau, pub, prev)));
    }
    function _markOk(uint256 tau) internal { (bool ok, bytes memory e) = _mark(tau); require(ok, string.concat("updateMarkPrice reverted ", vm.toString(abi.encodePacked(_sel(e))))); }

    // ------------------------------------------------------------------ orders (production entries: commitOrder / executeOrder)
    function _req(uint256 lots, uint256 margin, bool close, bool long_) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = bytes32(uint256(keccak256(abi.encode(_now(), lots, margin, close, gasleft()))));
        r.side = long_ ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT; r.sizeDelta = lots * LOT; r.marginDelta = margin; r.isClose = close;
        r.targetPrice = close ? (long_ ? 2e8 : 1) : (long_ ? 1 : 2e8);
        uint32 w = uint32(router.maxExecutionWindowSeconds());
        r.bounds = OrderV3Types.ExecutionBounds({submitBy: uint64(_now() + w), executionWindowSeconds: w, allowedExecutionModes: 7, expectedConfigHash: abi.decode(_sc(address(router.lifecycleBook()), abi.encodeWithSignature("currentExecutionConfigHash()")), (bytes32)), maxExecutionBountyAssets: type(uint256).max, maxExecutionNotionalAssets: type(uint256).max, maxGrossAccountDebitAssets: type(uint256).max, maxActionChargeAssets: type(uint256).max, maxExplicitFeesAssets: type(uint256).max, maxPostPositionSize: type(uint256).max, minPostSettlementBalanceAssets: 0, minPostPositionEquityAssets: 0, maxPostLeverageBps: type(uint32).max});
    }
    function _sc(address t, bytes memory d) internal view returns (bytes memory o) { (bool ok, bytes memory r) = t.staticcall(d); require(ok); o = r; }
    function _commit(address a, OrderV3Types.OrderRequest memory r) internal returns (bool ok, bytes memory ret) {
        vm.prank(a); (ok, ret) = address(router).call(abi.encodeWithSignature("commitOrder((bytes32,uint8,uint256,uint256,uint256,bool,(uint64,uint32,uint8,bytes32,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint32)))", r));
    }
    function _sel(bytes memory e) internal pure returns (bytes4 s) { if (e.length >= 4) s = bytes4(e); }
    function _x(uint256 v) internal pure returns (string memory) { return vm.toString(v); }
    /// commit, advance 1 s, execute the head order at basket `tau`. Returns (committed, executed, reason).
    function _order(address a, uint256 lots, uint256 margin, bool close, bool long_, uint256 tau) internal returns (bool okc, bool oke, bytes memory why) {
        bytes memory ec; (okc, ec) = _commit(a, _req(lots, margin, close, long_));
        if (!okc) { why = ec; return (false, false, why); }
        _adv(1);
        uint64 head = router.nextExecuteId(); uint256 pub = _now(); uint256 prev = lastTick; lastTick = pub;
        vm.prank(keeper); (oke, why) = address(router).call(abi.encodeWithSignature("executeOrder(uint64,bytes[])", head, _payload(tau, pub, prev)));
    }
    function _pos(address a) internal view returns (uint256 size, uint256 margin, uint256 entry) { (size, margin, entry,,,,) = engine.positions(a); }
    function _trader(uint256 n, uint256 cash) internal returns (address a) { a = address(uint160(0xC0FFEE00 + n)); vm.deal(a, 10 ether); _fund(a, cash); vm.prank(a); ch.deposit(a, cash); }

    function _snap(string memory tag) internal view {
        IHousePool.PoolLiquidityView memory v = pool.getPoolLiquidityView();
        console2.log(string.concat("SNAP|", tag, "|rawAssets=", _x(pool.rawAssets()), "|claims=", _x(engine.totalTraderClaimBalanceAssets()), "|sr=", _x(v.seniorPrincipalAssets), "|jr=", _x(v.juniorPrincipalAssets), "|hwm=", _x(v.seniorHighWaterMarkAssets), "|bucket=", _x(v.pendingRecapitalizationAssets), "|reserved=", _x(v.withdrawalReservedAssets), "|free=", _x(v.freeAssets), "|deficit=", _x(v.currentTerminalDeficitAssets), "|degraded=", v.degradedMode ? "1" : "0", "|markFresh=", v.markFresh ? "1" : "0", "|mark=", _x(engine.lastMarkPrice())));
    }

    // ------------------------------------------------------------------ LP actions (production entries on the deployed vaults)
    uint256 internal px; // current stand-in basket price (8 decimals) used by keeper settlements
    function _settle() internal returns (bool ok, bytes memory err) {
        uint256 pub = _now(); uint256 prev = lastTick; lastTick = pub;
        vm.prank(keeper);
        (ok, err) = address(router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", _payload(px, pub, prev)));
    }
    function _lpDeposit(address vault, address lp, uint256 amount) internal returns (uint256 id) {
        _fund(lp, amount); vm.deal(lp, 1 ether);
        vm.prank(lp); id = TrancheVault(vault).requestDeposit(amount, lp, lp);
    }
    function _claimDep(address vault, address lp, uint256 id) internal returns (uint256 shares) {
        uint256 a = TrancheVault(vault).claimableDepositRequest(id, lp);
        vm.prank(lp); shares = TrancheVault(vault).claimDeposit(id, a, lp, lp);
    }
    function _toEpoch(address vault, uint256 id) internal { uint256 t = TrancheVault(vault).depositEpochStart(id); if (_now() < t) { vm.warp(t); vm.roll(vm.getBlockNumber() + 1); } }
    function _requestExit(address vault, address lp) internal returns (uint256 id, uint256 shares) {
        shares = TrancheVault(vault).maxRequestRedeem(lp);
        vm.prank(lp); id = TrancheVault(vault).requestRedeem(shares, lp, lp);
        uint256 m = pool.lpEpochStart(id); if (_now() < m) { vm.warp(m); vm.roll(vm.getBlockNumber() + 1); }
    }

    function _cfgBump() internal view returns (IHousePool.PoolConfig memory c) {
        c = IHousePool.PoolConfig({seniorRateBps: pool.seniorRateBps() + 1, markStalenessLimit: pool.markStalenessLimit(), seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(), juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(), maxSeniorExposureAssets: pool.maxSeniorExposureAssets(), maxSeniorShareBps: pool.maxSeniorShareBps()});
    }
}
```

`poc/test-m175/M175Fork.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PoCBase} from "./PoCBase.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// Fork test of the DEPLOYED Plether perps v2.0.0 contracts (Arbitrum Sepolia 421614, block 326928451, 2026-11-07 12:00:00Z); time is moved to
/// Sunday 2026-11-15 23:00:00Z (market open) by a setup warp.
/// Production entries only: commitOrder/executeOrder, router.updateMarkPrice, router.settleLpEpoch, TrancheVault.requestDeposit/requestRedeem,
/// engine.recapitalizePool/clearDegradedMode (owner), pool.proposePoolConfig/finalizePoolConfig (owner).
/// Cheats: chainId/warp/roll (setup and passage of time), ArbSys code at 0x64, a Pyth mock at the deployed Pyth address (the price path),
/// vm.prank(owner) as the real owner address ONLY for the four owner calls above (the trusted role the impact names),
/// Mock USDG permissionless mint (deployed function) to fund actors. No other capability-granting cheat.
/// M175_EXPECT = frozen (default) | fixa | fixh selects the expected outcome; M175_ETCH=1 puts the patched HousePool runtime over the deployed one.
contract M175Fork is PoCBase {
    uint256 constant START = 1794783600; // Sun 2026-11-15 23:00:00Z
    uint256 constant P0 = 96_101_009; uint256 constant P1 = 190_000_000; uint256 constant P2 = 5_000_000;
    address A; address S2; address SLP; address[4] JLP;
    uint256 internal expectMode; // 0 frozen, 1 fixa, 2 fixh
    uint256 internal jid0;
    bool internal oStale; bool internal oBlocked; bool internal oHwmFell; bool internal oFundedAt1; uint256 internal oBucket1;

    struct Shock { uint256 P; uint256 C; uint256 L; int256 delta; uint256 X; uint256 Rmin; uint256 d; }

    function setUp() public {
        _setUpLive(START);
        string memory e = vm.envOr("M175_EXPECT", string("frozen"));
        expectMode = keccak256(bytes(e)) == keccak256("fixa") ? 1 : keccak256(bytes(e)) == keccak256("fixh") ? 2 : 0;
        if (vm.envOr("M175_ETCH", uint256(0)) == 1) {
            HousePool np = new HousePool(address(token), address(engine), address(housePoolSidecar()));
            vm.etch(address(pool), address(np).code);
            console2.log("RES|etched HousePool runtime bytes", address(np).code.length);
        }
    }
    function housePoolSidecar() internal view returns (address) { return address(uint160(0x0070cd3b39bc5a006784777048c7b690f10642cbf1)); }
    function _res(string memory k, string memory v) internal pure { console2.log(string.concat("RES|", k, "|", v)); }
    function _b(bool v) internal pure returns (string memory) { return v ? "true" : "false"; }

    // ------------------------------------------------------------------ phase 0: ordinary LPs on the deployed vaults
    function _lps() internal {
        px = P0; _adv(2); _markOk(P0);
        SLP = address(0x51B0); uint256 sid = _lpDeposit(address(senior), SLP, 3_000_000e6);
        uint256[4] memory jid;
        for (uint256 i; i < 4; ++i) { JLP[i] = address(uint160(0x71B0 + i)); jid[i] = _lpDeposit(address(junior), JLP[i], 1_000_000e6); }
        _toEpoch(address(senior), sid); _toEpoch(address(junior), jid[0]); _markOk(P0);
        (bool ok,) = _settle(); require(ok, "settle lps");
        _claimDep(address(senior), SLP, sid); for (uint256 i; i < 4; ++i) _claimDep(address(junior), JLP[i], jid[i]);
        _adv(3601);
    }
    function _o(address a, uint256 lots, uint256 m, bool close, bool long_, uint256 tau) internal {
        (bool c, bool e, bytes memory w) = _order(a, lots, m, close, long_, tau);
        require(c && e, string.concat("order failed ", vm.toString(abi.encodePacked(_sel(w)))));
    }
    // ------------------------------------------------------------------ phase 1: ordinary traders: path risk realised by two opposite positions
    function _shock() internal returns (Shock memory s) {
        _markOk(P0);
        A = _trader(1, 1_000_000e6); S2 = _trader(2, 100_000_000e6);
        _o(A, 400_000, 300_000e6, false, false, P0); _o(S2, 400_000, 45_000_000e6, false, true, P0);   // skew cliff: alternate sides
        _o(A, 432_000, 300_000e6, false, false, P0); _o(S2, 432_000, 45_000_000e6, false, true, P0);   // A SHORT 832,000 lots, S2 LONG 832,000 lots (about 80M notional each)
        _adv(5); px = P1; _o(A, 832_000, 0, true, false, P1);   // up-leg: SHORT closes in profit, paid from pool cash (latches degraded mode)
        _adv(5); px = P2; _o(S2, 832_000, 0, true, true, P2);   // down-leg: LONG closes in profit above remaining cash: whole payout becomes a claim
        ICfdEngineTypes.TerminalNavSnapshot memory t = engine.terminalNavSnapshot();
        s.P = pool.rawAssets(); s.C = engine.totalTraderClaimBalanceAssets(); s.L = t.maxDirectionalLiabilityAssets; s.delta = t.terminalLpPriceDeltaAssets;
        s.X = pool.getPoolLiquidityView().currentTerminalDeficitAssets; s.Rmin = s.C + s.L - s.P; s.d = s.Rmin - s.X;
    }
    function _assertWipe(Shock memory s) internal view {
        (uint256 sp, uint256 jp,,) = pool.getPendingTrancheState();
        require(s.P < s.C, "claims above cash"); require(sp == 0 && jp == 0, "tranches not wiped"); require(engine.degradedMode(), "no degraded latch");
        require(s.X > 0, "no terminal deficit");
    }
    // ------------------------------------------------------------------ phase 2: owner recapitalization (trusted role, real owner address)
    function _recap(uint256 amount) internal {
        token.mint(OWNER, amount);
        vm.startPrank(OWNER); token.approve(address(engine), amount); engine.recapitalizePool(amount); engine.clearDegradedMode(); vm.stopPrank();
    }
    function _log(Shock memory s) internal pure {
        _res("P cash", _x(s.P)); _res("C claims", _x(s.C)); _res("L max liability", _x(s.L)); _res("terminal LP delta (signed)", vm.toString(s.delta));
        _res("X deficit", _x(s.X)); _res("Rmin = C+L-P", _x(s.Rmin)); _res("d = Rmin - X", _x(s.d));
    }
    function _probe(uint256 sid, uint256 n) internal returns (uint256 fundedAt, uint256 reverts) {
        for (uint256 k; k < n; ++k) {
            _adv(3600); (bool ok, bytes memory e) = _settle();
            if (!ok) { ++reverts; require(_sel(e) == bytes4(0x86cca6b8), "unexpected revert"); }
            if (fundedAt == 0 && senior.claimableRedeemAssets(sid, SLP) != 0) fundedAt = k + 1;
        }
    }

    // ================================================================== tests
    /// control 0: the same senior LP exit before any shock is funded by one settlement
    function test_T0_control_healthyExit() public {
        _lps(); uint256 pre = senior.estimateRedeemAssets(senior.balanceOf(SLP));
        (uint256 sid,) = _requestExit(address(senior), SLP); _adv(1); (bool ok,) = _settle();
        _res("T0 healthy exit settle ok", _b(ok)); _res("T0 estimate before", _x(pre)); _res("T0 funded", _x(senior.claimableRedeemAssets(sid, SLP)));
        require(ok && senior.claimableRedeemAssets(sid, SLP) != 0, "healthy exit not funded");
    }
    /// wipe state through ordinary trader flows (price device: the Pyth mock), verified with the pool's own getters
    function test_T1_wipeState() public {
        _lps(); Shock memory s = _shock(); _log(s); _assertWipe(s); _snap("T1-wiped");
        (uint256 sp, uint256 jp,,) = pool.getPendingTrancheState(); _res("T1 preview senior principal", _x(sp)); _res("T1 preview junior principal", _x(jp));
        _res("T1 canAcceptTrancheDeposits senior", _b(pool.canAcceptTrancheDeposits(true))); _res("T1 canAcceptTrancheDeposits junior", _b(pool.canAcceptTrancheDeposits(false)));
        (uint256 sz,,) = _pos(A); (uint256 sz2,,) = _pos(S2); _res("T1 own positions open", _b(sz != 0 || sz2 != 0));
    }
    /// main: recapitalization R = Rmin + Y (Y <= X), senior exit queued before the first post-recap settlement
    function _main(uint256 y, bool queueExit, uint256 probes, uint256 levers, string memory tag) internal {
        _lps(); Shock memory s = _shock(); _log(s); _assertWipe(s);
        _res(string.concat(tag, " Y"), _x(y)); _res(string.concat(tag, " R = Rmin + Y"), _x(s.Rmin + y));
        _recap(s.Rmin + y); _snap(string.concat(tag, "-recapped"));
        _res(string.concat(tag, " third-party senior deposit refused?"), _b(_depositRefused(address(senior))));
        _res(string.concat(tag, " third-party junior deposit refused?"), _b(_depositRefused(address(junior))));
        uint256 sid; if (queueExit) { uint256 sh; (sid, sh) = _requestExit(address(senior), SLP); _res(string.concat(tag, " SLP exit shares"), _x(sh)); _res(string.concat(tag, " SLP exit estimate"), _x(senior.estimateRedeemAssets(sh))); }
        else { (jid0,) = _requestExit(address(junior), JLP[0]); }
        uint256 hwm0 = pool.seniorHighWaterMark();
        _adv(1); (bool ok1, bytes memory e1) = _settle(); _res(string.concat(tag, " settlement 1 ok"), _b(ok1)); if (!ok1) _res(string.concat(tag, " settlement 1 revert"), vm.toString(abi.encodePacked(_sel(e1))));
        _snap(string.concat(tag, "-settle1"));
        IHousePool.PoolLiquidityView memory v = pool.getPoolLiquidityView();
        oBucket1 = v.pendingRecapitalizationAssets; oStale = oBucket1 != 0; if (queueExit) oFundedAt1 = ok1 && senior.claimableRedeemAssets(sid, SLP) != 0;
        _res(string.concat(tag, " bucket after settle 1"), _x(v.pendingRecapitalizationAssets)); _res(string.concat(tag, " hwm before / after settle 1"), string.concat(_x(hwm0), " / ", _x(v.seniorHighWaterMarkAssets)));
        _res(string.concat(tag, " reserved - (C+L+B)"), _x(v.withdrawalReservedAssets - s.C - s.L - (s.L * engine.settlementBufferBps() + 9_999) / 10_000));
        if (queueExit) {
            (uint256 fundedAt, uint256 rev) = _probe(sid, probes);
            _res(string.concat(tag, " hourly probes after settle 1"), _x(probes)); _res(string.concat(tag, " probes reverting NoLpEpochProgress"), _x(rev)); _res(string.concat(tag, " SLP funded at probe (0=never)"), _x(fundedAt));
            _res(string.concat(tag, " SLP claimable"), _x(senior.claimableRedeemAssets(sid, SLP)));
            oBlocked = (rev == probes) && fundedAt == 0 && !oFundedAt1;
            vm.prank(SLP); try senior.cancelRedeemRequest(sid, SLP, SLP) { _res(string.concat(tag, " cancel"), "ok"); } catch (bytes memory e) { _res(string.concat(tag, " cancel reverts"), vm.toString(abi.encodePacked(_sel(e)))); }
            _snap(string.concat(tag, "-after-probes"));
            for (uint256 L_; L_ < levers; ++L_) _lever(sid, L_ + 1, tag);
        } else {
            // manifestation 1: later settlements replay the wipe; junior zero-NAV exits are the progress items
            uint256 hwmPrev = pool.seniorHighWaterMark();
            if (junior.redeemRefundPending(jid0, JLP[0])) { vm.prank(JLP[0]); junior.claimRedeemRefund(jid0, JLP[0], JLP[0]); }
            for (uint256 k; k < 4; ++k) {
                _adv(3600); (uint256 id,) = _requestExit(address(junior), JLP[(k + 1) % 4]); _adv(1); (bool ok,) = _settle();
                _snap(string.concat(tag, "-settle", _x(k + 2), ok ? "" : "-REV"));
                if (pool.seniorHighWaterMark() < hwmPrev) oHwmFell = true;
                hwmPrev = pool.seniorHighWaterMark();
                if (junior.redeemRefundPending(id, JLP[(k + 1) % 4])) { vm.prank(JLP[(k + 1) % 4]); junior.claimRedeemRefund(id, JLP[(k + 1) % 4], JLP[(k + 1) % 4]); }
            }
        }
        _res(string.concat(tag, " OUTCOME stale bucket after settle 1"), _b(oStale)); _res(string.concat(tag, " OUTCOME exit blocked over probes"), _b(oBlocked));
        _res(string.concat(tag, " OUTCOME exit funded at settlement 1"), _b(oFundedAt1)); _res(string.concat(tag, " OUTCOME HWM fell without a loss"), _b(oHwmFell));
    }
    function _depositRefused(address vault) internal returns (bool refused) {
        address t = address(0x7E57); _fund(t, 1_000e6); vm.deal(t, 1 ether);
        vm.prank(t); try TrancheVault(vault).requestDeposit(1_000e6, t, t) returns (uint256) { return false; } catch (bytes memory e) { console2.log(string.concat("RES|deposit refusal selector|", vm.toString(abi.encodePacked(_sel(e))))); return true; }
    }
    function _lever(uint256 sid, uint256 n, string memory tag) internal {
        { IHousePool.PoolConfig memory cfg = _cfgBump(); vm.prank(OWNER); pool.proposePoolConfig(cfg); }
        vm.warp(pool.poolConfigActivationTime()); vm.roll(vm.getBlockNumber() + 100);
        while (engine.isFadWindow() || engine.isOracleFrozen()) { vm.warp(_now() + 1 hours); vm.roll(vm.getBlockNumber() + 10); } // the owner waits for the weekly market open
        _res(string.concat(tag, " lever ", _x(n), " finalize time (unix)"), _x(_now()));
        vm.prank(OWNER); try pool.finalizePoolConfig() { _res(string.concat(tag, " lever finalize without fresh mark"), "ok"); return; } catch (bytes memory e) { _res(string.concat(tag, " lever ", _x(n), " finalize w/o fresh mark reverts"), vm.toString(abi.encodePacked(_sel(e)))); }
        _markOk(px);
        vm.prank(OWNER); pool.finalizePoolConfig();
        _snap(string.concat(tag, "-lever", _x(n)));
        (uint256 fundedAt,) = _probe(sid, 1); _res(string.concat(tag, " lever ", _x(n), " SLP funded at next probe"), _x(fundedAt == 0 ? 0 : 1));
    }

    function test_T2_main_blockedSeniorExit() public {
        _main(10_000_000e6, true, 30, 2, "T2");
        if (expectMode == 1) { require(!oStale && !oBlocked && oFundedAt1, "fixa: expected bucket cleared and exit funded at settlement 1"); }
        else { require(oStale && oBlocked, "frozen/fixh: expected stale bucket and blocked exit"); }
    }
    function test_T5_leversUntilFunded() public {
        _main(10_000_000e6, true, 30, 5, "T5");
    }
    /// sweep of the second cash amount Y around X (snapshot after the shock; each Y on the same state)
    function test_T6_sweepY() public {
        _lps(); Shock memory s = _shock(); _assertWipe(s); _res("T6 X", _x(s.X)); _res("T6 buffer B", _x((s.L * engine.settlementBufferBps() + 9_999) / 10_000));
        uint256[9] memory ys = [uint256(1_000_000e6), 10_000_000e6, 30_000_000e6, s.X - 1, s.X, s.X + 1, s.X + 29_000e6, s.X + 31_000e6, s.X + 1_000_000e6];
        uint256 snap = vm.snapshotState(); uint256 savedTick = lastTick; uint256 savedPx = px;
        for (uint256 i; i < ys.length; ++i) {
            vm.revertToState(snap); lastTick = savedTick; px = savedPx;
            _recap(s.Rmin + ys[i]);
            (uint256 sid,) = _requestExit(address(senior), SLP); _adv(1);
            (bool ok, bytes memory e) = _settle(); ok; e;
            _res(string.concat("T6 Y=", _x(ys[i]), " settle1 ok"), _b(ok)); _res(string.concat("T6 Y=", _x(ys[i]), " SLP funded"), _x(senior.claimableRedeemAssets(sid, SLP)));
            _res(string.concat("T6 Y=", _x(ys[i]), " Y-X (signed)"), vm.toString(int256(ys[i]) - int256(s.X)));
        }
    }
    /// the exit is queued only AFTER two ordinary drain settlements: it remains blocked for 24 hourly attempts, then a further 20M owner recapitalization releases it
    function _drainThenExit(uint256 pre, string memory tag) internal returns (uint256 sid) {
        _lps(); Shock memory s = _shock(); _assertWipe(s); _recap(s.Rmin + 10_000_000e6);
        for (uint256 k; k < pre; ++k) {
            (uint256 id,) = _requestExit(address(junior), JLP[k]); _adv(1); (bool ok,) = _settle(); require(ok, "progress settle");
            if (junior.redeemRefundPending(id, JLP[k])) { vm.prank(JLP[k]); junior.claimRedeemRefund(id, JLP[k], JLP[k]); }
            _adv(3600);
        }
        _snap(string.concat(tag, "-before-exit"));
        uint256 sh; (sid, sh) = _requestExit(address(senior), SLP); sh;
    }
    function test_T7_exitQueuedMidDrain_andSecondRecap() public {
        uint256 sid = _drainThenExit(2, "T7");
        (uint256 fundedAt, uint256 rev) = _probe(sid, 24);
        _res("T7 probes reverting after mid-drain exit (of 24)", _x(rev)); _res("T7 funded at probe (0=never)", _x(fundedAt)); _snap("T7-after-24-probes");
        oBlocked = rev == 24 && fundedAt == 0;
        // owner adds more cash: every recapitalization queues its whole amount, so free cash does not change
        token.mint(OWNER, 20_000_000e6); vm.startPrank(OWNER); token.approve(address(engine), 20_000_000e6); engine.recapitalizePool(20_000_000e6); vm.stopPrank();
        _snap("T7-after-second-owner-recap");
        (uint256 f2, uint256 r2) = _probe(sid, 3); _res("T7 after a further 20M owner recap: probes reverting (of 3)", _x(r2)); _res("T7 funded after 20M recap at probe (0=never)", _x(f2));
        if (expectMode == 1) require(!oBlocked, "fixa: exit should be funded"); else require(oBlocked, "frozen/fixh: exit should be blocked");
    }
    function test_T3_hwmReplay() public {
        _main(10_000_000e6, false, 0, 0, "T3");
        if (expectMode == 0) { require(oStale && oHwmFell, "frozen: expected stale bucket and HWM replay"); }
        else if (expectMode == 2) { require(oStale && !oHwmFell, "fixh: expected stale bucket, HWM held"); }
        else { require(!oStale && !oHwmFell, "fixa: expected no bucket and HWM held"); }
    }
    function test_T4_control_bigY() public { // Y > X + B: enough free cash remains to fund the exit; X remains pending in frozen/fixH
        _main(60_000_000e6, true, 6, 0, "T4");
        require(!oBlocked && oFundedAt1, "control: exit must be funded at settlement 1");
    }
}
```

`poc/fixA_writeoff.diff` (used only in step 5)
```diff
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -1595,7 +1595,7 @@
             }
         }
 
-        _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
+        _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation, markFresh);
     }
 
     function _getWithdrawalSnapshot()
@@ -1759,8 +1759,20 @@
     function _applyPendingClaimantBucketsLive(
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets,
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets,
-        bool allowRevenueContinuation
+        bool allowRevenueContinuation,
+        bool markFresh
     ) internal {
+        if (markFresh) {
+            // A claimant bucket larger than the equity it can settle has no cash
+            // behind the excess; the excess filled a terminal deficit or trader claims. Write it off so it cannot be
+            // reserved, block entry, or replay a wipe-and-bootstrap at later reconciles.
+            if (settleableClaimantBuckets.recapitalizationAssets < claimantBuckets.recapitalizationAssets) {
+                pendingRecapitalizationAssets = settleableClaimantBuckets.recapitalizationAssets;
+            }
+            if (settleableClaimantBuckets.revenueAssets < claimantBuckets.revenueAssets) {
+                pendingTradingRevenueAssets = settleableClaimantBuckets.revenueAssets;
+            }
+        }
         if (_pendingClaimantBucketAssets(settleableClaimantBuckets) == 0) {
             return;
         }
```

`poc/fixH_intent_mark.diff` (used only in step 5)
```diff
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -195,6 +195,9 @@
     PoolConfig public pendingPoolConfig;
     /// @notice Earliest Unix timestamp at which the pending configuration may be finalized, or zero if none.
     uint256 public poolConfigActivationTime;
+    // Appended after the last existing state variable so the deployed storage layout is preserved.
+    /// @notice Recapitalization recorded since the pending bucket was last empty; the bootstrap mark target (intent-as-mark-target variant).
+    uint256 public recapitalizationIntentAssets;
 
     modifier onlyPauserOrOwner() {
         if (msg.sender != owner() && msg.sender != pauser) {
@@ -1670,7 +1673,7 @@
         }
 
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-            pendingState, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+            pendingState, settleableClaimantBuckets, _recapitalizationIntentBuckets(), allowRevenueContinuation
         );
     }
 
@@ -1767,7 +1770,7 @@
 
         HousePoolPendingPreviewLib.PendingAccountingState memory state = _currentPendingAccountingState();
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-            state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+            state, settleableClaimantBuckets, _recapitalizationIntentBuckets(), allowRevenueContinuation
         );
         _decreasePendingClaimantBuckets(settleableClaimantBuckets);
 
@@ -1784,11 +1787,23 @@
         buckets.revenueAssets = pendingTradingRevenueAssets;
     }
 
+    /// @dev The bootstrap mark target is the recorded intent, not the residual still outstanding.
+    function _recapitalizationIntentBuckets()
+        internal
+        view
+        returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory buckets)
+    {
+        buckets.recapitalizationAssets = recapitalizationIntentAssets;
+    }
+
     function _decreasePendingClaimantBuckets(
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
     ) internal {
         pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
         pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
+        if (pendingRecapitalizationAssets == 0) {
+            recapitalizationIntentAssets = 0;
+        }
     }
 
     function _recordPendingClaimantInflow(
@@ -1797,6 +1812,7 @@
     ) internal {
         if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
             pendingRecapitalizationAssets += amount;
+            recapitalizationIntentAssets += amount;
         } else {
             pendingTradingRevenueAssets += amount;
         }
```

### Step 3. Run (vulnerable code)
```bash
chmod +x poc/run_fork.sh
poc/run_fork.sh ../plether-frozen frozen http://127.0.0.1:8545
```
Eight tests pass; each test asserts the vulnerable outcome for this build (`M175_EXPECT=frozen`), so a pass means the defect reproduced.

### Step 4. Expected output (vulnerable build; lines that prove the bug, all USDG with six decimals as integers)
```
RES|P cash|43428135992567
RES|C claims|75731545598389
RES|L max liability|11878653855744
RES|X deficit|43513122755958
RES|Rmin = C+L-P|44182063461566
RES|T1 preview senior principal|0
RES|T1 preview junior principal|0
RES|T2 R = Rmin + Y|54182063461566
SNAP|T2-recapped|rawAssets=97610199454133|claims=75731545598389|sr=14114376418228|jr=106002904195891|hwm=14114376418228|bucket=54182063461566|reserved=141821959550339|free=0|deficit=0|degraded=0|markFresh=1|mark=5000000
RES|T2 third-party senior deposit refused?|true
RES|T2 third-party junior deposit refused?|true
RES|T2 SLP exit shares|2987848707829685
RES|T2 settlement 1 ok|false
RES|T2 settlement 1 revert|0x86cca6b8
RES|T2 bucket after settle 1|54182063461566
RES|T2 reserved - (C+L+B)|54182063461566
RES|T2 hourly probes after settle 1|30
RES|T2 probes reverting NoLpEpochProgress|30
RES|T2 SLP funded at probe (0=never)|0
RES|T2 cancel reverts|0x3260a8c1
RES|T2 lever 1 finalize time (unix)|1795161601
RES|T2 lever 1 finalize w/o fresh mark reverts|0x40497d8e
RES|T2 OUTCOME stale bucket after settle 1|true
RES|T2 OUTCOME exit blocked over probes|true
RES|T2 OUTCOME exit funded at settlement 1|false
RES|T2 OUTCOME HWM fell without a loss|false
SNAP|T3-settle1|rawAssets=97610199454133|claims=75731545598389|sr=10668940705608|jr=0|hwm=54182063461566|bucket=43513122755958|reserved=131153018844731|free=0|deficit=0|degraded=0|markFresh=1|mark=5000000
RES|T3 bucket after settle 1|43513122755958
RES|T3 hwm before / after settle 1|14114376418228 / 54182063461566
SNAP|T3-settle2|rawAssets=97610199454133|claims=75731545598389|sr=10668940705608|jr=0|hwm=43513122755958|bucket=32844182050350|reserved=120484078139123|free=0|deficit=0|degraded=0|markFresh=1|mark=5000000
SNAP|T3-settle3|rawAssets=97610199454133|claims=75731545598389|sr=10668940705608|jr=0|hwm=32844182050350|bucket=22175241344742|reserved=109815137433515|free=0|deficit=0|degraded=0|markFresh=1|mark=5000000
SNAP|T3-settle4|rawAssets=97610199454133|claims=75731545598389|sr=10668940705608|jr=0|hwm=22175241344742|bucket=11506300639134|reserved=99146196727907|free=0|deficit=0|degraded=0|markFresh=1|mark=5000000
SNAP|T3-settle5|rawAssets=97610199454133|claims=75731545598389|sr=10668940705608|jr=0|hwm=11506300639134|bucket=837359933526|reserved=88477256022299|free=9132943431834|deficit=0|degraded=0|markFresh=1|mark=5000000
RES|T3 OUTCOME stale bucket after settle 1|true
RES|T3 OUTCOME exit blocked over probes|false
RES|T3 OUTCOME exit funded at settlement 1|false
RES|T3 OUTCOME HWM fell without a loss|true
RES|T4 SLP funded at probe (0=never)|1
RES|T4 SLP claimable|12895137321246
RES|T5 lever 1 finalize time (unix)|1795161601
RES|T5 lever 1 SLP funded at next probe|0
RES|T5 lever 2 finalize time (unix)|1795424401
RES|T5 lever 2 SLP funded at next probe|0
RES|T5 lever 3 finalize time (unix)|1795687201
RES|T5 lever 3 SLP funded at next probe|0
RES|T5 lever 4 finalize time (unix)|1795993201
RES|T5 lever 4 SLP funded at next probe|1
RES|T5 lever 5 finalize time (unix)|1796256001
SNAP|T5-lever5|rawAssets=95342524201014|claims=75731545598389|sr=8401265452489|jr=0|hwm=9060645167397|bucket=0|reserved=87639896088773|free=7702628112241|deficit=0|degraded=0|markFresh=1|mark=5000000
RES|T5 lever 5 SLP funded at next probe|1
RES|T6 buffer B|29696634640
RES|T6 Y=43513122755957 settle1 ok|false
RES|T6 Y=43513122755957 SLP funded|0
RES|T6 Y=43513122755957 Y-X (signed)|-1
RES|T6 Y=43513122755958 settle1 ok|false
RES|T6 Y=43513122755958 SLP funded|0
RES|T6 Y=43513122755958 Y-X (signed)|0
RES|T6 Y=43513122755959 settle1 ok|false
RES|T6 Y=43513122755959 SLP funded|0
RES|T6 Y=43513122755959 Y-X (signed)|1
RES|T6 Y=43542122755958 settle1 ok|false
RES|T6 Y=43542122755958 SLP funded|0
RES|T6 Y=43542122755958 Y-X (signed)|29000000000
RES|T6 Y=43544122755958 settle1 ok|true
RES|T6 Y=43544122755958 SLP funded|1303365360
RES|T6 Y=43544122755958 Y-X (signed)|31000000000
RES|T7 probes reverting after mid-drain exit (of 24)|24
RES|T7 funded at probe (0=never)|0
RES|T7 funded after 20M recap at probe (0=never)|1
```
Reading the lines: `T1 preview ... principal 0` and `X deficit`: both tranches are wiped with claims `C` above cash `P`. `T2 R = Rmin + Y` is the owner's recapitalization (54,182,063.461566 USDG). `T2-recapped`: `reserved` (141,821,959.550339) exceeds `rawAssets` (97,610,199.454133) because the whole `bucket` (54,182,063.461566) is reserved, so `free` is 0. `T2 settlement 1 ok|false` with selector 0x86cca6b8 (`HousePool__NoLpEpochProgress`) and 30 of 30 hourly settlements reverting, `T2 SLP funded at probe|0`, `T2 cancel reverts|0x3260a8c1` (`RedeemCancellationUnavailable`): the senior LP's exit is blocked. `T5` lines: the exit is funded only after owner lever 4 (finalize times are Unix seconds: Fri 2026-11-20 08:00Z, Mon 11-23 09:00Z, Thu 11-26 10:00Z, Sun 11-29 23:00Z). `T3-settle1..5`: the senior high-water mark falls by 10,668,940.705608 at every settlement while senior principal stays 10,668,940.705608 and no loss occurs. `T6`: the first settlement reverts at Y = X + 29,000 USDG and succeeds at Y = X + 31,000 (B = 29,696.634640). `T7`: an exit queued after two ordinary drain settlements is blocked 24 of 24 hourly settlements.

### Step 5. Fix and control checks
Patch A (write off the unbacked remainder at the first fresh reconcile):
```bash
poc/run_fork.sh ../plether-fixa fixa http://127.0.0.1:8545
```
Eight tests pass with the expectation flipped (`M175_EXPECT=fixa`); the outcome lines are:
```
RES|etched HousePool runtime bytes 24633
RES|T2 settlement 1 ok|true
RES|T2 bucket after settle 1|0
RES|T2 SLP funded at probe (0=never)|1
RES|T2 SLP claimable|2267675253119
RES|T3 bucket after settle 1|0
RES|T3 hwm before / after settle 1|14114376418228 / 54182063461566
RES|T7 probes reverting after mid-drain exit (of 24)|23
RES|T2 OUTCOME stale bucket after settle 1|false
RES|T2 OUTCOME exit blocked over probes|false
RES|T2 OUTCOME exit funded at settlement 1|true
RES|T2 OUTCOME HWM fell without a loss|false
RES|T3 OUTCOME stale bucket after settle 1|false
RES|T3 OUTCOME exit blocked over probes|false
RES|T3 OUTCOME exit funded at settlement 1|false
RES|T3 OUTCOME HWM fell without a loss|false
RES|T7 funded after 20M recap at probe (0=never)|1
```
The same exit is funded at the first settlement (2,267,675.253119 USDG claimable), the stale remainder is 0, and the mark is never replayed (`T3 OUTCOME HWM fell without a loss|false`).

Mark-target variant (stores the recapitalization intent and uses it as the bootstrap mark target; `poc/fixH_intent_mark.diff`):
```bash
poc/run_fork.sh ../plether-fixh fixh http://127.0.0.1:8545
```
Eight tests pass with the expectation `M175_EXPECT=fixh`; the outcome lines are:
```
RES|etched HousePool runtime bytes 24675
RES|T2 settlement 1 ok|false
RES|T2 bucket after settle 1|54182063461566
RES|T2 probes reverting NoLpEpochProgress|30
RES|T2 SLP funded at probe (0=never)|0
RES|T3 bucket after settle 1|43513122755958
RES|T3 hwm before / after settle 1|14114376418228 / 54182063461566
RES|T7 probes reverting after mid-drain exit (of 24)|24
RES|T2 OUTCOME stale bucket after settle 1|true
RES|T2 OUTCOME exit blocked over probes|true
RES|T3 OUTCOME stale bucket after settle 1|true
RES|T3 OUTCOME HWM fell without a loss|false
RES|T7 funded after 20M recap at probe (0=never)|0
```
The mark replay stops but the remainder stays queued and the senior exit stays blocked (30 of 30 reverts), so this variant does not remove the defect.

Controls inside the vulnerable build: `T0` (the same LP's healthy exit before the shock is funded in one settlement), `T4` (with Y = 60,000,000, above X + B, the exit is funded at settlement 1 and the remainder is still stale), `T6` (the sweep gives the exact boundary).

