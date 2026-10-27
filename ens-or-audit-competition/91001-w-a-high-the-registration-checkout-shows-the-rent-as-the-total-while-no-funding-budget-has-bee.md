# 91001 w a high the registration checkout shows the rent as the total while no funding budget has been fetched and admits the click on that figure

**Submitted on Aug 28th 2026 at 19:18:56 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91001
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's Hybrid Custody registration funds the smart account with `rent + commit leg cost + register leg cost` under one EIP-2612 permit. The payment screen discloses that with a network cost line, an "up to" qualifier on the total, and an affordability gate sized on the full budget. All three hang off a Rhinestone orchestrator quote. Before that quote first lands the qualifier is gone, the network cost reads `—`, and the gate falls back to the rent. A wallet that can cover the budget is admitted on `Total $8.00 USD`, and the machine then builds a permit for `20196054` units, `20.196054` USDC. A wallet that cannot is admitted on the same figure and refused afterwards, which is the failure the project already recorded on Sepolia.

## Vulnerability Details

The project records that failure and its cause at `packages/smart-account/DEBUGGING_INTENTS.md:288-297`:

```
The first real instance: `permit`/`transferFrom` for `20196054` against a wallet
holding `20000000`. ...

**Why it hits only some users.** The funding permit is signed for the whole HCA
budget — `registrationPrice + commitCost + registerCost` — while the pricing UI
gates on `registrationPrice` alone. On Sepolia the two legs have run to ~12 USDC
against an 8 USDC name ...
```

`signFundingPermitActor` now refuses when the wallet cannot cover the permit, and its error names the gap in the project's own words (`packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:627-637`):

```
`The funding amount covers the registration price plus the ` +
`execution costs of both the commit and register legs, so it is ` +
`larger than the price shown at checkout.`
```

That gate stops a doomed registration late. The disclosure that would stop it at the screen is the other half, and it is incomplete.

Every part of it hangs off one value. `computeRegistrationFunding` returns `null` until the budget quote resolves (`apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts:72`), and the screen binds to that null in four places:

```
TokenPickerContent.tsx:434   const requiredAmount = funding?.walletDebit ?? pricingData
TokenPickerContent.tsx:438   const displayTotal   = funding?.total       ?? pricingData
TokenPickerContent.tsx:491   {(funding || isQuotingFunding) && <NetworkCostRow isLoading={funding?.isLoading ?? true} networkFee={funding?.networkFee} />}
TokenPickerContent.tsx:575   <PaymentTotalRow isEstimate={!!funding} total={displayTotal} />
```

With `funding === null` the headline becomes the rent and loses its qualifier, because `isEstimate` comes from the same null. `PaymentTotalRow` gates the qualifier at `:30` and states the assumption that fails here at `:8-10`:

```
* an estimate, so the figure is an upper bound: hence "up to". Without a budget
* the total is exact rent and the hedge would be a lie, so it is dropped.
```

The total is not exact rent. The registration still funds the whole budget.

The network cost degrades rather than vanishing. While `budgetQuery.isLoading` is true the row renders with `networkFee === undefined` and prints `—` (`:358`, `NetworkCostRow.tsx:44`). After a failure with no budget cached behind it both operands are false and the row is not rendered at all.

The affordability gate falls back with them. `canNext` (`TokenPickerContent.tsx:457-462`, applied at `:584`) never requires the quote to have landed; its deciding condition compares the wallet against `requiredAmount`, which without a budget is the rent.

The state that matters is the one with no budget data for the key yet: a first arrival at the payment screen, a failure with nothing cached behind it, or a switch of the primary-name toggle, which is part of the query key by design (`hcaBudget.query.ts:142-144`). Once a budget has been fetched successfully for that key, React Query keeps serving it through a background refetch, so a remount while the data is still cached shows the real figure and is not affected. Reaching that first landing takes two `prepareTransaction` round trips to the orchestrator, with `retry: 1` (`hcaBudget.query.ts:37-42,158`).

