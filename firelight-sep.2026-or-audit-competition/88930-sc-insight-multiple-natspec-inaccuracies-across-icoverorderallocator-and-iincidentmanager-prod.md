# 88930 sc insight multiple natspec inaccuracies across icoverorderallocator and iincidentmanager produce misleading documentation for integrators

**Submitted on Aug 15th 2026 at 18:38:21 UTC by @blackgrease for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88930
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/interfaces/IIncidentManager.sol>
* **Impacts:**

## Description

Several public-facing NatSpec comments in `FirelightVault`, `ICoverOrderAllocator`, and `IIncidentManager` describe behavior that does not match the underlying implementation, or omit conditions that the implementation actually enforces. These are documentation-only defects as the code itself behaves consistently and correctly in every case, but an integrator or auditor relying on the documented interface rather than tracing the implementation would be misled about actual contract behavior.

### Identified instances

{% stepper %}
{% step %}

## `ICoverOrderAllocator.getEffectiveCapacityConfig()` NatSpec claims "current", but returns next period's configuration

```solidity
/// @notice Current capacity configuration.
function getEffectiveCapacityConfig() external view returns (CapacityConfig memory);
```

```solidity
function getEffectiveCapacityConfig() external view returns (CapacityConfig memory) {
    return _getEffectiveCapacityConfig(_currentPeriod() + 1);
}
```

This contradicts the interface's own documentation elsewhere in the same file, which correctly states the `+1` timing twice, e.g. for `CapacityConfig`:

```solidity
/// @notice Per-period capacity configuration (checkpointed; effective at currentPeriod() + 1).
struct CapacityConfig { ... }

/// @notice Emitted when a new capacity configuration checkpoint is pushed.
/// @param config The capacity configuration that becomes effective (currentPeriod() + 1, or 0 for the first one).
event CapacityConfigUpdated(CapacityConfig config);
```

The sibling function `getEffectiveProtocolConcentration()` documents the identical `+1` pattern correctly, confirming this is an isolated doc error rather than a deliberate convention.
{% endstep %}

{% step %}

## `FirelightVault.deposit()` and `mint()` NatSpec omit the active-incident block in enforced checks

```solidity
/// @notice Deposits assets into the vault and receive shares, with blocklist and pause checks.
function deposit(uint256 assets, address receiver) public override ... returns (uint256) {
    if (assets == 0) revert InvalidAmount();
    if (_hasActiveIncident()) revert CurrentPeriodHasActiveIncident();
    ...
}
```

```solidity
 /**
     * @notice Mints shares by depositing the required amount of assets into the vault, with blocklist and pause checks.
     */
    function mint( uint256 shares,  address receiver ) public ... returns (uint256) {
        if (shares == 0) revert InvalidAmount();

        if (_hasActiveIncident()) revert CurrentPeriodHasActiveIncident();
    ...
}
```

The documentation names two blocking conditions — blocklist and pause. The implementations have a third, independent undocumented condition: an active incident in the current or previous period causing the operation to revert with `CurrentPeriodHasActiveIncident()`.
{% endstep %}

{% step %}

## `ICoverOrderAllocator.InvalidProtocolConcentration()` NatSpec names the wrong trigger for a zero chain ID

```solidity
/// @notice Thrown when a concentration input is invalid (zero chainId/protocol or bps > 100%).
error InvalidProtocolConcentration();
```

```solidity
function _setProtocolConcentration(ProtocolConcentrationInput calldata c) internal {
    if (c.chainId == 0) revert InvalidChainId();
    if (bytes(c.protocol).length == 0) revert InvalidProtocolConcentration();
    if (c.maxProtocolConcentrationBps > BPS_DENOMINATOR) revert InvalidProtocolConcentration();
    ...
}
```

A zero `chainId` actually reverts with `InvalidChainId`, not `InvalidProtocolConcentration`.
{% endstep %}

{% step %}

## `IIncidentManager.IncidentPayoutWindowExpired()` NatSpec describes only the payment condition, but the error is also thrown when trying to cancel expired incidents

```solidity
/// @notice Reverts when an incident can no longer be paid because its payout window expired.
error IncidentPayoutWindowExpired(uint256 incidentId, uint256 incidentPeriod);
```

This error is thrown by `_validateIncidentPeriod`, which gates:

* `cancelPreAssessmentIncident`
* `cancelIncident` via `_activeIncidentWithStatus`
  {% endstep %}

{% step %}

## `IIncidentManager.updateIncidentReportURI()` NatSpec omits its actual third blocking condition

```solidity
/// @notice Updates the report URI for an active incident.
/// The report URI cannot be updated after the incident is CLOSED or CANCELED.
function updateIncidentReportURI(uint256 incidentId, string calldata reportURI) external;
```

The doc names two blocking statuses. The implementation has a third, distinct blocking condition: an expired payout window, which causes a revert with `IncidentPayoutWindowExpired` and can be triggered even while the incident's status is `OPEN`, `CONFIRMED`, or `UNDER_EVALUATION`.

