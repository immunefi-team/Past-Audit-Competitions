# Stacks II Attackathon

## Reports by Severity

<details>

<summary>High</summary>

* \#42747 \[BC-High] Large BTC transactions with many sbtc deposits can permanently crash/halt all signers
* \#40692 \[BC-High] Calling multiple withdrawals on a single transaction causes Signers to halt and the network to stop
* \#42752 \[BC-High] Signer can be DOSed through their libp2p component
* \#40806 \[BC-High] Users can submit deposits containing large \`reclaim\_scripts\` to DoS Emily and Signers

</details>

<details>

<summary>Medium</summary>

* \#42773 \[BC-Medium] Signers can be compromised by a libp2p DoS attack
* \#40731 \[BC-Medium] A malicious signer can force a panic in the coordinator by sending \`DkgFailure::BadPrivateShares\` with an invalid signer ID
* \#42404 \[BC-Medium] A signer can OOM kill other signers during DKG verification
* \#40655 \[BC-Medium] Malicious signers can give different votes to other Signers to prevent sBTC withdrawal
* \#41111 \[BC-Medium] A malicious signer could manipulate withdrawal decisions preventing accepted and rejected withdrawals from getting confirmed on Stacks chain

</details>

<details>

<summary>Low</summary>

* \#41014 \[BC-Low] The signer can submit multi-tx first to make the coordinator's submission fail
* \#42764 \[BC-Low] A BTC wallet on signer blocklists can cause network DoS
* \#40770 \[BC-Low] Unvalidated withdrawal events allow data manipulation and denial of service in Emily

</details>

<details>

<summary>Insight</summary>

* \#41202 \[BC-Insight] A malicious signer can force a failure of the signature round by providing a key ID they don't own
* \#41597 \[BC-Insight] Emily server can crash their connected Stacks node when processing a large number of events
* \#42750 \[BC-Insight] Subtraction overflow risk in WSTS FIRE coordinator
* \#41340 \[BC-Insight] There is insecure Exposure of TRUSTED\_REORG\_API\_KEY in Lambda and is can lead to Potential sBTC Withdrawal Manipulation

</details>

## Reports by Type

<details>

<summary>Blockchain/DLT</summary>

* \#41014 \[BC-Low] The signer can submit multi-tx first to make the coordinator's submission fail
* \#42747 \[BC-High] Large BTC transactions with many sbtc deposits can permanently crash/halt all signers
* \#41202 \[BC-Insight] A malicious signer can force a failure of the signature round by providing a key ID they don't own
* \#42773 \[BC-Medium] Signers can be compromised by a libp2p DoS attack
* \#40692 \[BC-High] Calling multiple withdrawals on a single transaction causes Signers to halt and the network to stop
* \#41597 \[BC-Insight] Emily server can crash their connected Stacks node when processing a large number of events
* \#42752 \[BC-High] Signer can be DOSed through their libp2p component
* \#40731 \[BC-Medium] A malicious signer can force a panic in the coordinator by sending \`DkgFailure::BadPrivateShares\` with an invalid signer ID
* \#42764 \[BC-Low] A BTC wallet on signer blocklists can cause network DoS
* \#42404 \[BC-Medium] A signer can OOM kill other signers during DKG verification
* \#40770 \[BC-Low] Unvalidated withdrawal events allow data manipulation and denial of service in Emily
* \#42750 \[BC-Insight] Subtraction overflow risk in WSTS FIRE coordinator
* \#40806 \[BC-High] Users can submit deposits containing large \`reclaim\_scripts\` to DoS Emily and Signers
* \#40655 \[BC-Medium] Malicious signers can give different votes to other Signers to prevent sBTC withdrawal
* \#41340 \[BC-Insight] There is insecure Exposure of TRUSTED\_REORG\_API\_KEY in Lambda and is can lead to Potential sBTC Withdrawal Manipulation
* \#41111 \[BC-Medium] A malicious signer could manipulate withdrawal decisions preventing accepted and rejected withdrawals from getting confirmed on Stacks chain

</details>
