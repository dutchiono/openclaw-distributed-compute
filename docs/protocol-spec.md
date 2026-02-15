# OpenClaw Distributed Compute Protocol Specification v1.0

**Status:** Draft  
**Version:** 1.0.0  
**Date:** February 15, 2026  
**Authors:** OpenClaw Core Team

---

## Abstract

The OpenClaw Distributed Compute Protocol (ODCP) enables coordination of compute workloads across a heterogeneous network of Mac devices running OpenClaw. This specification defines the communication protocols, job scheduling mechanisms, security model, and incentive structures for a fault-tolerant distributed supercomputing network.

**Goals:**
- Enable 100,000+ devices to function as a unified compute cluster
- Support embarrassingly parallel workloads (ML training, rendering, simulations)
- Provide sub-minute job submission to execution latency
- Achieve 99.9% job completion reliability through redundancy
- Maintain zero-trust security model

---

## 1. Network Architecture

### 1.1 Component Overview

```
┌─────────────────────────────────────────────────────────┐
│                   CONTROL PLANE                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Job API    │  │  Scheduler   │  │   Registry   │  │
│  │  (REST/WS)   │  │  (Matchmaker)│  │  (Discovery) │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
┌─────────────▄───┐  ┌─────▄──────┐  ┌──▄─────────────┐
│   Data Plane    │  │   Data     │  │   Data Plane    │
│   (P2P WebRTC)  │◄─┤   Storage  │─►│   (P2P WebRTC)  │
│                 │  │   (IPFS)   │  │                 │
│  ┌───────────┐  │  └────────────┘  │  ┌───────────┐  │
│  │  Agent    │  │                   │  │  Agent    │  │
│  │  Node 1   │  │                   │  │  Node N   │  │
│  └───────────┘  │                   │  └───────────┘  │
└─────────────────┘                   └─────────────────┘
```

### 1.2 Components

**Control Plane (Centralized)**
- **Job API Server:** REST/WebSocket API for job submission and monitoring
- **Scheduler:** Matches jobs to optimal nodes based on requirements and capabilities
- **Registry Service:** Node discovery, capability tracking, health monitoring
- **Metrics Aggregator:** Network statistics, performance tracking

**Data Plane (Hybrid P2P + CDN)**
- **WebRTC Mesh:** Direct peer-to-peer data transfer between nodes
- **IPFS Storage:** Distributed storage for large datasets and code bundles
- **Coordinator Relay:** Fallback for NAT traversal and initial handshake

**Agent (On-Device)**
- **Resource Monitor:** Tracks idle time, CPU/GPU/RAM availability
- **Execution Sandbox:** Isolated environment for running untrusted code
- **P2P Client:** WebRTC connection management and data transfer
- **Heartbeat Service:** Reports status to registry every 30s

---

## 2. Job Lifecycle

### 2.1 Job Submission

**REST API Endpoint**
```http
POST /v1/jobs
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "job_type": "ml_training",
  "name": "ResNet50 Training",
  "code_bundle": "ipfs://Qm...",
  "entry_point": "train.py",
  "requirements": {
    "nodes": 100,
    "gpu_memory_min": "8GB",
    "ram_min": "16GB",
    "max_runtime": "4h",
    "redundancy": 3
  },
  "priority": "normal",
  "callback_url": "https://your-app.com/webhook"
}
```

**Response**
```json
{
  "job_id": "job_0699220a1b2c3d4e",
  "status": "queued",
  "estimated_start": "2026-02-15T20:05:00Z",
  "estimated_completion": "2026-02-15T23:45:00Z",
  "cost_credits": 1847
}
```

### 2.2 Job States

```
┌─────────┐     ┌─────────┐     ┌──────────┐     ┌───────────┐
│ QUEUED  │────►│ PENDING │────►│ RUNNING  │────►│ COMPLETED │
└─────────┘     └─────────┘     └──────────┘     └───────────┘
                     │                │
                     │                │
                     ▄                ▄
                ┌─────────┐     ┌─────────┐
                │ FAILED  │     │ TIMEOUT │
                └─────────┘     └─────────┘
```

