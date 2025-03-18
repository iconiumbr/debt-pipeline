# Debt Processing Pipeline System

The Debt Processing Pipeline is a comprehensive system designed to process debt data submitted by clients through various methods. The pipeline handles data submission, storage, transformation, and final processing to ensure that the debt information is accurately recorded and tracked from start to finish, in an asynchronous way, allowing the system operator to scale up the components of the system to scale according to the variable demand through the day.

## Reasoning

This company is a debt collection bureau.
Its clients are businesses that want to collect overdue debts from individuals that have not been paid voluntarily. The clients submit the debt collection information, which includes basic details about the debtor they wish to collect from, such as their name, CPF (individual taxpayer registry number), the amount owed, and the date the payment was missed. Once the debt is registered in the collection bureau's system, the debts will be protested, the debtor will be notified, and a discount or installment payment plan may be negotiated.

## Scope

This system description will not go through the phase of collecting the debt, but for the sake of limitation of this scope, will be restricted only to receiving, processing and storing debt data that the clients sent to the bureau. The complexity is due to the multitude of data and file formats that the clients can use, and how the system may react to smoothly and accurately ingest this data, from small to very big clients, that requires precision and swiftness.

## System Components

### Batch Service

The batch service is used to track and maintain the status of the input debt data requests that the clients send to the bureau.

### Input Services

Used as border services, those components are exposed to the client to collect, either passively or actively, the debt data. These services answer synchronously, taking input and storing the raw data for intermediate processing.

### File Picker Process

In case of files delivered to FTP, it will prompt an event in S3 notification to SQS. Then the file picker process will check the file, check client details and from there it will either break down a large file into multiple sub-batches or register a single batch for the intermediate process.

### Intermediate Process

This service does the processing of the raw data, gathered by several sources in the input phase, does ETL work, prepares the data and injects it into a DynamoDB that will be used for the registering process.

### Registering Process

Takes data from the DynamoDB temporary storage, processes this data according to several required validations and inputs the final debt data into the debt storage system.

### Message Broker

For this model, we use Amazon SNS for basic relaying of inter-process messaging, signaling to different components like the batch service that there is a new file to be processed by the intermediate system, or that the batch service needs to update the batch status.

### Monitoring

All processes emit time series metrics to AWS Timestream for further analysis.

## Batch Processing Service

The Batch Processing Service is responsible for managing the status of client submissions and tracking their progress through the various stages of the debt processing pipeline. This service ensures that data is correctly handled, processed, and stored in the system.

### Key Responsibilities of the Batch Processing Service

#### Batch Creation

- Invoked by a sync call from the input stages, contains basic batch processing info including status, file location, client identification, batch number, number of debts received, processed and failed.
- Once a batch is created, a message is dispatched to the intermediate process queue.

#### Tracking and Status Updates

- The Batch Processing Service maintains the status of the batch throughout the entire pipeline.

#### Completion of Batch

- Once the registering process is completed, the batch status is finished and updated back to the batch.
- The batch will dispatch messages to the destination that needs to be notified about the results, either through SES for email or Event Bridge for further webhook or other methods.

#### Failure Notification

- In case of failure during processing of the input, such as bad format, once the message is received by the service, it will dispatch to the client through SES or Event Bridge.

## Input Phase

The Input Phase is responsible for receiving, storing, and registering incoming data from clients in different formats. Clients can submit data via file uploads or through individual REST insert requests. Once the data is submitted, it undergoes processing to transform it into a format suitable for further steps in the debt processing pipeline.

### 1 - Data Submission via REST API using Default Format

**Target:**
- For clients willing to use the default format either big or small clients.

**Requires:**
- Data to be in a defined format.

**Operation:**
- Client will input several debt lines via REST calls.
- Data is recorded in a temporary NoSQL DB.
- Upon finish, client will invoke submit request. Then data is written down to S3 CSV default format.

**Next Step:**
- Message is sent to batch service to create batch and further send message to the intermediate process queue.

### 2 - Data Submission via REST API for CSV using Default Format

**Target:**
- For clients willing to use the default format either big or small clients.

**Requires:**
- Data to be in a defined format.

**Operation:**
- Client will input a single CSV using a REST call.
- CSV is stored in S3.

**Next Step:**
- Message is sent to batch service to create batch and further send message to the intermediate process queue.

## Processing Phase

### 1 - File Picker Service

**Input:**
- The file picker monitors the SQS file picker queue.

**Operation:**
- Runs in parallel with multiple replicas.
- Once a message comes through, it will locate the file in S3.
- Assess who is the client, and if the file is not corrupted.
- Check if the file is big enough that demands it to be broken or maintain a single batch.
- If needed, break down smaller files and write down to S3.
- Identify the client processing type (default format, custom format by code).

**Next Step:**
- Message is sent to batch service to create batch and further send message to the intermediate process queue, as a single or multiple sub-batches.

### 2 - Intermediate Service

**Input:**
- The file picker monitors the SQS intermediate queue.

**Operation:**
- Runs in parallel with multiple replicas.
- Assesses the file exists and is correct.
- Checks from the Queue the file format to be processed.
- May need to fetch rules from external service to know how to process the file.
- Process the data into DynamoDB format expected by the registering process.

**Next Step:**
- Send message to the Batch service SQS queue reporting changes in the batch.
- If successful, send message to the registering process SQS queue to process the final request.
- If unsuccessful, send message to the batch service SQS queue informing of the failure on the file processing (format, corruption, etc.).

## Metrics Collection

Each service logs relevant metrics to a Time Series Database. This database provides a historical record of system activity, allowing for the monitoring and analysis of system performance.

Metrics tracked include:
- Number of lines processed per client.
- Number of failures.
- Number of debts processed during each injection run.
- Number of rejections.
