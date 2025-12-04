# Prompt
System Design AWS is in the very early phase, where new cloud services such as EC2, DynamoDB, and S3 are under development.
Before the launch of these cloud services, we need to build the billing infrastructure,
so AWS can charge customers for their use of different services. 

Task is to design this billing infrastructure. 

* Different AWS cloud services should be able to send usage data to the billing service at any granularity. 
* Billing service/infrastructure needs to generate bills for different accounts. 
* Account users should be able to view their service usage through AWS portal.

---
```mermaid
graph TD

%% =====================
%% PRODUCERS (AWS SERVICES)
%% =====================
subgraph Producers[Service Producers]
  EC2[EC2 Usage Reporter]
  S3[S3 Usage Reporter]
  DDB[DynamoDB Usage Reporter]
end

%% =====================
%% INGESTION LAYER
%% =====================
subgraph Ingestion[Metering Ingestion Layer]
  subgraph API[Usage Ingestion API]
    UAPI[Metering API Gateway]
    Auth[AuthN/AuthZ Layer Account Service tokens]
  end

  UBuf[Buffered Appender]
  Batcher[Batcher + Compressor]
  Kafka[(Kafka Cluster - Durable Log)]
end

%% =====================
%% STREAM PROCESSING
%% =====================
subgraph StreamProc[Streaming Aggregation Pipeline]
  Flink[Flink / Kinesis Analytics]
  StreamState[(Stream State Store)]
  RTView[Real-Time Usage DB DynamoDB/Redis]
end

%% =====================
%% BATCH PROCESSING
%% =====================
subgraph BatchProc[Batch Reconciliation Pipeline]
  RawStore[(Raw Usage Store - S3/Glue)]
  Spark[Batch Aggregation Spark/EMR]
  Audit[Reconciliation + Audit Engine]
  BillDB[(Billing DB - Authoritative)]
end

%% =====================
%% CONSUMERS / PORTAL
%% =====================
subgraph Consumers[Consumers]
  Portal[AWS Billing Portal]
  Analytics[Internal Analytics/Finance Systems]
end

%% =====================
%% DATA FLOW
%% =====================

%% Producers -> Ingestion
EC2 -->|Usage events JSON/Proto| UAPI
S3 -->|Usage events JSON/Proto| UAPI
DDB -->|Usage events JSON/Proto| UAPI

%% Ingestion pipeline
UAPI --> Auth --> UBuf --> Batcher --> Kafka

%% Streaming
Kafka --> Flink
Flink --> StreamState
StreamState --> RTView

%% Batch
Kafka --> RawStore
RawStore --> Spark --> Audit --> BillDB

%% Reconciliation feedback loop
BillDB -->|Reconciliation reports| StreamState

%% Consumer Access
RTView --> Portal
BillDB --> Portal
BillDB --> Analytics


```
---
```mermaid
graph TD

%% =====================
%% EC2 Sequence Numbers
%% =====================
subgraph EC2[EC2 Metering]
  EC2Instance[i-123 EC2 Instance]
  EC2Seq[Sequence No per Instance]
  EC2Event[Usage Event with seq_no]
end

EC2Instance --> EC2Seq --> EC2Event

%% =====================
%% S3 Sequence Numbers
%% =====================
subgraph S3[S3 Metering]
  S3Bucket[bucket-xyz S3 Bucket]
  S3Ops[Operations / Snapshots]
  S3Window[Assign to Usage Window]
  S3Seq[Sequence No per Bucket per Window]
  S3Event[Aggregated Usage Event with seq_no]
end

S3Bucket --> S3Ops --> S3Window --> S3Seq --> S3Event

%% =====================
%% Shared Ingestion
%% =====================
subgraph Ingestion[Metering Ingestion Layer]
  Kafka[(Kafka Durable Log)]
  StreamProc[Flink / Stream Processing]
  BatchProc[Spark / Batch Reconciliation]
end

EC2Event --> Kafka
S3Event --> Kafka
Kafka --> StreamProc
Kafka --> BatchProc

%% =====================
%% Real-Time and Batch
%% =====================
StreamProc --> RTView[Real-Time Usage DB]
BatchProc --> BillDB[Billing DB]

```