**State Transitions:**
- `QUEUED`: Job accepted, waiting for scheduler
- `PENDING`: Nodes allocated, downloading code bundle
- `RUNNING`: Active execution on assigned nodes
- `COMPLETED`: All tasks finished successfully
- `FAILED`: Unrecoverable error (e.g., invalid code)
- `TIMEOUT`: Exceeded max_runtime

### 2.3 Task Partitioning

Jobs are split into tasks by the scheduler:

```python
# Example: ML training job with 100 nodes
job = {
  "nodes": 100,
  "epochs": 50,
  "dataset_shards": 100
}

# Scheduler creates 100 tasks
tasks = [
  {
    "task_id": "task_001",
    "node_id": "node_abc123",
    "shard_index": 0,
    "epochs": "0-50",
    "checkpoint_url": "ipfs://Qm..."
  },
  # ... 99 more tasks
]
```

**Task Assignment Algorithm:**
1. Parse job requirements (GPU memory, RAM, runtime)
2. Query registry for available nodes matching requirements
3. Score nodes by: proximity, past reliability, current load
4. Assign tasks to top N nodes
5. Create redundant assignments (3x for critical jobs)

---

## 3. Communication Protocols

### 3.1 Node Registration

**WebSocket Connection**
```javascript
// Agent initiates persistent connection
ws://registry.openclaw.org/v1/register

// Initial handshake
{
  "type": "register",
  "node_id": "node_abc123",
  "device_info": {
    "model": "Mac15,12",
    "chip": "M4 Pro",
    "tflops": 9.3,
    "ram_gb": 48,
    "gpu_memory_gb": 48,
    "disk_free_gb": 250,
    "bandwidth_mbps": 500,
    "location": {"lat": 40.7128, "lon": -74.0060}
  },
  "capabilities": ["ml_training", "rendering", "simulation"],
  "availability": {
    "allocation_pct": 25,
    "idle_threshold_sec": 300,
    "max_power_watts": 15
  }
}

// Registry response
{
  "type": "registered",
  "node_id": "node_abc123",
  "auth_token": "eyJ...",
  "heartbeat_interval_sec": 30
}
```

### 3.2 Heartbeat Protocol

Agents send heartbeat every 30 seconds:

```json
{
  "type": "heartbeat",
  "node_id": "node_abc123",
  "timestamp": "2026-02-15T20:00:00Z",
  "status": "idle",
  "current_load": {
    "cpu_pct": 5,
    "gpu_pct": 0,
    "ram_pct": 40,
    "temp_celsius": 42
  },
  "tasks_completed": 247,
  "uptime_sec": 67320
}
```

**Failure Detection:**
- Missing 2 consecutive heartbeats → mark node `SUSPECTED`
- Missing 4 consecutive heartbeats → mark node `OFFLINE`
- Node returns → mark `ONLINE`, apply reputation penalty

### 3.3 Task Assignment

**Scheduler → Agent (via WebSocket)**
```json
{
  "type": "task_assigned",
  "task_id": "task_001",
  "job_id": "job_0699220a1b2c3d4e",
  "code_bundle": "ipfs://QmXn...",
  "entry_point": "train.py",
  "env_vars": {
    "SHARD_INDEX": "0",
    "TOTAL_SHARDS": "100",
    "CHECKPOINT_URL": "ipfs://QmYz..."
  },
  "timeout_sec": 14400,
  "result_destination": "ipfs://QmAb...",
  "redundancy_group": "rg_001"
}
```

**Agent → Scheduler (Acceptance)**
```json
{
  "type": "task_accepted",
  "task_id": "task_001",
  "estimated_start": "2026-02-15T20:05:23Z"
}
```

### 3.4 Task Execution Updates

**Progress Reporting (Every 60s)**
```json
{
  "type": "task_progress",
  "task_id": "task_001",
  "progress_pct": 45,
  "metrics": {
    "iterations": 2250,
    "loss": 0.342,
    "throughput_its": 50
  },
  "estimated_completion": "2026-02-15T22:30:00Z"
}
```