A related defect keeps the query from ever settling into a cached-fresh state: `hcaBudget.query.ts:154` asks for `staleTime: 60_000`, but the shared `resultQueryOptions` helper destructures `staleTime` and `gcTime` out of the options and never returns them (`packages/utils/src/tanstack-query/neverthrow/query.ts:120-147`), leaving the app default of `staleTime: 0` (`apps/manager/src/router.tsx:33-36`). The intended 60 seconds never applies.

Clicking Register calls `fetchQuery` (`TokenPickerContent.tsx:229-233`) and on success re-checks the wallet against the true `walletDebit` (`:238-249`). That fetch populates the cache, so the checkout behind the flow can re-render with the real total, but the user has already acted and no step asks them to confirm the new figure before the wallet prompt. When the re-fetch itself fails it is swallowed:

```
// A quote failure is not a funding failure. Fall through and let the
// machine (and its own pre-permit balance check) surface the problem.
.catch(() => null)
```

which leaves `walletDebitRaw` null and skips the balance check entirely.

`startRegistration` forwards the rent as `totalPrice` (`TokenPickerContent.tsx:212`) and `registrationUi.machine.ts:546-561` relays it as `price` with no `hcaBudget`. The override guard therefore fails (`registration.machine.ts:611`) and the machine quotes the budget again on its own (`:616`). The permit is sized from that independent quote (`:721`):

```
value: bigintMax(
  context.hcaBudget! - (context.hcaUsdcBalance ?? 0n),
  0n,
),
```

That value is the `value` of the EIP-712 permit the wallet is asked to sign (`registration.hca.actors.ts:689-708`) and the amount of `permit()` and `transferFrom(wallet, hca, value)` in the commit batch (`:824-828,841-842`). `DEBUGGING_INTENTS.md:288-290` records that pair being built for exactly `20196054` and failing against live chain state.

Nothing compares the machine's figure against the one the screen rendered, and nothing returns the user to the checkout when it comes back higher.

## Impact Details

A `.eth` name of five or more characters costs `8000021` USDC units for one year on the deployed Sepolia registrar, with zero premium when it has never been registered. The two legs cost `4000000` and `8196033` in the fixture the app tests against (`apps/manager/src/features/register-v2/data/queries/hcaBudget.query.test.ts:20-25`), consistent with the `~12 USDC` of legs against an `8 USDC` name that `DEBUGGING_INTENTS.md:293-295` reports on Sepolia. Rent plus legs is `20196054`, the exact value that document names as the first real instance.

The PoC renders the checkout component for that registration, for two wallets, in three budget states each:

| Wallet         | Budget quote | Screen                                                           | Register    |
| -------------- | ------------ | ---------------------------------------------------------------- | ----------- |
| 50.000000 USDC | in flight    | `Network cost —` … `Total $8.00 USD`                             | **enabled** |
| 50.000000 USDC | failed       | no network cost row … `Total $8.00 USD`                          | **enabled** |
| 50.000000 USDC | landed       | `Network cost $12.20` … `Total up to $20.20 USD`                 | enabled     |
| 20.000000 USDC | in flight    | `Network cost —` … `Total $8.00 USD`                             | **enabled** |
| 20.000000 USDC | failed       | no network cost row … `Total $8.00 USD`                          | **enabled** |
| 20.000000 USDC | landed       | `Network cost $12.20` … `Need $20.20` … `Total up to $20.20 USD` | disabled    |

The 50 USDC wallet is the one that proceeds. It is admitted on a screen reading `Total $8.00 USD`, and the shipped machine computes `20196054` for the permit, which step 4 shows. Both of the app's later gates pass for a wallet at that balance: the click-time check compares `20.196054` against 50 (`TokenPickerContent.tsx:244-249`), and `signFundingPermitActor` compares the same figure against the same balance (`registration.hca.actors.ts:627-637`). So the in-app approval is given on `8.00` and the typed data presented to the wallet carries `20.196054`.

