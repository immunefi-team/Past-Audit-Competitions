# 88741 sc high c 1 firelightvault rescuewithdrawfromblocklisted permanently poisons a still open withdrawal bucket freezing a staker s funds with no recovery path

**Submitted on Aug 14th 2026 at 19:28:30 UTC by @web3pavlou for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88741
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

**Severity selected:** Critical — *Permanent freezing of funds*\
**Asset:** `contracts/core/FirelightVault.sol`\
**Requires an attacker:** No. Every actor in the sequence performs its documented function correctly.

***

## Summary

`rescueWithdrawFromBlocklisted` uses `isWithdrawClaimed[period][from] = true` (`FirelightVault.sol:870`) as a bookkeeping marker meaning "these shares were moved away." But that mapping is not a marker — it is the hard claim gate in `claimWithdraw` (`:794`), it is never reset anywhere in the contract, and `periods[i]` is **unbounded**, so the still-open bucket `currentPeriod() + 1` can be flagged.

A staker rescued out of the open bucket, then un-blocklisted after a false-positive compliance flag, who makes a fresh withdrawal request in the same period lands back in that poisoned bucket. Their shares are burned, their assets are moved into `pendingWithdrawAssets`, and `claimWithdraw` rejects them permanently.

The result is worse than a loss to a counterparty: the funds become **unreachable by anybody**. In the PoC the vault ends holding **600,000 FXRP** while reporting `totalAssets() == 0` and `totalSupply() == 0`. The tokens are present, owned by no one, and claimable by no one.

The protocol documents the opposite guarantee:

> *"Authorized roles can rescue assets from blocklisted accounts and reassign them to a non-blocklisted address, **preserving the position** without violating compliance constraints."*\
> — Firelight docs, *Deployments and Withdrawals*

***

## Scope: this is not a privileged-actor report

The program excludes reports that require *"a privileged role holder to act within the permissions explicitly granted to that role... **unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint**."* The project restated the test in its competition Q\&A:

> *"If a trusted role operates strictly within its designated functions, that is expected behavior and not a vulnerability. However, **a bug may be valid if the role can bypass its documented boundaries or violate a system invariant** without needing to gain additional roles."*

This report satisfies that exception on its own terms, and I want to be precise about why:

**Nobody acts maliciously or incorrectly.** `BLOCKLIST_ROLE` flags an account, `RESCUER_ROLE` moves the pending withdrawal, `BLOCKLIST_ROLE` un-flags on review. Every call is the intended use of the function. There is no adversary in this report — the roles are the mechanism, not the attacker — and the victim is an ordinary staker whose only action is a second withdrawal request.

**The documented invariant is violated.** The protocol states that the rescue path reassigns a blocklisted account's assets *"**preserving the position** without violating compliance constraints."* The PoC shows the position is not preserved: it is destroyed. That is the invariant violation the exception is written for.

**No additional roles are needed.** The permanent write at `FirelightVault.sol:870` lands inside the rescue's ordinary execution. The rescuer cannot avoid it, cannot detect it, and no role in the system can undo it.

The defect is that correct, intended use of a compliance function permanently destroys a user's position and contradicts the protocol's own stated guarantee. A guard that malfunctions when used properly is a code defect, not a privilege question.

***

## Vulnerability details

### The flag is a claim gate, not a marker

`isWithdrawClaimed` appears in exactly five places in the entire contract:

| Line  | Use                                                                                                              |
| ----- | ---------------------------------------------------------------------------------------------------------------- |
| `794` | `claimWithdraw`—**the hard gate**: `if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);` |
| `806` | `claimWithdraw` — set after a genuine claim                                                                      |
| `864` | `rescueWithdrawFromBlocklisted` — guard on `from`                                                                |
| `865` | `rescueWithdrawFromBlocklisted` — guard on `to`                                                                  |
| `870` | `rescueWithdrawFromBlocklisted` — **set as a "moved away" marker**                                               |

There is **no reset**. `removeFromBlocklist` does not touch it. No admin function does. Line 870 writes a one-way, permanent value into the mapping that line 794 uses to decide whether a user may ever be paid.

### `periods[i]` is unbounded

```solidity
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;   // <-- permanent, on an arbitrary period
    ...
}
```

Nothing constrains `periods[i]` to a settled bucket. Since `_requestWithdraw` always credits `currentPeriod() + 1`, and a rescue must target a bucket that actually holds a balance (`_withdrawOf == 0` reverts), the natural target for an in-flight rescue **is** the open bucket. Flagging it is not an edge case — it is the ordinary path.

### The flag is also unnecessary

Zeroing `withdrawSharesOf[periods[i]][from]` on line 869 already prevents a later claim: `claimWithdraw` computes `assets` from that mapping and reverts `NoWithdrawalAmount(period)` when it is zero. Line 870 protects nothing. It exists only to poison the slot.

***

## Attack path

No attacker. Each step is a legitimate action by the party authorised to take it.

{% stepper %}
{% step %}

## Period P — Alice requests a partial withdrawal

Alice deposits 1,000,000 FXRP and later requests withdrawal of 400,000. The credit lands in the open bucket `P+1`.
{% endstep %}

