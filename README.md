# Web-Attack-Detection-Using-ELK

## Objective

This if my first cyber security project that I worked on. I mainly did it to learn how SIEMs work, how to forward, analyze, and respond to logs alongside with understanding the attackers mindset as well as how attacks are done in a real world scenario. Although it seemed like a simple "Attack and Analyze Log" project, it turned out to be a log more complicated as you will see in the "Steps" section. I learned a lot more than just Analyzing and attacking but rather configurating, troubleshooting, detection engineering, mapping attacks to MITRE ATT&CK, OWSAP and more.

## Skills Learned

- Log Analysis
- SIEM detection engineering fundamentals
- Elastic Security detection rule creation
- Alert tuning and validation
- Apache access log analysis
- HTTP request and parameter inspection
- Log ingestion and normalization
- Troubleshooting SIEM ingestion pipelines
- Basic SOC alert workflow
- Basic Penetration Testing on DVWA (Command Injection, SQL Injection, XSS (Stored))
- Configuration of Docker, ELK, Filebeat

## Tools Used

- Elasticsearch
- Kibana (Elastic Security / Detection Engine)
- Logstash
- Filebeat
- Apache Web Server
- DVWA (Damn Vulnerable Web Application)
- Docker & Docker Compose

## Project Overview

This project simulates a real-world SOC detection scenario by generating different attacks against a vulnerable web application and detecting them through log-based analysis. Apache access logs were ingested using Filebeat, processed through Logstash, and indexed into Elasticsearch. Detection rules were created in Kibana to identify SQL Injection attempts based on known payload patterns and request characteristics.

Alerts were validated by manually triggering SQLi attacks and confirming alert generation without excessive false positives.

## Detection Logic

The SQL Injection detection rule focuses on identifying suspicious query patterns commonly associated with SQLi attacks, including:

- Boolean-based payloads (`' OR '1'='1`)
- SQL keywords in URL parameters (`UNION`, `SELECT`, `SLEEP`)
- Abnormal request structures in HTTP GET/POST requests

The rule operates on Apache access logs ingested into the `filebeat-*` index pattern and triggers alerts at a defined interval to simulate SOC monitoring behavior.

## Steps

Add screenshots for the following stages:

- Filebeat ingesting Apache access logs  
- Logstash pipeline configuration  
- SQL Injection attempts in DVWA  
- Detection rule configuration in Kibana  
- Triggered SQLi alerts in Elastic Security  

Each screenshot should include a short caption explaining what is being shown and why it matters.

*Ref 1: SQL Injection attempt in DVWA generating malicious HTTP requests logged by Apache*
