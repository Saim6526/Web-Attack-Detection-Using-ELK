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
