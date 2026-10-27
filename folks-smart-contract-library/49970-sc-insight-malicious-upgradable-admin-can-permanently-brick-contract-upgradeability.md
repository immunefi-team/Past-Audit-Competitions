# #49970 \[SC-Insight] Malicious upgradable admin can permanently brick contract upgradeability

**Submitted on Jul 20th 2025 at 22:01:37 UTC by @danvinci\_20 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49970
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/Upgradeable.py>
* **Impacts:**
  * Unauthorized escalation of privileged roles which deviate from the original permissions
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

The contract’s `update_min_upgrade_delay()` function allows an address with the `upgradable_admin_role` to schedule a change to the minimum required delay before any upgrade can be scheduled.

However, there is no upper bound on the `min_upgrade_delay` value that can be set. A malicious actor with the upgradable admin role can set this delay to an extremely large number (`2^64 - 1`), effectively making it impossible to schedule upgrades for the foreseeable future.

This results in a permanent denial of governance and upgradeability. Since the upgrade scheduling function enforces a hard check that the timestamp must be at least the current time plus the active `min_upgrade_delay`, no upgrade will ever be schedulable if this delay is set too high. Moreover, because changes to the delay must themselves wait until the (delayed) timestamp is reached, there is no immediate way to override this malicious value.

```solidity
 @abimethod
    def schedule_contract_upgrade(self, program_sha256: Bytes32, timestamp: UInt64) -> None:
        """Schedule the upgrade of the contract.

        Args:
            program_sha256 (Bytes32): The SHA256 of the new program
            timestamp (UInt64): The timestamp to schedule the upgrade

        Raises:
            AssertionError: If the contract is not initialised
            AssertionError: If the caller does not have the upgradable admin role
            AssertionError: If the timestamp is not sufficiently in the future
        """
        self._only_initialised()
        self._check_sender_role(self.upgradable_admin_role())

        # ensure timestamp is sufficiently in the future
    @>>    self._check_schedule_timestamp(timestamp)

        # schedule contract upgrade, possibly overriding existing scheduled upgrade
        self.scheduled_contract_upgrade.value = ScheduledContractUpgrade(program_sha256.copy(), ARC4UInt64(timestamp))

        emit(UpgradeScheduled(program_sha256, ARC4UInt64(timestamp)))
```

```solidity
  @abimethod(readonly=True)
    def get_active_min_upgrade_delay(self) -> UInt64:
        """Clarifies the active minimum upgrade delay in cases where there was a scheduled update.

        Returns:
            The active minimum upgrade delay
        """
        min_upgrade_delay = self.min_upgrade_delay.value.copy()
  @>>      return (
            min_upgrade_delay.delay_1 if Global.latest_timestamp >= min_upgrade_delay.timestamp
            else min_upgrade_delay.delay_0
        ).native

    @subroutine
    def _check_schedule_timestamp(self, timestamp: UInt64) -> None:
    @>>    assert (
            timestamp >= Global.latest_timestamp + self.get_active_min_upgrade_delay()
        ), "Must schedule at least min upgrade delay time in future"
```

## Impact Details

A malicious admin can set the minimum upgrade delay to an extremely large value, making it impossible to schedule any future upgrades. Since delay changes themselves are also subject to this delay, the malicious value cannot be easily overridden. This results in a permanent loss of upgradability and governance control over the contract.

## Recommendations

Introduce an upper bound to the `min_upgrade_delay` parameter in `update_min_upgrade_delay()` to avoid malicious or accidental denial of upgrade functionality. For example

```solidity
MAX_MIN_UPGRADE_DELAY = UInt64(30 * 86400)  #  30 days

assert min_upgrade_delay <= MAX_MIN_UPGRADE_DELAY, "Min upgrade delay too large"
```

## Proof of Concept

## Proof of Concept

The following steps can demonstrate the issue:

1. Malicious admin calls:

```solidity
update_min_upgrade_delay(UInt64(2**64 - 1), Global.latest_timestamp + 1)
```

2. After 1 second (when new delay becomes active), any upgrade attempt will require:

```solidity
scheduled_timestamp >= Global.latest_timestamp + (2^64 - 1)
```

which is unreasonably far in the future and will fail `_check_schedule_timestamp()`.

This results in:

1. No upgrade being able to be schedulable,
2. No recovery path (since modifying the delay requires waiting the massive delay period),
3. The system being permanently locked
