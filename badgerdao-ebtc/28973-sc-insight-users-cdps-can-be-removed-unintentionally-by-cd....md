# 28973 - \[SC - Insight] Users CDPs can be removed unintentionally by CD...

Submitted on Mar 3rd 2024 at 21:16:50 UTC by @savi0ur for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28973

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/SortedCdps.sol>

Impacts:

* Permanent freezing of funds

## Description

## Bug Description

<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/SortedCdps.sol#L420-L455>

```solidity
function batchRemove(bytes32[] memory _ids) external override {
	_requireCallerIsCdpManager();
	uint256 _len = _ids.length;
	require(_len > 1, "SortedCdps: batchRemove() only apply to multiple cdpIds!");

	bytes32 _firstPrev = data.nodes[_ids[0]].prevId;
	bytes32 _lastNext = data.nodes[_ids[_len - 1]].nextId;

	require(
		_firstPrev != dummyId || _lastNext != dummyId,
		"SortedCdps: batchRemove() leave ZERO node left!"
	);

	for (uint256 i = 0; i < _len; ++i) {
		require(contains(_ids[i]), "SortedCdps: List does not contain the id");
	}

	// orphan nodes in between to save gas
	if (_firstPrev != dummyId) {
		data.nodes[_firstPrev].nextId = _lastNext;
	} else {
		data.head = _lastNext;
	}
	if (_lastNext != dummyId) {
		data.nodes[_lastNext].prevId = _firstPrev;
	} else {
		data.tail = _firstPrev;
	}

	// delete node & owner storages to get gas refund
	for (uint i = 0; i < _len; ++i) {
		delete data.nodes[_ids[i]];
		emit NodeRemoved(_ids[i]);
	}
	size = size - _len;
}
```

Function `batchRemove(bytes32[] memory _ids)` is used by CDP Manager to remove CDP IDs in batch. However, its assuming that `_ids` provided to remove are sorted in the same order as in the input array. If required ordering is not followed, it will also remove CDP's of the users which its not supposed to remove i.e., whose ID is not specified in `_ids`.

If this happened, it will remove CDP of some users, which then wont be able to claim their reward / collateral as their position is removed.

## Impact

Since there is no validation on `_ids` to be in sorted order in the same way as in the input array, if its provided in out of order, it will remove some additional CDP's that its not intending to delete. Which will make those additional CDP position non existent from the system and the user belonging to those positions wont be able to claim their rewards / collateral back.

## Recommendation

There should be a validation on `_ids` to be in a sorted order in the same way as in the input array. It can be done as below

```diff
function batchRemove(bytes32[] memory _ids) external override {
    _requireCallerIsCdpManager();
    uint256 _len = _ids.length;
    require(_len > 1, "SortedCdps: batchRemove() only apply to multiple cdpIds!");

+   uint256 nicr_prev = cdpManager.getCachedNominalICR(cdps[0]);
+   uint256 nicr_next;
+   for (uint i = 1; i < cdps.length; i++) {
+       nicr_next = cdpManager.getCachedNominalICR(cdps[i]);
+       require(nicr_prev <= nicr_next, "SortedCdps: List should be sorted");
+       nicr_prev = nicr_next;
+   }

    bytes32 _firstPrev = data.nodes[_ids[0]].prevId;
    bytes32 _lastNext = data.nodes[_ids[_len - 1]].nextId;

    require(
        _firstPrev != dummyId || _lastNext != dummyId,
        "SortedCdps: batchRemove() leave ZERO node left!"
    );

    for (uint256 i = 0; i < _len; ++i) {
        require(contains(_ids[i]), "SortedCdps: List does not contain the id");
    }

    // orphan nodes in between to save gas
    if (_firstPrev != dummyId) {
        data.nodes[_firstPrev].nextId = _lastNext;
    } else {
        data.head = _lastNext;
    }
    if (_lastNext != dummyId) {
        data.nodes[_lastNext].prevId = _firstPrev;
    } else {
        data.tail = _firstPrev;
    }

    // delete node & owner storages to get gas refund
    for (uint i = 0; i < _len; ++i) {
        delete data.nodes[_ids[i]];
        emit NodeRemoved(_ids[i]);
    }
    size = size - _len;
}
```

## References

<https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/SortedCdps.sol>

## Proof Of Concept

**Steps to Run using Foundry:**

* Paste following foundry code in `/ebtc-boost/packages/contracts/foundry_test/SortedCdps.t.sol`
* Run using `forge test --match-contract CDPOpsTest --match-test testBatchRemoveInOrderRemovesMore -vvvv`

```solidity
function testBatchRemoveInOrderRemovesMore() public {
    uint8 amntOfCdps = 5;

    // open some cdps
    uint256 collAmount = 30 ether;
    address user = _utils.getNextUserAddress();
    vm.startPrank(user);
    vm.deal(user, type(uint96).max);
    collateral.approve(address(borrowerOperations), type(uint256).max);
    collateral.deposit{value: 10000 ether}();
    uint256 borrowedAmount = _utils.calculateBorrowAmount(
        collAmount,
        priceFeedMock.fetchPrice(),
        COLLATERAL_RATIO
    );
    // Open X amount of CDPs
    for (uint256 cdpIx = 0; cdpIx < amntOfCdps; cdpIx++) {
        borrowerOperations.openCdp(borrowedAmount, HINT, HINT, collAmount + cdpIx * 1 ether);
    }
    vm.stopPrank();

    bytes32[] memory cdps = sortedCdps.getCdpsOf(user);
    // And check that amount of CDPs as expected
    assertEq(amntOfCdps, cdps.length);
    
    uint256 nicr_prev = cdpManager.getCachedNominalICR(cdps[0]);
    uint256 nicr_next;
    for (uint i = 1; i < cdps.length; i++) {
        nicr_next = cdpManager.getCachedNominalICR(cdps[i]);
        require(nicr_prev <= nicr_next, "SortedCdps: List should be sorted");
        nicr_prev = nicr_next;
    }

    bytes32[] memory _ids = new bytes32[](3);
    _ids[0] = cdps[0];
    _ids[1] = cdps[3];
    _ids[2] = cdps[1];

    vm.prank(address(cdpManager));
    sortedCdps.batchRemove(_ids);

    cdps = sortedCdps.getCdpsOf(user);
    assert(cdps.length == 1); // It should be 2 as we are deleting 3 ids out of 5
}
```
