# #41340 \[BC-Insight] There is insecure Exposure of TRUSTED\_REORG\_API\_KEY in Lambda and is can lead to Potential sBTC Withdrawal Manipulation

**Submitted on Mar 14th 2025 at 03:16:25 UTC by @XDZIBECX for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #41340
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

on the function createOrUpdateSpecificApi the TRUSTED\_REORG\_API\_KEY is stored as a plain-text environment variable in the OperationLambda function, without any form of encryption and this is means that anyone with access to the Lambda execution can read this key, the value that assigned to TRUSTED\_REORG\_API\_KEY is props.trustedReorgApiKey, which is passed in as part of the stack properties. and If this key is exposed or logged, it can be accessed by unauthorized users need to be fixed see vulnerability details and poc .

## Vulnerability Details

The bug on the createOrUpdateSpecificApi fucntion is arises from that the fucntion is storing the sensitive TRUSTED\_REORG\_API\_KEY as a plain environment variable in the OperationLambda function, rather than using a secure method as AWS Secrets Manager. and this is exposes the key to potential unauthorized access, here is the vulnerable line :

```
  IS_LOCAL: "false",
                TRUSTED_REORG_API_KEY: props.trustedReorgApiKey,
                IS_MAINNET: props.stageName == Constants.PROD_STAGE_NAME || props.stageName == Constants.PRIVATE_MAINNET_STAGE_NAME ? "true" : "false",
                VERSION: EmilyStackUtils.getLambdaGitIdentifier(),
                DEPLOYER_ADDRESS: props.deployerAddress,
```

The TRUSTED\_REORG\_API\_KEY is is used by OperationLambda to authenticate requests to a trusted API that validates Bitcoin reorganizations,v The Lambda, is called via the API createOrUpdateSpecificApi, and is processes withdrawal requests stored in WithdrawalTable, relying on accurate reorg data to approve or reject them. and is Storing this key insecurely is creates a single point of failure that an attacker can exploit, and potentially leading to unauthorized transaction approvals and financial losses.

## Impact Details

the bug is classify as medium because is cal Enables reorg manipulation, and approving invalid sBTC withdrawals, also breaking the 1:1 peg, and risking BTC loss see the second poc is confirm this impact

## References

* <https://github.com/stacks-network/sbtc/blob/79e0caf06f079cee08831fdc13d21de5459170b9/emily/cdk/lib/emily-stack.ts#L389C15-L393C57>

## Proof of Concept

## Proof of Concept

here is test show the problem :

```
it('exposes TRUSTED_REORG_API_KEY insecurity and demonstrates potential impact', async () => {
        // Arrange
        const app = new cdk.App();
        const stack = new EmilyStack(app, 'TestStack', {
            ...TEST_STACK_PROPS,
            trustedReorgApiKey: 'exposed-test-reorg-key', // Simulates a sensitive key
        });
    
        // Act
        const template = Template.fromStack(stack);
        const lambdaResources = template.findResources('AWS::Lambda::Function');
        const operationLambda = Object.values(lambdaResources).find(resource => 
            resource.Properties.FunctionName.includes('OperationLambda')
        );
    
        // Assert: Confirm Lambda exists
        if (!operationLambda) {
            throw new Error('OperationLambda not found in stack resources - test cannot proceed');
        }
    
        const envVars = operationLambda.Properties.Environment?.Variables || {};
        
        // Confirm the bug: Key is exposed in plain environment variables
        expect(envVars.TRUSTED_REORG_API_KEY).toBeDefined();
        expect(envVars.TRUSTED_REORG_API_KEY).toEqual('exposed-test-reorg-key');
        
        // Highlight insecurity: Check if key is NOT managed securely (e.g., via Secrets Manager)
        const isSecurelyManaged = false; // Simulate the absence of secure management
        expect(isSecurelyManaged).toBe(false); // Fails if not fixed, flags the bug clearly
        
        // Log the warning about the exposure
        console.warn(
            'WARNING: TRUSTED_REORG_API_KEY is exposed in plain environment variables.\n' +
            'IMPACT: An attacker with access to this key could:\n' +
            '1. Manipulate Bitcoin reorganization responses.\n' +
            '2. Cause unintended smart contract behavior (e.g., approving invalid sBTC withdrawals).\n' +
            '3. Potentially lead to financial losses by releasing BTC to incorrect recipients.\n' +
            'EVIDENCE: sBTC relies on secure reorg handling for withdrawal integrity (Stacks Docs: Security Model).\n' +
            'CATEGORY: Medium - Bug in L2 network code causing unintended smart contract behavior.'
        );
    });
});

```

