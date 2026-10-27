# Shardeum Core III

## Reports by Severity

<details>

<summary>Critical</summary>

* \#39979 \[BC-Critical] Total network shutdown via fixDeserializedWrappedEVMAccount call through binary\_repair\_oos\_accounts endpoint
* \#40007 \[BC-Critical] Drain node staking account due to improper validation of SetCertTime internal transaction
* \#39191 \[BC-Critical] JoinRoute: Attacker reachable input serialization
* \#39465 \[BC-Critical] Lack of authorization on InitClaimReward transaction allows attacker to prevent all nodes from being rewarded
* \#39511 \[BC-Critical] Malicious node can drain balance of other node's nominator evm address
* \#40000 \[BC-Critical] Improper input validation in fixDeserializedWrappedEVMAccount leads to DOS and total network shutdown
* \#39973 \[BC-Critical] Standard node rewarding flow can be blocked
* \#40005 \[BC-Critical] Removal of node out of network via remove by app gossip and signature duplications
* \#39994 \[BC-Critical] Tricking nodes into signing nearly-arbitrary data
* \#39355 \[BC-Critical] Tricking Legit node to sign their own apoptosis request payload
* \#39812 \[BC-Critical] Bypass \`SetCertTime\` transaction signature check #1
* \#39507 \[BC-Critical] Insufficient validation on ClaimReward transaction allows attacker to claim an inflated reward OR prevent all nodes from being rewarded
* \#39364 \[BC-Critical] Trusting heavily on "appData" enables infinite SHM duplication through double-spend exploit
* \#39875 \[BC-Critical] Lack of validation of node deactivation time in \`ClaimRewards\` allows to steal rewards
* \#39675 \[BC-Critical] Reward Exploitation via Unvalidated Node Status in "initRewardTX"
* \#39678 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to capitalization
* \#39679 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to ignored suffixes
* \#39791 \[BC-Critical] Filling the queue with "setCertTime" stop the network from processing new transactions
* \#39813 \[BC-Critical] Bypass \`SetCertTime\` transaction signature check #2
* \#39838 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to signature malleability
* \#39871 \[BC-Critical] Lack of consensus voting in best cycle calculation allows a malicious validator to fake cycle data and crash all nodes
* \#39876 \[BC-Critical] Receiving rewards multiple times for the same period
* \#39885 \[BC-Critical] Signature forgery on behalf of network nodes using binary\_sign\_app\_data endpoint
* \#39921 \[BC-Critical] accountDeserializer isn't type safe
* \#39873 \[BC-Critical] Lack of validation of node activation time in \`InitRewardTimes\` allows to steal rewards
* \#39811 \[BC-Critical] Inducing large memory allocation via /join endpoint

</details>

<details>

<summary>High</summary>

* \#39149 \[BC-High] EIP-2930 transactions with 20k-address overload the nodes and force the network into "safety" mode

</details>

<details>

<summary>Medium</summary>

* \#39395 \[BC-Medium] got.get without response limit
* \#39850 \[BC-Medium] Bypass TransferFromSecureAccount transaction validations
* \#39913 \[BC-Medium] No rate Limiting in resource-intensive endpoint

</details>

<details>

<summary>Insight</summary>

* \#39752 \[BC-Insight] There is an issue related to incorrect version parsing and comparison logic lead to incorrect node validation,
* \#39463 \[BC-Insight] \`multiSendWithHeader\` and \`sendWithHeader\` have JSON injection vulnerability
* \#39027 \[BC-Insight] Abusive Join request handler node
* \#39882 \[BC-Insight] Data unsubscribe same node replay
* \#39164 \[BC-Insight] Service point exhaustion
* \#39103 \[BC-Insight] Unchecked data size in "getStakeTxBlobFromEVMTx()" can use lots of CPU resources

</details>

## Reports by Type

<details>

<summary>Blockchain/DLT</summary>

* \#39979 \[BC-Critical] Total network shutdown via fixDeserializedWrappedEVMAccount call through binary\_repair\_oos\_accounts endpoint
* \#40007 \[BC-Critical] Drain node staking account due to improper validation of SetCertTime internal transaction
* \#39191 \[BC-Critical] JoinRoute: Attacker reachable input serialization
* \#39752 \[BC-Insight] There is an issue related to incorrect version parsing and comparison logic lead to incorrect node validation,
* \#39395 \[BC-Medium] got.get without response limit
* \#39465 \[BC-Critical] Lack of authorization on InitClaimReward transaction allows attacker to prevent all nodes from being rewarded
* \#39463 \[BC-Insight] \`multiSendWithHeader\` and \`sendWithHeader\` have JSON injection vulnerability
* \#39511 \[BC-Critical] Malicious node can drain balance of other node's nominator evm address
* \#40000 \[BC-Critical] Improper input validation in fixDeserializedWrappedEVMAccount leads to DOS and total network shutdown
* \#39973 \[BC-Critical] Standard node rewarding flow can be blocked
* \#40005 \[BC-Critical] Removal of node out of network via remove by app gossip and signature duplications
* \#39994 \[BC-Critical] Tricking nodes into signing nearly-arbitrary data
* \#39355 \[BC-Critical] Tricking Legit node to sign their own apoptosis request payload
* \#39812 \[BC-Critical] Bypass \`SetCertTime\` transaction signature check #1
* \#39507 \[BC-Critical] Insufficient validation on ClaimReward transaction allows attacker to claim an inflated reward OR prevent all nodes from being rewarded
* \#39364 \[BC-Critical] Trusting heavily on "appData" enables infinite SHM duplication through double-spend exploit
* \#39850 \[BC-Medium] Bypass TransferFromSecureAccount transaction validations
* \#39149 \[BC-High] EIP-2930 transactions with 20k-address overload the nodes and force the network into "safety" mode
* \#39027 \[BC-Insight] Abusive Join request handler node
* \#39882 \[BC-Insight] Data unsubscribe same node replay
* \#39875 \[BC-Critical] Lack of validation of node deactivation time in \`ClaimRewards\` allows to steal rewards
* \#39164 \[BC-Insight] Service point exhaustion
* \#39675 \[BC-Critical] Reward Exploitation via Unvalidated Node Status in "initRewardTX"
* \#39678 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to capitalization
* \#39679 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to ignored suffixes
* \#39791 \[BC-Critical] Filling the queue with "setCertTime" stop the network from processing new transactions
* \#39103 \[BC-Insight] Unchecked data size in "getStakeTxBlobFromEVMTx()" can use lots of CPU resources
* \#39813 \[BC-Critical] Bypass \`SetCertTime\` transaction signature check #2
* \#39838 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to signature malleability
* \#39871 \[BC-Critical] Lack of consensus voting in best cycle calculation allows a malicious validator to fake cycle data and crash all nodes
* \#39876 \[BC-Critical] Receiving rewards multiple times for the same period
* \#39885 \[BC-Critical] Signature forgery on behalf of network nodes using binary\_sign\_app\_data endpoint
* \#39913 \[BC-Medium] No rate Limiting in resource-intensive endpoint
* \#39921 \[BC-Critical] accountDeserializer isn't type safe
* \#39873 \[BC-Critical] Lack of validation of node activation time in \`InitRewardTimes\` allows to steal rewards
* \#39811 \[BC-Critical] Inducing large memory allocation via /join endpoint

</details>
