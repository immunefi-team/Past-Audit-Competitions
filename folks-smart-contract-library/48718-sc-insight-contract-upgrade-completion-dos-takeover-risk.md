# #48718 \[SC-Insight] Contract Upgrade Completion DoS/Takeover Risk

**Submitted on Jul 7th 2025 at 10:09:36 UTC by @Blobism for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48718
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/Upgradeable.py>
* **Impacts:**
  * Temporary denial of service for more than one block

## Description

## Brief/Intro

The Upgradeable contract module allows any user to complete contract upgrades, which sets the contract to an uninitialized state. This means that a malicious user could complete a contract upgrade before an authorized user, potentially rendering the contract unusable until it is initialized. Furthermore, if contract initialization permissions are not carefully managed, a malicious user could then initialize the contract and give themselves an admin role.

## Vulnerability Details

The Upgradeable method `complete_contract_upgrade` can be called by anyone, and it sets the `is_initialised` state of the contract to `False`. This means that if the child contract requires initialization to function, and a contract upgrade was scheduled, any malicious user could complete the contract upgrade, rendering the contract unusable until it gets initialized again. This is desired behavior according to the docstring, but it is highly undesireable from a security perspective, due to the potential for DoS attacks.

To be more specific, if the child contract relies on the method `self._only_initialised()` at all, it is very risky to allow any user to set the contract into an uninitialized state.

```python
@abimethod(allow_actions=["UpdateApplication"])
def complete_contract_upgrade(self) -> None:
    """Complete the scheduled upgrade
    Anyone can call this method. <----------------- the issue

    Raises:
        AssertionError: If the contract is not initialised
        AssertionError: If the complete upgrade timestamp is not met
        AssertionError: If the contract SHA256 is not valid
    """
    self._only_initialised()

    # ... <checks if upgrade is scheduled> ...

    # reset to new contract version
    del self.scheduled_contract_upgrade.value
    self.version += UInt64(1)
    self.is_initialised = False # <-------- contract uninitialized

    emit(UpgradeCompleted(program_sha256, ARC4UInt64(self.version)))
```

Furthermore, there is a risk of contract takeover if the initialization method is not carefully designed to guard against it. This comment describes what is expected of the child contract:

```python
"""
...
It is the responsibility of the child contract to grant the `{upgradable_admin_role}`. The `initialise()` ABI
method, which is left abstract, is the recommended place to do this. If the `{upgradable_admin_role}` is ungranted
then no account will be able to upgrade the contract.
"""
```

If the contract initializer is allowed to freely select who gets the admin role, a malicious user could complete the contract upgrade, then initialize the contract such that they are now the admin. This is more so an inherent risk of the design rather than a vulnerability, but it needs to be detailed more explicitly. One way to do this safely is provided in `InitialisableWithCreator`, which checks that the caller must be the contract creator.

So using `InitialisableWithCreator` removes the risk of contract takeover with a very barebones approach. However, the danger of contract takeover is generally increased by the fact that any malicious user can complete a contract upgrade themselves, then immediately attempt to initialize the contract.

Some potential fixes are:

* restrict upgrade completion to being callable only by authorized users
* be much more explicit in docs about the importance of securing the initialization method

## Impact Details

The definite vulnerability is a temporary DoS, because anyone can complete a contract upgrade after it has been initiated, leaving the contract in an uninitialized state.

An additional risk, which is dependent upon poor initialization implementation, is that there could be contract takeover following a malicious user completing the contract upgrade.

## References

See `contracts/library/Upgradeable.py`

## Proof of Concept

## Proof of Concept

This test in `Upgradeable.test.ts` is sufficient to demonstrate the issue. It shows how a user who is NOT the creator or an admin is able to complete the contract upgrade, setting the contract to being NOT initialized. While the test may suggest that it is desired behavior, this is **highly unadvisable** from a security perspective.

```typescript
test("succeeds and completes scheduled upgrade", async () => {
  const { timestamp: scheduledTimestamp } = (await client.state.global.scheduledContractUpgrade())!;
  expect(await getPrevBlockTimestamp(localnet)).toBeGreaterThanOrEqual(scheduledTimestamp);

  const oldVersion = (await client.state.global.version()) || 0n;
  const programSha256 = calculateProgramSha256(approvalProgramToUpdateTo, clearStateProgramToUpdateTo);

  const res = await localnet.algorand.send.appUpdateMethodCall({
    sender: user,
    appId,
    method: client.appClient.getABIMethod("complete_contract_upgrade"),
    approvalProgram: approvalProgramToUpdateTo,
    clearStateProgram: clearStateProgramToUpdateTo,
  });

  const updatedApp = await localnet.algorand.app.getById(appId);
  expect(updatedApp.approvalProgram).toEqual(approvalProgramToUpdateTo);
  expect(updatedApp.clearStateProgram).toEqual(clearStateProgramToUpdateTo);
  expect(res.confirmations[0].logs).toBeDefined();
  expect(res.confirmations[0].logs![0]).toEqual(
    getEventBytes("UpgradeCompleted(byte[32],uint64)", [programSha256, oldVersion + 1n]),
  );
  expect(await client.state.global.scheduledContractUpgrade()).toBeUndefined();
  expect(await client.state.global.version()).toEqual(oldVersion + 1n);
  expect(await client.state.global.isInitialised()).toBeFalsy();
});
```
