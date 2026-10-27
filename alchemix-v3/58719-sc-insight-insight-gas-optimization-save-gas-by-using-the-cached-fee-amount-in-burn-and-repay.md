# 58719 sc insight insight gas optimization save gas by using the cached fee amount in burn and repay in alchemist sol&#x20;

**Submitted on Nov 4th 2025 at 08:45:35 UTC by @chief\_hunter888 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58719
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## **Brief Summary:**

The current `burn()` implementation in **AlchemistV3.sol** does not cache the fee amount; instead, it recalculates it three separate times throughout the function. This redundancy increases gas consumption unnecessarily. Similarly, the `repay()` function calculates and stores the fee amount, but does not use the cached amount and recalculates it multiple times.

We recommend computing the fee **once**, caching it in a local `uint256` variable, and reusing that value wherever needed. This simple optimization reduces redundant arithmetic operations and improves overall execution efficiency.

As **Table 1** illustrates, this change yields tangible savings of **approximately 45,000–113,000 gas units** per successful `burn()` transaction. Similarly, **Table 2** illustrates gas savings of about **1,300-65,000 gas units**. Naturally, these savings do not apply to reverted transactions, as they never reach this computation.

Given the **central role of `AlchemistV3.sol` as the core protocol contract**, optimizing frequently used functions like `burn()` offers meaningful long-term benefits - improving performance, lowering transaction costs, and enhancing the protocol’s operational efficiency.

**Table 1:**

| Test Name                                           | Without Caching (gas) | With Caching (gas) | Gas Saved |
| --------------------------------------------------- | --------------------- | ------------------ | --------- |
| testBurn()                                          | 830,776               | 786,048            | 44,728    |
| testBurnNoLimit()                                   | 2,213,896             | 2,145,395          | 68,501    |
| testBurnSameBlock()                                 | 746,431               | 746,431            | 0         |
| testBurnWithEarmarkedDebt()                         | 2,873,567             | 2,760,338          | 113,229   |
| testBurnWithEarmarkedDebtFullyEarmarked()           | 2,094,367             | 2,021,995          | 72,372    |
| testBurnWithFee()                                   | 884,800               | 840,072            | 44,728    |
| testBurnZeroAmount()                                | 744,652               | 744,652            | 0         |
| testBurnZeroIdRevert()                              | 745,248               | 745,248            | 0         |
| testBurn\_variable\_burn\_amounts(uint256) (μ gas)  | 1,143,370             | 1,097,548          | 45,822    |
| testBurn\_variable\_burn\_amounts(uint256) (\~ gas) | 1,195,836             | 1,151,108          | 44,728    |

**Table 2:**

| Test Name                                                    | Without Caching (gas) | With Caching (gas) | Gas Saved |
| ------------------------------------------------------------ | --------------------- | ------------------ | --------- |
| testRepayInvalidIdRevert(uint256) (μ gas)                    | 718,902               | 718,902            | 0         |
| testRepayInvalidIdRevert(uint256) (\~ gas)                   | 718,902               | 718,902            | 0         |
| testRepaySameBlock()                                         | 719,774               | 719,774            | 0         |
| testRepayUnearmarkedDebtOnly()                               | 936,689               | 935,391            | 1,298     |
| testRepayUnearmarkedDebtOnly\_Variable\_Amount(uint256) (μ)  | 924,633               | 918,543            | 6,090     |
| testRepayUnearmarkedDebtOnly\_Variable\_Amount(uint256) (\~) | 916,299               | 915,001            | 1,298     |
| testRepayWithDifferentPrice()                                | 1,029,090             | 1,028,052          | 1,038     |
| testRepayWithEarmarkedDebt()                                 | 2,150,419             | 2,100,873          | 49,546    |
| testRepayWithEarmarkedDebtPartial()                          | 2,263,798             | 2,221,994          | 41,804    |
| testRepayWithEarmarkedDebtWithFee()                          | 2,229,805             | 2,180,259          | 49,546    |
| testRepayWithEarmarkedDebt\_MultiplePoke\_Broken()           | 2,322,198             | 2,257,168          | 65,030    |
| testRepayZeroAmount()                                        | 713,995               | 713,995            | 0         |
| testRepayZeroTokenIdRevert()                                 | 713,865               | 713,865            | 0         |

## Vulnerability Details

The inefficient code for burn and repay can be found in the`AlchemistV3.sol` in the range of lines line 483-486 and lines 534-542. In case of burn `convertDebtTokensToYield(credit) * protocolFee / BPS;` is unnecessarily calculated 3 times instead of once and stored as a variable. Similarly, for repay, the feeAmount is calculated 4 times and stored the first time calculated, but the variable is never used `creditToYield * protocolFee / BPS` we recommend to use the cached variable instead.

## Experimental Setup POC Details

**Step 1:** Run burn test with the code as is to extract initial reference values:

```
source .env && FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path "src/test/AlchemistV3.t.sol" --evm-version cancun --match-test testBurn
```

Similarly for repay it would be,

```
source .env && FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path "src/test/AlchemistV3.t.sol" --evm-version cancun --match-test testBurn
```

Before:

Sample output for burn

```
Ran 9 tests for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testBurn() (gas: 830776)
[PASS] testBurnNoLimit() (gas: 2213896)
[PASS] testBurnSameBlock() (gas: 746431)
[PASS] testBurnWithEarmarkedDebt() (gas: 2873567)
[PASS] testBurnWithEarmarkedDebtFullyEarmarked() (gas: 2094367)
[PASS] testBurnWithFee() (gas: 884800)
[PASS] testBurnZeroAmount() (gas: 744652)
[PASS] testBurnZeroIdRevert() (gas: 745248)
[PASS] testBurn_variable_burn_amounts(uint256) (runs: 256, μ: 1143370, ~: 1195836)
Suite result: ok. 9 passed; 0 failed; 0 skipped; finished in 2.48s (483.69ms CPU time)

Ran 1 test suite in 2.87s (2.48s CPU time): 9 tests passed, 0 failed, 0 skipped (9 total tests)
```

**Step 2:** Modify the `AlchemistV3.sol` to use caching to save gas and improve legibility:

E.g. modify the `burn()` function to look like this

