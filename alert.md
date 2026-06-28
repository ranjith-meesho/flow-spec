Monitoring Architecture Documentation

Overview

Complete monitoring stack for microservices with three pillars: metrics, logs, and traces. Designed for multi-BU deployments with cost optimization and comprehensive observability.

***
1. METRICS COLLECTION & STORAGE

Architecture Flow

Applications/Nodes 
    ↓
Exporters (Telegraf, Node Exporter, Stack Driver)
    ↓
VM Agent (Scrape Job - 5 min interval)
    ↓
VM Insert (Write Point)
    ↓
VM Storage (Time Series Database)
    ↓
VM Select (Query Endpoint)
    ↓
Grafana (Visualization)

Components

Data Sources
Application metrics
Node metrics
Custom metrics from services

Exporters
Telegraf: Primary exporter, superior performance characteristics
Node Exporter: System-level metrics
Stack Driver: Cloud-native metrics

Why Telegraf over Daemon Sets?
Single VM Agent eliminates single point of failure in containerized exporters
If one service becomes high-volume, entire daemonset won't crash
More resource-efficient deployment model

VM Agent (Victoria Metrics Agent)
Runs as pod with annotations
Dynamically discovers scrape targets via pod annotations
Configurable from central configuration file
Written in Go for efficient garbage collection
Implements scrape interval logic (default: 5 minutes)

Configuration:
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5m
    static_configs:
      - targets: ['localhost:8080']
    pod_annotation: 'prometheus.io/scrape=true'

VM Insert
Write endpoint for time series data
Handles ingestion of metrics from exporters
Distributes data to VM Storage nodes

VM Storage
Stateful deployment (critical for persistence)
Time series database
Stores metrics with labels and ID
Cost-optimized storage layer

Active Time Series Optimization:
Only stores data when values change
Highly efficient label-based compression
Reduces storage footprint compared to traditional TSDB

VM Select
Query endpoint for Grafana
Retrieves metrics from VM Storage
Handles PromQL queries
Returns time series data to visualization layer

Metrics Pipeline - Why This Architecture

Cost Optimization:
Victoria Metrics is 10-100x cheaper than alternatives
Non-cross-cluster setup reduces infrastructure costs
Active time series tracking prevents unnecessary storage

Performance:
Low CPU/Memory usage through Go implementation
Reduced GC pressure via tuned garbage collection
Handles high cardinality efficiently

***
2. LOGS COLLECTION & ANALYSIS

Architecture Flow

Applications write logs to node disk
    ↓
Fluent D (reads from disk path)
    ↓
Log parsing (timestamps, filters)
    ↓
Flush on schedule or trigger
    ↓
Elasticsearch (centralized storage)
    ↓
Log search & analysis dashboards

Components

Application Logging
Apps write logs to node disk at configured path
Fluent D monitors this path for new entries
Supports multiple log formats

Fluent D Collector
Reads logs from node disk paths
Parses based on timestamp and filters
Buffers and flushes logs
Handles log rotation gracefully

Configuration:
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/app.pos
  <parse>
    @type json
    time_format %iso8601
  </parse>
</source>

<match *>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  flush_interval 5s
</match>

Elasticsearch
Central log storage
Full-text search capability
Time-series log retention
Integration with monitoring dashboards

***
3. DISTRIBUTED TRACING

Architecture Flow

4 Microservices (calling each other)
    ↓
OpenTelemetry Instrumentation (in each pod)
    ↓
Sampling Strategy:
  - 1% of normal requests (info level)
  - 100% of error requests
    ↓
Tail Sampling (decode path before decision)
    ↓
Elasticsearch APM (storage)
    ↓
APM Dashboard & Analysis

Components

Instrumentation
OpenTelemetry SDK deployed in each service pod
Automatic instrumentation of RPC calls
Context propagation across service boundaries

Sampling Strategy
Two-tier sampling:
Head Sampling (initial decision): 1% of all requests
Tail Sampling (informed decision): 
Know the complete request path
100% sampling for requests with errors
Random sampling for successful requests

