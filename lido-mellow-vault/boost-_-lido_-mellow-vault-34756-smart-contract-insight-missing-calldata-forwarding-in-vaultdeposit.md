# Boost \_ Lido\_ Mellow Vault 34756 - \[Smart Contract - Insight] Missing calldata forwarding in Vaultde

Submitted on Fri Aug 23 2024 22:16:23 GMT-0400 (Atlantic Standard Time) by @marchev for [Boost | Lido: Mellow Vault](https://immunefi.com/bounty/boost-lido/)

Report ID: #34756

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x5E362eb2c0706Bd1d134689eC75176018385430B>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `Vault#deposit()` function uses a custom validator to validate deposits, but it only passes the function signature (`msg.sig`) and not the calldata. This prevents any validation based on the arguments provided in the deposit call, potentially allowing invalid or malicious deposits to bypass critical validation logic.

## Vulnerability Details

The `Vault` contract in this protocol allows for modular configuration using custom validators. However, in the `Vault#deposit()` function, only `msg.sig` (the function signature) is passed to the validator:

```solidity
	IValidator(configurator.validator()).validate(
		msg.sender,
		address(this),
		abi.encodeWithSelector(msg.sig) //@audit The deposit() calldata is not passed here
	);
```

This flaw means the validator cannot access the calldata (e.g., deposit amounts or address), restricting the ability to implement validation logic that depends on deposit arguments. This severely limits custom validation possibilities for deposits that require checking input data.

## Impact Details

Since the validator cannot inspect calldata, any custom logic based on deposit arguments, such as limiting deposit amounts or validating sender addresses, cannot be enforced. Thus, any restrictions that could be implemented in a custom validator will be bypassed or put the contract into a non-working state.

## References

<https://github.com/mellow-finance/mellow-lrt/blob/1c885ad9a2964ca88ad3e59c3a7411fc0059aa34/src/Vault.sol#L298-L302>

## Proof of concept

## Proof of Concept

The following coded PoC demonstrates how using a custom deposit validator that validates that the deposit beneficiary is not `address(0)` causes the `deposit()` function to revert. This is because only `msg.sig` is passed to the validator and no calldata whatsoever which causes the calldata decoding to fail.

Add the following test case to `VaultTest.t.sol`:

```sol
    function test_deposit_custom_validator_gets_no_calldata() external {
        Vault vault = new Vault("Mellow LRT Vault", "mLRT", admin);
        vm.startPrank(admin);
        vault.grantRole(vault.ADMIN_DELEGATE_ROLE(), admin);
        vault.grantRole(vault.OPERATOR(), operator);

        ManagedValidator validator = new ManagedValidator(admin);

        VaultConfigurator configurator = VaultConfigurator(
            address(vault.configurator())
        );
        configurator.stageValidator(address(validator));
        configurator.commitValidator();

        validator.setCustomValidator(address(vault), address(new NoZeroAddressDepositValidator()));

        _setUp(vault);
        vm.stopPrank();
        _initialDeposit(vault);
    }
```

Also, add the following custom deposit validator at the end of the `VaultTest.t.sol` file:

```sol

contract NoZeroAddressDepositValidator is IValidator {

    function validate(address from, address to, bytes calldata data) external view {
        console.log("calldata (size=%s):", data.length);
        console.logBytes(data);
        (address beneficiary,,,,) = abi.decode(data[4:], (address, uint256[], uint256, uint256, uint256));
        require(beneficiary != address(0), "deposit beneficiary cannot be address(0)");
    }
}
```

Run the PoC via:

```sh
$ forge test -vvv --fork-url $RPC_URL --fork-block-number 19845261 --match-test test_deposit_custom_validator_gets_no_calldata
```