```solidity
    /// @inheritdoc IAlchemistV3Actions
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientId);
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == _accounts[recipientId].lastMintBlock) revert CannotRepayOnMintBlock();

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before more is taken
        _sync(recipientId);

        uint256 debt;
        // Burning alAssets can only repay unearmarked debt
        _checkState((debt = _accounts[recipientId].debt - _accounts[recipientId].earmarked) > 0);

        uint256 credit = amount > debt ? debt : amount;

        // Must only burn enough tokens that the transmuter positions can still be fulfilled
        if (credit > totalSyntheticsIssued - ITransmuter(transmuter).totalLocked()) {
            revert BurnLimitExceeded(credit, totalSyntheticsIssued - ITransmuter(transmuter).totalLocked());
        }

        // Burn the tokens from the message sender
        TokenUtils.safeBurnFrom(debtToken, msg.sender, credit);

        // Debt is subject to protocol fee similar to redemptions
        uint256 protocolFeeAmount = convertDebtTokensToYield(credit) * protocolFee / BPS;
        _accounts[recipientId].collateralBalance -= protocolFeeAmount;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeAmount);
        _mytSharesDeposited -= protocolFeeAmount;

        // Update the recipient's debt.
        _subDebt(recipientId, credit);

        totalSyntheticsIssued -= credit;

        emit Burn(msg.sender, credit, recipientId);

        return credit;
    }
```

For `repay()` it would be:

```solidity
    /// @inheritdoc IAlchemistV3Actions
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientTokenId);
        Account storage account = _accounts[recipientTokenId];
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == account.lastMintBlock) revert CannotRepayOnMintBlock();

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(recipientTokenId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 yieldToDebt = convertYieldTokensToDebt(amount);
        uint256 credit = yieldToDebt > debt ? debt : yieldToDebt;
        uint256 creditToYield = convertDebtTokensToYield(credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
        cumulativeEarmarked -= earmarkPaidGlobal;

        // Debt is subject to protocol fee similar to redemptions
        uint256 feeAmount = creditToYield * protocolFee / BPS;
        if (feeAmount > account.collateralBalance) {
            revert("Not enough collateral to pay for debt fee");
        } else {
            account.collateralBalance -= feeAmount;
        }

        _subDebt(recipientTokenId, credit);

        // Transfer the repaid tokens to the transmuter.
        TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeAmount);
        _mytSharesDeposited -= feeAmount;

        emit Repay(msg.sender, amount, recipientTokenId, creditToYield);

        return creditToYield;
    }
```

**Step 3:** Run tests again:

```
After (Sample output burn):
Ran 9 tests for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testBurn() (gas: 786048)
[PASS] testBurnNoLimit() (gas: 2145395)
[PASS] testBurnSameBlock() (gas: 746431)
[PASS] testBurnWithEarmarkedDebt() (gas: 2760338)
[PASS] testBurnWithEarmarkedDebtFullyEarmarked() (gas: 2021995)
[PASS] testBurnWithFee() (gas: 840072)
[PASS] testBurnZeroAmount() (gas: 744652)
[PASS] testBurnZeroIdRevert() (gas: 745248)
[PASS] testBurn_variable_burn_amounts(uint256) (runs: 256, μ: 1097548, ~: 1151108)
Suite result: ok. 9 passed; 0 failed; 0 skipped; finished in 2.51s (457.79ms CPU time)
```

## Recommendation

Make the recommended changes to the `burn` and `repay` function of the `AlchemistV3.sol` contract and review the code base for further instances where substantial amounts of gas could be saved and increase usability of the protocol.

Implementing these optimizations in such a core contract not only yields tangible gas savings, that could be obtained via very easy and quick fixes, but also reinforces the protocol’s long-term efficiency, maintainability, code clarity, and trustworthiness. All key qualities for a system at the heart of user interactions.

## Proof of Concept

## Proof of Concept

`AlchemistV3.sol` with gas optimized `burn()` and `repay()` function. Please follow the gas optimization outlined in the description above, if you doubt the impact on gas savings.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "./interfaces/IAlchemistV3.sol";
import {ITokenAdapter} from "./interfaces/ITokenAdapter.sol";
import {ITransmuter} from "./interfaces/ITransmuter.sol";
import {IAlchemistV3Position} from "./interfaces/IAlchemistV3Position.sol";
import {IFeeVault} from "./interfaces/IFeeVault.sol";
import "./libraries/PositionDecay.sol";
import {TokenUtils} from "./libraries/TokenUtils.sol";
import {SafeCast} from "./libraries/SafeCast.sol";
import {Initializable} from "../lib/openzeppelin-contracts-upgradeable/contracts/proxy/utils/Initializable.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "./base/Errors.sol";
import {IERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IAlchemistTokenVault} from "./interfaces/IAlchemistTokenVault.sol";
import {IVaultV2} from "../lib/vault-v2/src/interfaces/IVaultV2.sol";

