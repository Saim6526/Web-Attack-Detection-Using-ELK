# Step 1: Identifying the Attack Surface (DVWA)
Before we break into DVWA, we need to understand the procedures we need to adopt. Think of it like this: if you want to break into a house, you need to have some knowledge of the house first. You don't just run at the front door.

#### **Reconnaissance vs. Enumeration**
- **Reconnaissance:** Finding out everything you can about the target before touching anything.

- **Enumeration:** Interacting with the target and pulling information out of it.

For the simplest form of recon, we check the IP of our machine and make sure it’s alive.

```
# Check your IP
ip a

# Ping the target to see if it's up
ping -c 3 <DVWA-IP>
```
Instead of guessing which ports to attack, we run an Nmap scan. This scan is the foundation of our entire attack.


`nmap -sV -sC -O <DVWA-IP>`
- `-sV`: Identifies the version of running services.

- `-sC`: Uses default NSE scripts for more info.

- `-O`: OS fingerprinting.

## What is Enumeration?
DVWA is dozens of vulnerable modules all bundled together. After the recon phase comes Enumeration. This is where we look for the specific "hooks" we can grab onto.

*Note: This is enumeration, not scanning. If we don't do this, we will be going in blind.*

### The Application Attack Surface
Check everything that could potentially take input or leak info:

- Login page behavior: How does it act?
- Session cookies: Are they predictable?
- URL structure: Look for parameters like ?id=, ?page=, ?user=, or ?cmd=. Parameters = Injection points.
- Hidden Parameters/Fields: These often leak functionality or logic.
- Forms: Check the methods (GET vs. POST) and input names.
- Headers: Look for X-Powered-By or Server headers.

### Technology Stack
This part helps us identify the vulnerabilities. We need to know:

- PHP/Apache/Debian **versions**
- **Backend DB**: Is it MySQL or MariaDB?
- **Public Files**: Are there files like robots.txt, phpinfo.php, or backup files lying around?

### Function-Level Enumeration
We cannot simply try to hack a website all at once. Instead, we need to approach it step by step, focusing on one part at a time. DVWA is designed this way so we can learn that process. For each module, we should examine it carefully and ask ourselves questions:

- SQL Injection: Does it error? Is it Blind? Boolean? UNION-based?
- File Upload: Is there a MIME check? Extension check? Is it only client-side?
- Command Injection: Are there input filters? A blacklist? Is the output encoded?
- XSS: Is it Reflected, Stored, or DOM-based?
- Brute Force: Are there rate limits? How does the login logic handle weak usernames?

### Hands-On: Attack Surface Enumeration (Step 0)
What are the things you would inspect on the site without running a single exploit? Before we touch Nmap, Burp Suite, or any modules, we just type stuff and watch reactions.

- Type ' in a form: Does it trigger an SQLi error?
- Type <script>: Does it hint at XSS?
- Type && ls: Does the app try to run it?
- Type an extreamly long input: Test for buffer or validation issues.

### Directory & Tech Leakage
Visit these common paths to see what works:

- `/robots.txt`
- `/phpinfo.php`
- `/config/ or /backup/`
- `/admin/ or /uploads/`

### Task: Testing the Fundamentals
I tested these theories on the lab setup to see what I could find.

Find the PHP Version: I navigated from http://localhost/DVWA/index.php to http://localhost/DVWA/phpinfo.php. 
This immediately leaked the server's configuration.

Check Cookies: I hit F12 to open DevTools, went to the Storage tab, and selected Cookies. This showed the PHPSESSID and security level settings.

Monitor Login Paths: Using the Network module in DevTools, I watched the traffic while logging in.

### What these findings mean:
Our enumeration revealed two primary endpoints used during login:

login.php (Authentication Endpoint): An attacker asks: "Is there SQLi here? Does it leak error messages? Does it set predictable cookies?"

index.php (Post-Auth Dashboard): This confirms the login worked, cookies were accepted, and no anti-CSRF protections are blocking us.

The basics of web enumeration are now complete.

Directory & Function Discovery (Web Recon II)
This phase moves beyond high-level network scanning. Here, we actively hunt for hidden directories, files, and functional modules to map out the application's attack surface.

## Discovery Techniques
To find out what we’re actually hacking, You can utilize three primary approaches:

- Manually Browsing: Just clicking through the app to see how it behaves.
- Spidering with Burp Suite: Using the automated spider to crawl the link structure and discover endpoints I might have missed.
- Directory Fuzzing: Using tools to brute-force common paths.
  - Tools: dirsearch, gobuster, or ffuf.

For DVWA, the goal is to map out the active components:

- `/vulnerabilities/`
- `/security.php`
- `/includes/`
- `/setup.php`

### Automated vs. Manual Enumeration
I attempted to automate discovery using dirsearch:


`dirsearch -u http://127.0.0.1/DVWA/`
*Note: The tool didn't return significant results for my specific configuration, so I manually enumerated the vulnerabilities modules. This is often necessary when automated tools fail to account for specific application configurations.*

### Mapping Modules to Attacker Frameworks
The final step of the Recon/Enumeration phase is mapping the discovered modules to the tactical framework an attacker actually follows. This turns a "list of buttons" into a "threat model."

| Module | Goal / Technique | Log Indicators |
| --- | --- | --- |
| **Brute Force** | Credential attacks, weak auth | Multiple failed logins, repeated sources |
| **SQL Injection** | Injection, direct data access, bypass login | Odd query strings, ', UNION, SELECT, OR 1=1 |
| **Command Injection** | RCE, gaining shell, pivoting | ping, netcat, strange commands in parameters |
| **XSS (Reflected/Stored/DOM)** | Browser hijacking, cookie theft | <script>, event handlers, encoded payloads |
| **File Inclusion / Upload** | Backdoors, RCE, traversal | ../../, PHP file uploads, shell.php |
| **CSRF** | User impersonation, account takeover | Legitimate UA but suspicious actions |
| **Weak Session IDs** | Predictable tokens, session hijacking | Token manipulation, session replay |
| **Crypto / API / Redir** | Token manipulation, redirect phishing | Abusing insecure API endpoints |
