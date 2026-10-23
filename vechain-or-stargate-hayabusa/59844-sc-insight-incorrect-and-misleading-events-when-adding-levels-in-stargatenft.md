# 59844 sc insight incorrect and misleading events when adding levels in stargatenft&#x20;

**Submitted on Nov 16th 2025 at 10:44:17 UTC by @blackgrease for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59844
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/libraries/Levels.sol>

## Description

**Affected Files:** `StargateNFT.sol` and `Levels`

The `StargateNFT` inherits the `Levels` library allowing levels to be added into the contract by an address with the `LEVEL_OPERATOR_ROLE` privilege (set by protocol). However, when adding a level the `addLevel` function emits incorrect/misleading events. Specifically, the function emits:

* `LevelUpdated`: misleading and incorrect because this is an addition operation and not updating an existing level.
* `LevelCirculatingSupplyUpdated`: misleading and incorrect because this is an addition operation (not an update). Also the `oldCirculatingSupply` value is 0 which isn't useful.
* `LevelCapUpdated`: again, not an updating operation; the `oldCap` parameter is meaningless.

A stack trace of the emitted events:

```
│   │   │   ├─ emit LevelUpdated(levelId: 4, name: "Thunder Boosted", isX: false, maturityBlocks: 7200, scaledRewardFactor: 1, vetAmountRequiredToStake: 500000000000000000000 [5e20])
│   │   │   ├─ emit LevelCirculatingSupplyUpdated(levelId: 4, oldCirculatingSupply: 0, newCirculatingSupply: 0)
│   │   │   ├─ emit LevelCapUpdated(levelId: 4, oldCap: 0, newCap: 1000000000 [1e9])
```

These events are emitted by `Levels::updateLevel`. Note: in version 3 of `StargateNFT` the `updateLevel` function is no longer exposed. Documentation confirms removal of setters: StargateNFT#L92 "Removed setters to optimize contract size: setStargateDelegation, setLegacyNodes, updateLevelCap, updateLevel".

The `addLevel` function should emit its own unique event that accurately represents the addition operation and contains meaningful data.

### The Associated Code

StargateNFT wrapper:

```solidity
function addLevel(
    DataTypes.LevelAndSupply memory _levelAndSupply
) public onlyRole(LEVEL_OPERATOR_ROLE) {
    Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
}
```

Levels library (relevant extract):

```solidity
function addLevel(
    DataTypes.StargateNFTStorage storage $,
    DataTypes.LevelAndSupply memory _levelAndSupply
) external { 
    // Increment MAX_LEVEL_ID
    $.MAX_LEVEL_ID++;

    // Override level ID to be the new MAX_LEVEL_ID (We do not care about the level id in the input)
    _levelAndSupply.level.id = $.MAX_LEVEL_ID;

    // Validate level fields
    _validateLevel(_levelAndSupply.level); 

    // Validate supply
    if (_levelAndSupply.circulatingSupply > _levelAndSupply.cap) { 
        revert Errors.CirculatingSupplyGreaterThanCap();
    }

    // Add new level to storage
    $.levels[_levelAndSupply.level.id] = _levelAndSupply.level;
    _checkpointLevelCirculatingSupply(
        $,
        _levelAndSupply.level.id,
        _levelAndSupply.circulatingSupply
    );
    $.cap[_levelAndSupply.level.id] = _levelAndSupply.cap;

    emit LevelUpdated(
        _levelAndSupply.level.id,
        _levelAndSupply.level.name,
        _levelAndSupply.level.isX,
        _levelAndSupply.level.maturityBlocks,
        _levelAndSupply.level.scaledRewardFactor,
        _levelAndSupply.level.vetAmountRequiredToStake
    ); //@audit-insight: incorrect/misleading event emitted
    emit LevelCirculatingSupplyUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.circulatingSupply); //@audit-insight: incorrect/misleading event emitted
    emit LevelCapUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.cap); //@audit-insight: incorrect/misleading event emitted
}
```

## Impact

This is classified as an Insight under "Code Optimizations and Enhancements". Incorrect or misleading event emission can confuse indexers, analytics services, and anyone relying on on-chain events to understand protocol state changes. The semantics of "add" differ from "update" and should be represented distinctly.

## Mitigation

Create and emit a new event that accurately represents level addition. Suggested event:

```solidity
event NewLevelAdded(
    uint8 indexed levelId,
    string name,
    bool isX,
    uint64 maturityBlocks,
    uint64 scaledRewardFactor,
    uint256 vetAmountRequiredToStake, 
    uint208 circulatingSupply,
    uint32 levelCap
);
```

Change `Levels::addLevel` to emit the new event instead of the three misleading events. Example patch:

```diff
-        emit LevelUpdated(_levelAndSupply.level.id, _levelAndSupply.level.name, _levelAndSupply.level.isX, _levelAndSupply.level.maturityBlocks, _levelAndSupply.level.scaledRewardFactor, _levelAndSupply.level.vetAmountRequiredToStake ); //@audit-fix: remove 
-        emit LevelCirculatingSupplyUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.circulatingSupply); //@audit-fix: remove 
-        emit LevelCapUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.cap); //@audit-fix: remove 
+        emit NewLevelAdded(
+            _levelAndSupply.level.id,
+            _levelAndSupply.level.name,
+            _levelAndSupply.level.isX,
+            _levelAndSupply.level.maturityBlocks,
+            _levelAndSupply.level.scaledRewardFactor, //@audit: no 0 check
+            _levelAndSupply.level.vetAmountRequiredToStake,
+            _levelAndSupply.circulatingSupply,
+            _levelAndSupply.cap
+        );
```