{% step %}

## Period P — Compliance flags Alice

`BLOCKLIST_ROLE` calls `addToBlocklist(alice)`.

*(Legitimate.)*
{% endstep %}

{% step %}

## Period P — Rescue moves the pending withdrawal

`RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, treasury, [P+1])`. Alice's credit moves to the treasury and `isWithdrawClaimed[P+1][alice]` is set to `true`.

*(Legitimate — this is precisely what the function is for.)*
{% endstep %}

{% step %}

## Period P — Compliance removes the false-positive flag

`BLOCKLIST_ROLE` calls `removeFromBlocklist(alice)`.

*(Legitimate.)*
{% endstep %}

{% step %}

## Period P — Alice withdraws the remaining position

Alice withdraws the remainder of her position — 600,000 FXRP. Her request is credited to bucket `P+1` again, and her shares are burned.
{% endstep %}

{% step %}

## Period P+2 — Treasury claims, Alice is permanently rejected

The treasury claims its rescued 400,000 successfully. Alice's `claimWithdraw(P+1)` reverts `AlreadyClaimedPeriod(P+1)`, and will revert forever.
{% endstep %}
{% endstepper %}

> **Timing.** Steps 3–5 must fall inside one period for the second request to land in the flagged bucket. Under today's **1-day** periods that is a same-day cycle; at the **30-day** periods the docs commit to for the feature-complete launch it is a month-wide window — and a compliance false positive resolved within a month is entirely ordinary. The PoC runs on the project's own `DEFAULT_CONFIG` duration of one week rather than the deployed one day: the requirement is only that the three steps share a period, so the defect is duration-independent and scales with whatever `periodConfigurationDuration` is configured.

***

## Impact

**Permanent freezing of funds.** Alice loses 600,000 FXRP — 60% of her stake — with no recovery path in the contract. Her shares are burned, so she holds no ERC-4626 claim, and her bucket credit is permanently unclaimable.

Critically, the funds are not merely transferred to the wrong party. The final state from the PoC:

| Quantity                  | Value                              |
| ------------------------- | ---------------------------------- |
| `token.balanceOf(vault)`  | **600,000,000,000** (600,000 FXRP) |
| `pendingWithdrawAssets()` | 600,000,000,000                    |
| `totalAssets()`           | **0**                              |
| `totalSupply()`           | **0**                              |

Because `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets`, the stranded amount is excluded from the vault's own accounting. Alice cannot claim it (the flag). No other staker can reach it (there are no shares and no other credits). It is orphaned inside the contract.

A second, lesser consequence of the same line: once `to` claims a period, `isWithdrawClaimed[period][to]` is `true`, and line 865 makes that address permanently unusable as a rescue destination for that period.

***

## Severity: permanence

`isWithdrawClaimed` has no reset anywhere in the contract, and a second rescue of the same account cannot clear it — line 864 reverts `AlreadyClaimedPeriod` on `from`. There is no recovery path in the deployed code.

One pre-emption, stated plainly: `FirelightVault` sits behind a transparent proxy, so in principle an upgrade could add a reset. I do not think that reduces the severity. The funds are unrecoverable by every function the contract exposes; recovery would require authoring, auditing and shipping new logic through governance, with the affected balance stranded meanwhile. If triage nonetheless treats an upgrade as a recovery path, the impact is *Temporary freezing of funds for at least 24 hours* (High) rather than Critical — but it is one of the two, not less.

I have also considered and rejected *Griefing*: griefing implies a deliberate actor accepting a cost to damage someone. Nobody chooses this outcome and nobody gains from it — the funds are destroyed, not captured.

## Prior coverage

Macro A-1's TMAAR describes the rescue *power* (*"can blocklist any account and move its shares and pending withdrawals to an address of choice… a targeted confiscation power"*) but does not identify this defect. It appears in none of the three Known Issues, none of the design notes on the program's scope page, and none of the 46 disclosed reports from the November 2025 competition. The single rescue-adjacent item there was Insight **#59288** — *"Repeated Array Access in `rescueWithdrawFromBlocklisted` Loop Causes Unnecessary Gas Consumption."* Someone read this exact loop and saw only the array indexing.

## Recommended mitigation

### Option A — remove the redundant flag (preferred, one line)

Delete line 870:

```diff
             withdrawSharesOf[periods[i]][to] += _withdrawOf;
             withdrawSharesOf[periods[i]][from] = 0;
-            isWithdrawClaimed[periods[i]][from] = true;
```

Zeroing the credit on the preceding line already causes any later `claimWithdraw` by `from` to revert `NoWithdrawalAmount`, so the flag protects nothing. Removing it restores the documented behaviour: a rescued staker who is later un-blocklisted can withdraw normally. The guards on lines 864–865 should remain — they correctly prevent rescuing a bucket that was genuinely claimed.

### Option B — bound the rescue to settled buckets

If the flag is retained for auditability, constrain the target so an open bucket can never be poisoned:

```solidity
if (periods[i] >= currentPeriod()) revert InvalidPeriod();
```

This mirrors the constraint `claimWithdraw` already applies at line 791 and makes the two functions consistent about which buckets are final.

***

## References

