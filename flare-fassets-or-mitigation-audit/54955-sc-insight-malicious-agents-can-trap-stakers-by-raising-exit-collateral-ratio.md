# #54955 \[SC-Insight] malicious agents can trap stakers by raising exit collateral ratio

**Submitted on Sep 19th 2025 at 14:41:53 UTC by @HarryBarz for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #54955
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/commit/abb3dde42ebb36fc052dc0658141005679f6add3>
* **Impacts:**
  * Permanent freezing of funds
  * Temporary freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

Malicious agents can exploit faulty OR logic in `setPoolExitCollateralRatioBIPS` to repeatedly increase the pool exit collateral ratio beyond reasonable limits, trapping stakers who cannot withdraw their funds due to impossibly high collateral requirements.

### Vulnerability Details

This issue's scope is under:

<https://github.com/flare-foundation/fassets/commit/abb3dde42ebb36fc052dc0658141005679f6add3?utm\\_source=immunefi>

which is supposed to be the fix for issue #45514 - <https://reports.immunefi.com/flare-fassets/45514-sc-medium-malicious-agents-can-trap-stakers-by-raising-the-exit-collateral-ratio>

But looking at the commit, the changes made there don't fix the bug in the codebase.

From the original bug report, the `setPoolExitCollateralRatioBIPS` function in `AgentUpdates.sol` uses faulty OR logic (`||`) that allows malicious agents to exploit only one of two validation conditions, enabling infinite escalation of the exit collateral ratio.

fassets/contracts/assetManager/library/AgentUpdates.sol

```solidity
require(_poolExitCollateralRatioBIPS <= currentExitCR * 3 / 2 ||
        _poolExitCollateralRatioBIPS <= minCR * 12 / 10,
        IncreaseTooBig());
```

This OR logic means only ONE condition needs to be true for the check to pass. The first condition allows up to 150% of current ratio, while the second allows up to 120% of minimum ratio. A malicious agent can repeatedly exploit the first condition while ignoring the second.

The internal function is called by the public `executeAgentSettingUpdate` function, which can be called by any agent vault owner to modify their pool's exit collateral ratio.

```solidity
function executeAgentSettingUpdate(
    address _agentVault,
    string memory _name
)
    external
    onlyAgentVaultOwner(_agentVault)
{
    // ... validation logic ...
    _executeUpdate(agent, hash, update.value);
    // ... event emission ...
}
```

`_executeUpdate` dispatches to the vulnerable function when the setting name is `"poolExitCollateralRatioBIPS"`:

```solidity
function _executeUpdate(
    Agent.State storage _agent,
    bytes32 _hash,
    uint256 _value
)
    private
{
    // ... other conditions ...
    } else if (_hash == POOL_EXIT_COLLATERAL_RATIO_BIPS) {
        AgentUpdates.setPoolExitCollateralRatioBIPS(_agent, _value);
    }
    // ... other conditions ...
}
```

The vulnerable OR logic (`||`) is still present in `AgentUpdates.sol`.

## Impact Details

Same as the original impact from the original report.

Malicious agents can trap stakers within collateral pools by exploiting the faulty OR logic to repeatedly inflate the exit collateral ratio beyond reasonable limits, preventing stakers from exiting their positions due to impossibly high collateral requirements. This creates an unfair competitive advantage for malicious agents who can bypass the intended 120% of minimum ratio rule while other agents must abide by it, and stakers have no way to detect or prevent this griefing attack. Even agents acting without malicious intent can inadvertently bypass the intended thresholds effectively freezing stakers' funds.

## Proof of Concept

{% stepper %}
{% step %}

### 1. Vulnerable check

The vulnerable code logic is still present:

```solidity
require(_poolExitCollateralRatioBIPS <= currentExitCR * 3 / 2 ||
        _poolExitCollateralRatioBIPS <= minCR * 12 / 10,
        IncreaseTooBig());
```

The OR logic (`||`) allows a call to succeed if either condition is true, enabling attacks that repeatedly satisfy only the first condition.
{% endstep %}

{% step %}

### 2. Initial setup

* minCR = 4000 (40% minimum collateral ratio)
* currentExitCR = 5500 (55% current exit collateral ratio)
* Attacker target: set `_poolExitCollateralRatioBIPS` = 7000 (70%)
  {% endstep %}

{% step %}

### 3. Call the public update

Call:

```
executeAgentSettingUpdate(agentVaultAddress, "poolExitCollateralRatioBIPS", value=7000)
```

This dispatches to `setPoolExitCollateralRatioBIPS` which performs the vulnerable require check.
{% endstep %}

{% step %}

### 4. Evaluate conditions

First condition: 7000 <= 5500 \* 3 / 2 → 7000 <= 8250 → TRUE

Second condition: 7000 <= 4000 \* 12 / 10 → 7000 <= 4800 → FALSE

OR evaluation: TRUE || FALSE = TRUE → the update succeeds and the ratio becomes 7000.
{% endstep %}

{% step %}

### 5. Repeat to escalate

New currentExitCR = 7000\
Next allowed via first condition: 7000 \* 3 / 2 = 10500

Attacker sets `_poolExitCollateralRatioBIPS = 10500`:

Check: 10500 <= 10500 \* 3 / 2 → 10500 <= 15750 → TRUE\
Second condition remains FALSE (10500 <= 4800 → FALSE)\
TRUE || FALSE = TRUE → update succeeds again.

This can be repeated indefinitely, each time increasing by up to 150% of the current exit ratio and bypassing the intended 120% of minimum ratio limit.
{% endstep %}
{% endstepper %}

## References

* <https://github.com/flare-foundation/fassets/commit/abb3dde42ebb36fc052dc0658141005679f6add3?utm\\_source=immunefi>
* <https://reports.immunefi.com/flare-fassets/45514-sc-medium-malicious-agents-can-trap-stakers-by-raising-the-exit-collateral-ratio>
* <https://github.com...658141005679f6add3> (Smart Contract - Fix of Report - 45514 — 18 September 2025)