This preserves accurate semantics and removes confusing `old*` fields that are meaningless for additions.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/e056f70903d89aea4fe1ddda4461a862>

## Proof of Concept

A runnable Foundry PoC is provided in the private gist at the link above. Run:

forge test --mt testAddLevelInccorectEventEmitted -C ./packages/contracts -vvvv

### PoC Description

{% stepper %}
{% step %}
Add a level to the contract using the `LEVEL_OPERATOR_ROLE`.
{% endstep %}

{% step %}
Retrieve the emitted event logs.
{% endstep %}

{% step %}
Confirm the stored event logs match what is (incorrectly) expected to be emitted from the contract.
{% endstep %}
{% endstepper %}

<details>

<summary>Stack Trace (for convenience)</summary>

```
[PASS] testAddLevelInccorectEventEmitted() (gas: 193024)
Traces:
  [193024] PoC_IncorrectEventsEmittedOnAddLevel::testAddLevelInccorectEventEmitted()
    ├─ [0] VM::recordLogs()
    │   └─ ← [Return]
    ├─ [0] VM::prank(PoC_IncorrectEventsEmittedOnAddLevel: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Return]
    ├─ [168235] ERC1967Proxy::fallback(LevelAndSupply({ level: Level({ name: "Thunder Boosted", isX: false, id: 4, maturityBlocks: 7200, scaledRewardFactor: 1, vetAmountRequiredToStake: 500000000000000000000 [5e20] }), circulatingSupply: 0, cap: 1000000000 [1e9] }))
    │   ├─ [163198] StargateNFT::addLevel(LevelAndSupply({ level: Level({ name: "Thunder Boosted", isX: false, id: 4, maturityBlocks: 7200, scaledRewardFactor: 1, vetAmountRequiredToStake: 500000000000000000000 [5e20] }), circulatingSupply: 0, cap: 1000000000 [1e9] })) [delegatecall]
    │   │   ├─ [153287] Levels::addLevel(106876110134940196446636019078460187759378049410576946175053451556559881877248 [1.068e77], 64) [delegatecall]
    │   │   │   ├─ emit LevelUpdated(levelId: 4, name: "Thunder Boosted", isX: false, maturityBlocks: 7200, scaledRewardFactor: 1, vetAmountRequiredToStake: 500000000000000000000 [5e20])
    │   │   │   ├─ emit LevelCirculatingSupplyUpdated(levelId: 4, oldCirculatingSupply: 0, newCirculatingSupply: 0)
    │   │   │   ├─ emit LevelCapUpdated(levelId: 4, oldCap: 0, newCap: 1000000000 [1e9])
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [0] VM::getRecordedLogs() [staticcall]
    │   └─ ← [Return] [([0xf100d70d7533f8cc876180c07c9da3b2fb3dd79fbcd54884d6264774f6bd9a8b, 0x0000000000000000000000000000000000000000000000000000000000000004], 0x00000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001c20000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000001b1ae4d6e2ef500000000000000000000000000000000000000000000000000000000000000000000f5468756e64657220426f6f737465640000000000000000000000000000000000, 0xc7183455a4C133Ae270771860664b6B7ec320bB1), ([0x2d3ae9d6214117349be592e75147c4a63aa26af4abdf41364cab8409def6c55f, 0x0000000000000000000000000000000000000000000000000000000000000004], 0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000, 0xc7183455a4C133Ae270771860664b6B7ec320bB1), ([0xa8fea641f11db999cf8191284c4fa96dc28a20a81368b7190b9fd772d2d2c153, 0x0000000000000000000000000000000000000000000000000000000000000004], 0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003b9aca00, 0xc7183455a4C133Ae270771860664b6B7ec320bB1)]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.17ms (341.85µs CPU time)
```

</details>

### Hardhat conversion to Foundry

{% stepper %}
{% step %}
Start: clone the repository

git clone <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa.git\\>
cd audit-comp-vechain-stargate-hayabusa/
{% endstep %}

{% step %}
Install dependencies:

forge install foundry-rs/forge-std
{% endstep %}

{% step %}
Install OpenZeppelin upgradeable contracts:

forge install OpenZeppelin/openzeppelin-contracts-upgradeable\@v5.0.2
{% endstep %}

{% step %}
Install OpenZeppelin contracts (used in test):

forge install OpenZeppelin/openzeppelin-contracts\@v5.0.2
{% endstep %}

{% step %}
Compile and run the PoC:

forge compile ./packages/contracts/ (warnings can be ignored)

forge test --mt testAddLevelInccorectEventEmitted -C ./packages/contracts -vvvv
{% endstep %}
{% endstepper %}

***

If you want, I can:

* produce a minimal patch (diff) that adds the NewLevelAdded event and replaces the three emits in `Levels::addLevel`, or
* open a suggested PR message and commit body consistent with the repo style.