The 20 USDC wallet shows what that figure controls. With the budget, the same screen names the real total, warns `Need $20.20`, and disables the button. Without it, the same screen admits the same wallet on `8.00`; the refusal then comes from the click-time check, or, if that quote also fails, from `signFundingPermitActor` after the flow has started.

The wallet debits the shortfall between the budget and the HCA's standing balance, so on an account that has not been pre-funded it is the whole `20196054`. The EIP-712 request carries the raw `value`, so a user who reads the typed data can see it. What the app never does is ask for a second confirmation: the click is taken on the rent, and no step between it and the wallet prompt requires the user to accept the larger figure.

The payment token is the deployment's MockUSDC on Sepolia, so there is no mainnet loss. The harm is what this program measures: what the app shows a user before they approve.

Selected impact: Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, specifically "showing the wrong recipient or amount on a transaction the user then approves".

`BOUNTY.md:600` records QA-03 against the Explorer and closes it with "Display-level only; a mismatch between the displayed total and the amount actually charged on-chain would be a new finding." This is the Manager and the mismatch is not display-level: `8.000021` is rendered as the total, and the transaction the app then constructs carries `20.196054`. `BOUNTY.md:395` separately keeps in scope "issues we fixed incorrectly or incompletely"; the cost disclosure was added for this gap and covers only the state where the quote has landed.

## References

* `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx:155-163,165-173,212,229-233,238-249,244-249,303-306,320-365,358,370-418,434,438,457-462,491,575,584`
* `apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx:8-10,30`
* `apps/manager/src/features/register-v2/workflow/pricing/components/NetworkCostRow.tsx:44`
* `apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts:52-56,72`
* `apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts:37-42,142-144,154,156-158`
* `apps/manager/src/features/register-v2/data/queries/hcaBudget.query.test.ts:20-25`
* `packages/utils/src/tanstack-query/neverthrow/query.ts:120-147`
* `apps/manager/src/router.tsx:33-36`
* `apps/manager/src/features/register-v2/state/registrationUi.machine.ts:546-561`
* `apps/manager/src/lib/smart-account/types.ts:13-19`
* `packages/transaction-manager/src/machines/registration/registration.machine.ts:611,616,721`
* `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:604,627-637,689-708,824-828,841-842`
* `packages/smart-account/src/providers/rhinestone/budget.ts:194-206,330`
* `packages/smart-account/DEBUGGING_INTENTS.md:288-297`

## Proof of Concept

### Steps to reproduce

{% stepper %}
{% step %}

## Clone the repository

