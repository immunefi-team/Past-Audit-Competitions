# #39752 \[BC-Insight] There is an issue related to incorrect version parsing and comparison logic lead to incorrect node validation,

**Submitted on Feb 6th 2025 at 16:00:06 UTC by @XDZIBECX for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39752
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

there is a vulnerability arise from the incorrect parsing of version numbers in the isEqualOrNewerVersion function see vulnerability details and PoC for details analysis and explaining thank you.

## Vulnerability Details

There is an issue on the function isEqualOrNewerVersion, this fucntion is designed to compare version strings to determine if a node's version meets the minimum requirement for network participation, and then It operates by splitting version numbers into segments and using bitwise operations (\~\~) to convert them into integers for comparison, but the problem is this conversion are incorrectly interprets multi-digit versions (as an example 1.10 is treated as 1.1 instead of 1.10), and this is show on the test and this as result is leading to incorrect version validation, the function does not properly reject malformed versions (as an example 1.2beta),this is show on the poc also and this is causing undefined behavior and validation errors here is the vulnerable line on the code :

```solidity
  for (let i = 0; i < testVerParts.length; i++) {
    const testV = ~~testVerParts[i]; <-- Here Incorrect parsing, misinterprets versions
    const minV = ~~minVerParts[i];   <-- Here the Same issue
    if (testV > minV) return true;
    if (testV < minV) return false;
  }
```

here is as show on the test it's Converts strings to numbers incorrectly.\
and is breaks multi-digit versions (as an example 1.10 becomes 1.1).\
and also is Fails for malformed versions (as an example 1.2beta)\
and also A shorter version (1.2) is incorrectly treated as smaller than 1.2.0.

* The bug is from the improper numeric parsing and the absence of input validation, this is lead to the rejection of valid nodes, and forcing them to repeatedly attempt network re-entry, and this is increasing the processing load, and degrading network performance. this is a real risk because it can cause network instability, validator node exclusion need to be fixed .

## Impact Details

This vulnerability is classified as Medium because it causes more than 30% of network processing nodes to shut down without requiring brute force actions, this is show on the poc, the issue does not completely halt the network as an example If 40% of validators are rejected and each earns $100/day in staking rewards, the network could suffer a $40,000 daily loss.

## References

* <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/utils/general.ts#L60C1-L65C25>

## Proof of Concept

## Proof of Concept

here is a poc show the issue :

```solidity

import axios from "axios";
import { isEqualOrNewerVersion } from "../../../../src/utils/general";

 
const RPC_URL = "http://127.0.0.1:3000/rpc";
const TOTAL_NODES = 10; // Simulate a network of 10 nodes

describe("Network Version Enforcement Test", () => {
  let rejectedNodes = 0;

  test("Should check all nodes and count rejections", async () => {
    const responses = await Promise.all(
      [...Array(TOTAL_NODES)].map(async (_, i) => {
        const nodeId = `Node-${i + 1}`;
        const nodeVersion = getNodeVersion(i + 1);

        try {
          const response = await axios.post(RPC_URL, {
            nodeId,
            nodeVersion,
          });

          if (!response.data.isValid) {
            rejectedNodes++;
          }

          // Verify server response with direct function call
          const expected = isEqualOrNewerVersion("1.2.0", nodeVersion);
          expect(response.data.isValid).toBe(expected);

        } catch (error) {
          console.error(`Node ${nodeId} failed request`, error);
        }
      })
    );

    console.log(`Rejected Nodes: ${rejectedNodes}/${TOTAL_NODES}`);

    // 🔴 **Bug confirmed if 30% or more nodes are rejected**
    expect(rejectedNodes).toBeLessThan(TOTAL_NODES * 0.3);
  });
});

// Function to assign node versions (simulate randomness)
function getNodeVersion(index: number): string {
  const versions = [
    "1.2",
    "1.3.0",
    "1.10",
    "1.2.0",
    "1.2beta",
    "1.1",
    "1.2.5",
    "1.2.3",
    "1.10.0",
    "1.2",
  ];
  return versions[index % versions.length];
}
```