```solidity
function updateIncidentReportURI(uint256 incidentId, string calldata reportURI) external onlyRole(CONFIG_ADMIN_ROLE) {
    Incident storage incident = _incident(incidentId);
    _validateIncidentPeriod(incidentId, incident.period);
    IncidentStatus status = incident.status;
    if (status == IncidentStatus.CLOSED || status == IncidentStatus.CANCELED) {
        revert ReportURIUpdateNotAllowed(incidentId, status);
    }
    _updateIncidentReportURI(incidentId, incident, reportURI);
}
```

{% endstep %}

{% step %}

## `IIncidentManager.InvalidCaptureTimestamp()` doc covers one of its two actual trigger conditions

```solidity
/// @notice Reverts when the vault resolves the capture timestamp to a period outside the payout window.
error InvalidCaptureTimestamp(uint48 captureTimestamp, uint256 capturePeriod, uint256 currentPeriod);
```

Two independent conditions share this one error: an expired period — which is documented — and a future-dated `captureTimestamp` being passed, which is undocumented.

```solidity
if (_isPeriodExpired(period) || captureTimestamp > block.timestamp) {
    revert InvalidCaptureTimestamp(captureTimestamp, period, $.vault.currentPeriod());
}
```

{% endstep %}
{% endstepper %}

## Impact

This issue falls under "Documentation Improvements" and the impact of the documentation discrepancies is on integrators, auditors, and off-chain tooling that reason about contract behavior from the NatSpec rather than tracing the implementation.

* An integrator calling `getEffectiveCapacityConfig()` expecting the presently-active configuration would silently receive next period's configuration instead.
* An integrator relying on `deposit()`/`mint()`'s doc would not expect either call to revert during an active incident, since only blocklist and pause status are documented as blocking.
* Error-handling logic built to distinguish `InvalidProtocolConcentration` from `InvalidChainId` would misclassify a zero-`chainId` submission.
* A reader relying on `IncidentPayoutWindowExpired`'s doc could reasonably assume cancellation, assessment, and report-URI updates remain available after the window closes, when none of them do.
* `updateIncidentReportURI`'s omitted third blocking condition could lead an integrator to expect a report URI update to succeed based on incident status alone, when the payout window independently blocks it.

As per Immunefi's guidelines for Insight issues, this reflects an area in the code that can be improved.

## Recommendations

Update each NatSpec comment to match its implementation:

### `FirelightVault.sol`

```diff
- /// @notice Deposits assets into the vault and receive shares, with blocklist and pause checks.
+ /// @notice Deposits assets into the vault and receive shares, with blocklist, pause, and
+ /// active-incident checks.
  function deposit(
      uint256 assets,
      address receiver
  ) public override whenNotPaused notBlocklisted(_msgSender()) notBlocklisted(receiver) nonReentrant returns (uint256){}

- /// @notice Mints shares by depositing the required amount of assets into the vault, with blocklist and pause checks.
+ /// @notice Mints shares by depositing the required amount of assets into the vault, with
+ /// blocklist, pause, and active-incident checks.
  function mint(
      uint256 shares,
      address receiver
  ) public override whenNotPaused notBlocklisted(_msgSender()) notBlocklisted(receiver) nonReentrant returns (uint256){}
```

### `ICoverOrderAllocator.sol`

```diff
- /// @notice Current capacity configuration.
+ /// @notice Capacity configuration effective at currentPeriod() + 1.
  function getEffectiveCapacityConfig() external view returns (CapacityConfig memory);

- /// @notice Thrown when a concentration input is invalid (zero chainId/protocol or bps > 100%).
+ /// @notice Thrown when a concentration input is invalid (zero protocol length or bps > 100%).
+ /// See {InvalidChainId} for a zero chainId.
  error InvalidProtocolConcentration();
```

### `IIncidentManager.sol`

```diff
- /// @notice Reverts when an incident can no longer be paid because its payout window expired.
+ /// @notice Reverts when an incident's payout window has expired, blocking not only payout but
+ /// also cancellation, assessment, and report URI updates.
  error IncidentPayoutWindowExpired(uint256 incidentId, uint256 incidentPeriod);

  /// @notice Updates the report URI for an active incident.
- /// The report URI cannot be updated after the incident is CLOSED or CANCELED.
+ /// The report URI cannot be updated after the incident is CLOSED or CANCELED, or after its
+ /// payout window has expired.
  function updateIncidentReportURI(uint256 incidentId, string calldata reportURI) external;

- /// @notice Reverts when the vault resolves the capture timestamp to a period outside the payout window.
+ /// @notice Reverts when the vault resolves the capture timestamp to a period outside the payout
+ /// window, or when the capture timestamp is later than the current block timestamp.
  error InvalidCaptureTimestamp(uint48 captureTimestamp, uint256 capturePeriod, uint256 currentPeriod);
```

## Proof of Concept

As per Insight rules, no runnable PoC is required and as this is a documentation report, there are no trigger conditions. The relevant NatSpec and implementation pairs are quoted in full under "Identified instances" above, each demonstrating the documented behavior against the actual code.