**Completion**
```json
{
  "type": "task_completed",
  "task_id": "task_001",
  "result_url": "ipfs://QmCd...",
  "metrics": {
    "iterations": 5000,
    "final_loss": 0.089,
    "runtime_sec": 8520,
    "gpu_hours": 2.37
  },
  "checksum": "sha256:a3f2..."
}
```

### 3.5 P2P Data Transfer

For large datasets, nodes transfer directly via WebRTC:

```javascript
// Node A requests data from Node B
{
  "type": "data_request",
  "from_node": "node_abc123",
  "to_node": "node_xyz789",
  "resource": "ipfs://QmEf.../checkpoint_epoch_25.pt",
  "size_bytes": 524288000
}

// Node B responds with WebRTC offer
{
  "type": "data_offer",
  "from_node": "node_xyz789",
  "to_node": "node_abc123",
  "resource": "ipfs://QmEf.../checkpoint_epoch_25.pt",
  "webrtc_offer": {
    "type": "offer",
    "sdp": "v=0\r\no=- 123456789 2 IN IP4 127.0.0.1..."
  }
}

// Data transfer via WebRTC DataChannel
// Falls back to IPFS gateway if P2P fails
```

---

## 4. Security Model

### 4.1 Zero-Trust Principles

1. **Code Isolation:** All job code runs in sandboxed containers
2. **Encrypted Transport:** TLS 1.3 for control plane, DTLS for P2P
3. **Code Signing:** All code bundles signed by submitter
4. **Resource Limits:** CPU/GPU/RAM/disk quotas enforced by agent
5. **Network Isolation:** No outbound network access from sandboxed code

### 4.2 Sandboxing Strategy

**macOS Sandbox Profile**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Version</key>
  <integer>1</integer>
  
  <!-- Allow computation but deny system access -->
  <key>AllowNetworkAccess</key>
  <false/>
  <key>AllowFileSystemWrites</key>
  <dict>
    <key>AllowedPaths</key>
    <array>
      <string>/tmp/openclaw-job-*</string>
    </array>
  </dict>
  <key>AllowSystemCalls</key>
  <array>
    <string>read</string>
    <string>write</string>
    <string>mmap</string>
  </array>
  <key>DenySystemCalls</key>
  <array>
    <string>exec</string>
    <string>fork</string>
    <string>socket</string>
  </array>
</dict>
</plist>
```

**Resource Limits (via cgroups/containers)**
```yaml
limits:
  cpu_cores: 4
  ram_mb: 8192
  gpu_memory_mb: 8192
  disk_mb: 10240
  runtime_sec: 14400
  network_out: 0  # No outbound network
```

### 4.3 Code Verification

**Code Bundle Structure**
```
job_bundle.tar.gz
├── manifest.json          # Job metadata
├── signature.asc          # GPG signature
├── train.py               # Entry point
├── requirements.txt       # Dependencies
└── data/
    └── config.yaml
```

**Manifest Schema**
```json
{
  "version": "1.0",
  "name": "ResNet50 Training",
  "entry_point": "train.py",
  "runtime": "python:3.11",
  "dependencies": ["torch==2.0.0", "numpy==1.24.0"],
  "checksum": "sha256:a3f2b1c9...",
  "signer": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "signature": "0x7f3a..."
}
```

**Verification Process:**
1. Download code bundle from IPFS
2. Verify GPG signature against manifest
3. Compute SHA256 checksum, compare to manifest
4. Check dependency allowlist (no malicious packages)
5. Scan for suspicious patterns (eval(), exec(), system calls)
6. If valid, extract to sandbox directory

### 4.4 Data Privacy

**For Job Submitters:**
- Code bundles can be encrypted (AES-256)
- Private jobs only assigned to trusted nodes
- Result encryption before IPFS upload

**For Node Operators:**
- No personal data leaves device
- Job code executed in isolated sandbox
- Opt-in to specific job types (e.g., only academic research)

---

## 5. Fault Tolerance

### 5.1 Redundancy Strategy

**Task Replication**
```python
# Critical jobs: 3x redundancy
if job.priority == "high":
    redundancy = 3
    