* `contracts/core/FirelightVault.sol:845-876` — `rescueWithdrawFromBlocklisted`
* `contracts/core/FirelightVault.sol:870` — the permanent flag write (root cause)
* `contracts/core/FirelightVault.sol:869` — the credit zeroing that already prevents a claim
* `contracts/core/FirelightVault.sol:788-810` — `claimWithdraw`; gate at `:794`
* `contracts/core/FirelightVault.sol:428` — `totalAssets()` excludes `pendingWithdrawAssets`
* `contracts/core/FirelightVault.sol:169` — `error AlreadyClaimedPeriod(uint256 period)`
* Firelight docs, *Deployments and Withdrawals* — "preserving the position"

***

## Proof of Concept

Foundry. Uses the shared `Base.t.sol` fixture, a Solidity port of the project's own `test/setup/fixtures.js`. This PoC calls only `deployVault()`; the allocator half of the fixture is unused here and included solely so the file matches the one used across my other submissions.

**`test/Foundry/Base.t.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "contracts/core/FirelightVault.sol";
import {CoverOrderAllocator} from "contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "contracts/core/CoverNFT.sol";
import {VaultRewardDistributor} from "contracts/core/VaultRewardDistributor.sol";
import {ICoverOrderAllocator} from "contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "contracts/core/interfaces/IAggregatorV3.sol";

import {MockFAsset} from "./mocks/MockFAsset.sol";
import {MockERC20} from "contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "contracts/test/MockAggregatorV3.sol";
import {MockCoverOrderAllocatorVault} from "contracts/test/MockCoverOrderAllocatorVault.sol";

/**
 * @title Base
 * @notice Solidity port of test/setup/fixtures.js — `deployVault` and
 *         `deployCoverOrderAllocator` — so the Hardhat suite and the Foundry suite
 *         exercise identical deployments.
 *
 * Two deliberate deviations from the JS, both noted where they matter:
 *
 *  1. Proxies are ERC1967Proxy rather than the Transparent proxies `upgrades.deployProxy`
 *     creates. Storage layout and delegatecall semantics are identical, and it removes the
 *     ProxyAdmin indirection that would otherwise make every `vm.prank` two-hop. 
 *
 *  2. The underlying is `MockFAsset`, not the real `deployFAsset` helper — contracts/fasset
 *     is pragma 0.8.23 and skipped in foundry.toml. MockFAsset reproduces
 *     `FAsset.transferFrom` line for line, including the round-UP fee, and defaults to a
 *     zero fee so it behaves exactly like a plain ERC20 until a test opts in.
 */
abstract contract Base is Test {
    // --- config (mirrors DEFAULT_CONFIG) ---
    uint8 internal constant DECIMALS = 6;
    string internal constant UNDERLYING = "fXRP";
    string internal constant LST = "stfXRP";
    uint256 internal constant INITIAL_DEPOSIT_LIMIT = 50_000_000_000; // 50k tokens
    uint48 internal constant PERIOD_DURATION = 604800; // 1 week

    // --- constants (mirrors the JS constants block) ---
    uint256 internal constant BPS = 10_000;
    uint256 internal constant LEV = 10_000;
    uint256 internal constant SECS_PER_YEAR = 365 days;
    string internal constant PROTOCOL_MORPHO = "Morpho";
    string internal constant PROTOCOL_AAVE = "Aave";
    bytes32 internal constant MARKET_A = bytes32("market-A");
    bytes32 internal constant MARKET_B = bytes32("market-B");

    // --- vault fixture ---
    MockFAsset internal token;
    FirelightVault internal vault;
    address internal deployer;
    address internal rescuer;
    address internal blocklister;
    address internal pauser;
    address internal limitUpdater;
    address internal periodConfigurationUpdater;
    address internal payoutSigner;
    address internal payoutAllowlister;
    address internal payoutReceiver;
    address internal incidentRole;
    address internal checkpointer;
    address[3] internal users;
    VaultRewardDistributor internal distributor;
    address internal distributorEOA;

    // --- allocator fixture ---
    CoverOrderAllocator internal allocator;
    CoverNFT internal coverNFT;
    MockCoverOrderAllocatorVault internal mockVault;
    MockAggregatorV3 internal priceFeed;
    MockERC20 internal usdc;
    MockERC20 internal usdt;
    address internal admin;
    address internal curator;
    address internal allocatorRole;
    address internal configAdmin;
    address internal adminRole;
    address internal premiumCollector;
    address internal firstLossBufferWallet;
    address internal buyer1;
    address internal buyer2;
    address internal beneficiary;
    bytes32 internal marketIdA;
    bytes32 internal marketIdB;
    bytes32 internal concHashMorpho;
    bytes32 internal concHashAave;

    // =====================================================================
    // deployVault  (port of fixtures.js::deployVault)
    // =====================================================================
    function deployVault() internal {
        // periodConfigurationAtTimestamp() rejects a zero epoch, and Foundry starts at
        // block.timestamp == 1. Warp to a realistic clock before deploying.
        if (block.timestamp < 1_700_000_000) vm.warp(1_700_000_000);

        deployer = _signer("deployer");
        rescuer = _signer("rescuer");
        blocklister = _signer("blocklister");
        pauser = _signer("pauser");
        limitUpdater = _signer("limitUpdater");
        periodConfigurationUpdater = _signer("periodConfigurationUpdater");
        users[0] = _signer("user1");
        users[1] = _signer("user2");
        users[2] = _signer("user3");

        token = new MockFAsset(UNDERLYING, UNDERLYING, DECIMALS);

        FirelightVault.InitParams memory p = FirelightVault.InitParams({
            defaultAdmin: deployer,
            limitUpdater: limitUpdater,
            blocklister: blocklister,
            pauser: pauser,
            periodConfigurationUpdater: periodConfigurationUpdater,
            rescuer: rescuer,
            depositLimit: INITIAL_DEPOSIT_LIMIT,
            periodConfigurationDuration: PERIOD_DURATION
        });

        FirelightVault impl = new FirelightVault();
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(impl), abi.encodeCall(FirelightVault.initialize, (IERC20(address(token)), LST, LST, abi.encode(p)))
        );
        vault = FirelightVault(address(proxy));

        // Payout wiring (mirrors the JS block after deployProxy)
        payoutSigner = _signer("payoutSigner");
        payoutAllowlister = _signer("payoutAllowlister");
        payoutReceiver = _signer("payoutReceiver");
        incidentRole = _signer("incident");
        checkpointer = _signer("checkpointer");

        vm.startPrank(deployer);
        vault.grantRole(vault.PAYOUT_ROLE(), payoutSigner);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), payoutAllowlister);
        vault.grantRole(vault.INCIDENT_ROLE(), incidentRole);
        vault.grantRole(vault.CHECKPOINT_ROLE(), checkpointer);
        vm.stopPrank();

        vm.prank(payoutAllowlister);
        vault.addToPayoutAllowlist(payoutReceiver);
    }

    // =====================================================================
    // deployCoverOrderAllocator  (port of fixtures.js::deployCoverOrderAllocator)
    // =====================================================================
    function deployCoverOrderAllocator() internal {
        if (block.timestamp < 1_700_000_000) vm.warp(1_700_000_000);

        admin = _signer("admin");
        curator = _signer("curator");
        allocatorRole = _signer("allocatorRole");
        configAdmin = _signer("configAdmin");
        adminRole = _signer("adminRole");
        premiumCollector = _signer("premiumCollector");
        firstLossBufferWallet = _signer("firstLossBufferWallet");
        buyer1 = _signer("buyer1");
        buyer2 = _signer("buyer2");
        beneficiary = _signer("beneficiary");

        // Mock vault — starts at currentPeriod = 1 to match production (period 0 never exists).
        // initialize() pushes the capacity-config checkpoint at effectivePeriod = 2, so tests
        // advance to period 2 before matching.
        mockVault = new MockCoverOrderAllocatorVault();
        mockVault.setCurrentPeriod(1);
        mockVault.setTotalAssets(0);
        for (uint256 i = 1; i <= 3; i++) {
            mockVault.setPeriodConfiguration(
                i, IFirelightVault.PeriodConfiguration({epoch: 0, duration: PERIOD_DURATION, startingPeriod: i})
            );
        }

        usdc = new MockERC20("MockUSDC", "mUSDC", 6);
        usdt = new MockERC20("MockUSDT", "mUSDT", 6);
        mockVault.setAsset(address(usdc));

        // $1.00, 18 decimals. Override with priceFeed.setAnswer(x) in feed-native decimals.
        priceFeed = new MockAggregatorV3(18, int256(10 ** 18));

        CoverNFT nftImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImpl),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "FLCOVER", "", admin, address(this), address(0), address(0))
                    )
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](2);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A});
        markets[1] = ICoverOrderAllocator.Market({chainId: 10, protocol: PROTOCOL_AAVE, market: MARKET_B});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](2);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: PROTOCOL_MORPHO, maxProtocolConcentrationBps: 6000
        });
        concs[1] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 10, protocol: PROTOCOL_AAVE, maxProtocolConcentrationBps: 4000
        });

        address[] memory premiumTokens = new address[](2);
        premiumTokens[0] = address(usdc);
        premiumTokens[1] = address(usdt);

        ICoverOrderAllocator.InitParams memory ip = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(mockVault)),
            premiumCollector: premiumCollector,
            coverNFT: coverNFT,
            priceFeedAdapter: IAggregatorV3(address(priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: admin,
            adminRole: adminRole,
            curatorRole: curator,
            allocatorRole: allocatorRole,
            configAdminRole: configAdmin,
            initialProtocolConcentrations: concs,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12000, // 1.2x, the MIN_CAR_BPS floor
                firstLossBufferToken: IERC20(address(usdc)),
                firstLossBuffer: firstLossBufferWallet,
                effectiveLeverage: 24000, // 2.0x multiplier (24000/12000)
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator aImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(new ERC1967Proxy(address(aImpl), abi.encodeCall(CoverOrderAllocator.initialize, (ip))))
        );

        // Same prank-consumption trap as above: cache the role first.
        bytes32 minterRole = coverNFT.MINTER_ROLE();
        vm.prank(admin);
        coverNFT.grantRole(minterRole, address(allocator));

        marketIdA = computeMarketId(1, PROTOCOL_MORPHO, MARKET_A);
        marketIdB = computeMarketId(10, PROTOCOL_AAVE, MARKET_B);
        concHashMorpho = computeProtocolConcentrationHash(1, PROTOCOL_MORPHO);
        concHashAave = computeProtocolConcentrationHash(10, PROTOCOL_AAVE);
    }

    // =====================================================================
    // deployFullStack — allocator wired to the REAL vault
    // =====================================================================
    /**
     * @dev `deployCoverOrderAllocator()` points the allocator at `MockCoverOrderAllocatorVault`,
     *      which returns hardcoded values and cannot execute a real `payout`. Anything that
     *      tests the *interaction* between vault state and allocator capacity needs the real
     *      vault instead.
     *
     *      `FirelightVault` does not inherit `IFirelightVault`, but every selector the allocator
     *      calls matches, so the interface cast resolves at runtime. (Struct nominal typing only
     *      binds at compile time; through an interface it is a plain ABI call.)
     *
     *      Capacity is deliberately isolated to the vault leg: the first-loss buffer wallet is
     *      left at zero balance, so `_computeAvailableCapacity` reduces to
     *      `totalAssetsAt(periodStart) * 1e12 * effectiveLeverage / minCAR`.
     *
     *      Call after `deployVault()`.
     */
    function deployFullStack() internal {
        admin = _signer("admin");
        curator = _signer("curator");
        allocatorRole = _signer("allocatorRole");
        configAdmin = _signer("configAdmin");
        adminRole = _signer("adminRole");
        premiumCollector = _signer("premiumCollector");
        firstLossBufferWallet = _signer("firstLossBufferWallet");
        buyer1 = _signer("buyer1");
        buyer2 = _signer("buyer2");
        beneficiary = _signer("beneficiary");

        usdc = new MockERC20("MockUSDC", "mUSDC", 6);
        priceFeed = new MockAggregatorV3(18, int256(10 ** 18)); // $1.00

        CoverNFT nftImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImpl),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "FLCOVER", "", admin, address(this), address(0), address(0))
                    )
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: PROTOCOL_MORPHO, maxProtocolConcentrationBps: 10000
        });

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(usdc);

        ICoverOrderAllocator.InitParams memory ip = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(vault)), // THE REAL VAULT
            premiumCollector: premiumCollector,
            coverNFT: coverNFT,
            priceFeedAdapter: IAggregatorV3(address(priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: admin,
            adminRole: adminRole,
            curatorRole: curator,
            allocatorRole: allocatorRole,
            configAdminRole: configAdmin,
            initialProtocolConcentrations: concs,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12000,
                firstLossBufferToken: IERC20(address(usdc)),
                firstLossBuffer: firstLossBufferWallet, // left at ZERO balance
                effectiveLeverage: 24000, // 2.0x multiplier
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator aImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(new ERC1967Proxy(address(aImpl), abi.encodeCall(CoverOrderAllocator.initialize, (ip))))
        );

        bytes32 minterRole = coverNFT.MINTER_ROLE();
        vm.prank(admin);
        coverNFT.grantRole(minterRole, address(allocator));

        marketIdA = computeMarketId(1, PROTOCOL_MORPHO, MARKET_A);
        concHashMorpho = computeProtocolConcentrationHash(1, PROTOCOL_MORPHO);
    }

    /// @dev MockAggregatorV3 pins `updatedAt` at construction, so any `vm.warp` past
    ///      `maxPriceAge` (7 days here) makes `PriceFeed.getPrice` revert `PriceFeedTooOld`.
    ///      Call before any commit/recommit that follows a warp.
    function refreshPriceFeed() internal {
        priceFeed.setUpdatedAt(block.timestamp);
    }

    /// @dev Capacity the allocator will compute, replicating `_computeAvailableCapacity`
    ///      with a zero first-loss buffer and a $1.00 / 18-decimal feed.
    function expectedCapacity(uint256 vaultAssets6dp) internal pure returns (uint256) {
        return (vaultAssets6dp * 10 ** 12) * 24000 / 12000;
    }

    // =====================================================================
    // Helpers (ports of the JS utility closures)
    // =====================================================================

    /// @dev fixtures.js::utils.mintAndApprove
    function mintAndApprove(uint256 amount, address user) internal {
        token.mintTo(user, amount);
        vm.prank(user);
        token.approve(address(vault), amount);
    }

    /**
     * @dev Wires the real `VaultRewardDistributor` — the protocol's ONLY on-chain path for
     *      returning premium to stakers. Premiums are collected in a premium token, swapped
     *      off-chain, and pushed back in as vault asset; the distributor then calls
     *      `checkpointTotalAssets()`, which is why the DISTRIBUTOR CONTRACT (not the EOA)
     *      needs CHECKPOINT_ROLE on the vault.
     *
     *      `distributeRewards` pulls with `safeTransferFrom(msg.sender, vault, amount)`, so
     *      the EOA holding DISTRIBUTOR_ROLE must hold the asset and approve the distributor.
     *
     *      Call after `deployVault()`.
     */
    function deployRewardDistributor() internal {
        distributorEOA = _signer("distributorEOA");

        VaultRewardDistributor dImpl = new VaultRewardDistributor();
        distributor = VaultRewardDistributor(
            address(
                new ERC1967Proxy(
                    address(dImpl),
                    abi.encodeCall(
                        VaultRewardDistributor.initialize,
                        (IFirelightVault(address(vault)), deployer, distributorEOA, address(0))
                    )
                )
            )
        );

        // NOTE: cache the role BEFORE vm.prank. `vault.CHECKPOINT_ROLE()` is an external
        // staticcall, Solidity evaluates arguments before the call, and vm.prank applies to
        // the NEXT call only — so inlining it consumes the prank and grantRole runs as the
        // test contract, reverting AccessControlUnauthorizedAccount(<test>, DEFAULT_ADMIN_ROLE).
        bytes32 checkpointRole = vault.CHECKPOINT_ROLE();
        vm.prank(deployer);
        vault.grantRole(checkpointRole, address(distributor));
    }

    /// @dev Push `amount` of vault asset in as period premium, exactly as production does.
    function distributePremium(uint256 amount) internal {
        token.mintTo(distributorEOA, amount);
        vm.startPrank(distributorEOA);
        token.approve(address(distributor), amount);
        distributor.distributeRewards(amount, address(token), amount, uint48(block.timestamp));
        vm.stopPrank();
    }

    /**
     * @dev The fixture keeps `fixtures.js`'s 50,000-token deposit limit (5e10 at 6 decimals).
     *      That is small enough that most PoCs trip `DepositLimitExceeded` before reaching
     *      the behaviour under test — `_depositFunds` checks `totalAssets() + assets > depositLimit`.
     *      Raise it through the real `DEPOSIT_LIMIT_UPDATE_ROLE` rather than editing the fixture,
     *      so the deployment stays identical to the Hardhat suite.
     *
     *      1e15 == 1 billion tokens: ample headroom, and small enough that
     *      `convertToShares(depositLimit)` inside `maxMint` cannot overflow `mulDiv`.
     */
    function raiseDepositLimit() internal {
        raiseDepositLimit(1_000_000_000e6);
    }

    function raiseDepositLimit(uint256 newLimit) internal {
        vm.prank(limitUpdater);
        vault.updateDepositLimit(newLimit);
    }

    /// @dev fixtures.js::fundAndApprove
    function fundAndApprove(address buyer, MockERC20 t, uint256 amount) internal {
        t.mint(buyer, amount);
        vm.prank(buyer);
        t.approve(address(allocator), amount);
    }

    /// @dev fixtures.js::advanceToPeriod (mock vault only)
    function advanceToPeriod(uint256 p) internal {
        mockVault.setCurrentPeriod(p);
    }

    /// @dev Advance the REAL vault by whole periods.
    function warpPeriods(uint256 n) internal {
        vm.warp(block.timestamp + n * PERIOD_DURATION);
    }

    /**
     * @dev Timestamp at which `period` begins, for the real vault.
     *
     * NOTE the type. `PeriodConfiguration` is declared TWICE in this codebase —
     * `FirelightVaultStorage.sol:51` and `IFirelightVault.sol:33` — and they are distinct
     * nominal types. `FirelightVault` inherits FirelightVaultStorage and does NOT implement
     * IFirelightVault, so this getter returns the STORAGE struct. Using the interface struct
     * here fails with "not implicitly convertible".
     *
     * Conversely `MockCoverOrderAllocatorVault is IFirelightVault`, so its
     * `setPeriodConfiguration` takes the INTERFACE struct. Both spellings are correct in
     * their own place; do not unify them.
     *
     * For the current period prefer `vault.currentPeriodStart()`, which needs no struct.
     */
    function periodStart(uint256 period) internal view returns (uint48) {
        FirelightVault.PeriodConfiguration memory c = vault.periodConfigurationAtNumber(period);
        return uint48(c.epoch + (period - c.startingPeriod) * c.duration);
    }

    /// @dev fixtures.js::computeMarketId
    function computeMarketId(uint64 chainId, string memory protocol, bytes32 market) internal pure returns (bytes32) {
        return keccak256(abi.encode(chainId, protocol, market));
    }

    /// @dev fixtures.js::computeProtocolConcentrationHash
    function computeProtocolConcentrationHash(uint64 chainId, string memory protocol) internal pure returns (bytes32) {
        return keccak256(abi.encode(chainId, protocol));
    }

    /**
     * @dev Leaf encoding used by `_settleCoverOrder`: OZ StandardMerkleTree double-hashing.
     *      For a SINGLE-order tree the root IS the leaf and the proof is empty, which is all
     *      most PoCs need. Multi-order trees need a real tree builder.
     */
    function settleLeaf(uint256 orderId, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, allocs))));
    }

    /// @dev Two-leaf tree, sorted-pair hashing (OZ MerkleProof convention).
    function merkleRoot2(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b ? keccak256(abi.encode(a, b)) : keccak256(abi.encode(b, a));
    }

    /// @dev Funded EOA, replacing the JS `randomSigner()`.
    function _signer(string memory label) internal returns (address a) {
        a = makeAddr(label);
        vm.deal(a, 1000 ether);
    }

    /// @dev Single-market allocation array, the shape most PoCs want.
    function oneMarket(bytes32 marketId, uint256 amount)
        internal
        pure
        returns (ICoverOrderAllocator.MarketCoverAllocation[] memory arr)
    {
        arr = new ICoverOrderAllocator.MarketCoverAllocation[](1);
        arr[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: amount});
    }

    function oneMarketInput(bytes32 marketId, uint32 rateAnnual, uint256 amount)
        internal
        pure
        returns (ICoverOrderAllocator.MarketAllocationInput[] memory arr)
    {
        arr = new ICoverOrderAllocator.MarketAllocationInput[](1);
        arr[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: rateAnnual, coverAmount: amount
        });
    }
}
```

