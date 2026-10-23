# Spectra Finance

## Reports by Severity

<details>

<summary>Medium</summary>

* \#44064 \[SC-Medium] Dispatcher incorrect validation causes principal tokens to be stuck in inheriting contract allowing attacker to steal user funds
* \#44167 \[SC-Medium] Incorrect balance check in PT redemption commands

</details>

<details>

<summary>Low</summary>

* \#43729 \[SC-Low] Silent execution failure on \`Dispatcher::\_dispatch\` due to unchecked return value on \`Dispatcher:TRANSFER\_NATIVE\` operation
* \#44175 \[SC-Low] Missing Success Check for payable(recipient).call
* \#44131 \[SC-Low] An attacker can steal frozen user ETH due to Dispatcher error
* \#43274 \[SC-Low] \`TRANSFER\_NATIVE\` Command in Dispatcher Does Not Check Return Value of Low-Level Call
* \#44170 \[SC-Low] Missing Check for Native ETH Transfer Success Allows Silent Failures and Potential Theft of Funds
* \#44173 \[SC-Low] Unchecked Low-Level Call in TRANSFER\_NATIVE in \`Dispatcher::\_dispatch\` Can Lead to Locked Ether and Potential Theft
* \#43912 \[SC-Low] Lack of ETH Success Transfer Checks in Dispatcher.sol
* \#43408 \[SC-Low] Not checking call success in \`TRANSFER\_NATIVE\`
* \#43490 \[SC-Low] TRANSFER\_NATIVE in Dispatcher can lead to loss of funds due to not checking user can receive ETH
* \#43380 \[SC-Low] Missing Error Check in TRANSFER\_NATIVE Command
* \#43528 \[SC-Low] fund freeze scenario
* \#43611 \[SC-Low] Unchecked ETH Transfer in TRANSFER\_NATIVE Command Risks Silent Failures
* \#44035 \[SC-Low] Lack of validation in native transfer allows attacker to steal user funds
* \#43987 \[SC-Low] Unchecked low-level ETH transfer in \`Dispatcher.sol\` may lead to undetected failures
* \#43856 \[SC-Low] Dispatcher.\_dispatch() does not revert on failure of transfer of funds when called with the TRANSFER\_NATIVE command
* \#44158 \[SC-Low] Dispatcher does not check if native transfers are successful
* \#44081 \[SC-Low] Users ETH could be stuck forever without a way to recover it
* \#43712 \[SC-Low] Silent ETH transfer failure in \`TRANSFER\_NATIVE\` command leads to permament locking of user funds
* \#43981 \[SC-Low] Silent ETH transfer failure in \`TRANSFER\_NATIVE\` command can permanently lock user funds
* \#44161 \[SC-Low] Return value of low level call not ckecked in \`Dispatcher.sol\` contract
* \#44101 \[SC-Low] \`\_dispatch()\` incorrectly assumes revert bubbling when transferring native tokens.
* \#43659 \[SC-Low] Silent ETH transfer failure in \`Dispatcher.sol\` leads to permament freezing of funds
* \#43469 \[SC-Low] Return value of low level call not checked can cause silent Reverts
* \#44091 \[SC-Low] Lack of ETH transfer check leads to stolen funds
* \#43803 \[SC-Low] Boolean success returned from address.call{value: amount}() not checked

</details>

<details>

<summary>Insight</summary>