# For each task, assign to 3 different nodes
for task in job.tasks:
    nodes = select_nodes(n=3, criteria=task.requirements)
    for node in nodes:
        assign_task(task, node, redundancy_group=task.id)
```

**Result Consensus**
- Wait for 2/3 nodes to complete
- Compare checksums of results
- If match: accept result
- If mismatch: assign to tiebreaker node

### 5.2 Failure Handling

**Node Failure During Execution**
```
Node A assigned task_001
↓
Node A goes offline (heartbeat timeout)
↓
Scheduler marks task_001 as FAILED
↓
Reassign task_001 to Node B (from standby pool)
↓
Node B completes task_001
```

**Checkpoint/Restart for Long Jobs**
```python
# Job code can save checkpoints
def train():
    for epoch in range(50):
        train_epoch(epoch)
        
        # Save checkpoint every 10 epochs
        if epoch % 10 == 0:
            save_checkpoint(f"checkpoint_epoch_{epoch}.pt")
            upload_to_ipfs(f"checkpoint_epoch_{epoch}.pt")
    
# On restart after failure, resume from last checkpoint
```

### 5.3 Network Partition Handling

**Split-Brain Prevention:**
- Centralized scheduler is source of truth
- Nodes cannot assign tasks to each other
- If node loses connection to scheduler for >5 minutes:
  - Pause current tasks
  - Save checkpoints
  - Wait for reconnection
  - Resume tasks after re-authentication

---

## 6. Incentive Mechanism

### 6.1 Credit System

**Earning Credits**
- 1 credit = 1 TFLOPS-hour contributed
- Credits earned in real-time as tasks complete
- Bonus multipliers:
  - High-reliability nodes: 1.5x
  - Long uptime (>30 days): 1.2x
  - Rare hardware (M4 Max): 1.3x

**Spending Credits**
- Submit jobs using credits instead of cash
- 1 credit = 1 TFLOPS-hour of compute consumed
- Priority queue access costs 2x credits

**Credit Account**
```json
{
  "user_id": "usr_abc123",
  "credits_balance": 1847,
  "credits_earned_lifetime": 5420,
  "credits_spent_lifetime": 3573,
  "contribution_rank": "top_15_pct"
}
```

### 6.2 Token Economics (Optional)

**CLAW Token Integration**
- Contributors can opt to receive CLAW tokens instead of credits
- Conversion rate: 1 credit = 10 CLAW tokens
- Tokens tradeable on DEXs
- Token holders get governance rights (protocol upgrades, fee structure)

**Staking for Priority Access**
- Stake 1000 CLAW tokens → priority job queue
- Stake 10,000 CLAW tokens → private job execution (trusted nodes only)
- Slash stake if node provides bad results (Byzantine behavior)

---

## 7. API Reference

### 7.1 Job Submission API

**Submit Job**
```http
POST /v1/jobs
Authorization: Bearer <api_key>

{
  "job_type": "ml_training | rendering | simulation",
  "name": "string",
  "code_bundle": "ipfs://...",
  "entry_point": "string",
  "requirements": {
    "nodes": "number",
    "gpu_memory_min": "string",
    "ram_min": "string",
    "max_runtime": "string",
    "redundancy": "number"
  },
  "priority": "low | normal | high",
  "callback_url": "string"
}
```

**Get Job Status**
```http
GET /v1/jobs/{job_id}
Authorization: Bearer <api_key>

Response:
{
  "job_id": "string",
  "status": "queued | pending | running | completed | failed",
  "progress_pct": "number",
  "tasks_completed": "number",
  "tasks_total": "number",
  "started_at": "ISO8601",
  "estimated_completion": "ISO8601",
  "result_urls": ["ipfs://..."]
}
```

**Cancel Job**
```http
DELETE /v1/jobs/{job_id}
Authorization: Bearer <api_key>

Response:
{
  "job_id": "string",
  "status": "cancelled",
  "tasks_cancelled": "number",
  "credits_refunded": "number"
}
```

### 7.2 Node Agent API

**Register Node**
```http
POST /v1/nodes/register
Content-Type: application/json