```solidity
// test/Foundry/mocks/MockFAsset.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/// Replica of contracts/fasset/fassetToken/implementation/FAsset.sol, which is pragma
/// 0.8.23 and skipped in foundry.toml. The transfer fee defaults to zero, so this behaves
/// as a plain ERC20 throughout this PoC.
contract MockFAsset is ERC20 {
    uint8   private immutable _decimals;
    uint256 public  transferFeeMillionths;
    address public  feeCollector = address(0xFEE);

    constructor(string memory n, string memory s, uint8 d) ERC20(n, s) { _decimals = d; }

    function decimals() public view override returns (uint8) { return _decimals; }
    function setTransferFeeMillionths(uint256 v) external { transferFeeMillionths = v; }
    function mintTo(address to, uint256 amount) external { _mint(to, amount); }
    function mint(address to, uint256 amount)   external { _mint(to, amount); }

    function _transferFeeAmount(uint256 amount) public view returns (uint256) {
        if (transferFeeMillionths == 0) return 0;
        return Math.mulDiv(amount, transferFeeMillionths, 1_000_000, Math.Rounding.Ceil);
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        uint256 fee = _transferFeeAmount(amount);
        _transfer(_msgSender(), to, amount - fee);
        if (fee > 0) _transfer(_msgSender(), feeCollector, fee);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        _spendAllowance(from, _msgSender(), amount);
        uint256 fee = _transferFeeAmount(amount);
        _transfer(from, to, amount - fee);
        if (fee > 0) _transfer(from, feeCollector, fee);
        return true;
    }
}
```

