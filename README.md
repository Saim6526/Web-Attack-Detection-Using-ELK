# Web-Attack-Detection-Using-ELK

# SQL Injection Detection & Alerting with Elastic SIEM

## Objective

The objective of this project was to design and implement a basic SOC-style detection capability for SQL Injection (SQLi) attacks using the Elastic Stack. The project focused on ingesting real web server logs, analyzing malicious SQLi patterns, and creating functional SIEM detection rules and alerts. This lab was built as a defensive, detection-engineering–focused project rather than a penetration testing exercise.

## Skills Learned

- SQL Injection attack patterns (error-based, boolean-based)
- SIEM detection engineering fundamentals
- Elastic Security detection rule creation
- Alert tuning and validation
- Apache access log analysis
- HTTP request and parameter inspection
- Log ingestion and normalization
- Troubleshooting SIEM ingestion pipelines
- Basic SOC alert workflow

## Tools Used

- Elasticsearch
- Kibana (Elastic Security / Detection Engine)
- Logstash
- Filebeat
- Apache Web Server
- DVWA (Damn Vulnerable Web Application)
- Docker & Docker Compose

## Project Overview

This project simulates a real-world SOC detection scenario by generating SQL Injection attacks against a vulnerable web application and detecting them through log-based analysis. Apache access logs were ingested using Filebeat, processed through Logstash, and indexed into Elasticsearch. Detection rules were created in Kibana to identify SQL Injection attempts based on known payload patterns and request characteristics.

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
