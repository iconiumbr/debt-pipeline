# Debt Processing Pipeline System

## Overview
The Debt Processing Pipeline is a comprehensive system designed to process debt data submitted by clients through various methods. It handles data submission, storage, transformation, and final processing to ensure accurate recording and tracking of debt information in an asynchronous manner. This allows for scalable system operations based on variable demand.

## Reasoning
The company is a debt collection bureau that helps businesses collect overdue debts. Clients submit debt collection information, including debtor details, CPF (individual taxpayer registry number), amount owed, and due date. Once the debt is registered, further actions such as protests, notifications, discounts, or installment plans may follow.

## Scope
This system focuses on receiving, processing, and storing debt data submitted by clients. It does not cover the debt collection phase. The complexity arises from handling multiple data and file formats, ensuring smooth ingestion for both small and large clients with precision and efficiency.

## System Components

### Batch Service
Tracks and maintains the status of debt data requests submitted by clients.

### Input Services
Border services exposed to clients to collect debt data passively or actively, storing raw data for intermediate processing.

### File Picker Process
- Detects file submissions via FTP using S3 notifications to SQS.
- Validates file details and client identity.
- Splits large files into multiple sub-batches or registers single batches for processing.

### Intermediate Process
- Gathers raw data from various input sources.
- Performs ETL operations.
- Prepares and injects data into DynamoDB for registration.

### Registering Process
- Takes data from DynamoDB.
- Validates and processes debt data.
- Inputs finalized data into the debt storage system.

### Message Broker
- Uses Amazon SNS for inter-process messaging.
- Notifies batch service and intermediate systems of new files.

### Monitoring
- All processes emit time-series metrics to AWS Timestream for analysis.

## Batch Processing Service

### Responsibilities
- **Batch Creation:**
  - Initiated via synchronous call from input stages.
  - Stores batch details (status, file location, client ID, batch number, debt count, etc.).
  - Dispatches messages to the intermediate process queue.

- **Tracking and Status Updates:**
  - Maintains batch status throughout the pipeline.

- **Completion of Batch:**
  - Updates batch status upon registering process completion.
  - Dispatches notifications via SES (email) or EventBridge (webhook/other methods).

- **Failure Notification:**
  - In case of errors (bad format, corruption, etc.), notifies clients via SES or EventBridge.

## Input Phase

### 1. Data Submission via REST API (Default Format)
**Target:** Small & large clients using the default format.

**Operation:**
- Client submits multiple debt lines via REST API.
- Data is stored in a temporary NoSQL DB.
- Client submits a request to finalize processing.
- Data is converted into S3 CSV format.

**Follow-up:**
- Message sent to batch service and intermediate process queue.

### 2. Data Submission via REST API (CSV Format)
**Target:** Clients using the default CSV format.

**Operation:**
- Client uploads a single CSV file via REST API.
- CSV is stored in S3.

**Follow-up:**
- Message sent to batch service and intermediate process queue.

### 3. File Submission via FTP
**Target:** Clients sending variable-sized files.

**Operation:**
- Clients submit CSV files to S3 via FTP.

**Follow-up:**
- S3 event triggers SQS message to file picker service.

### 4. Self-Service (Custom Format)
**Target:** Small clients with low debt counts.

**Operation:**
- Clients upload CSV/TXT/XLS via a web interface.
- First 100 columns are previewed for manual column mapping.
- File is stored in S3.

**Follow-up:**
- Message sent to batch service and intermediate process queue with mapping details.

## File Picker Service

### Input
- Monitors SQS file picket queue.

### Operation
- Runs in parallel with multiple replicas.
- Validates files and identifies clients.
- Splits large files if necessary.
- Determines processing type (default/custom format).

### Follow-up
- Sends messages to batch service and intermediate process queue.

## Intermediate Service

### Input
- Monitors SQS intermediate queue.

### Operation
- Runs in parallel with multiple replicas.
- Validates files and retrieves processing rules.
- Processes data into DynamoDB format.

### Processing Methods
- Custom code (Python, Go, etc.).
- AWS Glue / AWS ETL.
- EMR Hadoop (for large data sets).

### Follow-up
- Updates batch service on processing status.
- Sends success/failure messages to registering process.

## Registering Service

### Input
- Monitors SQS register queue.

### Operation
- Runs in parallel with multiple replicas.
- Retrieves batch data from DynamoDB.
- Validates and processes debt records.
- Stores failed records in S3 for review.

### Follow-up
- Updates batch service on completion status.

## Metrics Collection
- Logs key performance metrics to Amazon Timestream or Prometheus.
- Tracks:
  - Number of lines processed per client.
  - Failures and errors.
  - Debts processed per injection run.
  - Rejected debt records.

---
This markup provides a structured overview of the Debt Processing Pipeline System, detailing components, processes, and operations in a clear format.
