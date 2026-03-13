# DVWA Lab Logs Index

| Section | Phase | Quick Link |
| :--- | :--- | :--- |
| **01** | **Network Recon** | [Infrastructure Discovery (Recon)](#infrastructure-discovery-recon) |
| **02** | **App Enumeration** | [Application Discovery (Enumeration)](#application-discovery-enumeration) |
| **03** | **Path Discovery** | [Directory & Function Discovery](#directory--function-discovery) |
| **04** | **Threat Modeling** | [Threat Modeling: Mapping Modules](#threat-modeling-mapping-modules) |
| **05** | **SQL Injection** | [SQL Injection (SQLi): Database Compromise](#sql-injection-sqli-database-compromise) |
| **06** | **OS Command Execution** | [Command Injection: System Compromise](#command-injection-system-compromise) |
| **07** | **Session Hijacking** | [Stored XSS: User & Session Compromise](#stored-xss-user--session-compromise) |
| **08** | **Strategic Mapping** | [Strategic Framework Mapping](#strategic-framework-mapping) |
| **09** | **Lessons Learned** | [Defender Lessons Learned](#defender-lessons-learned) |

---

# Phase I: Reconnaissance & Enumeration

## Infrastructure Discovery (Recon)
Reconnaissance is gathering information about the target before interaction. For this lab, I focused on verifying the target's availability and open services.

**My Methodology:**
1. Check machine IP: `ip a`
2. Verify connectivity: `ping -c 3 <DVWA-IP>`
3. Service Discovery: `nmap -sV -sC -O <DVWA-IP>`

<img width="1745" height="1077" alt="Nmap_Scan" src="https://github.com/user-attachments/assets/fd38fe41-c973-410c-a76f-cdbc97c95449" />


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

<img width="1745" height="1077" alt="Dirsearch" src="https://github.com/user-attachments/assets/c5978a77-922b-4b8b-b91d-ebbde3670cc2" />

<img width="1745" height="1077" alt="phpinfo" src="https://github.com/user-attachments/assets/9534e61c-901c-41f6-8fa8-fbd61ffff80d" />


## Threat Modeling: Mapping Modules
The final part of this phase is mapping the discovered modules to a tactical framework.

| Module | Goal / Technique | Log Indicators |
| --- | --- | --- |
| **Brute Force** | Credential attacks | Multiple failed logins |
| **SQL Injection** | Data extraction/Login bypass | `'`, `UNION`, `OR 1=1` |
| **Command Injection** | Remote Code Execution | `;`, `&&`, `ping` in params |
| **XSS** | Cookie theft / Phishing | `<script>`, event handlers |

**Phase I (Recon & Enumeration) Complete.**

---

# 🛠 Lab Log: DVWA Penetration Testing (Phase II - Exploitation)

This log documents the transition from reconnaissance to active exploitation across the three primary domains of web security: the Database, the Operating System, and the User Session.

## SQL Injection (SQLi): Database Compromise
**Objective:** Determine if user-supplied input is reaching the database unsanitized and extract sensitive credentials.

### Vulnerability Confirmation (Low Security)
* **Initial Test:** Submitted a single quote (`'`) in the User ID field.
* **Observation:** The application returned a database syntax error, confirming that input is concatenated directly into the SQL query without parameterization.
* **Authentication Bypass:** Injected the payload `1 or 1=1`.
* **Result:** The backend query resolved as `SELECT * FROM users WHERE id = 1 OR 1=1;`. Because `1=1` is a tautology, the database returned every record in the table.

<img width="1745" height="1077" alt="1 or 1=1" src="https://github.com/user-attachments/assets/59ae27b7-e22f-49cc-875e-eecda095005e" />



### Client-Side Bypass (Medium Security)
* **The Obstacle:** The text input was replaced by a dropdown menu, restricting direct input.
* **The Bypass:** Used Browser DevTools (**F12**) to inspect the element and manually modify the `value` attribute of a dropdown option to my injection payload.
* **Extraction Payload:** `1 UNION SELECT user, password FROM users`
* **Result (Credential Leak):** Successfully dumped MD5 hashes for the entire user database.

<img width="1745" height="1077" alt="Union Select" src="https://github.com/user-attachments/assets/3369b1de-6230-4cdc-a1ec-1e462b131b84" />


**Exfiltrated Credentials:**
| Username | MD5 Hash | Decrypted Password |
| :--- | :--- | :--- |
| **admin** | `5f4dcc3b5aa765d6...` | password |
| **gordonb** | `e99a18c428cb38d5...` | abc123 |
| **1337** | `8d3533d75ae2c396...` | elite |

---

## Command Injection: System Compromise
**Objective:** Escalate from data access to Remote Code Execution (RCE) by abusing OS-level calls.

### Exploitation & Filter Evasion
* **Logic Analysis:** The "Ping" feature was hypothesized to run `shell_exec("ping -c 3 " . $target);`.
* **The Obstacle (Medium Security):** Common separators like `;` and `&&` were blacklisted and produced no results.
* **The Bypass (Filter Evasion):** Identified that the pipeline operator (`|`) was not filtered.
* **Payload:** `8.8.8.8 | whoami`
* **Result:** The server returned `www-data`. 
* **The Why:** Identifying the user as `www-data` confirms the service account context and marks the starting point for **Privilege Escalation**.



## Stored XSS: User & Session Compromise
**Objective:** Inject a persistent payload to execute JavaScript in the browser context of any user visiting the page.

### Exploitation & Client-Side Bypass
* **Attack Surface:** Identified the "Name" field as vulnerable to HTML injection (confirmed via `<h1>` tag injection).
* **Bypassing HTML Restrictions:** Modified the `maxlength` attribute in the browser's DevTools to bypass the client-side character limit.
* **Filter Evasion:** Basic `<script>` tags were blacklisted. I bypassed this by using an HTML event-handler payload.
* **Payload:** `<img src=x onerror=alert(1)>`
* **Result:** Persistent execution. Every time the Guestbook page is loaded, the script triggers in the victim's browser.


## Strategic Framework Mapping

| Phase | Module | MITRE ATT&CK | OWASP Top 10 |
| :--- | :--- | :--- | :--- |
| **Data** | SQL Injection | T1505 | A03: Injection |
| **System** | Command Injection | T1059 | A03: Injection |
| **Client** | Stored XSS | T1059 | A03: Injection / A07 |



## Defender Lessons Learned
1. **Never Trust the Client:** Relying on dropdown menus or `maxlength` attributes for security is ineffective, as they are easily bypassed via DevTools.
2. **Blacklists Fail:** Filtering specific characters (like `;`) or tags (like `<script>`) is insufficient. Security should rely on allow-listing or context-aware encoding.
3. **Defense in Depth:** Proper remediation requires server-side validation, output encoding, and a robust Content Security Policy (CSP).