{
  "node_id": "string",
  "device_info": {...},
  "capabilities": ["array"],
  "availability": {...}
}
```

**Update Availability**
```http
PATCH /v1/nodes/{node_id}/availability

{
  "status": "idle | busy | offline",
  "allocation_pct": "number"
}
```

### 7.3 WebSocket Events

**Client → Server**
- `register` - Node registration
- `heartbeat` - Node health update
- `task_accepted` - Task assignment confirmation
- `task_progress` - Task execution progress
- `task_completed` - Task finished
- `task_failed` - Task error

**Server → Client**
- `registered` - Registration confirmed
- `task_assigned` - New task for node
- `task_cancelled` - Cancel running task
- `shutdown` - Graceful shutdown request

---

## 8. Performance Targets

### 8.1 Latency

| Metric | Target | Measurement |
|--------|--------|-------------|
| Job submission → queued | < 500ms | p99 API response time |
| Job queued → running | < 60s | Time to first task execution |
| Task assignment → start | < 30s | Code download + sandbox setup |
| Heartbeat interval | 30s | Fixed interval |
| Task progress updates | 60s | Fixed interval |

### 8.2 Throughput

| Metric | Target (Alpha) | Target (Production) |
|--------|----------------|---------------------|
| Concurrent jobs | 1,000 | 100,000 |
| Active nodes | 1,000 | 100,000 |
| Tasks/second | 100 | 10,000 |
| Data transfer (P2P) | 1 GB/s | 100 GB/s |

### 8.3 Reliability

| Metric | Target |
|--------|--------|
| Job completion rate | 99.9% |
| Node uptime | 95% (median) |
| Scheduler uptime | 99.99% |
| Data availability (IPFS) | 99.9% |

---

## 9. Deployment Architecture

### 9.1 Control Plane Infrastructure

**Cloud Provider:** AWS (Multi-region)

```
Region: us-east-1 (Primary)
├── ELB (Application Load Balancer)
│   ├── Job API (ECS Fargate, 10 tasks)
│   └── WebSocket Gateway (ECS Fargate, 20 tasks)
├── Scheduler (ECS Fargate, 5 tasks)
├── Registry (RDS PostgreSQL Multi-AZ)
├── Metrics (CloudWatch + Prometheus)
└── IPFS Gateway (EC2, 5 nodes)

Region: eu-west-1 (Failover)
└── [Same stack, read replicas]
```

**Database Schema**
```sql
-- Nodes table
CREATE TABLE nodes (
    node_id VARCHAR(64) PRIMARY KEY,
    device_info JSONB,
    capabilities TEXT[],
    availability JSONB,
    status VARCHAR(20),
    last_heartbeat TIMESTAMP,
    reputation_score INTEGER,
    credits_earned BIGINT
);

-- Jobs table
CREATE TABLE jobs (
    job_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64),
    job_type VARCHAR(50),
    status VARCHAR(20),
    requirements JSONB,
    submitted_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    result_urls TEXT[]
);

