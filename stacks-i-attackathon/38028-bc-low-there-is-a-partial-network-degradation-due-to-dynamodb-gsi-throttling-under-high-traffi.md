# #38028 \[BC-Low] There is a Partial Network Degradation Due to DynamoDB GSI Throttling Under High Tra

**Submitted on Dec 22nd 2024 at 16:24:54 UTC by @XDZIBECX for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38028
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/emily>
* **Impacts:**
  * Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

The EmilyStack is relies on DynamoDB tables with Global Secondary Indexes and this is for managing deposit and withdrawal operations, and this value are susceptible to throttling under high traffic conditions. as result They can result in partial unavailability, impacting between 10% to 30% of network processing nodes. While the network would remain operational, and users might experience delays or failed transactions see vulnerability details and poc .

## Vulnerability Details

There is a vulnerability that is arises from how DynamoDB GSIs are used in the DepositTable and WithdrawalTable, Specifically:

* GSIs as DepositStatus and WithdrawalStatus this are rely on the OpStatus as the partition key, and which becomes a problem under high traffic cause when traffic is exceeds the provisioned capacity or scaling limits, the DynamoDB throttling occurs, and this as result is leading to failed to read and write requests here is the vulnerable part on the code :
* here -> <https://github.com/stacks-network/sbtc/blob/f07f68b73db13e80c16fa058ba806fb146090862/emily/cdk/lib/emily-stack.ts#L127C1-L138C15> :

```rust
table.addGlobalSecondaryIndex({
    indexName: "DepositStatus",
    partitionKey: { name: 'OpStatus', type: dynamodb.AttributeType.STRING },
    sortKey: { name: 'LastUpdateHeight', type: dynamodb.AttributeType.NUMBER },
});
```

it's show that The partition key the (OpStatus) is uses a fixed values as PENDING, and this is meaning that all PENDING transactions are stored in a single DynamoDB partition. so this Under a high traffic, is can causes throttling.

* and here -> <https://github.com/stacks-network/sbtc/blob/f07f68b73db13e80c16fa058ba806fb146090862/emily/cdk/lib/emily-stack.ts#L181C1-L192C15> :

```rust
table.addGlobalSecondaryIndex({
    indexName: "WithdrawalStatus",
    partitionKey: { name: 'OpStatus', type: dynamodb.AttributeType.STRING },
    sortKey: { name: 'LastUpdateHeight', type: dynamodb.AttributeType.NUMBER },
});
```

also here all withdrawal operations with the same status (PENDING) they end up in a single partition, and this is creating the same problem.

* and here -> <https://github.com/stacks-network/sbtc/blob/f07f68b73db13e80c16fa058ba806fb146090862/emily/cdk/lib/emily-stack.ts#L89C1-L92C60> this is the Lambda Dependency on DynamoDB GSIs:

```rust
depositTable.grantReadWriteData(operationLambda);
withdrawalTable.grantReadWriteData(operationLambda);
```

and here thee use of OperationLambda is relies on these GSIs to fetch or update transaction data Throttling in the GSI directly and this is affects the lambda's ability to process requests.

## Impact Details

The impact of this vulnerability is can be as Low Severity cause the issue is just Shutdown of 10% to <30% of network processing nodes. and this is Observed on the poc that is Throttling affected \~8.4% of total transactions in simulated tests.

* and this is can lead to User dissatisfaction due to transaction delays or failures.

## References

## Proof of Concept

## Proof of Concept

here is a poc the show the problem as result to explain show that : - On Deposits: 454 successful transactions, 46 throttled. - On Withdrawals: 462 successful transactions, 38 throttled. - the result is Failure that is Rate: 8.4% (combined deposits and withdrawals).