Why Tail Sampling?
Captures all error scenarios (errors at end of chain)
Reduces noise from successful transactions
Statistical representation of normal operation

Elasticsearch APM
Stores complete trace information
Supports service dependency mapping
Query interface for trace analysis
Integration with application metrics

Trace Context

For each request:
Trace ID (unique request identifier)
Span ID (service operation identifier)
Service name and operation name
Timing information
Error status and details

***
4. TELEGRAF OPERATOR & DEPLOYMENT

Pod Annotation Setup

apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: app
    image: myapp:latest

Telegraf Operator Workflow

Deployment Phase
Apply pod annotation: prometheus.io/scrape=true
Telegraf operator watches for pod creation

Sidecar Injection
Operator detects annotation
Calls webhook to inject Telegraf sidecar
Sidecar runs in same pod as application

Configuration
Central configuration file for entire cluster
Operator uses config to setup scrape jobs
Pod annotations override defaults

Metric Exposure
Telegraf sidecar exposes metrics on pod network
VM Agent discovers and scrapes these endpoints
Metrics pushed to Victoria Metrics

Operator Configuration File

apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-operator-config
data:
  telegraf.conf: |
    [[outputs.prometheus_client]]
      listen = ":9273"
    
    [[inputs.cpu]]
      percpu = true
    
    [[inputs.mem]]
    
    [[inputs.http]]
      urls = ["http://localhost:8080/metrics"]
      name_prefix = "app_"

***
5. ALERTING SYSTEM

Alert Triggers

Metric-Based Triggers:
5xx HTTP errors (single pod vs all pods analysis)
CPU usage threshold exceeded
Memory usage spike
GC pause time excessive
Network errors detected
Circuit breaker activation
Connection timeouts

VM Alert Configuration

groups:
- name: app_alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 5m
    annotations:
      summary: "High error rate detected"

Key Features:
Rules check data from VM Select query endpoint
Compares against threshold
Waits for scrape interval (5 min) before triggering
Sends to third-party alert managers

Alert Routing

Slack Integration
Direct channel notifications
Error details and context
Links to dashboards
Alert metadata included

PagerDuty Integration
Creates incidents for on-call teams
Supports escalation policies
Team assignment by service label
Integration with incident management

PagerDuty Routing Rules:

routing_rules:
  - service_id: auth-service
    escalation_policy: auth-team-oncall
  - service_id: payment-service
    escalation_policy: payment-team-oncall
  - service_id: api-gateway
    escalation_policy: platform-team-oncall

Multi-BU Setup:
Each Business Unit has separate Victoria Metrics instance
Cross-cluster: shared incident management (PagerDuty)
Non-cross-cluster: isolated routing per BU

Alert Troubleshooting Workflow

When alert triggers:

Check Alert Source
Is it one pod or all pods?
Node-specific or service-wide?

Cross-Reference with Logs
Search Elasticsearch for error patterns
Check application stack traces
Look for correlation with deployment

Infrastructure Analysis
New pod on problem node → Check node resources
Increase HPA delay if false positives
Reduce HPA threshold if pod recovery too slow

Common Causes & Fixes

   5xx Errors on New Pod:
Cause: Insufficient warmup time
Fix: Increase readiness probe delay
Configuration: initialDelaySeconds: 30

   HPA Thrashing:
Cause: Threshold too tight
Fix: Increase threshold or cooldown
Configuration: targetAverageUtilization: 75 (instead of 60)

   CPU Spikes:
Cause: GC pressure
Fix: Reduce scrape frequency
Victoria Metrics written in Go, tuned GC helps

***
6. COST OPTIMIZATION STRATEGIES

Victoria Metrics Advantages

Storage Efficiency
Stores metrics with labels and IDs
Active time series only (when values change)
10x cheaper than Prometheus alternatives

Compute Efficiency
Go implementation with tuned GC
Reduced CPU usage vs alternatives
Memory optimization via pooling

Deployment Model
Single VM Agent (not daemonset per pod)
Centralized control
Failure isolation

Non-Cross-Cluster Setup