Clone `immunefi-team/audit-comp-ens` and check out `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
{% endstep %}

{% step %}

## Install dependencies

Run `pnpm install --frozen-lockfile` at the repository root. Node 22 or newer.
{% endstep %}

{% step %}

## Install Foundry

Install Foundry so `anvil` and `cast` are on `PATH`.
{% endstep %}

{% step %}

## Save the PoC script

Save the script below as `poc.sh` in the repository root.
{% endstep %}

{% step %}

## Configure a Sepolia RPC endpoint

Export `POC_SEPOLIA_RPC` pointing at a Sepolia endpoint you control. The script refuses to run without it and does not fall back to the provider URL committed in the repository, because `BOUNTY.md:445` asks researchers not to exercise a recovered credential against a third party.
{% endstep %}

{% step %}

## Run the PoC

Run `bash poc.sh` from the repository root. It takes about three minutes, most of it in step 3 while Vite builds the checkout's module graph.
{% endstep %}

{% step %}

## Review checkout rendering

Read step 3. The checkout component is rendered for two wallets in three budget states. In every state with no successful budget for the key the total reads `Total $8.00 USD` and the Register button carries no `disabled` attribute, for both wallets. With the budget landed the same screen reads `Total up to $20.20 USD`, and for the 20 USDC wallet it adds `Need $20.20` and disables the button.
{% endstep %}

{% step %}

## Review the permit amount

Read step 4. The shipped registration machine, given the same name and duration, computes `20196054` for the funding permit.
{% endstep %}

{% step %}

## Compare the values

Compare with step 5. The script exits non-zero if the permit value is missing or does not equal the budget total.
{% endstep %}
{% endstepper %}

### What the PoC runs

Step 2 reads the rent from the deployed Sepolia registrar over a local anvil fork pinned at block 11584000, with no app code involved. Step 3 renders the real `TokenPickerContentBase` through the app's own Vite and Lingui configuration, supplying by hand the props `TokenPickerContent` passes it at `TokenPickerContent.tsx:320-365`, with `funding` produced by the real `computeRegistrationFunding` and the rent taken from step 2. It renders that component, not the wrapper, so the query lifecycle and the click handler are cited to source rather than exercised. Step 4 drives the real `registrationMachine` and records the value it computes for `signFundingPermit`.

Two props of that call differ and neither touches the three outputs read here: `footer`, the primary-name toggle, is omitted, and the wrapper's own shortfall `errorMessage` is forced to `null` rather than computed. For the landed underfunded row the wrapper would add its own message (`TokenPickerContent.tsx:303-306`), which would only make that row read worse.

What the PoC supplies rather than fetches, all marked in the script: the two per-leg orchestrator costs, passed through `QuoteLegCostUsdc`, the injection point `budget.ts:194-206` defines for the caller, because the orchestrator is a paid external service; an HCA balance of `0n`, the assumption that the account has not been pre-funded; two wallet balances; and the placeholder addresses and `price: 0n` the machine needs to reach the state under test but never uses to compute the permit. The rent is read from the chain and feeds both step 3 and step 4.

The PoC renders and computes. It does not sign, submit, or execute the batch: no wallet is involved, and no `permit` or `transferFrom` runs. That the permit value becomes the argument of both is cited to source. The wallet balances are React props, not on-chain balances; what step 3 shows is the screen and the button, and what step 4 shows is the figure the machine computes. The PoC renders the checkout and reads the real button state, but it does not click, so the `onNext` handler and its re-fetch are cited rather than executed. Nothing is broadcast to any network: anvil serves a local fork, which it populates by reading through the endpoint you supply.

The script writes two helper files inside the workspace and removes both, along with the anvil log, on exit. The checkout is left clean at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

```bash
#!/usr/bin/env bash
# ENS Manager: when the HCA funding quote has not landed, the payment screen
# drops the "up to" qualifier and the network cost, gates on the rent alone, and
# the registration machine still sizes the funding permit at the full budget.
#
# Run from the repository root of immunefi-team/audit-comp-ens at
# 1c9b47f18fcddd2e864dfe385c4171061c9811ae:
#
#   bash poc.sh
#
# Requires: node >= 22, `pnpm install --frozen-lockfile` already run, and
# Foundry (anvil + cast) on PATH.
#
# Nothing is broadcast to a public network. anvil serves a local fork; every
# step is a local read or an in-memory run of the shipped modules.
set -euo pipefail

PORT=8557
BLOCK=11584000
LABEL=poctest2026            # 11 characters, so the 5-or-more price tier
DURATION=31536000            # 1 year, seconds
RPC=http://127.0.0.1:$PORT
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc  # ENS_SEPOLIA_CONTRACTS.ETHRegistrar
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39       # the deployment's MockUSDC, SUPPORTED_TOKENS.USDC
# Supply your own Sepolia endpoint. The PoC deliberately does NOT default to the
# provider URL committed in apps/manager/src/lib/wagmi.ts:19: BOUNTY.md:445 asks
# researchers not to exercise a recovered credential against a third party.
SEPOLIA=${POC_SEPOLIA_RPC:?set POC_SEPOLIA_RPC to a Sepolia RPC endpoint you control}

command -v anvil >/dev/null || { echo "anvil not on PATH"; exit 1; }
command -v cast  >/dev/null || { echo "cast not on PATH";  exit 1; }

