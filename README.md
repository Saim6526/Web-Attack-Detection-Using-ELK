# Web-Attack-Detection-Using-ELK

## Project Overview

This project simulates a real-world SOC detection scenario by generating different attacks against a vulnerable web application and detecting them through log-based analysis. Apache access logs were ingested using Filebeat, processed through Logstash, and indexed into Elasticsearch. Detection rules were created in Kibana to identify SQL Injection attempts based on known payload patterns and request characteristics.

Alerts were validated by manually triggering SQLi attacks and confirming alert generation without excessive false positives.

## Tools Used

- Elasticsearch
- Kibana (Elastic Security / Detection Engine)
- Logstash
- Filebeat
- Apache Web Server
- DVWA (Damn Vulnerable Web Application)
- Docker & Docker Compose

