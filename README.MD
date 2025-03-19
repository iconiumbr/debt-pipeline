[Versão em Português (pt-BR)](ptbr.MD)
[Diagram SVG online](https://www.mermaidchart.com/raw/4182fac9-565c-4374-b955-4b713c8e491d?theme=light&version=v0.1&format=svg)
[Diagram SVG](graph.svg)

# Debt Processing Pipeline System

The Debt Processing Pipeline is a comprehensive system designed to process debt data submitted by clients through various methods. The pipeline handles data submission, storage, transformation, and final processing to ensure that the debt information is accurately recorded and tracked from start to finish, in an asynchronous way, allowing the system operator to scale up the components of the system to scale according to the variable demand through the day.

## Reasoning

This company is a debt collection bureau.  
Its clients are businesses that want to collect overdue debts from individuals that have not been paid voluntarily. The clients submit the debt collection information, which includes basic details about the debtor they wish to collect from, such as their name, CPF (individual taxpayer registry number), the amount owed, and the date the payment was missed. Once the debt is registered in the collection bureau's system, the debts will be protested, the debtor will be notified, and a discount or installment payment plan may be negotiated.

## Scope

This system description will not go thru the phase of collecting the debt, but for the sake of limitation of this scope, will be restricted only to receiving, processing and storing debt data that the clients sent to the bureau. The complexity is due the multitude of data and file formats that the clients can use, and how the system may react to smoothly and accurately ingest this data, from small to very big clients, that requires precision and swiftness.

## System Components

### a. Batch Service

The batch service is used to track and maintain the status of the input debt data requests that the clients send to the bureau.

### b. Input Services

Used as border services, those components are exposed to the client to collect, either passively or actively the debt data. These services answer synchronously, taking input and storing the raw data for intermediate processing.

### c. File Picker Process

In case of files delivered to FTP, it will prompt an event in S3 notification to SQS. Then the file picker process will check the file, check client details and from there it will either break down a large file into multiple sub-batches or register a single batch for the intermediate process.

### d. Intermediate Process

This service does the processing of the raw data, gathered by several sources in the input phase, does ETL work, prepare the data and inject it into a DynamoDB that will be used for the registering process.

### e. Registering Process

Takes data from the DynamoDB temporary storage, process this data according to several required validations and input the final debt data into the debt storage system.

### f. Message Broker

For this model, we use Amazon SNS for basic relying of inter-process messaging, signaling to different components like the batch service that there is a new file to be processed by the intermediate system, or that the batch service needs to update the batch status.

### g. Monitoring

All processes emit time series metrics to AWS Timestream for further analysis.

## Batch Processing Service

The Batch Processing Service is responsible for managing the status of client submissions and tracking their progress through the various stages of the debt processing pipeline. This service ensures that data is correctly handled, processed, and stored in the system.

### Key Responsibilities of the Batch Processing Service

#### Batch Creation

- Invoked by a sync call from the input stages, contains basic batch processing info including status, file location, client identification, batch number, number of debts received, processed and failed.
- Once a batch is created, a message is dispatched to the intermediate process queue.

#### Tracking and Status Updates

The Batch Processing Service maintains the status of the batch throughout the entire pipeline.

#### Completion of Batch

- Once the registering process is completed, the batch status is finished and updated back to the batch.
- The batch will dispatch messages to the destination that needs to be notified about the results, either thru SES for email or EventBridge for further webhook or other methods.

#### Failure Notification

- In case of failure during processing of the input, such as bad format, once the message is received by the service, it will dispatch to the client thru SES or EventBridge.

## Input Phase

The Input Phase is responsible for receiving, storing, and registering incoming data from clients in different formats. Clients can submit data via file uploads or through individual REST insert requests. Once the data is submitted, it undergoes processing to transform it into a format suitable for further steps in the debt processing pipeline.

### 1 - Data Submission via REST API using default format

#### Target

- For clients willing to use the default format either big or small clients.
- Requires data to be in a defined format.

#### Operation

- Client will input several debt lines via REST calls.
- Data is recorded in a temporary NoSQL DB.
- Upon finish, client will invoke submit request. Then data is written down to S3 CSV default format.

#### Next Step

- Message is sent to batch service to create batch and further send message to the intermediate process queue.

### 2 - Data Submission via REST API for CSV using default format

#### Target

- For clients willing to use the default format either big or small clients.
- Requires data to be in a defined format.

#### Operation

- Client will input a single CSV using a REST call.
- CSV is stored in S3.

#### Next Step

- Message is sent to batch service to create batch and further send message to the intermediate process queue.

### 3 - File Submission via FTP

#### Target

- For clients that send variable-sized files.
- Can be any format, defined by the agreement with the client, either default format or custom.
- Clients must have access to the FTP store.

#### Operation

- Client will input default format CSV files to S3 thru FTP.

#### Next Step

- S3 event notification dispatches message to SQS queue to the file picker service.

### 4 - Self-service small customized format

#### Target

- For small clients that have low-sized debt count.
- Requires a file (CSV, TXT, XLS).

#### Operation

- Client will use a website that allows the submit of the file.
- First 100 columns will be read, and the user is presented in the browser with the option to match required columns to their file.
- Once it's agreed, submit.
- File is recorded in S3.

#### Next Step

- Message is sent to batch service to create batch and further send message to the intermediate process queue. Includes basic information about the mapping.

## Processing Phase

The processing phase processes the input into the final destination.

### 1 - File Picker Service

#### Input

- The file picker monitors the SQS file picker queue.

#### Operation

- Runs in parallel with multiple replicas.
- Once a message comes thru, it will locate the file in S3.
- Assess who is the client, and if the file is not corrupted.
- Check if the file is big enough that demands it to be broken or maintain a single batch.
- If needed, break down smaller files and write down to S3.
- Identify the client processing type (default format, custom format by code).

#### Next Step

- Message is sent to batch service to create batch and further send message to the intermediate process queue, as a single or multiple sub-batches.

### 2 - Intermediate Service

#### Input

- The file picker monitors the SQS intermediate queue.

#### Operation

- Runs in parallel with multiple replicas.
- Assess the file exists and is correct.
- Checks from the Queue the file format to be processed.
- May need to fetch rules from an external service to know how to process the file.
- Process the data into DynamoDB format expected by the registering process.
  - The method can be discussed in a multitude of options including:
    - Custom code -- Python, Go, etc.
    - AWS Glue + AWS ETL.
    - EMR Hadoop? Is there such large data?

#### Next Step

- Send message to the Batch service SQS queue reporting changes in the batch.
- If successful, send message to the registering process SQS queue to process the final request.
- If Unsuccessful, send message to the batch service SQS queue informing of the failure on the file processing (format, corruption, etc.).

### 3 - Registering Service

*We don't know exactly what is the final destination, so this is just an assumption that the final store is just a black box that contains data and expects a specific format, like a service.*

#### Input

- The register service monitors the SQS register queue.

#### Operation

- Runs in parallel with multiple replicas.
- Get each message to process a batch.
- Will assess data from DynamoDB, perform validation and input values.
- Record eventual inconsistencies or failed debts and record to S3 for return.

#### Next Step

- Send message to the Batch service SQS queue reporting total processing numbers and the finished result, either success or fail.

## Metrics Collection

- Each service (Intermediate, picker, registering, batch) logs relevant metrics to a Time Series Database. This database provides a historical record of system activity, allowing for the monitoring and analysis of system performance.
- Amazon Timestream is used to store time-based metrics for all services in the pipeline. Alternatively, unmanaged services like Prometheus can be used.
- Metrics tracked include the number of lines processed per client, the number of failures, the number of debts processed during each injection run, and the number of rejections.
