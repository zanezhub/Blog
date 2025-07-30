---
title: "Credential Harvaster with anti-debugging"
date: 2025-07-30
---

This is from a phishing email, where the attacked sent a malicious URL to the service desk.
![[Pasted image 20250728084250.png]]

I checked the link in VirusTotal and it appears to be malicious : 

```
hXXps://www[.]google[.]com/url?q=https%3A%2F%2Fairblissco[.]com%2F.well-known%2Facme-challenge%2F007&sa=D&sntz=1&usg=AOvVaw1Z3C1siL9ee8nYVkm6Y3xG#?8077507749Family=c3RlZWxjYXNlQHNlcnZpY2Utbm93LmNvbQ==
```

The URL prefix `hXXps://www.google[.]com/url?q=` is used by Google to track clicks on links within Google Search results and other Google services.

As you can see there is a website that has a few parameters that might be malicious, it seems like the parameters are used to track different identifiers for the user that clicks the URL.
# Redirection
This redirects to this website, "hXXps://airblissco[.]com/.well-known/acme-challenge/007". The website only hosts a blank page and in the source code of the page you can see:
![[Pasted image 20250728084603.png]]

This code basically gets the URL from the past URL and gets the last value of the URL `c3RlZWxjYXNlQHNlcnZpY2Utbm93LmNvbQ==`, this is a base64 encoded string that contains the email of the victim that received the phishing email:
```
"c3RlZWxjYXNlQHNlcnZpY2Utbm93LmNvbQ" : "steelcase@service-now[.]com"
```

Now that we have this we need to access to the link: 
```
hXXps://jcxg[.]htuptunkecfn[.]es/gFm9iJn@aNqfcfa/steelcase@service-now.com
```

![[Pasted image 20250728085918.png]]![[Pasted image 20250728085923.png]]
![[Pasted image 20250728085930.png]]
# Anti Debugging Techniques
If you try to inspect the website it will trigger an anti debugging technique to stop analysts
![[Pasted image 20250728091738.png]]
This detects if the user is using inspect mode and then when the debugger jumps to the next instruction it will redirect you to one of the following websites:
- Walmart
- Etsy
- Flipkart
The code randomly chooses a website and uses the window.location.place with a hardcoded string to redirect you.

But if you see the code from the email this is the following:
```
hXXps://jcxg[.]htuptunkecfn[.]es/zxykirk2mahz7?id=12cfd321fe0c3cda67d141b10-cba59a289b4f842-7a774b31-f7b3eb7b-dd25cfa208f46-e2fb6d65-617f5869f974-98f58434e8787-a055ef9cf9d46a118034
```

![[Pasted image 20250728093315.png]]
![[Pasted image 20250728093323.png]]
# Analysis
## First snippet - (Malware.html)
The first script is obfuscated, after some time analyzing this I was able to know what the code was doing, this same code structure was reused multiple times by the attackers:
![[Pasted image 20250728093704.png]]
This starts creating a few variables that will be used for encryption using the `CryptoJS` lib, and it uses some variables to call functions to make it harder to read.
![[Pasted image 20250728094028.png]]
This code snippet is used to decrypt the payload for the next JavaScript code, then it tries to execute it using `eval`.

After decrypting the code, I realized that this is used as an anti debugging technique, as you can see it checks for BurpSuite and other forms of debugging,
- Blocks automated browsers/tools (Selenium, PhantomJS, Burp)
- Prevents users from easily opening devtools or viewing source via shortcuts or right-click
- Detects debugger usage and redirects away if detected
![[Pasted image 20250728095557.png]]
It then redirects you to a random website
![[Pasted image 20250728100107.png]]
## Second Snippet - (Malware.html)
First it starts declaring a lot of variables that all form part of the same payload
![[Pasted image 20250728105543.png]]
![[Pasted image 20250728105706.png]]
It add all the variables to prepare for the decryption, it uses the same snipped code we have seen previously. After decrypting this I got another JavaScript `Execute2.js` code that will be used in the browser.
## Execute2.js
The first snippet of the code is the same antidebug code we discussed previously. The only thing that is different is this `Base64` encoded payload. This code will be decoded and executed using 

```javascript
k = atob(payload);
document.write(k);
```

![[Pasted image 20250728110317.png]]
We will call the output of this `Execute3.js`
## Execute3.html
This is an HTML page, the first JavaScript is the same antidebugging code that we have seen.
The website is another credential harvester.

Now, the interesting part is another obfuscated JS code.
![[Pasted image 20250728114345.png]]
I did not bother to deobfuscate the code this time, all the scripts that were used are the same, so I was familiar enough to just decrypt this using a break point.

The output of this created another JavaScript code (`execute4.js`).
## Execute4.js
This gets information about the user and then it send it to two ULRs:
![[Pasted image 20250728145843.png]]

```text
/qxd9WI2swr3KhCMK2bXk7NO7IKX5nl86Sx7dTSErq3S0c1Qe36iTgb
```

This is likely to be an endpoint inside the same website.
![[Pasted image 20250728150930.png]]

```text
"hXXps://P6KrDf6gStSuT699hiwP8aVukpXibESwG4rsWrGDKikGEuufjK11zhc9XIY[.]hogardeguro[.]es/XiWHxInWxYzLJABzSLgHlSNYHYBPSHEMDZNXQKMOFVPKEZXQMWGKWSOLQFEX"
```

This should be where they're sending the MFA tokens and other information from their credential harvesters. 