/// @title  AlchemistV3
/// @author Alchemix Finance
contract AlchemistV3 is IAlchemistV3, Initializable {
    using SafeCast for int256;
    using SafeCast for uint256;
    using SafeCast for int128;
    using SafeCast for uint128;

    uint256 public constant BPS = 10_000;
    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public constant ONE_Q128 = uint256(1) << 128;

    /// @inheritdoc IAlchemistV3Immutables
    string public constant version = "3.0.0";

    /// @inheritdoc IAlchemistV3State
    address public admin;

    /// @inheritdoc IAlchemistV3State
    address public alchemistFeeVault;

    /// @inheritdoc IAlchemistV3Immutables
    address public debtToken;

    /// @inheritdoc IAlchemistV3State
    address public myt;

    /// @inheritdoc IAlchemistV3State
    uint256 public underlyingConversionFactor;

    /// @inheritdoc IAlchemistV3State
    uint256 public cumulativeEarmarked;

    /// @inheritdoc IAlchemistV3State
    uint256 public depositCap;

    /// @inheritdoc IAlchemistV3State
    uint256 public lastEarmarkBlock;

    /// @inheritdoc IAlchemistV3State
    uint256 public lastRedemptionBlock;

    /// @inheritdoc IAlchemistV3State
    uint256 public lastTransmuterTokenBalance;

    /// @inheritdoc IAlchemistV3State
    uint256 public minimumCollateralization;

    /// @inheritdoc IAlchemistV3State
    uint256 public collateralizationLowerBound;

    /// @inheritdoc IAlchemistV3State
    uint256 public globalMinimumCollateralization;

    /// @inheritdoc IAlchemistV3State
    uint256 public totalDebt;

    /// @inheritdoc IAlchemistV3State
    uint256 public totalSyntheticsIssued;

    /// @inheritdoc IAlchemistV3State
    uint256 public protocolFee;

    /// @inheritdoc IAlchemistV3State
    uint256 public liquidatorFee;

    /// @inheritdoc IAlchemistV3State
    uint256 public repaymentFee;

    /// @inheritdoc IAlchemistV3State
    address public alchemistPositionNFT;

    /// @inheritdoc IAlchemistV3State
    address public protocolFeeReceiver;

    /// @inheritdoc IAlchemistV3State
    address public underlyingToken;

    /// @inheritdoc IAlchemistV3State
    address public tokenAdapter;

    /// @inheritdoc IAlchemistV3State
    address public transmuter;

    /// @inheritdoc IAlchemistV3State
    address public pendingAdmin;

    /// @inheritdoc IAlchemistV3State
    bool public depositsPaused;

    /// @inheritdoc IAlchemistV3State
    bool public loansPaused;

    /// @inheritdoc IAlchemistV3State
    mapping(address => bool) public guardians;

    /// @dev Weight of earmarked amount / total unearmarked debt
    uint256 private _earmarkWeight;

    /// @dev Weight of redemption amount / total earmarked debt
    uint256 private _redemptionWeight;

    /// @dev Weight of redeemed collateral and fees / value of total collateral
    uint256 private _collateralWeight;

    /// @dev Earmarked scaled by survival
    uint256 private _survivalAccumulator;

    /// @dev Total locked collateral.
    /// Locked collateral is the collateral that cannot be withdrawn due to LTV constraints
    uint256 private _totalLocked;

    /// @dev Total yield tokens deposited
    /// This is used to differentiate between tokens deposited into a CDP and balance of the contract
    uint256 private _mytSharesDeposited;

    /// @dev User accounts
    mapping(uint256 => Account) private _accounts;

    /// @dev Historic redemptions
    mapping(uint256 => RedemptionInfo) private _redemptions;

    modifier onlyAdmin() {
        if (msg.sender != admin) {
            revert Unauthorized();
        }
        _;
    }

    modifier onlyAdminOrGuardian() {
        if (msg.sender != admin && !guardians[msg.sender]) {
            revert Unauthorized();
        }
        _;
    }

    modifier onlyTransmuter() {
        if (msg.sender != transmuter) {
            revert Unauthorized();
        }
        _;
    }

    constructor() initializer {}

    function initialize(AlchemistInitializationParams memory params) external initializer {
        _checkArgument(params.protocolFee <= BPS);
        _checkArgument(params.liquidatorFee <= BPS);
        _checkArgument(params.repaymentFee <= BPS);

        debtToken = params.debtToken;
        underlyingToken = params.underlyingToken;
        underlyingConversionFactor = 10 ** (TokenUtils.expectDecimals(params.debtToken) - TokenUtils.expectDecimals(params.underlyingToken));
        depositCap = params.depositCap;
        minimumCollateralization = params.minimumCollateralization;
        globalMinimumCollateralization = params.globalMinimumCollateralization;
        collateralizationLowerBound = params.collateralizationLowerBound;
        admin = params.admin;
        transmuter = params.transmuter;
        protocolFee = params.protocolFee;
        protocolFeeReceiver = params.protocolFeeReceiver;
        liquidatorFee = params.liquidatorFee;
        repaymentFee = params.repaymentFee;
        lastEarmarkBlock = block.number;
        lastRedemptionBlock = block.number;
        myt = params.myt;
    }

    /// @notice Emitted when a new Position NFT is minted.
    event AlchemistV3PositionNFTMinted(address indexed to, uint256 indexed tokenId);

    /// @notice Sets the NFT position token, callable by admin.
    function setAlchemistPositionNFT(address nft) external onlyAdmin {
        if (nft == address(0)) {
            revert AlchemistV3NFTZeroAddressError();
        }

        if (alchemistPositionNFT != address(0)) {
            revert AlchemistV3NFTAlreadySetError();
        }

        alchemistPositionNFT = nft;
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setAlchemistFeeVault(address value) external onlyAdmin {
        if (IFeeVault(value).token() != underlyingToken) {
            revert AlchemistVaultTokenMismatchError();
        }
        alchemistFeeVault = value;
        emit AlchemistFeeVaultUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setPendingAdmin(address value) external onlyAdmin {
        pendingAdmin = value;

        emit PendingAdminUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function acceptAdmin() external {
        _checkState(pendingAdmin != address(0));

        if (msg.sender != pendingAdmin) {
            revert Unauthorized();
        }

        admin = pendingAdmin;
        pendingAdmin = address(0);

        emit AdminUpdated(admin);
        emit PendingAdminUpdated(address(0));
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setDepositCap(uint256 value) external onlyAdmin {
        _checkArgument(value >= IERC20(myt).balanceOf(address(this)));

        depositCap = value;
        emit DepositCapUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setProtocolFeeReceiver(address value) external onlyAdmin {
        _checkArgument(value != address(0));

        protocolFeeReceiver = value;
        emit ProtocolFeeReceiverUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setProtocolFee(uint256 fee) external onlyAdmin {
        _checkArgument(fee <= BPS);

        protocolFee = fee;
        emit ProtocolFeeUpdated(fee);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setLiquidatorFee(uint256 fee) external onlyAdmin {
        _checkArgument(fee <= BPS);

        liquidatorFee = fee;
        emit LiquidatorFeeUpdated(fee);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setRepaymentFee(uint256 fee) external onlyAdmin {
        _checkArgument(fee <= BPS);

        repaymentFee = fee;
        emit RepaymentFeeUpdated(fee);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setTokenAdapter(address value) external onlyAdmin {
        _checkArgument(value != address(0));

        tokenAdapter = value;
        emit TokenAdapterUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setGuardian(address guardian, bool isActive) external onlyAdmin {
        _checkArgument(guardian != address(0));

        guardians[guardian] = isActive;
        emit GuardianSet(guardian, isActive);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setMinimumCollateralization(uint256 value) external onlyAdmin {
        _checkArgument(value >= FIXED_POINT_SCALAR);
        minimumCollateralization = value;

        emit MinimumCollateralizationUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setGlobalMinimumCollateralization(uint256 value) external onlyAdmin {
        _checkArgument(value >= minimumCollateralization);
        globalMinimumCollateralization = value;
        emit GlobalMinimumCollateralizationUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setCollateralizationLowerBound(uint256 value) external onlyAdmin {
        _checkArgument(value <= minimumCollateralization);
        _checkArgument(value >= FIXED_POINT_SCALAR);
        collateralizationLowerBound = value;
        emit CollateralizationLowerBoundUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function pauseDeposits(bool isPaused) external onlyAdminOrGuardian {
        depositsPaused = isPaused;
        emit DepositsPaused(isPaused);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function pauseLoans(bool isPaused) external onlyAdminOrGuardian {
        loansPaused = isPaused;
        emit LoansPaused(isPaused);
    }

    /// @inheritdoc IAlchemistV3State
    function getCDP(uint256 tokenId) external view returns (uint256, uint256, uint256) {
        (uint256 debt, uint256 earmarked, uint256 collateral) = _calculateUnrealizedDebt(tokenId);
        return (collateral, debt, earmarked);
    }

    /// @inheritdoc IAlchemistV3State
    function getTotalDeposited() external view returns (uint256) {
        return IERC20(myt).balanceOf(address(this));
    }

    /// @inheritdoc IAlchemistV3State
    function getMaxBorrowable(uint256 tokenId) external view returns (uint256) {
        (uint256 debt,, uint256 collateral) = _calculateUnrealizedDebt(tokenId);
        uint256 debtValueOfCollateral = convertYieldTokensToDebt(collateral);
        uint256 capacity = (debtValueOfCollateral * FIXED_POINT_SCALAR / minimumCollateralization);
        return debt > capacity  ? 0 : capacity - debt;
    }

    /// @inheritdoc IAlchemistV3State
    function mintAllowance(uint256 ownerTokenId, address spender) external view returns (uint256) {
        Account storage account = _accounts[ownerTokenId];
        return account.mintAllowances[account.allowancesVersion][spender];
    }

    /// @inheritdoc IAlchemistV3State
    function getTotalUnderlyingValue() external view returns (uint256) {
        return _getTotalUnderlyingValue();
    }

    /// @inheritdoc IAlchemistV3State
    function totalValue(uint256 tokenId) public view returns (uint256) {
        uint256 totalUnderlying;
        (,, uint256 collateral) = _calculateUnrealizedDebt(tokenId);
        if (collateral > 0) totalUnderlying += convertYieldTokensToUnderlying(collateral);
        return normalizeUnderlyingTokensToDebt(totalUnderlying);
    }

    /// @inheritdoc IAlchemistV3Actions
    function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
        _checkArgument(recipient != address(0));
        _checkArgument(amount > 0);
        _checkState(!depositsPaused);
        _checkState(_mytSharesDeposited + amount <= depositCap);

        // Only mint a new position if the id is 0
        if (tokenId == 0) {
            tokenId = IAlchemistV3Position(alchemistPositionNFT).mint(recipient);
            emit AlchemistV3PositionNFTMinted(recipient, tokenId);
        } else {
            _checkForValidAccountId(tokenId);
        }

        _accounts[tokenId].collateralBalance += amount;

        // Transfer tokens from msg.sender now that the internal storage updates have been committed.
        TokenUtils.safeTransferFrom(myt, msg.sender, address(this), amount);
        _mytSharesDeposited += amount;

        emit Deposit(amount, tokenId);

        return convertYieldTokensToDebt(amount);
    }

    /// @inheritdoc IAlchemistV3Actions
    function withdraw(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
        _checkArgument(recipient != address(0));
        _checkForValidAccountId(tokenId);
        _checkArgument(amount > 0);
        _checkAccountOwnership(IAlchemistV3Position(alchemistPositionNFT).ownerOf(tokenId), msg.sender);
        _earmark();

        _sync(tokenId);

        uint256 lockedCollateral = convertDebtTokensToYield(_accounts[tokenId].debt) * minimumCollateralization / FIXED_POINT_SCALAR;
        _checkArgument(_accounts[tokenId].collateralBalance - lockedCollateral >= amount);

        _accounts[tokenId].collateralBalance -= amount;

        // Assure that the collateralization invariant is still held.
        _validate(tokenId);

        // Transfer the yield tokens to msg.sender
        TokenUtils.safeTransfer(myt, recipient, amount);
        _mytSharesDeposited -= amount;

        emit Withdraw(amount, tokenId, recipient);

        return amount;
    }

    /// @inheritdoc IAlchemistV3Actions
    function mint(uint256 tokenId, uint256 amount, address recipient) external {
        _checkArgument(recipient != address(0));
        _checkForValidAccountId(tokenId);
        _checkArgument(amount > 0);
        _checkState(!loansPaused);
        _checkAccountOwnership(IAlchemistV3Position(alchemistPositionNFT).ownerOf(tokenId), msg.sender);

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before more is taken
        _sync(tokenId);

        // Mint tokens to recipient
        _mint(tokenId, amount, recipient);
    }

    /// @inheritdoc IAlchemistV3Actions
    function mintFrom(uint256 tokenId, uint256 amount, address recipient) external {
        _checkArgument(amount > 0);
        _checkForValidAccountId(tokenId);
        _checkArgument(recipient != address(0));
        _checkState(!loansPaused);
        // Preemptively try and decrease the minting allowance. This will save gas when the allowance is not sufficient.
        _decreaseMintAllowance(tokenId, msg.sender, amount);

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before more is taken
        _sync(tokenId);

        // Mint tokens from the tokenId's account to the recipient.
        _mint(tokenId, amount, recipient);
    }

    /// @inheritdoc IAlchemistV3Actions
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientId);
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == _accounts[recipientId].lastMintBlock) revert CannotRepayOnMintBlock();

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before more is taken
        _sync(recipientId);

        uint256 debt;
        // Burning alAssets can only repay unearmarked debt
        _checkState((debt = _accounts[recipientId].debt - _accounts[recipientId].earmarked) > 0);

        uint256 credit = amount > debt ? debt : amount;

        // Must only burn enough tokens that the transmuter positions can still be fulfilled
        if (credit > totalSyntheticsIssued - ITransmuter(transmuter).totalLocked()) {
            revert BurnLimitExceeded(credit, totalSyntheticsIssued - ITransmuter(transmuter).totalLocked());
        }

        // Burn the tokens from the message sender
        TokenUtils.safeBurnFrom(debtToken, msg.sender, credit);

        // Debt is subject to protocol fee similar to redemptions
        uint256 protocolFeeAmount = convertDebtTokensToYield(credit) * protocolFee / BPS;
        _accounts[recipientId].collateralBalance -= protocolFeeAmount;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeAmount);
        _mytSharesDeposited -= protocolFeeAmount;

        // Update the recipient's debt.
        _subDebt(recipientId, credit);

        totalSyntheticsIssued -= credit;

        emit Burn(msg.sender, credit, recipientId);

        return credit;
    }

    /// @inheritdoc IAlchemistV3Actions
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientTokenId);
        Account storage account = _accounts[recipientTokenId];
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == account.lastMintBlock) revert CannotRepayOnMintBlock();

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(recipientTokenId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 yieldToDebt = convertYieldTokensToDebt(amount);
        uint256 credit = yieldToDebt > debt ? debt : yieldToDebt;
        uint256 creditToYield = convertDebtTokensToYield(credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
        cumulativeEarmarked -= earmarkPaidGlobal;

        // Debt is subject to protocol fee similar to redemptions
        uint256 feeAmount = creditToYield * protocolFee / BPS;
        if (feeAmount > account.collateralBalance) {
            revert("Not enough collateral to pay for debt fee");
        } else {
            account.collateralBalance -= feeAmount;
        }

        _subDebt(recipientTokenId, credit);

        // Transfer the repaid tokens to the transmuter.
        TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeAmount);
        _mytSharesDeposited -= feeAmount;

        emit Repay(msg.sender, amount, recipientTokenId, creditToYield);

        return creditToYield;
    }

    /// @inheritdoc IAlchemistV3Actions
    function liquidate(uint256 accountId) external override returns (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        _checkForValidAccountId(accountId);
        (yieldAmount, feeInYield, feeInUnderlying) = _liquidate(accountId);
        if (yieldAmount > 0) {
            return (yieldAmount, feeInYield, feeInUnderlying);
        } else {
            // no liquidation amount returned, so no liquidation happened
            revert LiquidationError();
        }
    }

    /// @inheritdoc IAlchemistV3Actions
    function batchLiquidate(uint256[] memory accountIds)
        external
        returns (uint256 totalAmountLiquidated, uint256 totalFeesInYield, uint256 totalFeesInUnderlying)
    {
        if (accountIds.length == 0) {
            revert MissingInputData();
        }

        for (uint256 i = 0; i < accountIds.length; i++) {
            uint256 accountId = accountIds[i];
            if (accountId == 0 || !_tokenExists(alchemistPositionNFT, accountId)) {
                continue;
            }
            (uint256 underlyingAmount, uint256 feeInYield, uint256 feeInUnderlying) = _liquidate(accountId);
            totalAmountLiquidated += underlyingAmount;
            totalFeesInYield += feeInYield;
            totalFeesInUnderlying += feeInUnderlying;
        }

        if (totalAmountLiquidated > 0) {
            return (totalAmountLiquidated, totalFeesInYield, totalFeesInUnderlying);
        } else {
            // no total liquidation amount returned, so no liquidations happened
            revert LiquidationError();
        }
    }

    /// @inheritdoc IAlchemistV3Actions
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;

        // observed transmuter pre-balance -> potential cover
        uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0;
        uint256 coverDebt = convertYieldTokensToDebt(deltaYield);

        // cap cover so we never consume beyond remaining earmarked
        uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;

       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;

        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

        lastRedemptionBlock = block.number;

        // consume the observed cover so it can't be reused
        if (deltaYield != 0) {
            uint256 usedYield = convertDebtTokensToYield(coverToApplyDebt);
            lastTransmuterTokenBalance = transmuterBal > usedYield ? transmuterBal - usedYield : transmuterBal;
        }

        // move only the net collateral + fee
        uint256 collRedeemed  = convertDebtTokensToYield(amount);
        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
        uint256 totalOut      = collRedeemed + feeCollateral;

        // update locked collateral + collateral weight
        uint256 old = _totalLocked;
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
        _mytSharesDeposited -= collRedeemed + feeCollateral;

        emit Redemption(redeemedDebtTotal);
    }

    ///@inheritdoc IAlchemistV3Actions
    function reduceSyntheticsIssued(uint256 amount) external onlyTransmuter {
        totalSyntheticsIssued -= amount;
    }

    ///@inheritdoc IAlchemistV3Actions
    function setTransmuterTokenBalance(uint256 amount) external onlyTransmuter {
        lastTransmuterTokenBalance = amount;
    }

    /// @inheritdoc IAlchemistV3Actions
    function poke(uint256 tokenId) external {
        _checkForValidAccountId(tokenId);
        _earmark();
        _sync(tokenId);
    }

    /// @inheritdoc IAlchemistV3Actions
    function approveMint(uint256 tokenId, address spender, uint256 amount) external {
        _checkAccountOwnership(IAlchemistV3Position(alchemistPositionNFT).ownerOf(tokenId), msg.sender);
        _approveMint(tokenId, spender, amount);
    }

    /// @inheritdoc IAlchemistV3Actions
    function resetMintAllowances(uint256 tokenId) external {
        // Allow calls from either the token owner or the NFT contract
        if (msg.sender != address(alchemistPositionNFT)) {
            // Direct call - verify caller is current owner
            address tokenOwner = IERC721(alchemistPositionNFT).ownerOf(tokenId);
            if (msg.sender != tokenOwner) {
                revert Unauthorized();
            }
        }
        // increment version to start the mapping from a fresh state
        _accounts[tokenId].allowancesVersion += 1;
        // Emit event to notify allowance clearing
        emit MintAllowancesReset(tokenId);
    }

    /// @inheritdoc IAlchemistV3State
    function convertYieldTokensToDebt(uint256 amount) public view returns (uint256) {
        return normalizeUnderlyingTokensToDebt(convertYieldTokensToUnderlying(amount));
    }

    /// @inheritdoc IAlchemistV3State
    function convertDebtTokensToYield(uint256 amount) public view returns (uint256) {
        return convertUnderlyingTokensToYield(normalizeDebtTokensToUnderlying(amount));
    }

    /// @inheritdoc IAlchemistV3State
    function convertYieldTokensToUnderlying(uint256 amount) public view returns (uint256) {
        return IVaultV2(myt).convertToAssets(amount);
    }

    /// @inheritdoc IAlchemistV3State
    function convertUnderlyingTokensToYield(uint256 amount) public view returns (uint256) {
        return IVaultV2(myt).convertToShares(amount);
    }

    /// @inheritdoc IAlchemistV3State
    function normalizeUnderlyingTokensToDebt(uint256 amount) public view returns (uint256) {
        return amount * underlyingConversionFactor;
    }

    /// @inheritdoc IAlchemistV3State
    function normalizeDebtTokensToUnderlying(uint256 amount) public view returns (uint256) {
        return amount / underlyingConversionFactor;
    }

    /// @dev Mints debt tokens to `recipient` using the account owned by `tokenId`.
    /// @param tokenId     The tokenId of the account to mint from.
    /// @param amount    The amount to mint.
    /// @param recipient The recipient of the minted debt tokens.
    function _mint(uint256 tokenId, uint256 amount, address recipient) internal {
        _addDebt(tokenId, amount);

        totalSyntheticsIssued += amount;

        // Validate the tokenId's account to assure that the collateralization invariant is still held.
        _validate(tokenId);

        _accounts[tokenId].lastMintBlock = block.number;

        // Mint the debt tokens to the recipient.
        TokenUtils.safeMint(debtToken, recipient, amount);

        emit Mint(tokenId, amount, recipient);
    }

    /**
     * @notice Force repays earmarked debt of the account owned by `accountId` using account's collateral balance.
     * @param accountId The tokenId of the account to repay from.
     * @param amount The amount to repay in debt tokens.
     * @return creditToYield The amount of yield tokens repaid.
     */
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }

    /// @dev Fetches and applies the liquidation amount to account `tokenId` if the account collateral ratio touches `collateralizationLowerBound`.
    /// @dev Repays earmarked debt if it exists
    /// @dev If earmarked repayment restores account to healthy collateralization, no liquidation is performed. Caller receives a repayment fee.
    /// @param accountId  The tokenId of the account to to liquidate.
    /// @return amountLiquidated  The amount (in yield tokens) removed from the account `tokenId`.
    /// @return feeInYield The additional fee as a % of the liquidation amount to be sent to the liquidator
    /// @return feeInUnderlying The additional fee as a % of the liquidation amount, denominated in underlying token, to be sent to the liquidator
    function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
        // Query transmuter and earmark global debt
        _earmark();
        // Sync current user debt before deciding how much needs to be liquidated
        _sync(accountId);

        Account storage account = _accounts[accountId];

        // Early return if no debt exists
        if (account.debt == 0) {
            return (0, 0, 0);
        }

        // In the rare scenario where 1 share is worth 0 underlying asset
        if (IVaultV2(myt).convertToAssets(1e18) == 0) {
            return (0, 0, 0);
        }

        // Calculate initial collateralization ratio
        uint256 collateralInUnderlying = totalValue(accountId);
        uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        // If account is healthy, nothing to liquidate
        if (collateralizationRatio > collateralizationLowerBound) {
            return (0, 0, 0);
        }

        // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
    }

    /// @dev Performs the actual liquidation logic when collateralization is below the lower bound
    /// @param accountId The tokenId of the account to to liquidate.
    /// @param collateralInUnderlying The total collateral value of the account in debt tokens.
    /// @param repaidAmountInYield The amount of debt repaid in yield tokens.
    /// @return amountLiquidated The amount of yield tokens liquidated.
    /// @return feeInYield The fee in yield tokens to be sent to the liquidator.
    /// @return feeInUnderlying The fee in underlying tokens to be sent to the liquidator.
    function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

        // Handle outsourced fee from vault
        if (outsourcedFee > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee);
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }
        }

        emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
        return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    }

    /// @dev Handles repayment fee calculation and account deduction
    /// @param accountId The tokenId of the account to force a repayment on.
    /// @param repaidAmountInYield The amount of debt repaid in yield tokens.
    /// @return fee The fee in yield tokens to be sent to the liquidator.
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }

    /// @dev Increases the debt by `amount` for the account owned by `tokenId`.
    ///
    /// @param tokenId   The account owned by tokenId.
    /// @param amount  The amount to increase the debt by.
    function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

        account.rawLocked = lockedCollateral + toLock;
        _totalLocked += toLock;
        account.debt += amount;
        totalDebt += amount;
    }

    /// @dev Subtracts the debt by `amount` for the account owned by `tokenId`.
    ///
    /// @param tokenId   The account owned by tokenId.
    /// @param amount  The amount to decrease the debt by.
    function _subDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        // For cases when someone above minimum LTV gets liquidated.
        if (toFree > _totalLocked) {
            toFree = _totalLocked;
        }

        account.debt -= amount;
        totalDebt -= amount;
        _totalLocked -= toFree;
        account.rawLocked = lockedCollateral - toFree;

        // Clamp to avoid underflow due to rounding later at a later time
        if (cumulativeEarmarked > totalDebt) {
            cumulativeEarmarked = totalDebt;
        }
    }

    /// @dev Set the mint allowance for `spender` to `amount` for the account owned by `tokenId`.
    ///
    /// @param ownerTokenId   The id of the account granting approval.
    /// @param spender The address of the spender.
    /// @param amount  The amount of debt tokens to set the mint allowance to.
    function _approveMint(uint256 ownerTokenId, address spender, uint256 amount) internal {
        Account storage account = _accounts[ownerTokenId];
        account.mintAllowances[account.allowancesVersion][spender] = amount;
        emit ApproveMint(ownerTokenId, spender, amount);
    }

    /// @dev Decrease the mint allowance for `spender` by `amount` for the account owned by `ownerTokenId`.
    ///
    /// @param ownerTokenId The id of the account owner.
    /// @param spender The address of the spender.
    /// @param amount  The amount of debt tokens to decrease the mint allowance by.
    function _decreaseMintAllowance(uint256 ownerTokenId, address spender, uint256 amount) internal {
        Account storage account = _accounts[ownerTokenId];
        account.mintAllowances[account.allowancesVersion][spender] -= amount;
    }

    /// @dev Checks an expression and reverts with an {IllegalArgument} error if the expression is {false}.
    ///
    /// @param expression The expression to check.
    function _checkArgument(bool expression) internal pure {
        if (!expression) {
            revert IllegalArgument();
        }
    }

    /// @dev Checks if owner == sender and reverts with an {UnauthorizedAccountAccessError} error if the result is {false}.
    ///
    /// @param owner The address of the owner of an account.
    /// @param user The address of the user attempting to access an account.
    function _checkAccountOwnership(address owner, address user) internal pure {
        if (owner != user) {
            revert UnauthorizedAccountAccessError();
        }
    }

    /// @dev reverts {UnknownAccountOwnerIDError} error by if no owner exists.
    ///
    /// @param tokenId The id of an account.
    function _checkForValidAccountId(uint256 tokenId) internal view {
        if (!_tokenExists(alchemistPositionNFT, tokenId)) {
            revert UnknownAccountOwnerIDError();
        }
    }

    /**
     * @notice Checks whether a token id is linked to an owner. Non blocking / no reverts.
     * @param nft The address of the ERC721 based contract.
     * @param tokenId The token id to check.
     * @return exists A boolean that is true if the token exists.
     */
    function _tokenExists(address nft, uint256 tokenId) internal view returns (bool exists) {
        if (tokenId == 0) {
            // token ids start from 1
            return false;
        }
        try IERC721(nft).ownerOf(tokenId) {
            // If the call succeeds, the token exists.
            exists = true;
        } catch {
            // If the call fails, then the token does not exist.
            exists = false;
        }
    }

    /// @dev Checks an expression and reverts with an {IllegalState} error if the expression is {false}.
    ///
    /// @param expression The expression to check.
    function _checkState(bool expression) internal pure {
        if (!expression) {
            revert IllegalState();
        }
    }

    /// @dev Checks that the account owned by `tokenId` is properly collateralized.
    /// @dev If the account is undercollateralized then this will revert with an {Undercollateralized} error.
    ///
    /// @param tokenId The id of the account owner.
    function _validate(uint256 tokenId) internal view {
        if (_isUnderCollateralized(tokenId)) revert Undercollateralized();
    }

    /// @dev Update the user's earmarked and redeemed debt amounts.
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];

        // Collateral to remove from redemptions and fees
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
        account.collateralBalance -= collateralToRemove;

        // Redemption survival now and at last sync
        // Survival is the amount of earmark that is left after a redemption
        uint256 redemptionSurvivalOld = PositionDecay.SurvivalFromWeight(account.lastAccruedRedemptionWeight);
        if (redemptionSurvivalOld == 0) redemptionSurvivalOld = ONE_Q128;
        uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
        // Survival during current sync window
        uint256 survivalRatio = _divQ128(redemptionSurvivalNew, redemptionSurvivalOld);
        // User exposure at last sync used to calculate newly earmarked debt pre redemption
        uint256 userExposure = account.debt > account.earmarked ? account.debt - account.earmarked : 0;
        uint256 earmarkRaw = PositionDecay.ScaleByWeightDelta(userExposure, _earmarkWeight - account.lastAccruedEarmarkWeight);

        // Earmark survival at last sync
        // Survival is the amount of unearmarked debt left after an earmark
        uint256 earmarkSurvival = PositionDecay.SurvivalFromWeight(account.lastAccruedEarmarkWeight);
        if (earmarkSurvival == 0) earmarkSurvival = ONE_Q128;
        // Decay snapshot by what was redeemed from last sync until now
        uint256 decayedRedeemed = _mulQ128(account.lastSurvivalAccumulator, survivalRatio);
        // What was added to the survival accumulator in the current sync window
        uint256 survivalDiff = _survivalAccumulator > decayedRedeemed ? _survivalAccumulator - decayedRedeemed : 0;

        // Unwind accumulated earmarked at last sync
        uint256 unredeemedRatio = _divQ128(survivalDiff, earmarkSurvival);
        // Portion of earmark that remains after applying the redemption. Scaled back from 128.128
        uint256 earmarkedUnredeemed = _mulQ128(userExposure, unredeemedRatio);
        if (earmarkedUnredeemed > earmarkRaw) earmarkedUnredeemed = earmarkRaw;

        // Old earmarks that survived redemptions in the current sync window
        uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio);
        // What was redeemed from the newly earmark between last sync and now
        uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed;
        // Total overall earmarked to adjust user debt
        uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

        account.earmarked = exposureSurvival + earmarkedUnredeemed;
        account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;

        // Update locked collateral
        account.rawLocked = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        // Advance account checkpoint
        account.lastCollateralWeight = _collateralWeight;
        account.lastAccruedEarmarkWeight = _earmarkWeight;
        account.lastAccruedRedemptionWeight = _redemptionWeight;

        // Snapshot G for this account
        account.lastSurvivalAccumulator = _survivalAccumulator;
    }

    /// @dev Earmarks the debt for redemption.
    function _earmark() internal {
        if (totalDebt == 0) return;
        if (block.number <= lastEarmarkBlock) return;

        // Yield the transmuter accumulated since last earmark (cover)
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

        // Proper saturating subtract in DEBT units
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;

        lastTransmuterTokenBalance = transmuterCurrentBalance;

        uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
        if (amount > liveUnearmarked) amount = liveUnearmarked;

        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
            _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

            cumulativeEarmarked += amount;
        }

        lastEarmarkBlock = block.number;
    }

    /// @dev Gets the amount of debt that the account owned by `owner` will have after a sync occurs.
    ///
    /// @param tokenId The id of the account owner.
    ///
    /// @return The amount of debt that the account owned by `owner` will have after an update.
    /// @return The amount of debt which is currently earmarked fro redemption.
    /// @return The amount of collateral that has yet to be redeemed.
    function _calculateUnrealizedDebt(uint256 tokenId)
        internal
        view
        returns (uint256, uint256, uint256)
    {
        Account storage account = _accounts[tokenId];

        // Local copies
        uint256 earmarkWeightCopy = _earmarkWeight;
        uint256 survivalAccumulatorCopy   = _survivalAccumulator;

        // Simulate earmark since lastEarmarkBlock
        if (block.number > lastEarmarkBlock) {
            uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
            uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

            uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

            // cover in DEBT units
            uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
            amount = amount > coverInDebt ? amount - coverInDebt : 0;

            uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
            if (amount > liveUnearmarked) amount = liveUnearmarked;

            if (amount > 0 && liveUnearmarked != 0) {
                // Previous earmark survival
                uint256 previousSurvival = PositionDecay.SurvivalFromWeight(earmarkWeightCopy);
                if (previousSurvival == 0) previousSurvival = ONE_Q128;

                // Fraction of unearmarked debt being earmarked now in UQ128.128
                uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

                survivalAccumulatorCopy += _mulQ128(previousSurvival, earmarkedFraction);
                earmarkWeightCopy += PositionDecay.WeightIncrement(amount, liveUnearmarked);
            }
        }

        // Redemption survival now and at last sync
        // Survival is the amount of earmark that is left after a redemption
        uint256 redemptionSurvivalOld = PositionDecay.SurvivalFromWeight(account.lastAccruedRedemptionWeight);
        if (redemptionSurvivalOld == 0) redemptionSurvivalOld = ONE_Q128;
        uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
        // Survival during the current sync window
        uint256 survivalRatio = _divQ128(redemptionSurvivalNew, redemptionSurvivalOld);

        // User exposure at last sync used to calculate newly earmarked debt pre redemption
        uint256 userExposure = account.debt > account.earmarked ? account.debt - account.earmarked : 0;
        uint256 earmarkRaw = PositionDecay.ScaleByWeightDelta(userExposure, earmarkWeightCopy - account.lastAccruedEarmarkWeight);

        // Earmark survival at last sync
        // Survival is the amount of unearmarked debt left after an earmark
        uint256 earmarkSurvival = PositionDecay.SurvivalFromWeight(account.lastAccruedEarmarkWeight);
        if (earmarkSurvival == 0) earmarkSurvival = ONE_Q128;
        // Decay snapshot by what was redeemed from last sync until now
        uint256 decayedRedeemed = _mulQ128(account.lastSurvivalAccumulator, survivalRatio);
        // What was added to the survival accumulator in the current sync window
        uint256 survivalDiff  = survivalAccumulatorCopy > decayedRedeemed ? survivalAccumulatorCopy - decayedRedeemed : 0;

        // Unwind accumulated earmarked at last sync
        uint256 unredeemedRatio = _divQ128(survivalDiff, earmarkSurvival);
        // Portion of earmark that remains after applying the redemption. Scaled back from 128.128
        uint256 earmarkedUnredeemed = _mulQ128(userExposure, unredeemedRatio);
        if (earmarkedUnredeemed > earmarkRaw) earmarkedUnredeemed = earmarkRaw;

        // Old earmarks that survived redemptions in the current sync window
        uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio);

        // What was redeemed from the newly earmark between last sync and now
        uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed;
        // Total overall earmarked to adjust user debt
        uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

        uint256 newDebt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;
        uint256 newEarmarked = exposureSurvival + earmarkedUnredeemed;

        // Collateral from fees and redemptions
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
        uint256 newCollateral = account.collateralBalance - collateralToRemove;

        return (newDebt, newEarmarked, newCollateral);
    }

    /// @dev Checks that the account owned by `tokenId` is properly collateralized.
    /// @dev Returns true only if the account is undercollateralized
    ///
    /// @param tokenId The id of the account owner.
    function _isUnderCollateralized(uint256 tokenId) internal view returns (bool) {
        uint256 debt = _accounts[tokenId].debt;
        if (debt == 0) return false;

        uint256 collateralization = totalValue(tokenId) * FIXED_POINT_SCALAR / debt;
        return collateralization < minimumCollateralization;
    }

    /// @dev Calculates the total value of the alchemist in the underlying token.
    /// @return totalUnderlyingValue The total value of the alchemist in the underlying token.
    function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
        uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }

    /// @inheritdoc IAlchemistV3State
    function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {
        if (debt >= collateral) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt if debt is greater than collateral
            return (collateral, debt, 0, outsourcedFee);
        }

        if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt in high ltv global environment
            return (debt, debt, 0, outsourcedFee);
        }

        // fee is taken from surplus = collateral - debt
        uint256 surplus = collateral > debt ? collateral - debt : 0;

        fee = (surplus * feeBps) / BPS;

        // collateral remaining for margin‐restore calc
        uint256 adjCollat = collateral - fee;

        // compute m*d  (both plain units)
        uint256 md = (targetCollateralization * debt) / FIXED_POINT_SCALAR;

        // if md <= adjCollat, nothing to liquidate
        if (md <= adjCollat) {
            return (0, 0, fee, 0);
        }

        // numerator = md - adjCollat
        uint256 num = md - adjCollat;

        // denom = m - 1  =>  (targetCollateralization - FIXED_POINT_SCALAR)/FIXED_POINT_SCALAR
        uint256 denom = targetCollateralization - FIXED_POINT_SCALAR;

        // debtToBurn = (num * FIXED_POINT_SCALAR) / denom
        debtToBurn = (num * FIXED_POINT_SCALAR) / denom;

        // gross collateral seize = net + fee
        grossCollateralToSeize = debtToBurn + fee;
    }

    // Math helpers for Q128.128
    function _mulQ128(uint256 aQ, uint256 bQ) private pure returns (uint256 z) {
        if (aQ == 0 || bQ == 0) return 0;
        uint256 lo;
        uint256 hi;
        assembly {
            // 512-bit product [hi lo] = aQ * bQ
            let mm := mulmod(aQ, bQ, not(0))
            lo := mul(aQ, bQ)
            hi := sub(sub(mm, lo), lt(mm, lo))
        }
        // floor((a*b) / 2^128)
        z = (hi << 128) | (lo >> 128);
        // if there are non-zero low bits, round up
        if (lo & ((uint256(1) << 128) - 1) != 0) {
            unchecked {
                z += 1;
            }
        }
    }

    function _divQ128(uint256 numerQ128, uint256 denomQ128) private pure returns (uint256) {
        if (numerQ128 == 0) return 0;
        unchecked {
            // Fast path: shifting is safe if numerQ128 < 2^128
            if (numerQ128 <= type(uint256).max >> 128) {
                return (numerQ128 << 128) / denomQ128;
            }
            // Slow path: numerQ128 can only be 2^128 here.
            uint256 q = numerQ128 / denomQ128; // 0 or 1 in our domain
            uint256 r = numerQ128 - q * denomQ128; // remainder
            return (q << 128) + ((r << 128) / denomQ128);
        }
    }
}
```
