# 31524 - \[SC - High] Rounding down in getClaimableFlux leads to less...

Submitted on May 21st 2024 at 01:08:34 UTC by @SAAJ for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31524

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

`getClaimableFlux` carried out multiplication on result of division leads to less amount minting in `nftClaim` function of `FluxToken` contract.

## Vulnerability Details

`getClaimableFlux` function is called in `nftClaim` function to have the minting amount cached before `mint` is called. `claimableFlux` variable in `getClaimableFlux` function have mathematics operation which violates safe practice of carrying out multiplication before division that can lead to issue of rounding down.

## Impact Details

The rounding down issue in `getClaimableFlux` function will lead to minting less amount of `FLUX` tokens to be claimed against the `eth` deposited. The minting of less token due to rounding down will cause depositor to have less reward for claiming causing direct loss.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L224>

## Recommendation

The recommendation is made to carry out all division and multiplication separately to avoid loss of precision due to rounding down. The recommended code will mint exact value that the depositor is eligible for claiming against the amount they deposited.

```diff
    function getClaimableFlux(uint256 _amount, address _nft) public view returns (uint256 claimableFlux) {
        uint256 bpt = calculateBPT(_amount);

        uint256 veMul = IVotingEscrow(veALCX).MULTIPLIER();
        uint256 veMax = IVotingEscrow(veALCX).MAXTIME();
        uint256 fluxPerVe = IVotingEscrow(veALCX).fluxPerVeALCX();
        uint256 fluxMul = IVotingEscrow(veALCX).fluxMultiplier();

        // Amount of flux earned in 1 yr from _amount assuming it was deposited for maxtime
-        claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;

+	 claimableFlux_modified = (bpt * veMul * veMax * (fluxPerVe + BPS)) / (BPS * fluxMul * veMax); // all division after multiplication

        // Claimable flux for alchemechNFT is different than patronNFT
        if (_nft == alchemechNFT) {
            claimableFlux = (claimableFlux * alchemechMultiplier) / BPS;
        }
    }
```

## Proof of Concept

This simple test clearly demonstrates loss of rewards for user against amount deposited due to rounding down issue.

All the values are considered in default context as used in the `VotingEscrow` contract that were passed in the calculation of `claimableFlux` variable in `getClaimableFlux` function.

The test have also modified code for `claimableFlux` variable named `claimableFlux_modified` to show the difference in result of mathematical operation carried out in normal code and modified code context.

Normal code as used by protocol carried out multiplication on result of division, while modified code group together multiplication and division to have more precise value.

```
    // forge t --mt test_rewardFlux -vv
    function test_rewardFlux() external view {
        uint256 bpt = 1000e18 * 40; // 1000 tokens passed as param to calculateBPT() of FluxToken
        uint256 BPS = 10_000; // value taken from VotingEscrow contract
        uint256 veMul = 2; // value taken from VotingEscrow contract
        uint256 veMax = 365 days; // value taken from VotingEscrow contract
        uint256 fluxPerVe = 5000;// value taken from VotingEscrow contract
        uint256 fluxMul = 4;// value taken from VotingEscrow contract
        uint256 alchemechNFT = 5;// value taken from VotingEscrow contract
        uint256 _nft = 5; // value assumed for VotingEscrow contract

        // Amount of flux earned in 1 yr from _amount assuming it was deposited for maxtime
        uint256 claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
        console.log("claimableFlux: %e", claimableFlux); // value on multiply result of division

        uint256 claimableFlux_modified = (bpt * veMul * veMax * (fluxPerVe + BPS)) / (BPS * fluxMul * veMax); // all division after multiplication
        console.log("claimableFlux_modified: %e", claimableFlux_modified); // value when all division is after multiply

        if (_nft == alchemechNFT) {
            claimableFlux = (claimableFlux * alchemechNFT) / BPS;
            console.log("Nft claimableFlux: %e", claimableFlux); // resultant value on default code

            claimableFlux_modified = (claimableFlux_modified * alchemechNFT) / BPS;
            console.log("Nft claimableFlux_modified: %e", claimableFlux_modified); // resultant value on modified code
        }
    }
```

The result clearly shows difference in amount claimed by user for normal code against the modified code. The difference is due to loss of precision arising from rounding down causing direct loss.

```
 [PASS] test_rewardFlux() (gas: 7126)
Logs:
  claimableFlux: 2.9999999999999989116e22
  claimableFlux_modified: 3e22
  Nft claimableFlux: 1.4999999999999994558e19
  Nft claimableFlux_modified: 1.5e19

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.75ms (278.70µs CPU time)

Ran 1 test suite in 147.11ms (1.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
