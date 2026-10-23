# 59330 sc insight rescuer role not assigned during initialization

**Submitted on Nov 11th 2025 at 07:50:57 UTC by @VinayVig for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59330
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The RESCUER\_ROLE is never assigned during the initialization of the FirelightVault contract. As a result, no address initially has permission to perform rescue operations (such as recovering shares or pending withdrawals from blocklisted accounts). This could cause operational delays or configuration issues post-deployment.

## Vulnerability Details

In the initialize() function, the contract grants roles like DEFAULT\_ADMIN\_ROLE, DEPOSIT\_LIMIT\_UPDATE\_ROLE, BLOCKLIST\_ROLE, PAUSE\_ROLE, and PERIOD\_CONFIGURATION\_UPDATE\_ROLE based on the provided InitParams. However, the RESCUER\_ROLE is missing from this setup:

```solidity
  function initialize(
        IERC20 _asset,
        string memory _name,
        string memory _symbol,
        bytes memory _initParams
    ) public initializer {
        InitParams memory initParams = abi.decode(_initParams, (InitParams));
        __ERC20_init(_name, _symbol);
        __ERC4626_init(_asset);
        __Pausable_init();
        __ReentrancyGuard_init();
        __AccessControl_init();

        if (address(_asset) == address(0)) {
            revert InvalidAssetAddress();
        }

        if (initParams.depositLimit == 0) {
            revert InvalidDepositLimit();
        }

        if (initParams.periodConfigurationDuration == 0) {
            revert InvalidPeriodConfigurationDuration();
        }

        if (initParams.defaultAdmin == address(0)) {
            revert InvalidAdminAddress();
        }

        depositLimit = initParams.depositLimit;
        _addPeriodConfiguration(Time.timestamp(), initParams.periodConfigurationDuration);
        contractVersion = 1;

        _grantRole(DEFAULT_ADMIN_ROLE, initParams.defaultAdmin);
        if (initParams.limitUpdater != address(0)) {
            _grantRole(DEPOSIT_LIMIT_UPDATE_ROLE, initParams.limitUpdater);
        }

        if (initParams.blocklister != address(0)) {
            _grantRole(BLOCKLIST_ROLE, initParams.blocklister);
        }

        if (initParams.pauser != address(0)) {
            _grantRole(PAUSE_ROLE, initParams.pauser);
        }

        if (initParams.periodConfigurationUpdater != address(0)) {
            _grantRole(PERIOD_CONFIGURATION_UPDATE_ROLE, initParams.periodConfigurationUpdater);
        }
@>       // @audit-info where is RESCUER_ROLE
    }
```

## Impact Details

Although this does not pose a direct financial risk, it can delay the rescue of funds from blocklisted addresses and reduce contract responsiveness during critical recovery operations. If the admin forgets to assign the role, rescue functionality remains unusable indefinitely.

## References

initialize() function in FirelightVault.sol rescueSharesFromBlocklisted() and rescueWithdrawFromBlocklisted() functions

## Proof of Concept

## Proof of Concept

1. Deploy the FirelightVault contract using the provided initialize() function parameters.
2. Observe that RESCUER\_ROLE is never assigned within the initializer.
3. Verify post-deployment that no address holds the RESCUER\_ROLE:

```solidity
bool hasRole = vault.hasRole(RESCUER_ROLE, someAddress);
// returns false
```

4. This confirms that rescue functionality is unavailable until the admin manually assigns the role.
