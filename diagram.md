---
config:
  layout: fixed
---
flowchart TD
    A["Client Submission"] -- REST API --> B1["Default Format REST"] & B2["CSV REST Upload"]
    A -- FTP Upload --> B3["File Submission via FTP"]
    A -- Web Portal --> B4["Custom Format Upload"]
    B1 -- Store in NoSQL --> C1["Temp Storage"]
    B2 -- Store in S3 --> C2["S3 Bucket"]
    B3 -- S3 Event Notification --> C3["SQS File Picker"]
    C1 -- Submit Request --> D["Batch Service"]
    C3 -- File Picker --> E["File Picker Service"]
    G["SQS Intermediate Queue"] -- Triggers processing --> H["Intermediate Service"]
    H -- ETL Processing --> I["DynamoDB Storage"]
    H -- Metrics Logging --> T["Time Series DB"]
    J["SQS Register Queue"] -- Process debts --> K["Registering Service"]
    K -- Validate & Store --> L["Debt Storage System"]
    K -- Metrics Logging --> T
    E -- Create N batches --> D
    D -- Metrics Logging --> T
    B3 -- Metrics Logging --> T
    K -- Update status --> D
    D -- Send Mail / webhook --> N["Notification System"]
    H -- Register Queue --> J
    D -- Register New Job --> G
    B4 -- Store in S3 --> C2
    B2 -- Create batch --> D
    B4 -- Create batch --> D
    L@{ shape: db}
