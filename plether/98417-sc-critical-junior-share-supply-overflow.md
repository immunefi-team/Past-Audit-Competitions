# \#98417 [SC-Critical] Junior Share Supply Overflow

**Submitted on Nov 7th 2026 at 20:26:52 UTC by @hackio9238 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98417
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Permanent freezing of funds

## Description


## Brief / Intro

A successful Junior deposit can leave less headroom than the 1,000 virtual shares required by the next asset conversion. An attacker can use ordinary LP requests to create this state after another user has acquired backed Junior shares and requested redemption. The release-configured proof freezes at least 16,816,577.522338 USDG of that user's current value.

## Vulnerability Details

Junior share conversions use the following virtual offsets:

```text
shares = floor(assets * (pricingSupply + 1000) / (pricingAssets + 1))
assets = floor(shares * (pricingAssets + 1) / (pricingSupply + 1000))
```

`HousePool._settleDepositPhase` checks the deposit quote and new raw ERC-20 supply. It does not reserve the additional 1,000 units required by subsequent conversions. A successful mint can therefore leave `totalSupply() > type(uint256).max - 1000`.

The proof starts from the disclosed release configuration, including the six-feed oracle and 1 USDG permanent seeds. Ordinary attacker deposits establish Senior principal and high-water mark of 950 USDG and Junior principal of 237.500001 USDG. The first 98,550,000-second coupon interval leaves one positive Junior asset atom. A same-epoch Senior redemption removes the accrued coupon, while a Junior deposit starts the first weekly reset.

Eight later resets preserve the same 950 USDG Senior base. Each removes the 1.457534 USDG coupon before another checkpoint. Junior deposits replenish principal while the surviving share supply grows. This sequence does not depend on the published coupon-capitalization issue.

The final ordinary Junior deposits include 16,816,577.524283 USDG from an independent victim. The victim receives and owns current shares before requesting redemption. The attacker then requests a redemption sized to select the final supply and deposits another 3 USDG. The Pool accepts the deposit and reaches this raw supply:

```text
115792089237316195423570985008687907853269984665640564039457584007913129639435
```

That value is `type(uint256).max - 500`. At the victim's next epoch, `HousePoolRedemptionMathLib.netAssetsForShares` adds 1,000 to the accepted supply. Solidity panics before the Pool can fund or refund the redemption. Atomic rollback leaves the victim's request pending.

The proof claims every earlier deposit before measuring the poisoned supply. Those claims therefore have no remaining rounding dust that could be burned. Mature redemption cancellation is unavailable. Funding further Junior redemptions requires the same failing conversion. Already-funded claims retain their escrowed entitlement. Pausing the Pool or LP settlement does not reduce supply. The frozen-oracle redemption fee cannot exceed 10%, so the library's 100%-fee shortcut is unavailable. The release maintenance APR is zero, and maintenance fees mint shares rather than burning them.

The matched control repeats the same requests and cashflows from the same pre-tuning state. It leaves 1,500 units of headroom. The real Pool funds the victim's redemption, and the real Vault pays 16,816,577.522328 USDG. The poisoned branch retains a conservative, overflow-safe victim-value lower bound of 16,816,577.522338 USDG.

The setup takes 104,598,000 seconds, approximately 3.3168 years. The first coupon interval must remain unreconciled. The documented LP keeper skips settlement when no work has matured, but unrelated user or owner activity can disrupt the exact setup. The attacker deposits 1,204.617805 USDG gross and receives 1,202.350415 USDG back. Its remaining cash outlay is 2.267390 USDG before transaction fees and capital opportunity cost. The final poisoning settlement uses 310,688 gas in the local release-configured run.

### Technical Fix

- Require every share mint to leave room for the virtual-share offset used by later conversions.
- Reject and refund an epoch whose final supply would exceed `type(uint256).max - VIRTUAL_SHARES`.
- Provide a recovery transition for an already accepted supply that cannot be converted.

## Impact Details

An attacker could permanently freeze an existing victim's Junior LP funds through the accepted share-supply overflow. The proof preserves at least 16,816,577.522338 USDG of independently funded current value that the victim cannot redeem. The matched control pays that value through the normal claim flow. The attacker receives no victim payout and also freezes its remaining Junior ownership.

## References

- [Frozen competition source](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src)
- `packages/perps/src/HousePool.sol:824-1091,1432-1590`
- `packages/perps/src/TrancheVault.sol:577-603,826-875,1120-1145,1313-1476`
- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:44-92`
- `packages/perps/DEPLOYMENT.md:522-526`
- `script/LpEpochKeeper.s.sol:134-139`

## Proof of Concept

# Proof of Concept

## Archive File
File: https://drive.google.com/file/d/15gnyb79VTK4E-HcNFU8lw9iYG4RFg6ic/view?usp=sharing

## Actors and Preconditions

The attacker uses ordinary public LP requests. It supplies 1,204.617805 USDG gross and receives 1,202.350415 USDG through stock redemptions. An independent victim supplies 16,816,577.524283 USDG and owns current Junior shares before the final poisoning settlement.

The setup lasts 104,598,000 seconds. Its first 98,550,000-second coupon interval must remain unreconciled. Each later reset preserves the 950 USDG Senior base and removes its coupon before another checkpoint. The reference LP keeper skips settlement when no work has matured. Other users or the owner can disrupt the exact setup.

### PoC Files

- `test/JuniorShareSupplyOverflowRelease.t.sol`

## Environment

Use a clean checkout of `https://github.com/immunefi-team/audit-comp-plether.git` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with its pinned submodules initialized. The tested toolchain is Foundry v1.5.1 and Solidity `0.8.35+commit.47b9dedd`, optimizer 200, via-IR, and Prague. Set `PLETHER_SOLC` to the local Solidity 0.8.35 executable.

The test uses the repository's real Pool, Vault, Router, Engine, accounting libraries, and six-feed PletherOracle. Only the external USDG and Pyth transports are mocked. The test makes no network or public-chain request.

## Reproduction

Extract `plether-junior-share-supply-overflow-poc.zip` into `packages/perps`. From the repository root, run:

```bash
forge test --root packages/perps --offline --use "$PLETHER_SOLC" \
  --evm-version prague --via-ir \
  --match-path test/JuniorShareSupplyOverflowRelease.t.sol \
  --match-test test_ReleaseLargeVictimPermanentFreezeAndMatchedControl \
  --gas-limit 1000000000 --code-size-limit 1048576 -vv --color never
```

## Expected Result

The test passes after proving that the poisoned branch cannot fund the matured victim request and that the matched branch pays it. The large transaction gas allowance covers the complete multi-transaction test scenario. The test separately records the gas used by each final public settlement or claim.

```text
[PASS] test_ReleaseLargeVictimPermanentFreezeAndMatchedControl()
attack horizon seconds: 104598000
attacker gross deposits: 1204.617805
attacker returned principal: 1202.350415
victim current assets before poisoning: 16816577.524283
victim post-poison value lower bound: 16816577.522338
uint256 headroom: 500
poisoning pool settlement gas: 310688
matched tuning settlement gas: 310811
matched victim funding settlement gas: 212727
matched victim claim gas: 152866
control headroom: 1500
control victim payout: 16816577.522328
```

The victim-value calculation is a conservative lower bound using actual post-poison supply and Junior principal. The proof also checks that every earlier deposit claim is complete, mature cancellation is unavailable, and settlement still panics after more than 24 hours. The independent source review covers the remaining supported burn, pause, fee, and refund paths.
