In this section I will explain the tools I used along side with the architecture and infastructure of this setup.
I will paste a diagram that I made on draw.io below for a visual representation along side with making it easier for me to explain, and anyone who goes through this section, to understand.

<img width="632" height="606" alt="image" src="https://github.com/user-attachments/assets/11a588c1-6296-45bc-9b82-71a54b95c8ba" />

### **Architecture & Infrastructure**

**The Target Environment (**Red Team Zone**)**

Virtualization: Hosted on a Kali Linux VM via Oracle.

Networking: Configured using NAT (Network Address Translation). This ensures environment isolation while allowing the VM to communicate with the Host through a virtual gateway.

Application Stack (LAMP):
- Linux (Kali)
- Apache (Web Server) - Listening on Port 80
- MariaDB (Database) - Listening on Port 3306
- PHP (Application Logic)
Target: DVWA (Damn Vulnerable Web App) used for simulating SQL Injection and Command Injection attacks.


**The Monitoring Pipeline (Log Shipping)**

To ensure real-time visibility, a telemetry pipeline was established:

- Agent: Filebeat is installed on the Kali VM as a lightweight log shipper.
- Data Source: Filebeat monitors /var/log/apache2/access.log and /var/log/mysql/error.log.
- Transport: Data is shipped via the Lumberjack protocol to the host machine on TCP Port 5044.


**The SIEM Stack (**Blue Team Zone**)**

The defense infrastructure is containerized using Docker Compose to ensure modularity and efficient resource sharing.

- Logstash: Ingests raw logs from Filebeat. It uses custom Grok filters to parse fields and identify malicious patterns (e.g., UNION, SELECT, ;, &&).
- Elasticsearch: A NoSQL search engine that indexes and stores parsed logs for rapid querying (Port 9200).
- Kibana: The visualization layer used for threat hunting and dashboarding (Port 5601).


🛠️ Tools Used

- Hypervisor: Oracle
- Containers: Docker & Docker Compose
- SIEM: Elastic Stack (ELK)
- Scanning: Nmap (Service Versioning & Default Scripts)
- Vulnerability Lab: DVWA
