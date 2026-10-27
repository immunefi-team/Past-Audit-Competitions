# Boost \_ Shardeum\_ Ancillaries 34492 - \[Websites and Applications - Insight] DoS via unbounded tx id

Submitted on Tue Aug 13 2024 22:19:03 GMT-0400 (Atlantic Standard Time) by @Minato7namikazi for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34492

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/relayer-distributor/tree/dev>

Impacts:

* Taking down the application/website
* Temporarily disabling user to access target site, such as: Locking up the victim from login, Cookie bombing, etc.

## Description

## In api.ts in relayer-distributor

there is critical missed check in the `validateRequestData` function within the handling of the `txIdList` parameter for the `/originalTx` and `/receipt endpoints.`

## Vulnerability Details

The code correctly parses the txIdList string into an array of transaction IDs. However, it doesn't check the length of the resulting array. This means a malicious actor could submit a very large txIdList, potentially causing a denial-of-service (DoS) attack by overwhelming the server with database queries.

1. The code does parse the txIdList string into an array:

```typescript
let txIdListArr: string[] = [];
try {
  txIdListArr = StringUtils.safeJsonParse(txIdList);
} catch (e) {
  reply.send(
    Crypto.sign({
      success: false,
      error: `Invalid txIdList ${txIdList}`,
    })
  );
  return;
}
```

2. The code then iterates over each txId in the array:

```typescript
for (const txId of txIdListArr) {
  // ... (validation and database query for each txId)
}
```

3. there is indeed no check on the length of txIdListArr before starting the loop.

## Solution

Add a check to limit the number of transaction IDs allowed in the txIdList array. This limit should be consistent with the other limits imposed by the API like ( MAX\_ORIGINAL\_TXS\_PER\_REQUEST and MAX\_RECEIPTS\_PER\_REQUEST )

## Impact Details

Denial of Service (DoS):

* An attacker could send a request with an extremely large number of transaction IDs.
* The server would attempt to process each ID, leading to a high number of database queries.
* This could overwhelm the database, causing slowdowns or crashes.
* The affected endpoint could become unresponsive, denying service to legitimate users.

Resource Exhaustion:

* Processing a large number of IDs could consume significant CPU and memory resources.
* This could impact the performance of the entire server, affecting other services and users.

## Proof of concept

## Minimal PoC

```
import crypto from 'crypto';

// Simulated database query function
async function simulateDbQuery(txId: string): Promise<any> {
  await new Promise(resolve => setTimeout(resolve, 100)); // Simulate 100ms database query
  return { id: txId, data: 'Some data' };
}

// Vulnerable function similar to the original code
async function processTransactionList(txIdList: string): Promise<any[]> {
  let txIdListArr: string[];
  try {
    txIdListArr = JSON.parse(txIdList);
  } catch (e) {
    throw new Error(`Invalid txIdList ${txIdList}`);
  }

  const results = [];
  for (const txId of txIdListArr) {
    if (typeof txId !== 'string' || txId.length !== 64) {
      throw new Error(`Invalid txId ${txId} in the List`);
    }
    const result = await simulateDbQuery(txId);
    results.push(result);
  }

  return results;
}

// Function to generate a large list of fake transaction IDs
function generateLargeTxIdList(count: number): string[] {
  return Array(count).fill(0).map(() => crypto.randomBytes(32).toString('hex'));
}

// PoC demonstration
async function runPoC() {
  console.time('Processing time');

  const smallList = generateLargeTxIdList(10);
  const largeList = generateLargeTxIdList(1000);

  try {
    console.log('Processing small list (10 items)...');
    await processTransactionList(JSON.stringify(smallList));
    console.log('Small list processed successfully.');

    console.log('\nProcessing large list (1000 items)...');
    await processTransactionList(JSON.stringify(largeList));
    console.log('Large list processed successfully.');
  } catch (error) {
    console.error('Error:', error.message);
  }

  console.timeEnd('Processing time');
}

runPoC();
```

To run this PoC:

1. Save it as a `.ts` file (e.g., `txListVulnerabilityPoC.ts`)
2. (`npm install -g typescript ts-node`)
3. Run it with `ts-node txListVulnerabilityPoC.ts`

This is a minimal PoC to demonstrates the difference from high level :

1. A small list of 10 transaction IDs processes quickly.
2. A large list of 1000 transaction IDs takes significantly longer to process, simulating the potential for a DoS attack.

Expected output will be similar to:

```
Processing small list (10 items)...
Small list processed successfully.

Processing large list (1000 items)...
Large list processed successfully.
Processing time: ~100100ms
```

The large difference in processing time between the small and large lists demonstrates how an attacker could potentially overwhelm the server by sending a request with a very large number of transaction IDs.

This minimal PoC proves from high level that without a limit on the number of transaction IDs that can be processed in a single request, an attacker could cause significant resource consumption and potentially a denial of service.
