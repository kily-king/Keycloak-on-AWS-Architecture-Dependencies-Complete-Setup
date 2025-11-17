Expectations 
----------------
Mature the observability platform
mature the security posture and security operations within the platform
mature the FinOps Platform

Questions from the management
--------------------------------
- What Metrics we monitor currently? 
- Where do we derive that from? ODS - Grafana - kibana - scripts? what is the current state?
- What is our proposal going forward for observability? What all we would be offering as default platform offering?
- What is the meaning of maturing the security operation, current gaps and do we know what do we want to fix?
- How are we going to mature the FinOps platform? Which tool we are planning? What all we are planning to monitor costing for?

We are currently running our environment in AWS cloud and want to use the same for supporting the above to the customers using all AWS services, below are the tools we are using currently.



EC2 data
-------------
CPU utilization
Network traffic
Disk
Inbound & Outbound Traffic ( IOPS/Bytes/Sec )
Disk Read/Write


CloudWatch
----------------
Incoming Log Events
Incoming Bytes
Delivery errors / Throttling
Forwarded log events / Bytes

Prometheus
------------
Scrape Duration
Grafana Metrics

EBS
---------
Volumes read /write 
Volume idle time / queue length

Lambda - No data
---------


CloudWatch logs - no data
--------------- 

errors 
Forwarded log events
Incoming Bytes / log events

RDS - No data
-------
---Cluster Metrics
------------------
CPU utilization 
Database Connections
Available RAM
---Instance metrics
-------------------
CPU utilization 
Database Connections
Storage Space
RAM
Read/Write Throughput
Read/Write Latency
Read/Write operations

Kubernetes - CoreDNS
---------------
health Status
DNS requests
Average Packet size
Request by type
Request by return code
Total Forward Requests
DNS errors
cache hits / misses
Cache Size
Request Latency
DNS Request Duration
DNS response size
DNS request size

Istio Ztunnel  - No data
--------------


EKS Pods by Node
---------------------
CPU Usage
Memory usage
Note Networking


----------------------------------------------------------------------------------------

Splunk Log Delivery
----------------------
Heavy Forwarder EC2 
- Logs
- Metrics
- Logs error/warn
- Metrics error/warn
Heavy Forwarder  ASG
- networkIn
- NetworkOut
- I/O%
- CPU Utilization
- StatusCheckFailed
Backup to S3
- bytes
- Success


NY - Prod
---------
EKS - HTTP requests received

Kafka
--------
- BytesInPerSec
- BytesOutPerSec

ABP error
CPA error


ABP Custom Metrics Durations
ABP Misc
ABP Logs
ABP Observability


Alerts
------------------
Kafka Alerts
Redis Prod Alerts
Application Alerts
infrastructure alerts


CloudWatch Log -  storage and alerting.
EventBridge for triggering workflows on specific API events.
OpenSearch for advanced querying.
CloudWatch metrics
X-ray - Traces
CloudTrail - API calls and account activity






Based on the above all information prepare me details document to present to my management.