echo "=== [1] local Sepolia fork, pinned ==="
anvil --fork-url "$SEPOLIA" --fork-block-number $BLOCK --port $PORT >/tmp/poc-anvil.log 2>&1 &
ANVIL_PID=$!
trap 'kill $ANVIL_PID 2>/dev/null || true; rm -f packages/transaction-manager/_poc_permit.ts apps/manager/_poc_display.mjs /tmp/poc-anvil.log' EXIT
for _ in $(seq 1 60); do cast block-number --rpc-url $RPC >/dev/null 2>&1 && break; sleep 1; done
printf '  fork block                 %s\n' "$(cast block-number --rpc-url $RPC)"

echo
echo "=== [2] the rent, read from the chain ==="
printf '  isAvailable(%s)   %s\n' "$LABEL" "$(cast call $REGISTRAR 'isAvailable(string)(bool)' "$LABEL" --rpc-url $RPC)"
PRICE_RAW=$(cast call $REGISTRAR 'getRegisterPrice(string,uint64,address)(uint256,uint256)' "$LABEL" $DURATION $USDC --rpc-url $RPC)
RENT_BASE=$(echo "$PRICE_RAW" | sed -n '1p' | awk '{print $1}')
RENT_PREMIUM=$(echo "$PRICE_RAW" | sed -n '2p' | awk '{print $1}')
printf '  getRegisterPrice base      %s USDC units\n' "$RENT_BASE"
printf '  getRegisterPrice premium   %s USDC units\n' "$RENT_PREMIUM"

echo
echo "=== [3] the real checkout, rendered in each budget state (takes ~90s) ==="
cat > apps/manager/_poc_display.mjs <<'JS'
import { createServer } from 'vite'
import { createElement } from 'react'
import { renderToStaticMarkup } from 'react-dom/server'
import { i18n } from '@lingui/core'
import { I18nProvider } from '@lingui/react'

i18n.loadAndActivate({ locale: 'en', messages: {} })

const server = await createServer({
  configFile: 'vitest.config.ts', root: process.cwd(),
  server: { middlewareMode: true }, appType: 'custom', logLevel: 'error',
})
const { computeRegistrationFunding } = await server.ssrLoadModule(
  '/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts')
const { TokenPickerContentBase } = await server.ssrLoadModule(
  '/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx')

const RENT_RAW = BigInt(process.env.POC_RENT_RAW)          // read from the chain in step 2
const LEGS = 4_000_000n + 8_196_033n                       // the two orchestrator legs
const BUDGET = { total: RENT_RAW + LEGS, registrationPrice: RENT_RAW }

// Exactly the props `TokenPickerContent` passes down at TokenPickerContent.tsx:320-365,
// except `footer` (the primary-name toggle, which touches none of these three
// outputs) and the wrapper's own shortfall `errorMessage`, whose presence would
// only make the landed row read worse.
const propsFor = (walletRaw, budget, isLoading, isFetching) => {
  const f = computeRegistrationFunding({
    budget, walletBalanceRaw: walletRaw, hcaBalanceRaw: 0n, decimals: 6 })
  return {
    label: process.env.POC_LABEL,
    pricingLoading: false,
    pricingData: Number(RENT_RAW) / 1e6,
    isInPriceCooldown: false,                              // premium is 0, see step 2
    errorMessage: null,
    selectedToken: 'USDC',
    onSelectCoin: () => {}, onNext: () => {},
    stablecoinBalances: [{
      symbol: 'USDC', address: process.env.POC_USDC, decimals: 6,
      balance: String(walletRaw),
      formattedBalance: (Number(walletRaw) / 1e6).toFixed(2),
    }],
    isLoadingBalances: false, isConnected: true,
    funding: f ? { networkFee: f.networkFee, total: f.total, walletDebit: f.walletDebit, isLoading: isFetching } : undefined,
    isQuotingFunding: isLoading,
  }
}

const show = (name, props) => {
  const html = renderToStaticMarkup(
    createElement(I18nProvider, { i18n }, createElement(TokenPickerContentBase, props)))
  const btn = html.slice(html.lastIndexOf('<button'))
  const disabled = /\sdisabled(?:=|\s|>)/.test(btn.slice(0, btn.indexOf('>')))
  const text = html.replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim()
  console.log(`    ${name}`)
  console.log(`      screen          ${JSON.stringify(text)}`)
  console.log(`      Register button ${disabled ? 'disabled' : 'ENABLED'}`)
}

