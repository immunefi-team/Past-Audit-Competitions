# 31597 - \[SC - High] Loss of precision while calculating claimable f...

Submitted on May 21st 2024 at 16:09:53 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31597

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L38>

```solidity
/// @notice Multiplier for the slope of the decay
uint256 public constant MULTIPLIER = 2;
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L44>

```solidity
int256 internal constant iMULTIPLIER = 2;
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1157-L1160>

```solidity
function _calculatePoint(LockedBalance memory _locked, uint256 _time) internal pure returns (Point memory point) {
    if (_locked.end > _time && _locked.amount > 0) {
        point.slope = _locked.maxLockEnabled ? int256(0) : (int256(_locked.amount) * iMULTIPLIER) / iMAXTIME;
        point.bias = _locked.maxLockEnabled
            ? ((int256(_locked.amount) * iMULTIPLIER) / iMAXTIME) * (int256(_locked.end - _time))
            : (point.slope * (int256(_locked.end - _time)));
    }
}
```

As we can see, `iMULTIPLIER` and `MULTIPLIER` is set to 2, which is not sufficient to preserve the precision.

`point.slope` and `point.bias` are calculated as follow:

```
point.slope = _locked.maxLockEnabled ? int256(0) : (int256(_locked.amount) * iMULTIPLIER) / iMAXTIME;

point.bias = _locked.maxLockEnabled
            ? ((int256(_locked.amount) * iMULTIPLIER) / iMAXTIME) * (int256(_locked.end - _time))
            : (point.slope * (int256(_locked.end - _time)));
```

When `(int256(_locked.amount) * iMULTIPLIER) < iMAXTIME` and `((int256(_locked.amount) * iMULTIPLIER) < iMAXTIME`, it results in zero. So for the locked amounts which is small enough when multiplied by `iMULTIPLIER=2` end up being lower than `iMAXTIME`, will not be counted in bias and slope calculation. At that point in time, there `slope` and `bias` will be zero.

*For example,*

```
_locked.amount = (iMAXTIME / 2) - 1 = (365 days / 2) - 1 = 15767999

_locked.maxLockEnabled = false

point.slope = (int256(_locked.amount) * iMULTIPLIER) / iMAXTIME;
			= (15767999 * 2) / 365 days
			= 0

point.bias  = (point.slope * (int256(_locked.end - _time)))
			= (0 * (int256(_locked.end - _time)))
			= 0
```

As we can, both `slope` and `bias` become zero.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L224>

```solidity
function getClaimableFlux(uint256 _amount, address _nft) public view returns (uint256 claimableFlux) {
    uint256 bpt = calculateBPT(_amount);

    uint256 veMul = IVotingEscrow(veALCX).MULTIPLIER();
    uint256 veMax = IVotingEscrow(veALCX).MAXTIME();
    uint256 fluxPerVe = IVotingEscrow(veALCX).fluxPerVeALCX();
    uint256 fluxMul = IVotingEscrow(veALCX).fluxMultiplier();

    // Amount of flux earned in 1 yr from _amount assuming it was deposited for maxtime
    claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul; //@audit

    // Claimable flux for alchemechNFT is different than patronNFT
    if (_nft == alchemechNFT) {
        claimableFlux = (claimableFlux * alchemechMultiplier) / BPS;
    }
}
```

Similarly, in `getClaimableFlux` function, `claimableFlux` is calculated as

```
claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
```

Here also, `veMul = 2` as its fetched from `VotingEscrow's` multiplier.

## Impact

Precision loss while calculating claimable flux and point's `bias` and `slope`.

## Recommendation

Increase `MULTIPLIER` and `iMULTIPLIER` precision from `2` to `1e18`, for higher precision.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

## Proof Of Concept

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/FluxToken.t.sol`
* Run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract FluxTokenTest --match-test testPrecisionLoss -vv`

```solidity
function testPrecisionLoss() external {
    uint256 tokenId = 4;
    address ownerOfPatronNFT = IAlEthNFT(patronNFT).ownerOf(tokenId);
    // address ownerOfAlchemechNFT = IAlchemechNFT(alchemechNFT).ownerOf(tokenId);

    assertEq(flux.balanceOf(ownerOfPatronNFT), 0, "owner should have no flux");
    // assertEq(flux.balanceOf(ownerOfAlchemechNFT), 0, "owner should have no flux");

    uint256 tokenData1 = IAlEthNFT(patronNFT).tokenData(tokenId);
    uint256 patronTotal = flux.getClaimableFlux(tokenData1, patronNFT);
    // console.log("patronTotal:", patronTotal);

    uint256 _bpt = flux.calculateBPT(tokenData1);

    uint256 veMul = 200000;//IVotingEscrow(veALCX).MULTIPLIER();
    uint256 veMax = IVotingEscrow(veALCX).MAXTIME();
    uint256 fluxPerVe = IVotingEscrow(veALCX).fluxPerVeALCX();
    uint256 fluxMul = IVotingEscrow(veALCX).fluxMultiplier();

    // Amount of flux earned in 1 yr from _amount assuming it was deposited for maxtime
    uint claimableFlux = (((_bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
    console.log("patronTotal  :", patronTotal);
    console.log("claimableFlux:", claimableFlux);
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract FluxTokenTest --match-test testPrecisionLoss -vv

Ran 1 test for src/test/FluxToken.t.sol:FluxTokenTest
[PASS] testPrecisionLoss() (gas: 47036)
Logs:
  patronTotal  : 7499756087469342000
  claimableFlux: 749975608747561793994000
```

As we can see, by increasing MULTIPLIER, precision has increased. We have used `200000` as `MULTIPLIER` here to show precision issue as it was set `2` initially.