Benefits:
Each BU maintains own Victoria Metrics instance
No shared single point of failure
Independent scaling per business unit
Cost isolation per BU

Trade-offs:
Separate infrastructure overhead
Multiple instances to maintain
Limited cross-BU correlation

***
7. MULTI-BU ARCHITECTURE

Deployment Structure

Business Unit A
├── Victoria Metrics (own instance)
├── Telegraf Operator
├── VM Agent
└── Alerting Rules

Business Unit B
├── Victoria Metrics (own instance)
├── Telegraf Operator
├── VM Agent
└── Alerting Rules

Shared Services
├── PagerDuty (centralized on-call)
├── Elasticsearch (centralized logs)
├── Kibana (centralized log search)
└── Slack (centralized notifications)

Configuration Per BU

VM Storage: Separate stateful deployment
Telegraf Operator: Cluster-specific config
Alert Rules: BU-specific thresholds
Dashboards: Grafana per BU + shared

Cross-BU Monitoring

Elasticsearch logs (searchable across all BUs)
Shared APM dashboard for trace correlation
Centralized PagerDuty for incident tracking
Unified alert notifications in Slack

***
8. TROUBLESHOOTING GUIDE

No Metrics Appearing

Check Steps:
Verify pod has annotation: prometheus.io/scrape=true
Confirm telegraf sidecar injected: kubectl describe pod <pod>
Check VM Agent logs: kubectl logs -l app=vm-agent
Verify metrics endpoint: curl http://pod:port/metrics
Check VM Insert connectivity: telnet vm-insert:8480

Alerts Not Firing

Check VM Alert Rule Status
select alert_name, status from alerts
Verify for: duration has elapsed

Verify Data in VM Storage
Query VM Select: select_over_time(metric[5m])
Check timestamp freshness

PagerDuty Integration
Test webhook URL
Verify routing rule matches service label
Check escalation policy assignment

High Memory Usage

Victoria Metrics Tuning
--maxNumConcurrentMerges=2
--maxMetrics=1000000
--maxUniqueTimeseries=100000

Telegraf Configuration
Reduce collection interval
Filter unnecessary metrics
Limit label cardinality

Slow Trace Queries

Check Elasticsearch Cluster
Verify index size
Check shard allocation
Monitor disk space

Tail Sampling Tuning
Increase error detection threshold
Reduce info sampling ratio if dataset too large
Archive old traces to cold storage

***
9. KEY METRICS TO MONITOR

Application Metrics
Request rate (RPS)
Error rate (5xx, 4xx)
Response latency (p50, p95, p99)
Throughput

Infrastructure Metrics
CPU utilization
Memory usage
Disk I/O
Network throughput
Pod restart count

Database Metrics
Query latency
Connection pool utilization
Lock wait time
Replication lag

Distributed Tracing
Span duration (per service)
Error rate per service
Service dependency graph
Hot spots in request flow

***
10. DASHBOARDS & VISUALIZATION

Grafana Dashboards

Standard Dashboards:
System Overview (CPU, Memory, Disk, Network)
Application Health (Error Rate, Latency, Throughput)
Pod Performance (per-pod metrics)
Service Dependencies (trace visualization)
Alert Status (active/resolved alerts)

Custom Dashboards:
Business Unit specific KPIs
Cost tracking per service
Performance trending
Capacity planning

Log Analysis (Kibana)

Pre-built Searches:
Errors in last 24 hours
Slow queries
Failed transactions
Exception tracking

Alerting on Logs:
Alert on error count threshold
Anomaly detection
Pattern matching

***
Quick Start Checklist

Deploy Victoria Metrics (VM Insert, VM Storage, VM Select)
Configure Telegraf Operator with central config
Deploy VM Agent with scrape job configuration
Configure Fluent D for log collection
Deploy OpenTelemetry collectors (1% sampling)
Set up Elasticsearch for logs and APM
Configure VM Alert rules with thresholds
Integrate PagerDuty with routing rules
Create Grafana dashboards
Set up Kibana log search
Test end-to-end flow (generate traffic)
Document runbook for each alert type