for (const [who, walletRaw] of [
  ['wallet holding 50.000000 USDC, which covers the budget', 50_000_000n],
  ['wallet holding 20.000000 USDC, which covers only the rent', 20_000_000n],
]) {
  console.log(`  ${who}`)
  show('budget quote in flight', propsFor(walletRaw, undefined, true, true))
  show('budget quote failed   ', propsFor(walletRaw, undefined, false, false))
  show('budget quote landed   ', propsFor(walletRaw, BUDGET, false, false))
}
await server.close()
process.exit(0)   // vite holds handles open after close; nothing further to run
JS
( cd apps/manager && POC_RENT_RAW=$RENT_BASE POC_LABEL=$LABEL POC_USDC=$USDC node _poc_display.mjs )

echo
echo "=== [4] what the registration machine sizes the funding permit at ==="
cat > packages/transaction-manager/_poc_permit.ts <<'TS'
import { estimateHcaBudget } from '@ens-apps/smart-account'
import { fromResultAsync } from '@ens-apps/utils/xstate/neverthrow'
import { errAsync, okAsync, ResultAsync } from 'neverthrow'
import { createPublicClient, http, type PublicClient } from 'viem'
import { sepolia } from 'viem/chains'
import { createActor } from 'xstate'
import { registrationMachine } from './src/machines/registration/registration.machine'

const publicClient = createPublicClient({
  chain: sepolia, transport: http(process.env.POC_RPC as string),
}) as unknown as PublicClient

// The only values not read here: the two per-leg costs the Rhinestone
// orchestrator returns, supplied through `QuoteLegCostUsdc`, the injection
// point `budget.ts:194-206` defines for the caller. The orchestrator is a paid
// external service. These are the values `hcaBudget.query.test.ts:22-23`
// records, and which match the ~12 USDC of legs against an 8 USDC name
// that `DEBUGGING_INTENTS.md:292-294` reports on Sepolia.
const COMMIT_LEG = 4_000_000n
const REGISTER_LEG = 8_196_033n

let permitValue: bigint | undefined
let breakdown: Record<string, unknown> | undefined

const machine = registrationMachine.provide({
  actors: {
    estimateHcaBudget: fromResultAsync((input: { name: string; duration: bigint }) =>
      ResultAsync.fromPromise(
        estimateHcaBudget({
          publicClient, chainId: sepolia.id,
          label: input.name.replace(/\.eth$/, ''), duration: input.duration,
          hcaBalanceUsdc: 0n,
          quoteLegCostUsdc: async (leg) => ({
            spendUsdc: leg === 'commit' ? COMMIT_LEG : REGISTER_LEG,
          }),
        }),
        (e) => (e instanceof Error ? e : new Error(String(e))),
      ).map((b) => { breakdown = { ...b }; return b }),
    ),
    readHcaUsdcBalance: fromResultAsync(() => okAsync(0n)),   // assume the HCA is not pre-funded
    // Stops the run at the point of interest and records the value the machine
    // computed. Nothing is signed and nothing is submitted.
    signFundingPermit: fromResultAsync((input: { value: bigint }) => {
      permitValue = input.value
      return errAsync(new Error('POC: recorded the permit value; stopping here'))
    }),
  },
})

const actor = createActor(machine, { input: { chainId: sepolia.id } })
actor.start()
actor.send({
  type: 'START_REGISTRATION',
  name: `${process.env.POC_LABEL}.eth`,
  duration: BigInt(process.env.POC_DURATION as string),
  token: 'USDC',
  price: 0n,
  signer: { type: 'rhinestone', account: { getAddress: () => '0x1111111111111111111111111111111111111111' } },
  approvalSigner: { type: 'eoa' },
  accountAddress: '0x1111111111111111111111111111111111111111',
  publicClient,
} as never)

