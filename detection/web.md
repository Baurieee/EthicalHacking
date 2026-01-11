# DVWA Web Exploitation 

## What this is 


* **Stored/Reflected XSS**: attacker-controlled JavaScript runs in the victim’s browser under the DVWA origin -> can read/act as the user (session theft, forced actions). 
* **Session hijacking**: using a stolen `PHPSESSID` cookie to impersonate the victim. 
* **CSRF**: victim is logged in, browser auto-sends cookies, attacker forces a state change (password change). 
* **Command injection**: user input reaches `system()`/shell -> separators like `;` allow extra commands (RCE). 
* **File upload abuse**: upload HTML for XSS or upload a server-side shell (e.g. PHP) if validation is weak. 

---

## What to monitor on the server

### Web logs 

Apache/Nginx access logs show the payloads and endpoints:

* XSS indicators: `<script>`, `document.cookie`, `onerror=`, `%3Cscript%3E`
* CSRF indicators: requests to `/dvwa/vulnerabilities/csrf/` with `password_new=...`
* Command injection indicators: `;`, `&&`, `|`, backticks, `$(`, URL-encoded versions (`%3B`, `%26%26`)
* Upload indicators: POST to upload endpoint + then GET to `/hackable/uploads/...`

## what helps

* `HttpOnly` cookies reduce `document.cookie` theft impact (XSS still bad, but cookie theft is harder). 
* CSRF tokens + POST-only for sensitive actions blocks the DVWA-style CSRF. 
* Disable shell execution from web apps / sanitize input; command injection is usually full compromise. 
* Strict upload validation + store uploads outside webroot prevents easy webshell execution. 
