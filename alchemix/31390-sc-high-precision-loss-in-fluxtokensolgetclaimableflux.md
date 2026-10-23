# 31390 - \[SC - High] Precision Loss in FluxTokensolgetClaimableFlux

Submitted on May 17th 2024 at 23:44:47 UTC by @gladiator111 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31390

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

Precision Loss is there in `FluxToken.sol::getClaimableFlux` resulting in getting less Flux Token

## Vulnerability Details

`Note - Please adjust the Severity Level or impact as you seem proper. I have selected the most close impact from the impact list.`

In the function `FluxToken.sol::getClaimableFlux`

```solidity
// @audit Precision loss
    function getClaimableFlux(uint256 _amount, address _nft) public view returns (uint256 claimableFlux) {
        uint256 bpt = calculateBPT(_amount);                                  // .4 times amount

        uint256 veMul = IVotingEscrow(veALCX).MULTIPLIER();                   // 2
        uint256 veMax = IVotingEscrow(veALCX).MAXTIME();                      // 365 days
        uint256 fluxPerVe = IVotingEscrow(veALCX).fluxPerVeALCX();            // 5000 or 50%
        uint256 fluxMul = IVotingEscrow(veALCX).fluxMultiplier();             //4 or 4x

        // Amount of flux earned in 1 yr from _amount assuming it was deposited for maxtime
        claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;  
        //((( 40*amount * 2) / 31536000) * 31536000 * ( 5000 + 10000 ))/ 10000 / 4

        // Claimable flux for alchemechNFT is different than patronNFT
        if (_nft == alchemechNFT) {
            claimableFlux = (claimableFlux * alchemechMultiplier) / BPS;      // flux * 5 /10000  or 0.05% flux
        }
    }
```

The calculation of claimable flux leads to precision loss because of `division before multiply`

```solidity
 claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
```

Notice the veMax is divided before multiplying which leads to precision loss.

## Impact Details

User will get less Flux because of precision Loss.

## Suggestion / Recommendation

```diff
-  claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
+  claimableFlux = (((bpt * veMul)) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul / veMax;  
```

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L224>

## Proof of Concept

Add the following to the FluxToken.t.sol and run using

```bash
forge test --match-test testGetClaimableFluxPrecisionLoss -vvvv --fork-url $FORK_URL
```

```solidity
 function testGetClaimableFluxPrecisionLoss() public {
        uint256 claimableFlux = flux.getClaimableFlux(350000, patronNFT);
        uint256 correctClaimableFlux = CorrectedClaimableFlux(350000, patronNFT);
        uint256 claimableFluxWithDifferentAmount = flux.getClaimableFlux(1e18, patronNFT);
        uint256 correctClaimableFluxWithDifferentAmount = CorrectedClaimableFlux(1e18, patronNFT);
        console.log(claimableFlux);
        console.log(correctClaimableFlux);
        console.log(claimableFluxWithDifferentAmount);
        console.log(correctClaimableFluxWithDifferentAmount);
        assert(claimableFlux != correctClaimableFlux);
        assert(claimableFluxWithDifferentAmount != correctClaimableFluxWithDifferentAmount);
    }

    function CorrectedClaimableFlux(uint256 _amount, address _nft) public view returns (uint256 claimableFlux) {
        // Values are Hardcoded for simplification

        uint256 bpt = 40*_amount;/*calculateBPT(_amount);*/                                  // .4 times amount

        uint256 veMul = 2; /*IVotingEscrow(veALCX).MULTIPLIER();*/                   // This equals 2
        uint256 veMax = 365 days; /*IVotingEscrow(veALCX).MAXTIME(); */              // This equals 365 days
        uint256 fluxPerVe = 5000;/*IVotingEscrow(veALCX).fluxPerVeALCX();*/          // This equals 5000 or 50%
        uint256 fluxMul = 4;/*IVotingEscrow(veALCX).fluxMultiplier();*/              // This equals 4 or 4x

        // Amount of flux earned in 1 yr from _amount assuming it was deposited for maxtime
        // Precision Loss fixed here
        claimableFlux = (((bpt * veMul)) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul / veMax;  
        //((( 40*amount * 2) / 31536000) * 31536000 * ( 5000 + 10000 ))/ 10000 / 4

        // Claimable flux for alchemechNFT is different than patronNFT
        if (_nft == alchemechNFT) {
            claimableFlux = (claimableFlux * 5) / BPS;      // flux * 5 /10000  or 0.05% flux
        }
    }
```