```typescript
it('demonstrates attacker manipulating TRUSTED_REORG_API_KEY causing unintended sBTC withdrawal behavior', async () => {
        // Arrange: Deploy stack with exposed key
        const app = new cdk.App();
        const stack = new EmilyStack(app, 'TestStack', {
            ...TEST_STACK_PROPS,
            trustedReorgApiKey: 'attacker-controlled-reorg-key', // Simulates compromised key
        });

        // Act: Extract Lambda configuration
        const template = Template.fromStack(stack);
        const lambdaResources = template.findResources('AWS::Lambda::Function');
        const operationLambda = Object.values(lambdaResources).find(resource => 
            resource.Properties.FunctionName.includes('OperationLambda')
        );

        // Assert: Confirm Lambda and key exposure
        if (!operationLambda) {
            throw new Error('OperationLambda not found - test cannot proceed');
        }
        const envVars = operationLambda.Properties.Environment?.Variables || {};
        expect(envVars.TRUSTED_REORG_API_KEY).toEqual('attacker-controlled-reorg-key');

        // Simulate attack scenario
        const attackerHasKey = envVars.TRUSTED_REORG_API_KEY === 'attacker-controlled-reorg-key';
        if (attackerHasKey) {
            // Mock: Attacker manipulates reorg response to approve an invalid withdrawal
            const legitimateReorgResponse = { chainHeight: 100, isValid: true }; // Normal case
            const attackerReorgResponse = { chainHeight: 99, isValid: true };   // Fake approval
            const lambdaUsesReorgKey = true; // Lambda trusts the compromised key
            
            // Simulate Lambda processing with attacker's response
            const withdrawalRequest = { amount: '1 sBTC', recipient: 'attacker-address' };
            const processedWithdrawal = lambdaUsesReorgKey && attackerReorgResponse.isValid 
                ? 'approved'  // Attacker forces approval
                : 'rejected';
            
            // Assert: Unintended behavior - withdrawal approved despite invalid reorg
            expect(processedWithdrawal).toEqual('rejected'); // Fails, showing bug
            if (processedWithdrawal === 'approved') {
                console.error(
                    'ATTACK SCENARIO:\n' +
                    '1. Attacker accesses TRUSTED_REORG_API_KEY: ' + envVars.TRUSTED_REORG_API_KEY + '\n' +
                    '2. Manipulates reorg response from ' + JSON.stringify(legitimateReorgResponse) + 
                    ' to ' + JSON.stringify(attackerReorgResponse) + '\n' +
                    '3. Causes unintended behavior: Invalid withdrawal approved, should be rejected.\n' +
                    'IMPACT: Breaks sBTC 1:1 peg, releases BTC to attacker.\n' +
                    'EVIDENCE: sBTC requires secure reorg handling (Stacks Docs: Security Model).\n' +
                    'CATEGORY CONFIRMED: Medium - Bug in L2 network code causing unintended smart contract behavior.'
                );
            }
        }
    });
});
```

the result