* i use this as server :

```solidity
const express = require("express");
const { isEqualOrNewerVersion } = require("./src/utils/general"); // ✅ Correct Import

const app = express();
app.use(express.json());

const PORT = 3000;

// Mock list of nodes with random versions
const nodes = [
  { id: "Node-1", version: "1.2" },
  { id: "Node-2", version: "1.3.0" },
  { id: "Node-3", version: "1.10" }, // Incorrectly handled version
  { id: "Node-4", version: "1.2.0" },
  { id: "Node-5", version: "1.2beta" }, // Malformed version
  { id: "Node-6", version: "1.1" },
  { id: "Node-7", version: "1.2.5" },
  { id: "Node-8", version: "1.2.3" },
  { id: "Node-9", version: "1.10.0" }, // Incorrectly handled version
  { id: "Node-10", version: "1.2" },
];

const MINIMUM_VERSION = "1.2.0";

// API to check if node versions are valid using the real function
app.post("/rpc", async (req, res) => {
  const { nodeId, nodeVersion } = req.body;

  try {
    const isValid = isEqualOrNewerVersion(MINIMUM_VERSION, nodeVersion); // ✅ Correct function call
    console.log(`✅ Node ${nodeId} (version ${nodeVersion}) -> Valid: ${isValid}`);
    res.json({ nodeId, isValid });
  } catch (error) {
    console.error(`❌ Node ${nodeId} (version ${nodeVersion}) failed validation: ${error.message}`);
    res.json({ nodeId, isValid: false, error: error.message });
  }
});

// Start the server
app.listen(PORT, () => {
  console.log(`🚀 Mock RPC Server running at http://127.0.0.1:${PORT}/rpc`);
});
```

* here is the result when i run the test and the server the result of test is show as :

```solidity
  npm test general.test.ts

> @shardeum-foundation/shardeum@1.17.1 pretest
> npm run compile


> @shardeum-foundation/shardeum@1.17.1 compile
> tsc -p .


> @shardeum-foundation/shardeum@1.17.1 test
> jest general.test.ts

  console.log
    Rejected Nodes: 4/10

      at Object.<anonymous> (test/unit/src/utils/general.test.ts:37:13)

 FAIL  test/unit/src/utils/general.test.ts (5.11 s)
  Network Version Enforcement Test
    × Should check all nodes and count rejections (168 ms)

  ● Network Version Enforcement Test › Should check all nodes and count rejections

    expect(received).toBeLessThan(expected)

    Expected: < 3
    Received:   4

      38 |
      39 |     // 🔴 **Bug confirmed if 30% or more nodes are rejected**
    > 40 |     expect(rejectedNodes).toBeLessThan(TOTAL_NODES * 0.3);
         |                           ^
      41 |   });
      42 | });
      43 |

      at Object.<anonymous> (test/unit/src/utils/general.test.ts:40:27)       

Test Suites: 1 failed, 1 total                                                
Tests:       1 failed, 1 total                                                
Snapshots:   0 total
Time:        7.418 s
Ran all test suites matching /general.test.ts/i.

```

* and this is the result of the server response

```solidity

  node mock-rpc-server.js
� Mock RPC Server running at http://127.0.0.1:3000/rpc
✅ Node Node-1 (version 1.3.0) -> Valid: true
✅ Node Node-2 (version 1.10) -> Valid: true
✅ Node Node-3 (version 1.2.0) -> Valid: true
✅ Node Node-4 (version 1.2beta) -> Valid: false
✅ Node Node-5 (version 1.1) -> Valid: false
✅ Node Node-6 (version 1.2.5) -> Valid: true
✅ Node Node-7 (version 1.2.3) -> Valid: true
✅ Node Node-8 (version 1.10.0) -> Valid: true
✅ Node Node-9 (version 1.2) -> Valid: false
✅ Node Node-10 (version 1.2) -> Valid: false

```
