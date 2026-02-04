# AWS Data Orchestration Workshop

## Table of Contents
1. [ETL vs ELT Fundamentals](#etl-vs-elt-fundamentals)
2. [AWS Step Functions Overview](#aws-step-functions-overview)
3. [AWS Glue Jobs](#aws-glue-jobs)
4. [AWS Glue Crawlers](#aws-glue-crawlers)
5. [Data Lake Architecture: Raw/Staging/Analytics Layers](#data-lake-architecture)
6. [Basic Logging in CloudWatch](#basic-logging-in-cloudwatch)
7. [Amazon EventBridge](#amazon-eventbridge)
8. [Hands-on Lab](#hands-on-lab)

---

## ETL vs ELT Fundamentals

### ETL (Extract, Transform, Load)

ETL represents the traditional approach to data processing where data is **extracted** from source systems, **transformed** through various processing steps outside the target system, and then **loaded** into the final destination. This methodology requires transformation to happen before loading, which means you need a staging area for processing the data.

ETL works particularly well for structured data scenarios and complex transformations where you need precise control over the data processing logic. This approach has been the cornerstone of traditional data warehousing for decades, providing reliable and predictable data processing patterns.

### ELT (Extract, Load, Transform)

ELT takes a different approach by **extracting** data from source systems, **loading** the raw data directly into the target system first, and then **transforming** the data within the target system itself. This approach leverages the processing power of modern cloud-based systems and data lakes.

The key advantage of ELT is that raw data is preserved in its original form, allowing for more flexible transformations later. This approach has become increasingly popular with big data and cloud environments where storage is abundant and processing power can be scaled on demand.

### When to Use Each

ETL is most suitable when you have limited storage in your target system, need to implement complex transformation logic before loading, have strict data quality requirements, or must meet specific compliance and governance needs that require data validation before storage.

ELT shines in cloud-based data lake environments where you're processing big data, need flexible schema requirements, or want to take advantage of cost-effective storage solutions. The ability to store raw data and transform it later provides greater flexibility for evolving business requirements.

---

## AWS Step Functions Overview

AWS Step Functions provides a powerful way to orchestrate distributed applications and microservices using visual workflows. Think of it as a conductor for your cloud applications, coordinating the execution of multiple services in a specific sequence or pattern.

The service uses **state machines** defined in JSON format to represent your workflow logic. Each state machine consists of individual **states** that represent different steps in your process, connected by **transitions** that determine how the workflow moves from one step to another based on conditions or results.

Step Functions supports several types of states that give you flexibility in designing your workflows. **Task** states perform actual work by invoking services like Lambda functions or Glue jobs. **Choice** states implement branching logic based on conditions in your data. **Wait** states introduce delays for specified time periods. **Parallel** states allow you to execute multiple branches concurrently, while **Map** states let you iterate over arrays of items. Additional states like **Pass**, **Fail**, and **Succeed** help with testing, debugging, and controlling workflow termination.

### Example State Machine for Data Pipeline
```json
{
  "Comment": "Data Processing Pipeline",
  "StartAt": "CrawlRawData",
  "States": {
    "CrawlRawData": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:glue:startCrawler",
      "Parameters": {
        "Name": "raw-data-crawler"
      },
      "Next": "WaitForCrawler"
    },
    "WaitForCrawler": {
      "Type": "Wait",
      "Seconds": 30,
      "Next": "CheckCrawlerStatus"
    },
    "CheckCrawlerStatus": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:glue:getCrawler",
      "Parameters": {
        "Name": "raw-data-crawler"
      },
      "Next": "IsCrawlerComplete"
    },
    "IsCrawlerComplete": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.Crawler.State",
          "StringEquals": "READY",
          "Next": "ProcessData"
        }
      ],
      "Default": "WaitForCrawler"
    },
    "ProcessData": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "raw-to-staging-etl"
      },
      "End": true
    }
  }
}
```


---

## AWS Glue Jobs

AWS Glue provides serverless ETL capabilities that eliminate the need to provision and manage infrastructure for data transformation tasks. The service automatically scales resources based on your workload requirements, making it ideal for both small and large-scale data processing operations.

Glue supports three main job types to accommodate different processing needs. **Spark Jobs** leverage Apache Spark for large-scale data processing and complex transformations across distributed datasets. **Python Shell Jobs** are designed for lightweight Python scripts that don't require the full Spark framework. **Ray Jobs** cater specifically to machine learning workloads that benefit from Ray's distributed computing capabilities.

The service integrates seamlessly with the AWS Glue Data Catalog, providing automatic schema discovery and metadata management. Job bookmarks help track processed data to avoid reprocessing the same records, while auto-scaling ensures optimal resource utilization based on your data volume and processing requirements.

### Example Glue ETL Script
```python
import boto3
import pandas as pd
from awsglue.utils import getResolvedOptions
import sys

# Get job parameters
args = getResolvedOptions(sys.argv, ['JOB_NAME', 'INPUT_PATH', 'OUTPUT_PATH'])

# Initialize AWS clients
s3 = boto3.client('s3')
glue = boto3.client('glue')

def process_customer_data():
    # Read CSV from S3
    input_path = args['INPUT_PATH']
    df = pd.read_csv(input_path)
    
    # Data transformations
    # 1. Clean and standardize data
    df['email'] = df['email'].str.lower().str.strip()
    df['name'] = df['name'].str.title()
    
    # 2. Add processing timestamp
    df['processed_date'] = pd.Timestamp.now()
    
    # 3. Filter active customers only
    df = df[df['status'] == 'active']
    
    # 4. Remove duplicates
    df = df.drop_duplicates(subset=['customer_id'])
    
    # Write to S3 as Parquet
    output_path = args['OUTPUT_PATH']
    df.to_parquet(output_path, index=False)
    
    print(f"Processed {len(df)} customer records")
    return len(df)

if __name__ == "__main__":
    record_count = process_customer_data()
    print(f"Job completed successfully. Processed {record_count} records.")
```

### Best Practices
- Partition data for better performance
- Use appropriate worker types and counts
- Enable CloudWatch logging for debugging

---

## AWS Glue Crawlers

Crawlers serve as the automated discovery mechanism for your data infrastructure, eliminating the manual effort required to catalog and maintain metadata about your datasets. These intelligent services connect to your specified data stores, systematically scan through your data to determine its structure and schema, then create or update corresponding tables in the AWS Glue Data Catalog.

The crawler process is remarkably sophisticated in handling schema evolution. As your data structure changes over time, crawlers automatically detect these modifications and update the catalog accordingly. This ensures that your metadata remains current without requiring manual intervention, which is particularly valuable in dynamic data environments where schemas frequently evolve.

When configuring a crawler, you specify several key parameters that control its behavior. The data stores define where the crawler should look for data, supporting various sources including S3, RDS, and DynamoDB. An IAM role provides the necessary permissions to access your data sources securely. You can set up schedules to run crawlers automatically at specified intervals, and designate an output database where the discovered metadata will be stored.

### Best Practices
- Use exclusion patterns to avoid unwanted files
- Set appropriate schedule based on data arrival
- Use partitioning for large datasets

---

## Data Lake Architecture

### Three-Layer Architecture

#### Raw Layer (Bronze)
**Purpose**: Store data in its original, unprocessed format
- **Location**: `s3://bucket/raw/`
- **Format**: Original format (JSON, CSV, Avro, etc.)
- **Schema**: No schema enforcement
- **Partitioning**: By ingestion date and source system

```
s3://data-lake/raw/
├── sales/
│   └── year=2024/month=01/day=15/
│       ├── sales_001.json
│       └── sales_002.json
├── customers/
│   └── year=2024/month=01/day=15/
│       └── customers.csv
└── products/
    └── year=2024/month=01/day=15/
        └── products.parquet
```

#### Staging Layer (Silver)
**Purpose**: Cleaned, validated, and standardized data
- **Location**: `s3://bucket/staging/`
- **Format**: Parquet (optimized for analytics)
- **Schema**: Enforced and documented
- **Partitioning**: Optimized for query patterns

```
s3://data-lake/staging/
├── sales/
│   └── year=2024/month=01/
│       └── sales.parquet
├── customers/
│   └── year=2024/month=01/
│       └── customers.parquet
└── products/
    └── year=2024/month=01/
        └── products.parquet
```

#### Analytics Layer (Gold)
**Purpose**: Business-ready, aggregated data for reporting
- **Location**: `s3://bucket/analytics/`
- **Format**: Parquet with optimized schema
- **Schema**: Star/snowflake schema design
- **Partitioning**: By business dimensions

```
s3://data-lake/analytics/
├── sales_summary/
│   └── year=2024/month=01/
│       └── monthly_sales.parquet
├── customer_metrics/
│   └── year=2024/month=01/
│       └── customer_kpis.parquet
└── product_performance/
    └── year=2024/month=01/
        └── product_metrics.parquet
```

### Data Flow Example
```
Raw Data → Crawler → Data Catalog → Glue Job → Staging → Glue Job → Analytics
```

---

## Basic Logging in CloudWatch

Amazon CloudWatch serves as AWS's centralized monitoring and logging service, automatically collecting and storing log data from your orchestration services. For data pipelines, CloudWatch provides visibility into job execution, error tracking, and performance monitoring without requiring additional configuration.

When you run AWS Glue jobs, CloudWatch automatically creates log groups to capture different types of information. Output logs contain general job execution details and custom log messages you write in your code, while error logs specifically capture exceptions and failures that occur during processing. This automatic logging helps you troubleshoot issues and monitor job performance over time.

### Example CloudWatch Log from Glue Job
```
2024-01-15 14:30:15,123 INFO [main] Starting customer data processing job
2024-01-15 14:30:16,456 INFO [main] Loaded 15,432 records from raw_database.customers
2024-01-15 14:30:18,789 INFO [main] Applied data transformations successfully
2024-01-15 14:30:19,012 INFO [main] Removed 23 duplicate records
2024-01-15 14:30:21,345 INFO [main] Writing 15,409 records to staging layer
2024-01-15 14:30:23,678 INFO [main] Job completed successfully in 8.5 seconds
2024-01-15 14:30:23,901 INFO [main] Output written to s3://bucket/staging/customers/
```

CloudWatch automatically stores these logs in `/aws/glue/jobs/output` and `/aws/glue/jobs/error` log groups, making them searchable and accessible through the AWS console or CLI.

---

## Amazon EventBridge

Amazon EventBridge serves as a serverless event bus that enables event-driven orchestration by connecting different AWS services and applications. In the context of data orchestration, EventBridge acts as the trigger mechanism that initiates workflows based on specific events or schedules.

EventBridge excels at scheduling workflows through two primary mechanisms. First, it can respond to real-time events such as file uploads to S3, database changes, or custom application events, automatically triggering Step Functions state machines or other processing services. Second, it supports scheduled events using cron expressions, allowing you to run data pipelines at specific times or intervals.

For data orchestration workflows, EventBridge typically receives events when new data arrives (like S3 object creation events) and routes these events to Step Functions state machines that coordinate the subsequent processing steps. This creates a fully automated pipeline where data processing begins immediately when new data becomes available, without requiring manual intervention or constant polling.

### Simple Scheduling Example
```json
{
  "Rules": [
    {
      "Name": "DailyDataProcessing",
      "ScheduleExpression": "cron(0 2 * * ? *)",
      "Targets": [
        {
          "Id": "1",
          "Arn": "arn:aws:states:region:account:stateMachine:DataPipeline"
        }
      ]
    }
  ]
}
```

---

## Hands-on Lab

### Lab Objective
Build an event-driven data orchestration pipeline that processes customer data through all three layers.



### Lab Steps

# AWS Orchestration Workshop - Hands-on Lab

## Workshop Objective

Build a complete data orchestration pipeline using AWS services that processes data through multiple stages and makes it available for analysis in Athena.

## Architecture Requirements

Your pipeline must implement the following flow:

![Architecture Diagram](image.png)

## Tasks

### Task 1: Data Ingestion Setup
- Create **only one S3 bucket** with appropriate folder structure for data layers (raw/, processed/)
- Build a Glue job that retrieves data from any source of your choice (CSV file, API, sample data generation, etc.)
- Store the raw data in S3 in **Parquet format** in the **raw/** folder
- The data should contain at least 5 columns and 100+ records

### Task 2: Raw Data Cataloging
- Create a Glue crawler that discovers the raw data schema
- Configure the crawler to create a table in a database called `workshop_raw`
- Ensure the table is queryable in Athena

### Task 3: Data Transformation
- Build a second Glue job that reads the raw data
- Apply at least 3 different transformations (filtering, aggregation, data type conversion, etc.)
- Write the transformed data to S3 in **Parquet format** in a different location

### Task 4: Processed Data Cataloging
- Create another Glue crawler for the processed data
- Configure it to create a table in a database called `workshop_processed`
- Verify the processed table is available in Athena

### Task 5: Athena Query Validation
- Add an Athena query step that runs after the crawlers complete
- Execute a simple query like `SELECT COUNT(*) FROM workshop_processed.your_table`
- Verify the query returns expected results to confirm the pipeline worked correctly

### Task 6: Pipeline Orchestration
- Create a Step Functions state machine that orchestrates the entire pipeline
- The workflow must execute the jobs and crawlers in the correct sequence
- Include proper error handling for failed jobs
- Add wait states to handle crawler completion timing

### Task 7: Testing and Validation
- Execute the complete pipeline through Step Functions
- Verify both raw and processed tables are created in Athena
- Run sample queries on both tables to confirm data integrity
- Check CloudWatch logs for any errors or warnings

### Task 8: Automated Triggering
- Configure EventBridge to automatically trigger your Step Functions pipeline on a schedule or S3 events

## Optional Tasks

### Optional Task 1: Orchestration Monitoring
- Investigate how to monitor Step Functions execution steps (DynamoDB status tables, custom logging, etc.)

### Optional Task 2: Pipeline Notifications  
- Design a notification system for pipeline success/failure events using AWS services

### Optional Task 3: Cost Optimization
- Research how to estimate and optimize costs for your Glue jobs and Step Functions executions

### Optional Task 4: Security Enhancement
- Investigate how to implement least privilege access for your pipeline components using IAM roles

## Success Criteria

Your workshop is complete when:
- [ ] Step Functions successfully orchestrates the entire pipeline
- [ ] Raw data is stored in S3 as Parquet files
- [ ] Processed data is stored in S3 as Parquet files  
- [ ] Both datasets are queryable in Athena through Glue Data Catalog
- [ ] Pipeline handles errors gracefully
- [ ] All components are properly logged in CloudWatch

## Constraints

- All data storage must be in **Parquet format**
- Use only the services covered in the workshop (Step Functions, Glue Jobs, Glue Crawlers, S3, Athena)
- Pipeline must be fully automated through Step Functions
- Include appropriate IAM roles and permissions
- Use region: N. Virginia (us-east-1)

## Resources

- Use the workshop documentation as reference
- AWS Glue Developer Guide
- Step Functions Developer Guide
- **Remember: The goal is to learn by doing, NOT COPY SOLUTIONS!**


