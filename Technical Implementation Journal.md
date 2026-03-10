# 📑 Project Navigation Index

| Section | Phase | Quick Link |
| :--- | :--- | :--- |
| **01** | **Kali Linux Setup** | [Environment Initialization](#environment-initialization-kali-linux) |
| **02** | **LAMP Stack** | [Building the LAMP Stack](#building-the-lamp-stack) |
| **03** | **DVWA Setup** | [DVWA Deployment & Configuration](#dvwa-deployment--configuration) |
| **04** | **Forensic Logging** | [The Attacker Mindset Logging Setup](#the-attacker-mindset-logging-setup) |
| **05** | **Evidence Gathering** | [Live SQLi Testing & Evidence Collection](#2-live-sqli-testing--evidence-collection) |
| **06** | **Log Ingestion** | [Log Ingestion & Filebeat Configuration](#log-ingestion--filebeat-configuration) |
| **07** | **SIEM (Docker ELK)** | [Host Machine SIEM Setup (Dockerized ELK)](#host-machine-siem-setup-dockerized-elk) |
| **08** | **Networking** | [Networking: Connecting the VM to the Host](#networking-connecting-the-vm-to-the-host) |
| **09** | **Visualization** | [Final Validation & Kibana Visualization](#final-validation--kibana-visualization) |

---


# Environment Initialization: Kali Linux
The project started with setting up the Kali Linux environment. I opted to install the ISO directly rather than using a pre-packaged virtual machine to ensure full control over the installation process.

Once the OS was settled, the first essential step was a full system refresh:


`sudo apt update && sudo apt upgrade`
This ensures all tools have the latest security and dependency fixes, preventing old failures when running modern code later on.

## Building the LAMP Stack
With the base OS ready, the next phase was installing the LAMP stack (Linux, Apache, MariaDB, PHP) required to host DVWA. Each component plays a specific role:

Apache2: The web server that hosts the app and generates the primary HTTP access/error logs.

MariaDB: The database where SQLi attacks are targeted; this produces the database activity we need to monitor.

PHP & Modules: The engine that interprets the DVWA code and handles logic between the web server and the database.

Installation Command:
`sudo apt install -y apache2 mariadb-server php php-mysqli php-gd php-xml php-mbstring php-curl libapache2-mod-php git unzip ccze`

Verifying Services:
It's important to check if the servers are actually running after installation.

```
# Check status
sudo systemctl status apache2 --no-pager
sudo systemctl status mariadb --no-pager
```
```
# If not running, start them manually
sudo systemctl start apache2
sudo systemctl start mariadb
```
DVWA Deployment & Configuration
After the stack was live, I cloned DVWA into the web root and set up the necessary permissions so the web server could interact with it properly.

1. Clowning & Permissions
```
cd /tmp
git clone https://github.com/digininja/DVWA.git
sudo mv DVWA /var/www/html/
sudo chown -R www-data:www-data /var/www/html/DVWA
sudo chmod -R 755 /var/www/html/DVWA
```
2. Database & User Setup
I had to create a dedicated database and user to allow DVWA to function and produce realistic activity for our logs.

  SQL
-- Inside MariaDB prompt (sudo mysql)
```
CREATE DATABASE dvwa;
CREATE USER 'dvwauser'@'localhost' IDENTIFIED BY 'dvwapass';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwauser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
3. Application Config
I modified the DVWA config template (/var/www/html/DVWA/config/config.inc.php) to match these database credentials:
```
PHP
$_DVWA[ 'db_server' ]   = 'localhost';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ]     = 'dvwauser';
$_DVWA[ 'db_password' ] = 'dvwapass';
```
Finally, I imported the database schema to complete the setup:


`sudo mysql -u dvwauser -p dvwa < /var/www/html/DVWA/dvwa.sql`

## The "Attacker Mindset" Logging Setup
Setting up logs is crucial because the main goal is to understand the hacker's mindset and the evidence they leave behind. This data is what we use to build playbooks for SIEM/IDS/IPS.

Key Logs Collected:
- Apache access.log: Shows HTTP requests and payloads in parameters (essential for injection detection).
- Apache error.log: Reveals server-side warnings and misconfigurations.
- MariaDB slow_query_log: Captures queries that exceed a time threshold—great for spotting heavy UNION or sleep-based injections.

1. Logging Verification & Advanced Configuration
Before moving to the SIEM, I had to ensure Apache and MariaDB were actually producing the necessary logs. This is the foundation for database activity monitoring and critical for intrusion detection.

Apache Log Verification
I first confirmed the location and format of the Apache logs. These are the primary source to detect injection attempts by seeing payloads in the parameters.

```
# Check vhost config to see where logs are directed
sudo apachectl -S
sudo grep -n "CustomLog\|ErrorLog" -R /etc/apache2 -n
```
```
# List default files to verify existence
ls -l /var/log/apache2/
```
```
# Ensure logrotate is enabled to prevent disk space issues
cat /etc/logrotate.d/apache2
MariaDB: Enabling Forensic Logging
```
Standard database logging is often too quiet. For this project, I needed to see every query during testing and identify slow/heavy queries that might indicate automated scanning or complex injections.


Temporary General Logs (For Testing Only)
General logs capture every single query. It’s very verbose, so I only turn it on during active testing.

```
# Turn on and set the log file path
sudo mysql -e "SET GLOBAL general_log = 'ON'; SET GLOBAL general_log_file = '/tmp/mysqld-general.log';"
```
```
# Confirm variables are set
sudo mysql -e "SHOW VARIABLES LIKE 'general_log'; SHOW VARIABLES LIKE 'general_log_file';"
```
```
# Tail the log in real-time
sudo tail -f /tmp/mysqld-general.log
```
```
# Turn off when done
sudo mysql -e "SET GLOBAL general_log = 'OFF';"
```

Persistent Slow Query Log
I enabled the slow query log to catch heavy UNION or SLEEP usage. I added these lines to the end of /etc/mysql/mariadb.conf.d/50-server.cnf:

```
Ini, TOML
# Slow query log (enable for forensics/analysis)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```
Log Permissions and Restart
After the config changes, I had to set the correct permissions and restart the service.

```
# Create directory and set permissions
sudo mkdir -p /var/log/mysql
sudo touch /var/log/mysql/mysql-slow.log
sudo chown mysql:adm /var/log/mysql/mysql-slow.log
sudo chmod 640 /var/log/mysql/mysql-slow.log
```
```
# Apply changes
sudo systemctl restart mariadb
```
```
# Verify settings are applied
sudo mysql -e "SHOW VARIABLES LIKE 'slow_query_log'; SHOW VARIABLES LIKE 'slow_query_log_file';"
```

2. Live SQLi Testing & Evidence Collection
With logging properly configured, I ran a controlled SQLi test to capture raw evidence. This data helps gain the raw evidence needed to build detections later.

Step 1: Real-time Monitoring
I opened a terminal to watch the web and SQL logs live while I performed the attack in the browser.

```
# Watch web + sql logs live
sudo tail -F /var/log/apache2/access.log /var/log/apache2/error.log /tmp/mysqld-general.log
```
Step 2: Executing the Attack
In DVWA (Security: Low), I entered the following payload into the SQL Injection section:
`' OR '1'='1' --`


Step 3: Forensic Evidence Gathering
After stopping the live tail, I recorded the logs in a safe folder and extracted the most relevant snippets for the final report.
```
# Create evidence directory and copy logs
mkdir -p ~/dvwa-evidence
sudo cp /var/log/apache2/access.log ~/dvwa-evidence/apache-access.log
sudo cp /var/log/apache2/error.log ~/dvwa-evidence/apache-error.log
sudo cp /tmp/mysqld-general.log ~/dvwa-evidence/mysql-general.log 2>/dev/null || true
```

```
# Extract specific snippets for analysis
sudo tail -n 60 /var/log/apache2/access.log > ~/dvwa-evidence/apache-access-tail.txt
sudo tail -n 200 /tmp/mysqld-general.log > ~/dvwa-evidence/mysql-general-tail.txt
```
Step 4: Archiving
Finally, I compressed the evidence. This archive can be moved to a central collector or used for future SIEM ingestion.

```
# Compress and save
tar -czvf ~/dvwa-evidence.tgz -C ~ dvwa-evidence
```
```
# Verify archive
ls -lh ~/dvwa-evidence.tgz
By logging these interactions, I’ve established the base for Database Activity Monitoring, which is critical for identifying abnormal queries that bypass standard web filters.
```

## Log Ingestion & Filebeat Configuration
**Understanding the Ingestion Pipeline**

Before diving into the setup, I had to understand what Ingestion actually accomplishes. It isn't just “moving files”; it's a four-step pipeline:

- Harvest – Reading raw files from sources (Apache, MariaDB, System).
- Parse / Process – Adding structure (converting messy text into JSON).
- Store – Sending data to a searchable index (Elasticsearch).
- Visualize / Alert – Creating dashboards and rules (Kibana).

For this stage of the lab, I started with Minimal Local Ingestion.
This means Filebeat harvests the logs and writes them as structured JSON events to a local file before we eventually ship them to the ELK stack.

### Installation: The Kali Repo “Fix”

Installing Filebeat on Kali Linux is tricky. Kali’s default repositories focus on offensive security tools, not monitoring agents.

To get the latest version of Filebeat, I had to manually add the Elastic Official APT Repository.

Import the GPG Key & Add Repo
```
# Import the Elastic GPG key for security
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg

# Add the repository to your sources list
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Update and install
sudo apt update && sudo apt install filebeat -y
```
### Configuration (filebeat.yml)

I configured Filebeat to monitor our LAMP stack logs.

One important detail: Filebeat throws an error if multiple output namespaces are enabled simultaneously.

1. Defining Inputs

I told Filebeat exactly which “apartments” (directories) to watch.
```
# /etc/filebeat/filebeat.yml

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/apache2/*.log
      - /var/log/mysql/*.log
    fields:
      source_type: web_and_db
```
2. Local File Output (The “Dry Run”)

Instead of sending logs directly to the SIEM, I directed the output to a local JSON file for verification.
```
output.file:
  path: "/var/log/filebeat"
  filename: filebeat
  rotate_every_kb: 10240
  number_of_files: 7
```

**Tip:**
Make sure to comment out output.elasticsearch or output.logstash while using output.file.
If multiple outputs exist, Filebeat will fail to start.

### Verification & Testing

After saving the configuration, I enabled the service and generated some test traffic to verify that logs were being captured.

```
sudo systemctl enable filebeat
sudo systemctl start filebeat
```
```
# Generate traffic to the web server
curl http://127.0.0.1/DVWA/
curl http://127.0.0.1/DVWA/login.php
```
To confirm success, I checked the Filebeat log directory. A new .json file appeared containing timestamped events from Apache and MariaDB.

`sudo ls /var/log/filebeat/`

### The Forensic Cheat Sheet

Once logs are converted to JSON, additional tools help with threat hunting and analysis.


- jq -	Prettifies and extracts specific fields from JSON log events
- grep / egrep -	Searches for attack patterns like UNION or SELECT
- awk	Extracts - specific columns such as IP addresses
- goaccess -	Terminal-based log visualizer that creates a temporary dashboard

Example Threat Hunting Command
```
# Extract only the raw log message and file path from Filebeat JSON
cat /var/log/filebeat/filebeat | jq '.message, .log.file.path'
```

### Conceptual: Implementing Automated Alerts

Note: I originally planned to implement this using Cron, but later focused on SIEM integration instead.
However, this is how a “Poor Man’s SIEM” could be implemented without ELK.

The Logic:
- The Script:a Bash script runs: `grep -Ei "UNION|SELECT|UPDATE|DELETE"` against the newest Filebeat JSON log file.
- The Action If a match is found, the script pipes that log line into: `/var/log/alerts/critical_sqli.log`
- The Schedule Using crontab -e, run the script every 60 seconds: `* * * * * /home/user/scripts/detect_sqli.sh`

This creates a basic automated alerting system that triggers whenever a SQL injection signature is detected within the ingestion pipeline.


## Host Machine SIEM Setup (Dockerized ELK)
1. Docker Infrastructure Setup
To keep the environment clean, all SIEM-related files are stored in a dedicated directory. The docker-compose.yml file defines our three core services: Elasticsearch (storage), Logstash (processing), and Kibana (visualization).

```
# Create and enter the SIEM directory
mkdir -p ~/siem && cd ~/siem
```
```
# Create the orchestration file
nano docker-compose.yml
Docker Compose Configuration
YAML
version: '3.7'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.13
    container_name: es
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.13
    container_name: logstash
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5044:5044"   # Beats input
      - "5000:5000"   # Optional syslog
    depends_on:
      - elasticsearch
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.13
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - elk

volumes:
  esdata:
    driver: local

networks:
  elk:
    driver: bridge
```

2. Logstash Pipeline Configuration
The Logstash pipeline is the "brain" of the ingestion process. It listens for incoming Filebeat traffic, parses the raw strings into searchable fields using Grok, and tags them for easy filtering.

```
mkdir -p logstash/pipeline
nano logstash/pipeline/01-beats.conf
Pipeline Logic (01-beats.conf)
Code snippet
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse Apache logs using the COMBINEDAPACHELOG pattern
  if [log] and [log][file] and [log][file][path] =~ "apache2" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
      overwrite => [ "message" ]
    }
    date {
      match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
      remove_field => ["timestamp"]
    }
    mutate { add_tag => ["apache"] }
  } 
  
  # Tag MySQL logs for future parsing
  else if [log] and [log][file] and [log][file][path] =~ "mysql" {
    mutate { add_tag => ["mysql"] }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
  }
  # Enable for debugging in terminal
  stdout { codec => rubydebug }
}

```

3. Deployment & Service Health Checks
With the configurations in place, we launch the stack and verify that the "heartbeat" of each service is healthy.

```
# Start the containers in detached mode
docker-compose up -d

# Verify container status
docker-compose ps
Verifying Elasticsearch & Cluster Health
Bash
# Check if ES is responding
curl -s http://localhost:9200

# Confirm cluster health (should return "green")
curl -X GET "localhost:9200/_cluster/health?pretty"
```

### Networking: Connecting the VM to the Host
To ship logs from the Kali VM to the Host SIEM, we must bridge the network gap.

- Identify Host IP: Run hostname -I | awk '{print $1}' on the host machine. (Let's call this [REDACTED_HOST_IP]).
- Update VM Filebeat: In the Kali VM, edit /etc/filebeat/filebeat.yml and set the Logstash output to [REDACTED_HOST_IP]:5044.
- Check Host Listener: Ensure the host is listening for incoming logs:
`sudo netstat -tulnp | grep 5044`

### Final Validation & Kibana Visualization
The final step is to create the data view in Kibana so we can actually see the attacks.

Access Kibana: Open http://localhost:5601 in your browser.

Create Index Pattern: * Go to Stack Management -> Index Patterns -> Create Index Pattern.

Name: filebeat-*.

Select @timestamp as the time field.

Manual Log Test: Run a manual injection on the VM to trigger a log:
echo "SQLi_TEST_LOG $(date)" | sudo tee -a /var/log/apache2/access.log

### Troubleshooting the Connection
If logs are missing, use these commands to find the bottleneck:

- Check Filebeat Connection: sudo filebeat test output
- Check Logstash Logs: docker logs logstash --follow
- Check VM Log Paths: ls -l /var/log/apache2/

Project Status: SIEM Online
The lab should now be fully operational. Offensive actions taken in the Kali VM are captured by Filebeat, processed via Logstash, and visualized in Kibana. This setup provides the foundation for advanced Penetration Testing and Threat Hunting research.
