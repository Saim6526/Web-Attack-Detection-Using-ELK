# Phase I: Reconnaissance & Enumeration



## Infrastructure Discovery (Recon)
Reconnaissance is gathering information about the target before interaction. For this lab, I focused on verifying the target's availability and open services.

**My Methodology:**
1. Check machine IP: `ip a`
2. Verify connectivity: `ping -c 3 <DVWA-IP>`
3. Service Discovery: `nmap -sV -sC -O <DVWA-IP>`



## Application Discovery (Enumeration)
Enumeration moves beyond scanning; it is the active process of "hooking" into the application logic to find attack vectors.

### The Attack Surface Reference
When enumerating a web app, we look for:
- **Injection Points:** URL parameters (`?id=`), hidden fields, and forms.
- **Tech Stack:** PHP, Apache, and Database versions.
- **Entry Points:** Login behaviors and upload endpoints.

### 🛠 Lab Log: Testing the Fundamentals
*I applied the above principles to my local DVWA instance with the following results:*

* **Version Leakage:** I navigated to `/phpinfo.php`. 
    * *Result:* Confirmed PHP version and server configuration.
* **Cookie Inspection:** Opened DevTools (F12) > Storage > Cookies.
    * *Result:* Found `PHPSESSID` and security level tokens.
* **Login Flow:** Monitored the Network tab during a login attempt.
    * *Result:* Identified `login.php` as the auth endpoint and `index.php` as the post-auth landing.


## Directory & Function Discovery
Moving deeper into the application to find hidden paths.

**Discovery Techniques:**
- **Manual Browsing:** Clicking every link to see app behavior.
- **Directory Fuzzing:** Using `dirsearch` or `gobuster` to find hidden folders like `/admin` or `/config`.

> **My Experience:** I ran `dirsearch -u http://127.0.0.1/DVWA/`. Because my specific config was non-standard, the tool didn't catch everything, so I supplemented this with manual enumeration of the `/vulnerabilities/` sub-directories.




## Threat Modeling: Mapping Modules
The final part of this phase is mapping the discovered modules to a tactical framework.

| Module | Goal / Technique | Log Indicators |
| --- | --- | --- |
| **Brute Force** | Credential attacks | Multiple failed logins |
| **SQL Injection** | Data extraction/Login bypass | `'`, `UNION`, `OR 1=1` |
| **Command Injection** | Remote Code Execution | `;`, `&&`, `ping` in params |
| **XSS** | Cookie theft / Phishing | `<script>`, event handlers |

**Phase I (Recon & Enumeration) Complete.**
