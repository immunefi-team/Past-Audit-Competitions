# \#97644 [SC-Insight] `setTerminalNavBook` does not validate the book's size quantum, contrary to its NatSpec and SECURITY.md

**Submitted on Nov 2nd 2026 at 15:21:07 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97644
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Target:** `packages/perps/src/CfdEngine.sol` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
**Category:** Insight — Security best practices / Documentation improvements

## Description

Three places document that wiring the terminal NAV book checks the size quantum:

- `CfdEngine.sol:402-404` (NatSpec of `setTerminalNavBook`): *"The book must be contract code immutably bound to this Engine, the same uint32 price cap, and the canonical 100-token size quantum."*
- SECURITY.md:87-90: *"The Engine accepts only an empty book immutably bound to itself with matching `CAP_PRICE` and the canonical `1e20` size quantum."*
- SECURITY.md:289-291: *"Engine wiring validates the bound Engine, price cap, size quantum, and completely empty book before accepting it once."*

The implementation (`CfdEngine.sol:405-425`) relies only on `candidate.validateEngineBinding()`. That function (`TerminalNavBookV2.sol:155-169`) checks the bound Engine, `CAP_PRICE`, and that the book is empty. Neither function reads `ITerminalNavBookV2.SIZE_QUANTUM()` or compares it with `CfdTypes.SIZE_QUANTUM` (`1e20`).

The setter is owner-only, so this is not an exploitable vulnerability. It is a defense-in-depth gap where the documented guarantee is not enforced. Because the book is set once and has no repair path, a book built with a non-canonical quantum would be permanent.

By inspection of `TerminalNavBookV2.sol:215-216` (not exercised in the PoC): `terminalNavSnapshot()` reconciles the book's totals multiplied by `SIZE_QUANTUM` against the engine's side open interest and entry notional. With a non-canonical quantum, it would revert with `CfdEngine__InvalidTerminalNavBook` once a position exists, so the misconfiguration would surface only after wiring, when it can no longer be corrected.

## Proof of concept

**Files:**
- `packages/perps/test/perps/poc/TerminalNavBookQuantumNotValidated.t.sol`
- `TerminalNavBookV2WrongQuantum.sol`: a test-only copy of `src/TerminalNavBookV2.sol` that is byte-identical except for the contract name and `SIZE_QUANTUM = 1e19`. No in-scope source is modified.

```bash
git submodule update --init lib/forge-std lib/openzeppelin-contracts lib/morpho-blue
cd packages/perps
FOUNDRY_TEST=test/perps/poc forge test --match-path test/perps/poc/TerminalNavBookQuantumNotValidated.t.sol -vv
```

```
[PASS] test_control_engineBindingAndCapAreEnforced()
[PASS] test_setTerminalNavBook_acceptsNonCanonicalSizeQuantum()
```

- **`test_setTerminalNavBook_acceptsNonCanonicalSizeQuantum`:** a fresh engine accepts a book whose `SIZE_QUANTUM()` is `1e19`.
- **`test_control_engineBindingAndCapAreEnforced`:** the checks that *are* enforced still work. A book bound to another engine, and a book with a different price cap, both revert with `CfdEngine__InvalidTerminalNavBook`. A canonical book is accepted.

## Recommendation

Enforce the documented check in `setTerminalNavBook`:

```solidity
if (candidate.SIZE_QUANTUM() != CfdTypes.SIZE_QUANTUM) {
    revert CfdEngine__InvalidTerminalNavBook();
}
```

Alternatively, add the comparison inside `TerminalNavBookV2.validateEngineBinding()`. If the check is intentionally omitted, correct the NatSpec and SECURITY.md wording.


## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Insight PoC — CfdEngine.setTerminalNavBook does not validate the book's SIZE_QUANTUM, although its NatSpec
// (CfdEngine.sol:403-404) and SECURITY.md:289-291 state that wiring validates "the canonical 100-token size quantum".
// TerminalNavBookV2WrongQuantum.sol is a test-only copy of src/TerminalNavBookV2.sol identical except the contract
// name and SIZE_QUANTUM (1e19 instead of CfdTypes.SIZE_QUANTUM = 1e20). In-scope source is not modified.
// Run: cd packages/perps && FOUNDRY_TEST=test/perps/poc forge test --match-path \
//        test/perps/poc/TerminalNavBookQuantumNotValidated.t.sol -vv