* \#44084 \[SC-Insight] Incorrect Nat spec in \`calcIBTsToTokenizeForCurvePool\` and \`calcIBTsToTokenizeForCurvePoolCustomProp\`
* \#43195 \[SC-Insight] \`Dispatcher.sol\` uses \`initializer\` modifier instead of \`onlyInitializing\`
* \#44083 \[SC-Insight] Inconsistency in \`CurvePoolUtil\`
* \#43402 \[SC-Insight] Function \`getPTUnderlyingUnit\` could be marked external
* \#43464 \[SC-Insight] Refactoring \`Router.sol\` for gas savings and reducing code redundancy from two different \`Router::execute()\` which can result in undesirable outcomes for potentially delayed tra...
* \#43314 \[SC-Insight] Oracle functions mislead integrators as it is not compatible with Chainlink Price feed behaviour
* \#43971 \[SC-Insight] Incorrect NatSpec Tag in removeRateOracle() Misrepresents Function Reference

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#43729 \[SC-Low] Silent execution failure on \`Dispatcher::\_dispatch\` due to unchecked return value on \`Dispatcher:TRANSFER\_NATIVE\` operation
* \#44175 \[SC-Low] Missing Success Check for payable(recipient).call
* \#44084 \[SC-Insight] Incorrect Nat spec in \`calcIBTsToTokenizeForCurvePool\` and \`calcIBTsToTokenizeForCurvePoolCustomProp\`
* \#43195 \[SC-Insight] \`Dispatcher.sol\` uses \`initializer\` modifier instead of \`onlyInitializing\`
* \#44131 \[SC-Low] An attacker can steal frozen user ETH due to Dispatcher error
* \#43274 \[SC-Low] \`TRANSFER\_NATIVE\` Command in Dispatcher Does Not Check Return Value of Low-Level Call
* \#44170 \[SC-Low] Missing Check for Native ETH Transfer Success Allows Silent Failures and Potential Theft of Funds
* \#44173 \[SC-Low] Unchecked Low-Level Call in TRANSFER\_NATIVE in \`Dispatcher::\_dispatch\` Can Lead to Locked Ether and Potential Theft
* \#44083 \[SC-Insight] Inconsistency in \`CurvePoolUtil\`
* \#43912 \[SC-Low] Lack of ETH Success Transfer Checks in Dispatcher.sol
* \#43402 \[SC-Insight] Function \`getPTUnderlyingUnit\` could be marked external
* \#43464 \[SC-Insight] Refactoring \`Router.sol\` for gas savings and reducing code redundancy from two different \`Router::execute()\` which can result in undesirable outcomes for potentially delayed tra...
* \#43408 \[SC-Low] Not checking call success in \`TRANSFER\_NATIVE\`
* \#43490 \[SC-Low] TRANSFER\_NATIVE in Dispatcher can lead to loss of funds due to not checking user can receive ETH
* \#43380 \[SC-Low] Missing Error Check in TRANSFER\_NATIVE Command
* \#43528 \[SC-Low] fund freeze scenario
* \#43611 \[SC-Low] Unchecked ETH Transfer in TRANSFER\_NATIVE Command Risks Silent Failures
* \#44035 \[SC-Low] Lack of validation in native transfer allows attacker to steal user funds
* \#43987 \[SC-Low] Unchecked low-level ETH transfer in \`Dispatcher.sol\` may lead to undetected failures
* \#43856 \[SC-Low] Dispatcher.\_dispatch() does not revert on failure of transfer of funds when called with the TRANSFER\_NATIVE command
* \#44158 \[SC-Low] Dispatcher does not check if native transfers are successful
* \#44081 \[SC-Low] Users ETH could be stuck forever without a way to recover it
* \#43712 \[SC-Low] Silent ETH transfer failure in \`TRANSFER\_NATIVE\` command leads to permament locking of user funds
* \#43981 \[SC-Low] Silent ETH transfer failure in \`TRANSFER\_NATIVE\` command can permanently lock user funds
* \#44064 \[SC-Medium] Dispatcher incorrect validation causes principal tokens to be stuck in inheriting contract allowing attacker to steal user funds
* \#44161 \[SC-Low] Return value of low level call not ckecked in \`Dispatcher.sol\` contract
* \#44101 \[SC-Low] \`\_dispatch()\` incorrectly assumes revert bubbling when transferring native tokens.
* \#43659 \[SC-Low] Silent ETH transfer failure in \`Dispatcher.sol\` leads to permament freezing of funds
* \#43314 \[SC-Insight] Oracle functions mislead integrators as it is not compatible with Chainlink Price feed behaviour
* \#43469 \[SC-Low] Return value of low level call not checked can cause silent Reverts
* \#44091 \[SC-Low] Lack of ETH transfer check leads to stolen funds
* \#43803 \[SC-Low] Boolean success returned from address.call{value: amount}() not checked
* \#43971 \[SC-Insight] Incorrect NatSpec Tag in removeRateOracle() Misrepresents Function Reference
* \#44167 \[SC-Medium] Incorrect balance check in PT redemption commands

</details>
