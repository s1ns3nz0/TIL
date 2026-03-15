# 1. Data Modeling with Amazon DynamoDB
Link: https://www.youtube.com/watch?v=kQ-DSjtCb90
## Amazon DynamoDB: Built to Scale
### Performance at scale
- Consistent, single-digit millisecond read and write performance
- Nearly unlimited throughput and storage
### Enterprise ready
- Data encryption at rest
- Global replication
- Up to 99.999% availability SLA
### No servers to manage
- Fully managed, scale-to-zero serverless database
- Massive scalability
### Built-in integration with others AWS services
- Logging, monitoring, and analytics
- APplicaitons that span multiple AWS services
> DynamoDB doesn't have JOIN
## Why Model?
- Single Table Philosophy
![](DynamoDB.png)
- DynmaoDB can support your relational access patterns
- Define a model for your access patterns
- Best Practice: Consider single table design
# 2. DynamoDB Core Components
Ref:
https://medium.com/@AlexanderObregon/deep-dive-into-aws-dynamodb-understanding-the-core-features-dc39cb3e14f2
## 2.1 Core Advantages 
### 1) Fully Managed - Zero Infrastructure Work
- DynamoDB eliminates manul databases administration entirely - hardware provisioning, configuration, replication, patching and scaling are all handled by AWS automatically.
### 2) Consistent Performance at Any Scale
- DynmaoDB is engineered to deliver single-digit millisecond response times regardless of scale
### 3) Flexible Schema
- DynamoDB supports both key-value and document data models, and its schema-less design means each item in a table can have different set of attributes
## 2.2 Features
### 1) Two Capacity Modes
| Mode | Best For | Pricing|
|------|----------|--------|
|Provisioned|Predictable, steady traffic| Pay for reserved capacity|
|On-Demand|Unpredictable or spiky traffic | Pay per request only |
### 2) DAX(DynamoDB Accelerator)
- DAX is an in-memory cache for DynamoDB that reduces read response times from milliseconds to microseconds, delivering up to 10x performance improvement even at millions of requests per second
### 3) DynmaoDB Streams
- Streams capture a time-ordered sequence of itme-level modifications in any DynamoDB table and store this information up to 24 hours - enabling event-driven architecture
```
DynamoDB change → Stream → Lambda
                              │
                              ├── Sync to OpenSearch
                              ├── Trigger notifications
                              └── Replicate to other services
```
### 4) Global Tables (Multi-Region Repliacation)
- Global Tables allows fully replicated, multi-region, multi-master DynamoDB tables without building or maintaining your own replication solution.
```
Write in Seoul → auto-replicated → Virginia + Ireland (~1 second)
Regional outage → Route 53 failover → other region serves traffic ✅
```
### 5) Transactions(ACID)
- DynamoDB transaction provides ACID properties, allowing you to group multiple operations across multiple items into a single all-or-nothing operation
### 6) Two Consistency Model
| Model | Speed | Use When |
|-------|-------|----------|
|Eventually Consistent| Fast, cheap (0.5x RCU) | General reads, high traffic |
|Strongly Consistent| Slower, Full RCU cost | Banking, inventory, critical data |

### 7) Security Built-In

Security in DynamoDB is multi-faceted — covering network isolation using Amazon VPC, encryption at rest using AWS KMS, fine-grained IAM access control, and SSL/TLS encryption in transit. 
```
Encryption at rest    → KMS (AWS-managed or customer-managed key)
Encryption in transit → SSL/TLS always on
Network isolation     → VPC Endpoint (traffic never hits public internet)
Access control        → IAM policies per table, per operation, per item
```
### 8) Secondary Indexes
```
Local Secondary Index (LSI)
  → Same partition key, different sort key
  → Query within a single partition

Global Secondary Index (GSI)
  → Completely different partition + sort key
  → Query by any attribute efficiently
  → Sparse indexes for subset queries
```
### 9) Point-In-Time-Recovery(PITR)
- DynamoDB offers continuous backups with point-in-time recovery, allowing you to restore your table to any point within the last 35 days for regulatory compliance and data protection.

# 3. DynamoDB of Identity Service
```
┌──────────────────────────────────────────────────────────────┐
│              Identity Architecture with DynamoDB             │
│                                                              │
│  Client                                                      │
│    │  1. Login request                                       │
│    ▼                                                         │
│  API Gateway (/auth/login)                                   │
│    │                                                         │
│    ▼                                                         │
│  Auth Lambda                                                 │
│    │  2. Verify credentials via Cognito                      │
│    │  3. Create session in DynamoDB                          │
│    │  4. Return sessionId + JWT                              │
│    ▼                                                         │
│  DynamoDB (Identity Store)                                   │
│    ├── Sessions (TTL auto-expiry)                            │
│    ├── API Keys (hashed, with permissions)                   │
│    └── RBAC (users → roles → permissions)                    │
│                                                              │
│  Subsequent requests:                                        │
│  Client → API Gateway → Lambda Authorizer                    │
│                │                                             │
│                │  lookup session/key/permissions             │
│                ▼                                             │
│          DynamoDB  ──► cache hit: DAX (~μs)                  │
│                    ──► cache miss: DynamoDB (~ms)            │
│                │                                             │
│                ▼                                             │
│          Allow/Deny → Backend Lambda                         │
└──────────────────────────────────────────────────────────────┘
```
# 4. When to use DynamoDB?
## 4.1 DynamoDB vs RDS/Aurora
| | DynamoDB | RDS / Aurora |
|---|---|---|
| **Data model** | Key-value / document | Relational tables |
| **Schema** | Flexible — no fixed schema | Fixed — ALTER TABLE needed |
| **Queries** | Access patterns defined upfront | Any SQL query, flexible |
| **Joins** | Not supported | Native |
| **Scale** | Infinite horizontal | Vertical (bigger instance) |
| **Latency** | Single-digit ms | 1-10ms (good hardware) |
| **Transactions** | Limited (25 items max) | Full ACID across all rows |
| **Cost at scale** | Cheaper for high traffic | Expensive (always-on instances) |
| **Serverless** | Native | Aurora Serverless only |
| **Learning curve** | Steep (new paradigm) | Gentle (SQL is familiar) |

## 4.2 Choose DynamoDB When
**Choose DynamoDB when:**
- Access patterns are well-known and fixed
- Scale is unpredictable or very high
- Serverless / Lambda architecture
- Simple relationships (no complex joins)
- Need global distribution (Global Tables)
- Session/cache/identity data