import {BasePerpTest} from "../BasePerpTest.sol";
import {TerminalNavBookV2WrongQuantum} from "./TerminalNavBookV2WrongQuantum.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

contract TerminalNavBookQuantumNotValidatedPoC is BasePerpTest {

    uint256 internal constant CANONICAL_QUANTUM = 1e20; // CfdTypes.SIZE_QUANTUM

    /// The engine accepts a book whose size quantum differs from the canonical one.
    function test_setTerminalNavBook_acceptsNonCanonicalSizeQuantum() public {
        CfdEngine freshEngine = _deployEngine(_riskParams());
        TerminalNavBookV2WrongQuantum book =
            new TerminalNavBookV2WrongQuantum(address(freshEngine), uint32(freshEngine.CAP_PRICE()));
        assertTrue(book.SIZE_QUANTUM() != CANONICAL_QUANTUM, "book uses a non-canonical quantum");

        freshEngine.setTerminalNavBook(address(book)); // documented to be rejected; succeeds
        assertEq(address(freshEngine.terminalNavBook()), address(book), "wrong-quantum book wired");
    }

    /// Controls: the checks that ARE enforced (engine binding, price cap) still reject bad books.
    function test_control_engineBindingAndCapAreEnforced() public {
        CfdEngine freshEngine = _deployEngine(_riskParams());
        CfdEngine otherEngine = _deployEngine(_riskParams());

        TerminalNavBookV2 boundElsewhere = new TerminalNavBookV2(address(otherEngine), uint32(freshEngine.CAP_PRICE()));
        vm.expectRevert(ICfdEngineTypes.CfdEngine__InvalidTerminalNavBook.selector);
        freshEngine.setTerminalNavBook(address(boundElsewhere));

        TerminalNavBookV2 wrongCap = new TerminalNavBookV2(address(freshEngine), uint32(freshEngine.CAP_PRICE()) - 1);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__InvalidTerminalNavBook.selector);
        freshEngine.setTerminalNavBook(address(wrongCap));

        TerminalNavBookV2 good = new TerminalNavBookV2(address(freshEngine), uint32(freshEngine.CAP_PRICE()));
        freshEngine.setTerminalNavBook(address(good));
        assertEq(good.SIZE_QUANTUM(), CANONICAL_QUANTUM);
    }

}
```


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {ITerminalNavBookV2} from "@plether/perps/interfaces/ITerminalNavBookV2.sol";

/// @dev Minimal exact-state surface consumed by the immutable terminal-book coordinator.
interface ITerminalNavEngineView {

    function CAP_PRICE() external view returns (uint256);

    function clearinghouse() external view returns (address);

    function positions(
        address account
    )
        external
        view
        returns (
            uint256 size,
            uint256 margin,
            uint256 entryPrice,
            uint256 maxProfitAssets,
            CfdTypes.Side side,
            uint64 lastUpdateTime,
            int256 vpiAccrued
        );

    function positionEntryCostAssetAtoms(
        address account
    ) external view returns (uint256);

    function traderClaimBalanceAssets(
        address account
    ) external view returns (uint256);

    function sides(
        uint256 index
    ) external view returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);

    function lastMarkPrice() external view returns (uint256);

    function lastMarkTime() external view returns (uint64);

    function totalTraderClaimBalanceAssets() external view returns (uint256);

    function degradedMode() external view returns (bool);

}

/// @title TerminalNavBookV2
/// @notice Aggregates exact, account-local, collateral-capped terminal price PnL without iterating positions.
/// @dev Each active Engine account contributes one piecewise-affine curve over the immutable uint32 price domain. A
///      curve has a global affine base and at most one suffix event. Suffix events are accumulated in an eight-level
///      radix-16 prefix tree, making mutations and reads independent of active-position count. The immutable Engine is
///      the only writer; there is intentionally no owner, repair, migration, or arbitrary update path.
/// @custom:security-contact contact@plether.com
contract TerminalNavBookV2WrongQuantum is ITerminalNavBookV2 {

    /// @inheritdoc ITerminalNavBookV2
    uint256 public constant override SIZE_QUANTUM = 1e19; // test-only copy: non-canonical quantum

    /// @notice Number of radix-16 nibbles in a uint32 breakpoint key.
    uint8 private constant RADIX_LEVELS = 8;

    /// @notice Maximum positive value representable by an `int112` coefficient.
    uint256 private constant MAX_INT112 = uint256(int256(type(int112).max));

    /// @notice Maximum positive value representable by an `int144` coefficient.
    uint256 private constant MAX_INT144 = uint256(int256(type(int144).max));

    /// @inheritdoc ITerminalNavBookV2
    address public immutable override ENGINE;

    /// @inheritdoc ITerminalNavBookV2
    uint32 public immutable override CAP_PRICE;

    /// @notice Canonical account curves, keyed by the Engine account domain.
    mapping(address account => CurveRecord curve) private _curves;

    /// @notice Hash commitments to canonical account curves; zero denotes no stored curve.
    mapping(address account => bytes32 curveHash) private _curveHashes;

    /// @notice Aggregate coefficient that applies over the complete price domain.
    Coeff private _base;

    /// @notice Sparse aggregate suffix events by radix prefix level and right-aligned prefix.
    mapping(uint8 level => mapping(uint32 prefix => Coeff coefficient)) private _radixNodes;

    /// @notice Number of canonical active account curves.
    uint64 private _activeCurveCount;

    /// @notice Monotonic effective-mutation counter.
    uint64 private _bookVersion;

    /// @notice Gross lots budget used to prove every packed aggregate slope remains representable.
    uint112 private _totalLots;

    /// @notice Gross entry-basis budget used to prove every packed aggregate intercept remains representable.
    uint144 private _totalEntryCostAssetAtoms;

    /// @notice Gross effective collectible-cap budget used in packed-intercept safety checks.
    uint144 private _totalEffectiveCapAssetAtoms;

    /// @notice Internal piecewise-affine representation of one canonical account curve.
    struct CurveEncoding {
        Coeff base;
        Coeff eventCoefficient;
        uint32 breakpoint;
        bool hasEvent;
    }

    /// @notice Exact Engine state used to derive one canonical account curve.
    struct EngineCurveInput {
        uint112 lots;
        uint144 entryCostAssetAtoms;
        uint144 collectibleCapAssetAtoms;
        CfdTypes.Side side;
    }

    /// @notice Restricts every mutation to the immutable Engine.
    modifier onlyEngine() {
        _requireEngine();
        _;
    }

    /// @notice Reverts unless the current caller is the immutable Engine.
    function _requireEngine() private view {
        if (msg.sender != ENGINE) {
            revert TerminalNavBookV2__Unauthorized();
        }
    }

    /// @notice Binds an empty terminal book to one Engine and one uint32 oracle-price domain.
    /// @param engine_ Sole address authorized to insert, replace, and remove curves.
    /// @param capPrice_ Maximum accepted mark and curve-domain endpoint, with 8 decimals.
    constructor(
        address engine_,
        uint32 capPrice_
    ) {
        if (engine_ == address(0)) {
            revert TerminalNavBookV2__ZeroEngine();
        }
        if (capPrice_ == 0) {
            revert TerminalNavBookV2__ZeroCapPrice();
        }

        ENGINE = engine_;
        CAP_PRICE = capPrice_;
    }

    /// @inheritdoc ITerminalNavBookV2
    function validateEngineBinding() external view returns (bool valid) {
        if (msg.sender != ENGINE || _activeCurveCount != 0 || _bookVersion != 0 || _totalLots != 0) {
            return false;
        }

        ITerminalNavEngineView engine = ITerminalNavEngineView(ENGINE);
        (uint256 longMaxProfit, uint256 longOpenInterest, uint256 longEntryNotional, uint256 longMargin) =
            engine.sides(0);
        (uint256 shortMaxProfit, uint256 shortOpenInterest, uint256 shortEntryNotional, uint256 shortMargin) =
            engine.sides(1);
        return engine.CAP_PRICE() == CAP_PRICE && longMaxProfit == 0 && longOpenInterest == 0 && longEntryNotional == 0
            && longMargin == 0 && shortMaxProfit == 0 && shortOpenInterest == 0 && shortEntryNotional == 0
            && shortMargin == 0 && _totalEntryCostAssetAtoms == 0 && _totalEffectiveCapAssetAtoms == 0
            && _base.slope == 0 && _base.intercept == 0;
    }

    /// @inheritdoc ITerminalNavBookV2
    function authenticateEngineState(
        address account
    ) external view onlyEngine returns (bytes32 expectedHash) {
        (bool hasPosition, EngineCurveInput memory input) = _curveInputFromEngine(account);
        expectedHash = hasPosition ? _curveHash(account, _canonicalize(input)) : bytes32(0);
        bytes32 actualHash = _curveHashes[account];
        if (actualHash != expectedHash) {
            revert ICfdEngineTypes.CfdEngine__TerminalNavBookHashMismatch(account, expectedHash, actualHash);
        }
    }

    /// @inheritdoc ITerminalNavBookV2
    function syncFromEngine(
        address account,
        bytes32 expectedOldHash
    ) external onlyEngine returns (bytes32 newHash, uint64 newBookVersion) {
        if (account == address(0)) {
            revert TerminalNavBookV2__ZeroAccount();
        }

        bytes32 actualOldHash = _curveHashes[account];
        if (expectedOldHash != actualOldHash) {
            revert TerminalNavBookV2__CurveHashMismatch(account, expectedOldHash, actualOldHash);
        }

        (bool hasPosition, EngineCurveInput memory input) = _curveInputFromEngine(account);
        if (!hasPosition) {
            if (actualOldHash == bytes32(0)) {
                return (bytes32(0), _bookVersion);
            }
            return (bytes32(0), _removeCurve(account, actualOldHash, _curves[account]));
        }
        return _setCurve(account, actualOldHash, input);
    }

    /// @inheritdoc ITerminalNavBookV2
    function terminalNavSnapshot() external view returns (ICfdEngineTypes.TerminalNavSnapshot memory snapshot) {
        ITerminalNavEngineView engine = ITerminalNavEngineView(ENGINE);
        (uint256 longMaxProfit, uint256 longOpenInterest, uint256 longEntryNotional,) = engine.sides(0);
        (uint256 shortMaxProfit, uint256 shortOpenInterest, uint256 shortEntryNotional,) = engine.sides(1);
        uint256 totalOpenInterest = longOpenInterest + shortOpenInterest;
        uint256 totalEntryNotional = longEntryNotional + shortEntryNotional;
        if (
            uint256(_totalLots) * SIZE_QUANTUM != totalOpenInterest
                || uint256(_totalEntryCostAssetAtoms) * SIZE_QUANTUM != totalEntryNotional
                || ((_activeCurveCount == 0) != (totalOpenInterest == 0))
        ) {
            revert ICfdEngineTypes.CfdEngine__InvalidTerminalNavBook();
        }

        snapshot.hasOpenPositions = totalOpenInterest != 0;
        uint256 markPrice = engine.lastMarkPrice();
        uint64 markTime = engine.lastMarkTime();
        if (markPrice > CAP_PRICE || (snapshot.hasOpenPositions && (markPrice == 0 || markTime == 0))) {
            revert ICfdEngineTypes.CfdEngine__InvalidTerminalNavBook();
        }
        if (snapshot.hasOpenPositions) {
            snapshot.terminalLpPriceDeltaAssets = _terminalLpPriceDeltaAssetAtoms(uint32(markPrice));
        }
        snapshot.markPrice = uint32(markPrice);
        snapshot.markTime = markTime;
        snapshot.totalTraderClaimsAssets = engine.totalTraderClaimBalanceAssets();
        snapshot.maxDirectionalLiabilityAssets = longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
        snapshot.bookVersion = _bookVersion;
        snapshot.degradedMode = engine.degradedMode();
    }

    function _setCurve(
        address account,
        bytes32 actualOldHash,
        EngineCurveInput memory next
    ) private returns (bytes32 newHash, uint64 newBookVersion) {
        CurveRecord memory nextRecord = _canonicalize(next);
        newHash = _curveHash(account, nextRecord);
        if (newHash == bytes32(0)) {
            revert TerminalNavBookV2__ZeroCurveHash();
        }
        if (newHash == actualOldHash) {
            return (newHash, _bookVersion);
        }

        CurveRecord memory oldRecord = _curves[account];
        bool replacing = actualOldHash != bytes32(0);
        (uint112 nextTotalLots, uint144 nextTotalEntryCost, uint144 nextTotalCap) =
            _nextTotals(oldRecord, nextRecord, replacing);

        if (replacing) {
            _replaceCurve(oldRecord, nextRecord);
        } else {
            _applyCurve(nextRecord, true);
        }

        _totalLots = nextTotalLots;
        _totalEntryCostAssetAtoms = nextTotalEntryCost;
        _totalEffectiveCapAssetAtoms = nextTotalCap;
        if (!replacing) {
            if (_activeCurveCount == type(uint64).max) {
                revert TerminalNavBookV2__AggregateBoundsExceeded();
            }
            unchecked {
                ++_activeCurveCount;
            }
        }

        newBookVersion = _incrementBookVersion();
        _curves[account] = nextRecord;
        _curveHashes[account] = newHash;

        emit CurveSet(
            account,
            actualOldHash,
            newHash,
            newBookVersion,
            nextRecord.lots,
            nextRecord.entryCostAssetAtoms,
            nextRecord.effectiveCapAssetAtoms,
            nextRecord.side
        );
    }

    function _removeCurve(
        address account,
        bytes32 actualOldHash,
        CurveRecord memory oldRecord
    ) private returns (uint64 newBookVersion) {
        _applyCurve(oldRecord, false);

        _totalLots -= oldRecord.lots;
        _totalEntryCostAssetAtoms -= oldRecord.entryCostAssetAtoms;
        _totalEffectiveCapAssetAtoms -= oldRecord.effectiveCapAssetAtoms;
        unchecked {
            --_activeCurveCount;
        }

        newBookVersion = _incrementBookVersion();
        delete _curves[account];
        delete _curveHashes[account];

        emit CurveRemoved(account, actualOldHash, newBookVersion);
    }

    /// @inheritdoc ITerminalNavBookV2
    function terminalLpPriceDeltaAssetAtoms(
        uint32 markPrice
    ) external view returns (int256 terminalDeltaAssetAtoms) {
        return _terminalLpPriceDeltaAssetAtoms(markPrice);
    }

    function _terminalLpPriceDeltaAssetAtoms(
        uint32 markPrice
    ) private view returns (int256 terminalDeltaAssetAtoms) {
        if (markPrice > CAP_PRICE) {
            revert TerminalNavBookV2__MarkPriceAboveCap(markPrice, CAP_PRICE);
        }

        int256 aggregateSlope = int256(_base.slope);
        int256 aggregateIntercept = int256(_base.intercept);
        uint32 prefix;

        for (uint8 level = 1; level <= RADIX_LEVELS; ++level) {
            uint8 shift = uint8((RADIX_LEVELS - level) * 4);
            uint8 digit = uint8((markPrice >> shift) & 0x0f);
            uint32 childBase = prefix << 4;

            for (uint8 siblingDigit = 0; siblingDigit < digit; ++siblingDigit) {
                Coeff storage sibling = _radixNodes[level][childBase | uint32(siblingDigit)];
                aggregateSlope += int256(sibling.slope);
                aggregateIntercept += int256(sibling.intercept);
            }

            prefix = childBase | uint32(digit);
        }

        Coeff storage exactLeaf = _radixNodes[RADIX_LEVELS][prefix];
        aggregateSlope += int256(exactLeaf.slope);
        aggregateIntercept += int256(exactLeaf.intercept);

        terminalDeltaAssetAtoms = aggregateSlope * int256(uint256(markPrice)) + aggregateIntercept;
    }

    /// @inheritdoc ITerminalNavBookV2
    function curveOf(
        address account
    ) external view returns (CurveRecord memory curve) {
        return _curves[account];
    }

    /// @inheritdoc ITerminalNavBookV2
    function curveHashOf(
        address account
    ) external view returns (bytes32 curveHash) {
        return _curveHashes[account];
    }

    /// @inheritdoc ITerminalNavBookV2
    function bookState() external view returns (BookState memory state) {
        state.capPrice = CAP_PRICE;
        state.activeCurveCount = _activeCurveCount;
        state.bookVersion = _bookVersion;
        state.totalLots = _totalLots;
        state.totalEntryCostAssetAtoms = _totalEntryCostAssetAtoms;
        state.totalEffectiveCapAssetAtoms = _totalEffectiveCapAssetAtoms;
        state.base = _base;
    }

    /// @inheritdoc ITerminalNavBookV2
    function radixNode(
        uint8 level,
        uint32 prefix
    ) external view returns (Coeff memory coefficient) {
        if (level == 0 || level > RADIX_LEVELS || (level < RADIX_LEVELS && prefix >= uint32(1) << (level * 4))) {
            revert TerminalNavBookV2__InvalidRadixNode(level, prefix);
        }
        return _radixNodes[level][prefix];
    }

    /// @notice Validates a candidate curve and removes collateral that can never be collected inside the price domain.
    function _canonicalize(
        EngineCurveInput memory next
    ) private view returns (CurveRecord memory record) {
        if (next.lots == 0) {
            revert TerminalNavBookV2__ZeroLots();
        }

        uint256 maximumEntryCostAssetAtoms = uint256(next.lots) * uint256(CAP_PRICE);
        if (uint256(next.entryCostAssetAtoms) > maximumEntryCostAssetAtoms) {
            revert TerminalNavBookV2__EntryCostAboveCap(next.entryCostAssetAtoms, maximumEntryCostAssetAtoms);
        }

        uint256 maximumCollectibleAssetAtoms = next.side == CfdTypes.Side.LONG
            ? maximumEntryCostAssetAtoms - uint256(next.entryCostAssetAtoms)
            : uint256(next.entryCostAssetAtoms);
        uint144 effectiveCap = next.collectibleCapAssetAtoms;
        if (uint256(effectiveCap) > maximumCollectibleAssetAtoms) {
            effectiveCap = uint144(maximumCollectibleAssetAtoms);
        }

        record = CurveRecord({
            lots: next.lots,
            entryCostAssetAtoms: next.entryCostAssetAtoms,
            effectiveCapAssetAtoms: effectiveCap,
            side: next.side
        });
    }

    /// @notice Reconstructs one exact canonical curve candidate from live Engine and clearinghouse state.
    function _curveInputFromEngine(
        address account
    ) private view returns (bool hasPosition, EngineCurveInput memory input) {
        ITerminalNavEngineView engine = ITerminalNavEngineView(ENGINE);
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(account);
        if (size == 0) {
            return (false, input);
        }
        if (size % SIZE_QUANTUM != 0) {
            revert ICfdEngineTypes.CfdEngine__InvalidTerminalNavBook();
        }

        uint256 lots = size / SIZE_QUANTUM;
        uint256 entryCostAssetAtoms = engine.positionEntryCostAssetAtoms(account);
        IMarginClearinghouse clearinghouse = IMarginClearinghouse(engine.clearinghouse());
        uint256 candidateCapAssetAtoms =
            clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
        if (
            lots > type(uint112).max || entryCostAssetAtoms > type(uint144).max
                || candidateCapAssetAtoms > type(uint144).max
        ) {
            revert ICfdEngineTypes.CfdEngine__InvalidTerminalNavBook();
        }

        input = EngineCurveInput({
            lots: uint112(lots),
            entryCostAssetAtoms: uint144(entryCostAssetAtoms),
            collectibleCapAssetAtoms: uint144(candidateCapAssetAtoms),
            side: side
        });
        hasPosition = true;
    }

    /// @notice Computes post-replacement gross budgets before any accumulator writes occur.
    function _nextTotals(
        CurveRecord memory oldRecord,
        CurveRecord memory nextRecord,
        bool replacing
    ) private view returns (uint112 nextLots, uint144 nextEntryCost, uint144 nextCap) {
        uint256 oldLots = replacing ? uint256(oldRecord.lots) : 0;
        uint256 oldEntryCost = replacing ? uint256(oldRecord.entryCostAssetAtoms) : 0;
        uint256 oldCap = replacing ? uint256(oldRecord.effectiveCapAssetAtoms) : 0;

        uint256 widenedLots = uint256(_totalLots) - oldLots + uint256(nextRecord.lots);
        uint256 widenedEntryCost =
            uint256(_totalEntryCostAssetAtoms) - oldEntryCost + uint256(nextRecord.entryCostAssetAtoms);
        uint256 widenedCap = uint256(_totalEffectiveCapAssetAtoms) - oldCap + uint256(nextRecord.effectiveCapAssetAtoms);

        if (widenedLots > MAX_INT112 || widenedEntryCost + widenedCap > MAX_INT144) {
            revert TerminalNavBookV2__AggregateBoundsExceeded();
        }

        nextLots = uint112(widenedLots);
        nextEntryCost = uint144(widenedEntryCost);
        nextCap = uint144(widenedCap);
    }

    /// @notice Adds or subtracts one canonical account curve from the global base and sparse suffix accumulator.
    function _applyCurve(
        CurveRecord memory record,
        bool add
    ) private {
        CurveEncoding memory encoding = _encode(record);
        _base = _updatedCoefficient(_base, encoding.base, add);

        if (encoding.hasEvent) {
            _applyEvent(encoding.breakpoint, encoding.eventCoefficient, add);
        }
    }

    /// @notice Replaces one encoded curve while coalescing every shared radix-prefix write.
    /// @dev An unchanged breakpoint touches eight nodes once each. A relocation touches at most sixteen distinct nodes,
    ///      and fewer when the old and new keys share high-order prefixes.
    function _replaceCurve(
        CurveRecord memory oldRecord,
        CurveRecord memory nextRecord
    ) private {
        CurveEncoding memory oldEncoding = _encode(oldRecord);
        CurveEncoding memory nextEncoding = _encode(nextRecord);

        Coeff memory baseWithoutOld = _updatedCoefficient(_base, oldEncoding.base, false);
        _base = _updatedCoefficient(baseWithoutOld, nextEncoding.base, true);

        if (!oldEncoding.hasEvent) {
            if (nextEncoding.hasEvent) {
                _applyEvent(nextEncoding.breakpoint, nextEncoding.eventCoefficient, true);
            }
            return;
        }
        if (!nextEncoding.hasEvent) {
            _applyEvent(oldEncoding.breakpoint, oldEncoding.eventCoefficient, false);
            return;
        }

        uint32 oldPrefix;
        uint32 nextPrefix;
        for (uint8 level = 1; level <= RADIX_LEVELS; ++level) {
            uint8 shift = uint8((RADIX_LEVELS - level) * 4);
            oldPrefix = (oldPrefix << 4) | uint32((oldEncoding.breakpoint >> shift) & 0x0f);
            nextPrefix = (nextPrefix << 4) | uint32((nextEncoding.breakpoint >> shift) & 0x0f);

            if (oldPrefix == nextPrefix) {
                Coeff storage shared = _radixNodes[level][oldPrefix];
                Coeff memory withoutOld = _updatedCoefficient(shared, oldEncoding.eventCoefficient, false);
                Coeff memory updated = _updatedCoefficient(withoutOld, nextEncoding.eventCoefficient, true);
                _storeRadixNode(level, oldPrefix, updated);
            } else {
                _applyEventNode(level, oldPrefix, oldEncoding.eventCoefficient, false);
                _applyEventNode(level, nextPrefix, nextEncoding.eventCoefficient, true);
            }
        }
    }

    /// @notice Adds or removes one suffix event from its eight radix ancestors.
    function _applyEvent(
        uint32 breakpoint,
        Coeff memory eventCoefficient,
        bool add
    ) private {
        uint32 prefix;
        for (uint8 level = 1; level <= RADIX_LEVELS; ++level) {
            uint8 shift = uint8((RADIX_LEVELS - level) * 4);
            prefix = (prefix << 4) | uint32((breakpoint >> shift) & 0x0f);
            _applyEventNode(level, prefix, eventCoefficient, add);
        }
    }

    /// @notice Applies one event delta to one already-resolved radix node.
    function _applyEventNode(
        uint8 level,
        uint32 prefix,
        Coeff memory eventCoefficient,
        bool add
    ) private {
        Coeff storage stored = _radixNodes[level][prefix];
        Coeff memory updated = _updatedCoefficient(stored, eventCoefficient, add);
        _storeRadixNode(level, prefix, updated);
    }

    /// @notice Writes one packed radix node or deletes an exact zero to preserve storage refunds and sparse reads.
    function _storeRadixNode(
        uint8 level,
        uint32 prefix,
        Coeff memory updated
    ) private {
        if (updated.slope == 0 && updated.intercept == 0) {
            delete _radixNodes[level][prefix];
        } else {
            Coeff storage stored = _radixNodes[level][prefix];
            stored.slope = updated.slope;
            stored.intercept = updated.intercept;
        }
    }

    /// @notice Encodes `min(uncapped LP price PnL, account collectible cap)` as base plus one suffix event.
    function _encode(
        CurveRecord memory record
    ) private view returns (CurveEncoding memory encoding) {
        int112 lots = int112(record.lots);
        int144 entryCost = int144(record.entryCostAssetAtoms);
        uint256 cap = uint256(record.effectiveCapAssetAtoms);

        if (record.side == CfdTypes.Side.LONG) {
            encoding.base = Coeff({slope: lots, intercept: -entryCost});

            uint256 maximumCollectible = uint256(record.lots) * uint256(CAP_PRICE) - record.entryCostAssetAtoms;
            if (cap < maximumCollectible) {
                uint256 eventIntercept = uint256(record.entryCostAssetAtoms) + cap;
                encoding.hasEvent = true;
                encoding.breakpoint = _checkedBreakpoint(_ceilDiv(eventIntercept, record.lots));
                encoding.eventCoefficient = Coeff({slope: -lots, intercept: int144(int256(eventIntercept))});
            }
        } else if (cap == uint256(record.entryCostAssetAtoms)) {
            encoding.base = Coeff({slope: -lots, intercept: entryCost});
        } else {
            uint256 eventIntercept = uint256(record.entryCostAssetAtoms) - cap;
            encoding.base = Coeff({slope: 0, intercept: int144(int256(cap))});
            encoding.hasEvent = true;
            encoding.breakpoint = _checkedBreakpoint(_ceilDiv(eventIntercept, record.lots));
            encoding.eventCoefficient = Coeff({slope: -lots, intercept: int144(int256(eventIntercept))});
        }
    }

    /// @notice Applies a signed coefficient delta after widening and validates the packed result.
    function _updatedCoefficient(
        Coeff memory current,
        Coeff memory delta,
        bool add
    ) private pure returns (Coeff memory updated) {
        int256 slopeDelta = add ? int256(delta.slope) : -int256(delta.slope);
        int256 interceptDelta = add ? int256(delta.intercept) : -int256(delta.intercept);
        int256 widenedSlope = int256(current.slope) + slopeDelta;
        int256 widenedIntercept = int256(current.intercept) + interceptDelta;

        if (
            widenedSlope < type(int112).min || widenedSlope > type(int112).max || widenedIntercept < type(int144).min
                || widenedIntercept > type(int144).max
        ) {
            revert TerminalNavBookV2__CoefficientOverflow();
        }

        updated.slope = int112(widenedSlope);
        updated.intercept = int144(widenedIntercept);
    }

    /// @notice Computes a deployment- and account-domain-separated hash of a canonical curve record.
    function _curveHash(
        address account,
        CurveRecord memory record
    ) private view returns (bytes32) {
        return keccak256(
            abi.encode(
                address(this),
                CAP_PRICE,
                account,
                record.lots,
                record.entryCostAssetAtoms,
                record.effectiveCapAssetAtoms,
                record.side
            )
        );
    }

    /// @notice Proves a derived breakpoint is inside the immutable domain before narrowing to uint32.
    function _checkedBreakpoint(
        uint256 breakpoint
    ) private view returns (uint32 checkedBreakpoint) {
        if (breakpoint > CAP_PRICE) {
            revert TerminalNavBookV2__BreakpointAboveCap(breakpoint, CAP_PRICE);
        }
        checkedBreakpoint = uint32(breakpoint);
    }

    /// @notice Increments the monotonic version without allowing wraparound.
    function _incrementBookVersion() private returns (uint64 newBookVersion) {
        if (_bookVersion == type(uint64).max) {
            revert TerminalNavBookV2__BookVersionOverflow();
        }
        unchecked {
            newBookVersion = ++_bookVersion;
        }
    }

    /// @notice Returns `ceil(numerator / denominator)` without an overflow-prone additive round-up.
    function _ceilDiv(
        uint256 numerator,
        uint256 denominator
    ) private pure returns (uint256 quotient) {
        quotient = numerator / denominator;
        if (numerator % denominator != 0) {
            unchecked {
                ++quotient;
            }
        }
    }

    }
```