-- Tasks table
CREATE TABLE tasks (
    task_id VARCHAR(64) PRIMARY KEY,
    job_id VARCHAR(64) REFERENCES jobs(job_id),
    node_id VARCHAR(64) REFERENCES nodes(node_id),
    status VARCHAR(20),
    assigned_at TIMESTAMP,
    completed_at TIMESTAMP,
    result_url TEXT,
    redundancy_group VARCHAR(64)
);
```

### 9.2 Data Plane (IPFS)

**IPFS Pinning Infrastructure**
- 10 dedicated IPFS nodes across 3 regions
- Pin all code bundles for 30 days
- Pin job results for 7 days (or until user downloads)
- Use Pinata/Filebase for backup pinning

---

## 10. Monitoring & Observability

### 10.1 Metrics

**Network Metrics**
- Total nodes online
- Aggregate TFLOPS available
- Jobs queued / running / completed
- Tasks per second
- Credit distribution (earned vs spent)

**Node Metrics (Per Node)**
- Uptime percentage
- Tasks completed
- Average task duration
- Failure rate
- Reputation score

**Job Metrics (Per Job)**
- Queue time
- Execution time
- Cost in credits
- Completion rate
- Result size

### 10.2 Dashboards

**Public Dashboard** (compute.openclaw.org/stats)
- Live node count and map
- Aggregate TFLOPS
- Jobs completed today
- Top contributors

**Admin Dashboard** (internal)
- Node health overview
- Job queue depth
- Task failure rate
- Infrastructure costs

---

## 11. Future Extensions

### 11.1 Phase 2 Features

**Private Jobs**
- End-to-end encrypted code bundles
- Trusted node whitelist
- Result encryption before upload

**GPU Acceleration**
- Metal API integration for M-series GPUs
- Multi-GPU task scheduling
- GPU memory management

**Advanced Scheduling**
- Gang scheduling (start all tasks simultaneously)
- Preemption (pause low-priority jobs for urgent ones)
- Affinity rules (keep related tasks on same node)

### 11.2 Phase 3+ Features

**Federated Learning**
- Model aggregation without data sharing
- Privacy-preserving gradient updates
- Differential privacy guarantees

**Cross-Platform Support**
- Linux nodes (NVIDIA GPUs)
- Windows nodes (DirectML)
- ARM servers (AWS Graviton)

**Marketplace**
- Spot market for compute credits
- Auction-based pricing
- Premium tier for guaranteed SLAs

---

## 12. Compliance & Legal

### 12.1 Terms of Service

**Node Operators Must:**
- Only run agents on devices they own
- Not tamper with agent software
- Report security vulnerabilities responsibly

**Job Submitters Must:**
- Not submit malicious code
- Respect compute quotas
- Pay credits for commercial use

### 12.2 Content Policy

**Prohibited Workloads:**
- Cryptocurrency mining
- Password cracking / brute force attacks
- Illegal content generation
- DDoS attack coordination
- Spam/malware distribution

**Allowed Workloads:**
- Scientific research
- AI/ML training
- Creative rendering
- Data analysis
- Simulations

---

## 13. References

- WebRTC: https://www.w3.org/TR/webrtc/
- IPFS: https://docs.ipfs.tech/
- macOS Sandboxing: https://developer.apple.com/documentation/security/app_sandbox
- Distributed Systems Principles: M. van Steen, A. Tanenbaum

---

## Appendix A: Example Job Code

**Simple ML Training Job**
```python
#!/usr/bin/env python3
# train.py - Entry point for distributed training

import os
import torch
import torch.nn as nn
from torchvision import models, datasets, transforms

def main():
    # Read environment variables set by scheduler
    shard_index = int(os.environ['SHARD_INDEX'])
    total_shards = int(os.environ['TOTAL_SHARDS'])
    
    # Load model
    model = models.resnet50(pretrained=True)
    
    # Load data shard
    transform = transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
    ])
    dataset = datasets.ImageFolder('/tmp/openclaw-job-*/data', transform=transform)
    
    # Split dataset into shards
    shard_size = len(dataset) // total_shards
    start_idx = shard_index * shard_size
    end_idx = start_idx + shard_size
    subset = torch.utils.data.Subset(dataset, range(start_idx, end_idx))
    
    # Train
    loader = torch.utils.data.DataLoader(subset, batch_size=32, shuffle=True)
    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
    criterion = nn.CrossEntropyLoss()
    
    for epoch in range(10):
        for batch_idx, (data, target) in enumerate(loader):
            optimizer.zero_grad()
            output = model(data)
            loss = criterion(output, target)
            loss.backward()
            optimizer.step()
            
            if batch_idx % 100 == 0:
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {loss.item()}")
    
    # Save result
    torch.save(model.state_dict(), f'/tmp/openclaw-job-*/result_shard_{shard_index}.pt')
    print("Training complete!")

if __name__ == '__main__':
    main()
```

---

**End of Specification**

---

## Changelog

**v1.0.0 (2026-02-15)**
- Initial draft specification
- Core protocol design
- Security model
- API reference