**`test/Foundry/RescueClaimFlag.t.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Base} from "./Base.t.sol";
import {console2} from "forge-std/Test.sol";
import {FirelightVault} from "contracts/core/FirelightVault.sol";

contract RescueClaimFlag is Base {
    uint256 constant STAKE = 1_000_000e6;
    uint256 constant FIRST =   400_000e6; // rescued away
    // the remaining 600_000e6 is what gets frozen

    address alice;
    address treasury;

    function setUp() public {
        deployVault();
        raiseDepositLimit();
        alice    = users[0];
        treasury = users[1];
    }

    function testRescueOfOpenBucketPermanentlyFreezesALaterWithdrawal() public {
        mintAndApprove(STAKE, alice);
        vm.prank(alice);
        vault.deposit(STAKE, alice);

        uint256 bucket = vault.currentPeriod() + 1;

        // 1. Alice requests a partial withdrawal. It lands in the still-open bucket.
        uint256 sharesForFirst = vault.previewWithdraw(FIRST);
        vm.prank(alice);
        vault.redeem(sharesForFirst, alice, alice);
        assertEq(vault.withdrawSharesOf(bucket, alice), sharesForFirst, "credit in the open bucket");

        // 2. Compliance flags Alice. Legitimate use of BLOCKLIST_ROLE.
        vm.prank(blocklister);
        vault.addToBlocklist(alice);

        // 3. RESCUER_ROLE moves the pending withdrawal to the treasury.
        //    Legitimate use — and exactly what the function exists for.
        uint256[] memory periods = new uint256[](1);
        periods[0] = bucket;
        vm.prank(rescuer);
        vault.rescueWithdrawFromBlocklisted(alice, treasury, periods);

        assertEq(vault.withdrawSharesOf(bucket, alice), 0, "alice's credit moved");
        assertGt(vault.withdrawSharesOf(bucket, treasury), 0, "treasury received it");
        assertTrue(
            vault.isWithdrawClaimed(bucket, alice),
            "<-- the defect: permanent flag on an OPEN bucket"
        );

        // 4. The flag was a false positive. Legitimate reversal.
        vm.prank(blocklister);
        vault.removeFromBlocklist(alice);
        assertTrue(vault.isWithdrawClaimed(bucket, alice), "removeFromBlocklist does NOT reset it");

        // 5. Alice withdraws the rest of her position, same period, same bucket.
        uint256 remaining = vault.balanceOf(alice);
        vm.prank(alice);
        uint256 owed = vault.redeem(remaining, alice, alice);
        assertEq(vault.balanceOf(alice), 0, "shares burned");
        assertGt(vault.withdrawSharesOf(bucket, alice), 0, "credited back into the flagged bucket");
        console2.log("alice is owed        :", owed);

        // 6. Two periods on, the bucket is claimable.
        warpPeriods(2);

        //    The treasury's rescued portion claims fine — the rescue itself worked.
        vm.prank(treasury);
        uint256 treasuryGot = vault.claimWithdraw(bucket);
        console2.log("treasury claimed     :", treasuryGot);
        assertEq(treasuryGot, FIRST, "rescue delivered its intended outcome");

        //    Alice cannot.
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, bucket)
        );
        vault.claimWithdraw(bucket);

        // 7. The funds are orphaned: present in the vault, owned by nobody, reachable by nobody.
        uint256 stranded = token.balanceOf(address(vault));
        console2.log("vault token balance  :", stranded);
        console2.log("pendingWithdrawAssets:", vault.pendingWithdrawAssets());
        console2.log("totalAssets()        :", vault.totalAssets());
        console2.log("totalSupply()        :", vault.totalSupply());

        assertEq(stranded, STAKE - FIRST, "600,000 FXRP still held by the vault");
        assertEq(vault.pendingWithdrawAssets(), stranded, "all of it locked as 'pending'");
        assertEq(vault.totalAssets(), 0, "vault reports it holds nothing");
        assertEq(vault.totalSupply(), 0, "no shares exist to claim it");
    }

    /// Control A — identical two-part withdrawal, no blocklist and no rescue.
    /// Isolates the rescue as the cause rather than the split withdrawal.
    function testControlWithoutTheRescueBothPartsClaimFine() public {
        mintAndApprove(STAKE, alice);
        vm.prank(alice);
        vault.deposit(STAKE, alice);

        uint256 bucket = vault.currentPeriod() + 1;

        uint256 sharesForFirst = vault.previewWithdraw(FIRST);
        vm.prank(alice);
        vault.redeem(sharesForFirst, alice, alice);

        uint256 remaining = vault.balanceOf(alice);
        vm.prank(alice);
        vault.redeem(remaining, alice, alice);

        warpPeriods(2);

        vm.prank(alice);
        uint256 got = vault.claimWithdraw(bucket);
        console2.log("alice claimed (no rescue):", got);
        assertEq(got, STAKE, "two requests into one bucket claim fine on their own");
        assertEq(vault.pendingWithdrawAssets(), 0, "nothing stranded");
    }

    /// Control B — the rescue happens, but Alice waits a period before withdrawing again.
    /// Her request lands in a DIFFERENT bucket and claims normally, proving the loss is
    /// specifically the same-period bucket collision.
    function testControlWaitingOnePeriodAvoidsTheCollision() public {
        mintAndApprove(STAKE, alice);
        vm.prank(alice);
        vault.deposit(STAKE, alice);

        uint256 flagged = vault.currentPeriod() + 1;

        uint256 sharesForFirst = vault.previewWithdraw(FIRST);
        vm.prank(alice);
        vault.redeem(sharesForFirst, alice, alice);

        vm.prank(blocklister);
        vault.addToBlocklist(alice);

        uint256[] memory periods = new uint256[](1);
        periods[0] = flagged;
        vm.prank(rescuer);
        vault.rescueWithdrawFromBlocklisted(alice, treasury, periods);

        vm.prank(blocklister);
        vault.removeFromBlocklist(alice);

        // Wait one period, so the new request goes to a different bucket.
        warpPeriods(1);
        uint256 safeBucket = vault.currentPeriod() + 1;
        assertTrue(safeBucket != flagged, "different bucket");

        uint256 remaining = vault.balanceOf(alice);
        vm.prank(alice);
        vault.redeem(remaining, alice, alice);

        warpPeriods(2);

        vm.prank(alice);
        uint256 got = vault.claimWithdraw(safeBucket);
        console2.log("alice claimed (waited one period):", got);
        assertEq(got, STAKE - FIRST, "no loss when the buckets do not collide");
    }
}
```

