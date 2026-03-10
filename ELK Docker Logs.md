# Defensive Analysis: Log Parsing & Forensic Integrity

Once the logs reach the SIEM, the priority shifts from **Ingestion** to **Integrity**. A SIEM is only as powerful as its parsing logic. If the logs aren't structured, detection rules cannot be automated.

---

## Log Structure & GROK Validation
Before building dashboards, I verified that the **Logstash GROK filters** were correctly "breaking" the raw Apache strings into meaningful JSON fields. 

### The Difference: Success vs. Failure
If GROK parsing is successful, the data is searchable and indexable. If it fails, the SIEM treats the entire log as a single, unreadable string.

| Feature | Successful Parsing | GROK Failure (`_grokparsefailure`) |
| :--- | :--- | :--- |
| **Searchability** | Can filter by `clientip` or `response` | Must use slow "full-text" searches |
| **Alerting** | Can trigger on `response: 404` | Cannot isolate status codes |
| **Structure** | Organized into specific JSON fields | Entire log dumped into the `message` field |

### SIEM Verification Results
After auditing the Kibana "Discover" tab, I confirmed that the following fields are being correctly extracted from the DVWA traffic:

* `clientip`: "127.0.0.1"
* `verb`: "GET"
* `request`: "/DVWA/vulnerabilities/sqli/…"
* `response`: "200"
* `httpversion`: "1.1"

---

## Elasticsearch Optimization: Fields vs. Keywords
I audited the index mapping to ensure efficient searching. Elasticsearch stores fields in two ways to balance speed and flexibility:

1.  **Standard Field (`clientip`)**: Used for full-text search.
2.  **.keyword Field (`clientip.keyword`)**: Used for exact matches, aggregations, and grouping in dashboards.

*Note: These are not duplicates; they are a mechanical necessity for any professional SIEM pipeline to allow for high-speed data visualization.*

---

## Log Source Intelligence
Understanding what each log file provides is essential for **Correlation** and **Incident Response**.



### A. Apache Access Logs (`access.log`)
This is the **Primary Detection Source**. It records the "What, Who, and Where" of the transport layer.
* **Best for:** Detecting SQLi payloads in URI parameters, Brute Force attempts (repeated 401/200 codes), and XSS scripts in GET requests.
* **Example Pattern:** `POST /DVWA/vulnerabilities/xss_s/ HTTP/1.1 200`

### B. Apache Error Logs (`error.log`)
This provides the **Context**. It records the "Behind the Scenes" of the application.
* **Best for:** Confirming exploitation. For example, if an SQLi attack is successful, the error log might show database driver warnings or PHP crashes that confirm the app’s logic was broken.
* **Useful for:** Correlation and Impact Analysis.

---

## Security Takeaway
By ensuring every log is parsed into a structured format, we transition from "looking at text files" to **Database Activity Monitoring**. This structure is what allows us to write automated detection rules that can block an attacker in real-time.

# SIEM Hardening & Automated Detection Engineering

This phase documents the transition from manual log searching to **Automated Alerting**. This involved hardening the ELK stack, resolving complex authentication dependencies, and deploying high-fidelity detection rules.

---

## Hardening the Stack: Security & Encryption
By default, many ELK installations run without security. To enable the **Detection Engine**, I had to implement mandatory encryption and authentication.

### The "Permission Denied" Hurdle
When first accessing the Rules page, I encountered a `Detection Engine Permission Required` error. This was due to missing encryption keys and security configurations.

**Resolution Steps:**
1. **Enabled X-Pack Security:** Updated `docker-compose.yml` to set `xpack.security.enabled=true`.
2. **Credential Management:** Generated system passwords using:
   `docker exec -it es /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto`
3. **Encryption Key Generation:** Used OpenSSL to generate 32-byte base64 keys for Kibana:
   `openssl rand -base64 32`

### Configuration Fix (Kibana Environment)
*Note: GitHub and Docker environment variables must be capitalized for the system to parse them correctly.*

```
- XPACK_SECURITY_ENCRYPTIONKEY=KEY_1
- XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=KEY_2
- XPACK_REPORTING_ENCRYPTIONKEY=KEY_3
- xpack.security.authc.api_key.enabled=true
```

---

## Detection Rule Creation: SQL Injection

With the engine online, I created a Custom Query Rule designed to flag signatures of SQL manipulation.

### Rule Logic (KQL)
This rule monitors the `filebeat-*` index for common SQL injection characters and keywords that should never appear in standard web application traffic.

```
message : (
  "*union*select*" OR
  "*select*from*" OR
  "*information_schema*" OR
  "*sleep(*" OR
  "*benchmark(*" OR
  "*order by*" OR
  "*--*" OR
  "*#*"
)
```

### Rule Metadata
- **Name:** Possible SQL Injection Attempt (Web Logs)  
- **Severity:** High (Risk Score: 70)  
- **MITRE ATT&CK Mapping:** T1190 - Exploit Public-Facing Application  
- **Schedule:** Runs every 5 minutes with a 5-minute look-back to ensure no data gaps.

---

## Troubleshooting the Ingestion Pipeline (401 Error)

After enabling security, the logs stopped flowing.

### Diagnostic Process
- **Checked Filebeat:** Confirmed it was still harvesting logs locally.  
- **Checked Logstash Logs:** Identified a `401 Authentication Error`.

### The Fix
Logstash now required credentials to communicate with Elasticsearch. I updated the output section of the Logstash pipeline:

```
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    user => "elastic"
    password => "ELASTIC_PASSWORD"
    index => "filebeat-%{+YYYY.MM.dd}"
  }
}
```

**Note:** I also resolved a *Pipeline Conflict* by consolidating multiple conflicting `.conf` files into a single primary pipeline.

---

## SOC Playbook: SQLi Response

I established a standard operating procedure for when this alert fires:

| Step | Action | Description |
|-----|------|-------------|
| 1 | Trigger | ELK Alert | High-severity alert fires based on SQL keywords |
| 2 | Investigate | Source Audit | Identify the Source IP and cross-reference `error.log` for successful injection evidence |
| 3 | Contain | IP Blocking | Temporarily block the source IP at the firewall or rate-limit the endpoint |
| 4 | Remediate | Code Fix | Move from dynamic queries to Prepared Statements (Parameterized Queries) |

---

## Results & Lessons Learned

**Persistence Matters:**  
Most security features in professional SIEM platforms are disabled by default. Understanding how to configure encryption keys is a critical skill for a security engineer.

**Authentication Chains:**  
Securing one part of the stack (Elasticsearch) requires updating the entire pipeline (Logstash and Filebeat).

**High-Fidelity Alerts:**  
The custom KQL rule produced **zero false positives** during testing while successfully detecting every manual injection attempt.

**Lab Project Status:** Fully Operational.

---

### Final Touches for your GitHub
- **Table of Contents:** Add a link to this section in your **Project Navigation Index**.
- **Sensitive Information:** Replace real passwords with placeholders before publishing.