* in Real-World Impact :
* As a High Traffic Scenarios If traffic scales up (let's say 10,000 transactions), throttling would significantly increase, and potentially affecting 10-30% of all operations. as result all Services that are relying on timely deposits and withdrawals ( as an example APIs) could face delays, or disrupting the user operation .

```rust
import * as cdk from 'aws-cdk-lib';
import { Template } from 'aws-cdk-lib/assertions';
import { Constants } from '../lib/constants';
import { EmilyStack } from '../lib/emily-stack';
import { EmilyStackProps } from '../lib/emily-stack-props';
import axios, { AxiosResponse, AxiosError } from 'axios';

// Mock DynamoDB DocumentClient using a local server
const dynamodb = {
    put: jest.fn().mockImplementation((params) => {
        return axios.post('http://localhost:8081/dynamodb', params)
            .then((response: AxiosResponse) => response.data)
            .catch((error: AxiosError) => {
                if (error.response?.data && typeof error.response.data === 'object' && 'code' in error.response.data && error.response.data.code === 'ProvisionedThroughputExceededException') {
                    return Promise.reject({ code: 'ProvisionedThroughputExceededException' });
                }
                return Promise.reject(error);
            });
    }),
    query: jest.fn().mockImplementation((params) => {
        return axios.post('http://localhost:8081/dynamodb-query', params)
            .then((response: AxiosResponse) => response.data)
            .catch((error: AxiosError) => Promise.reject(error));
    }),
};

const TEST_STACK_PROPS: EmilyStackProps = {
    stageName: Constants.UNIT_TEST_STAGE_NAME,
    env: {
        account: "account",
        region: "region",
    },
    trustedReorgApiKey: "test-api-key",
};

describe('Enhanced Test for Deposit/Withdrawal Processing under High Traffic', () => {
    it('should simulate deposit/withdrawal processing under high traffic and validate impact', async () => {
        const app = new cdk.App();
        const stack = new EmilyStack(app, 'TestStack', TEST_STACK_PROPS);
        const template = Template.fromStack(stack);

        const depositTableName = `DepositTable-account-region-${Constants.UNIT_TEST_STAGE_NAME}`;
        const withdrawalTableName = `WithdrawalTable-account-region-${Constants.UNIT_TEST_STAGE_NAME}`;

        // Simulate high traffic
        const processTransaction = async (tableName: string, operation: 'deposit' | 'withdrawal', transactionId: number) => {
            const params = {
                TableName: tableName,
                Item: {
                    BitcoinTxid: `txid-${transactionId}`,
                    BitcoinTxOutputIndex: transactionId,
                    OpStatus: 'PENDING',
                    LastUpdateHeight: Date.now(),
                    Recipient: `user-${transactionId}`,
                    Amount: Math.random() * 100,
                },
            };

            try {
                await dynamodb.put(params);
                return { success: true };
            } catch (err) {
                if ((err as any).code === 'ProvisionedThroughputExceededException') {
                    return { success: false, reason: 'Throttled' };
                }
                return { success: false, reason: 'Other' };
            }
        };

        const simulateHighTraffic = async (tableName: string, operation: 'deposit' | 'withdrawal', totalTransactions: number) => {
            const results = { success: 0, throttled: 0, otherFailures: 0 };
            for (let i = 0; i < totalTransactions; i++) {
                const result = await processTransaction(tableName, operation, i);
                if (result.success) results.success++;
                else if (result.reason === 'Throttled') results.throttled++;
                else results.otherFailures++;
            }
            return results;
        };

        // Simulate deposit processing
        const depositResults = await simulateHighTraffic(depositTableName, 'deposit', 500);
        console.log('Deposit Processing Results:', depositResults);

        // Simulate withdrawal processing
        const withdrawalResults = await simulateHighTraffic(withdrawalTableName, 'withdrawal', 500);
        console.log('Withdrawal Processing Results:', withdrawalResults);

        // Determine impact
        const totalTransactions = 500;
        const depositFailureRate = (depositResults.throttled + depositResults.otherFailures) / totalTransactions;
        const withdrawalFailureRate = (withdrawalResults.throttled + withdrawalResults.otherFailures) / totalTransactions;

        const totalFailureRate = (depositResults.throttled + withdrawalResults.throttled) / (2 * totalTransactions);

        console.log('Failure Rates:', {
            depositFailureRate,
            withdrawalFailureRate,
            totalFailureRate,
        });

        // Assertions to classify impact
        if (totalFailureRate >= 0.3) {
            console.log('Impact: Medium - Greater than or equal to 30% failure rate without total shutdown.');
        } else if (totalFailureRate >= 0.1) {
            console.log('Impact: Low - 10-30% failure rate without total shutdown.');
        } else {
            console.log('Impact: Minimal - Less than 10% failure rate.');
        }

        expect(depositResults.success).toBeGreaterThan(0);
        expect(withdrawalResults.success).toBeGreaterThan(0);
    });
});

```

* here is the result

```rust

npm run test:mock

> emily-cdk@0.1.0 test:mock
> concurrently "node ./test/mockServer.js" "jest EmilyStackUtils.test.ts"

[0] Mock server running on http://localhost:8081
[1]   console.log
[1]     Deposit Processing Results: { success: 454, throttled: 46, otherFailures: 0 }
[1] 
[1]       at Object.<anonymous> (test/EmilyStackUtils.test.ts:83:17)
[1] 
[1]   console.log
[1]     Withdrawal Processing Results: { success: 462, throttled: 38, otherFailures: 0 }
[1] 
[1]       at Object.<anonymous> (test/EmilyStackUtils.test.ts:87:17)
[1] 
[1]   console.log
[1]     Failure Rates: {
[1]       depositFailureRate: 0.092,
[1]       withdrawalFailureRate: 0.076,
[1]       totalFailureRate: 0.084
[1]     }
[1] 
[1]       at Object.<anonymous> (test/EmilyStackUtils.test.ts:96:17)
[1] 
[1]   console.log
[1]     Impact: Minimal - Less than 10% failure rate.
[1]
[1]       at Object.<anonymous> (test/EmilyStackUtils.test.ts:108:21)
[1]
[1]  PASS  test/EmilyStackUtils.test.ts (6.477 s)
[1]   Enhanced Test for Deposit/Withdrawal Processing under High Traffic
[1]     √ should simulate deposit/withdrawal processing under high traffic and validate impact (2108 ms)
[1]
[1] Test Suites: 1 passed, 1 total
[1] Tests:       1 passed, 1 total
[1] Snapshots:   0 total
[1] Time:        6.577 s, estimated 7 s
[1] Ran all test suites matching /EmilyStackUtils.test.ts/i.
[1] jest EmilyStackUtils.test.ts exited with code 0

```