### Output

```
$ forge test --match-contract RescueClaimFlag -vv


Ran 3 tests for test/Foundry/RescueClaimFlag.t.sol:RescueClaimFlag
[PASS] testControlWaitingOnePeriodAvoidsTheCollision() (gas: 799827)
Logs:
  alice claimed (waited one period): 600000000000

[PASS] testControlWithoutTheRescueBothPartsClaimFine() (gas: 545496)
Logs:
  alice claimed (no rescue): 1000000000000

[PASS] testRescueOfOpenBucketPermanentlyFreezesALaterWithdrawal() (gas: 673039)
Logs:
  alice is owed        : 600000000000
  treasury claimed     : 400000000000
  vault token balance  : 600000000000
  pendingWithdrawAssets: 600000000000
  totalAssets()        : 0
  totalSupply()        : 0

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 11.65ms (4.31ms CPU time)
```

The three runs isolate the defect precisely:

* **Control A** — the same split withdrawal with no rescue claims the full **1,000,000**. The two-request pattern is not the problem.
* **Control B** — the rescue happens, but Alice waits one period; her request lands in a different bucket and she recovers **600,000** in full. The bucket collision is the problem.
* **The finding** — same rescue, same-period second request: the treasury's rescued portion claims correctly, Alice's reverts forever, and **600,000 FXRP** is left in a vault that reports holding nothing.