for (let i = 0; i < 120 && permitValue === undefined; i++) {
  await new Promise((r) => setTimeout(r, 250))
}

const u = (v: unknown) => (typeof v === 'bigint' ? `${v}` : String(v))
console.log('POC  budget.registrationPrice   ', u(breakdown?.registrationPrice), '  <- read from the fork')
console.log('POC  budget.commitCost          ', u(breakdown?.commitCost))
console.log('POC  budget.registerCost        ', u(breakdown?.registerCost))
console.log('POC  budget.total               ', u(breakdown?.total))
console.log('POC  permit value the machine   ', u(permitValue))
console.log('POC  computed for signing          registration.machine.ts:721, hcaBudget - hcaUsdcBalance')
console.log('POC                                the same value is the argument of permit() and')
console.log('POC                                transferFrom() at registration.hca.actors.ts:824-828,841-842')

if (permitValue === undefined) { console.error('POC FAILED: no permit value was computed'); process.exit(1) }
if (permitValue !== breakdown?.total) { console.error('POC FAILED: permit value != budget total'); process.exit(1) }
process.exit(0)
TS
POC_RPC=$RPC POC_LABEL=$LABEL POC_DURATION=$DURATION \
  ./e2e/node_modules/.bin/tsx packages/transaction-manager/_poc_permit.ts 2>/dev/null | sed -n 's/^POC //p'

echo
echo "=== [5] the gap ==="
printf '  the screen headline while the quote is outstanding     %s USDC units\n' "$RENT_BASE"
echo "  the permit the machine then sizes                      20196054 USDC units"
echo "  never shown under that headline                        12196033 USDC units (the two legs)"
```

Transcript, with `POC_SEPOLIA_RPC` set:

```
=== [1] local Sepolia fork, pinned ===
  fork block                 11584000

=== [2] the rent, read from the chain ===
  isAvailable(poctest2026)   true
  getRegisterPrice base      8000021 USDC units
  getRegisterPrice premium   0 USDC units

=== [3] the real checkout, rendered in each budget state (takes ~90s) ===
  wallet holding 50.000000 USDC, which covers the budget
    budget quote in flight
      screen          "poctest2026.eth Network cost — info Select payment S USDC $50.00 available Stables accepted Total $8.00 USD Register name"
      Register button ENABLED
    budget quote failed   
      screen          "poctest2026.eth Select payment S USDC $50.00 available Stables accepted Total $8.00 USD Register name"
      Register button ENABLED
    budget quote landed   
      screen          "poctest2026.eth Network cost $12.20 info Select payment S USDC $50.00 available Stables accepted Total up to $20.20 USD Register name"
      Register button ENABLED
  wallet holding 20.000000 USDC, which covers only the rent
    budget quote in flight
      screen          "poctest2026.eth Network cost — info Select payment S USDC $20.00 available Stables accepted Total $8.00 USD Register name"
      Register button ENABLED
    budget quote failed   
      screen          "poctest2026.eth Select payment S USDC $20.00 available Stables accepted Total $8.00 USD Register name"
      Register button ENABLED
    budget quote landed   
      screen          "poctest2026.eth Network cost $12.20 info Select payment S USDC $20.00 available Need $20.20 Stables accepted Total up to $20.20 USD Register name"
      Register button disabled

=== [4] what the registration machine sizes the funding permit at ===
 budget.registrationPrice    8000021   <- read from the fork
 budget.commitCost           4000000
 budget.registerCost         8196033
 budget.total                20196054
 permit value the machine    20196054
 computed for signing          registration.machine.ts:721, hcaBudget - hcaUsdcBalance
                               the same value is the argument of permit() and
                               transferFrom() at registration.hca.actors.ts:824-828,841-842

=== [5] the gap ===
  the screen headline while the quote is outstanding     8000021 USDC units
  the permit the machine then sizes                      20196054 USDC units
  never shown under that headline                        12196033 USDC units (the two legs)
```