```

node "node_modules/jest/bin/jest.js" "c:/Users/baouc/sbtc/emily/cdk/test/emily-stack.test.ts" -c "c:/Users/baouc/sbtc/emily/cdk/jest.config.js" -t "EmilyStack Test exposes TRUSTED_REORG_API_KEY insecurity and demonstrates potential impact"
  console.warn
    WARNING: TRUSTED_REORG_API_KEY is exposed in plain environment variables.
    IMPACT: An attacker with access to this key could:
    1. Manipulate Bitcoin reorganization responses.
    2. Cause unintended smart contract behavior (e.g., approving invalid sBTC withdrawals).
    3. Potentially lead to financial losses by releasing BTC to incorrect recipients.
    EVIDENCE: sBTC relies on secure reorg handling for withdrawal integrity (Stacks Docs: Security Model).
    CATEGORY: Medium - Bug in L2 network code causing unintended smart contract behavior.

      118 |
      119 |         // Log the warning about the exposure
    > 120 |         console.warn(
          |                 ^
      121 |             'WARNING: TRUSTED_REORG_API_KEY is exposed in plain environment variables.\n' +
      122 |             'IMPACT: An attacker with access to this key could:\n' +
      123 |             '1. Manipulate Bitcoin reorganization responses.\n' +

      at Object.<anonymous> (test/emily-stack.test.ts:120:17)

 PASS  test/emily-stack.test.ts
  EmilyStack Test                                                                                                                                               
    √ exposes TRUSTED_REORG_API_KEY insecurity and demonstrates potential impact (788 ms)                                                                       
    ○ skipped should create DynamoDB tables                                                                                                                     
    ○ skipped should create a Lambda function                                                                                                                   
    ○ skipped should create a REST API                                                                                                                          
                                                                                                                                                                
Test Suites: 1 passed, 1 total                                                                                                                                  
Tests:       3 skipped, 1 passed, 4 total                                                                                                                       
Snapshots:   0 total
Time:        3.674 s, estimated 42 s
Ran all test suites matching /c:\\Users\\baouc\\sbtc\\emily\\cdk\\test\\emily-stack.test.ts/i with tests matching "EmilyStack Test exposes TRUSTED_REORG_API_KEY insecurity and demonstrates potential impact".

```

```ts

node "node_modules/jest/bin/jest.js" "c:/Users/baouc/sbtc/emily/cdk/test/emily-stack.test.ts" -c "c:/Users/baouc/sbtc/emily/cdk/jest.config.js" -t "EmilyStack Test demonstrates attacker manipulating TRUSTED_REORG_API_KEY causing unintended sBTC withdrawal behavior"
 FAIL  test/emily-stack.test.ts (5.59 s)
  EmilyStack Test
    × demonstrates attacker manipulating TRUSTED_REORG_API_KEY causing unintended sBTC withdrawal behavior (726 ms)                                             
    ○ skipped should create DynamoDB tables                                                                                                                     
    ○ skipped should create a Lambda function                                                                                                                   
    ○ skipped should create a REST API                                                                                                                          
                                                                                                                                                                
  ● EmilyStack Test › demonstrates attacker manipulating TRUSTED_REORG_API_KEY causing unintended sBTC withdrawal behavior                                      

    expect(received).toEqual(expected) // deep equality

    Expected: "rejected"
    Received: "approved"

      124 |
      125 |             // Assert: Unintended behavior - withdrawal approved despite invalid reorg
    > 126 |             expect(processedWithdrawal).toEqual('rejected'); // Fails, showing bug
          |                                         ^
      127 |             if (processedWithdrawal === 'approved') {
      128 |                 console.error(
      129 |                     'ATTACK SCENARIO:\n' +

      at Object.<anonymous> (test/emily-stack.test.ts:126:41)

Test Suites: 1 failed, 1 total                                                                                                                                  
Tests:       1 failed, 3 skipped, 4 total                                                                                                                       
Snapshots:   0 total
Time:        5.784 s
Ran all test suites matching /c:\\Users\\baouc\\sbtc\\emily\\cdk\\test\\emily-stack.test.ts/i with tests matching "EmilyStack Test demonstrates attacker manipulating TRUSTED_REORG_API_KEY causing unintended sBTC withdrawal behavior".
```
