# Distributed Review Processing System on AWS

A scalable distributed system for processing large collections of Amazon product reviews using AWS cloud services.

The system automatically distributes review analysis across multiple Worker instances, performs **Sentiment Analysis** and **Named Entity Recognition (NER)**, detects sarcasm, aggregates the results, and generates HTML reports for the user.

---

# System Architecture

```
                    +------------------+
                    |    LocalApp      |
                    +---------+--------+
                              |
                     Upload Input Files
                              |
                              v
                         Amazon S3
                              |
                              |
                     APP_TO_MANAGER
                          (SQS)
                              |
                              v
                    +------------------+
                    |     Manager      |
                    +------------------+
                     /        |        \
                    /         |         \
              Worker      Worker      Worker
                    \         |         /
                     \        |        /
                  WORKER_TO_MANAGER
                          (SQS)
                              |
                              v
                     Aggregate Results
                              |
                              v
                         Amazon S3
                              |
                              v
                        HTML Reports
```

---

# Project Overview

The project implements a distributed review-processing pipeline that automatically scales according to the workload.

Each review is analyzed independently using Natural Language Processing (NLP), allowing the system to process thousands of reviews in parallel.

The Manager coordinates the execution while Worker instances perform the computational tasks.

---

# Features

- Automatic Manager creation
- Dynamic Worker allocation
- Distributed task scheduling
- Asynchronous communication using Amazon SQS
- File storage using Amazon S3
- Sentiment Analysis
- Named Entity Recognition (NER)
- Sarcasm detection
- Automatic HTML report generation
- Fault-tolerant distributed execution

---

# Workflow

## Step 1 – Local Application

The Local Application is the entry point of the system.

It performs the following actions:

- Starts a Manager instance if one is not already running.
- Creates a unique S3 bucket.
- Uploads the input review files.
- Sends a processing request to the Manager through Amazon SQS.
- Waits until processing is complete.
- Downloads the generated output files.
- Converts the output into HTML reports.
- Cleans all temporary AWS resources.

---

## Step 2 – Manager

The Manager coordinates the entire distributed system.

It creates three SQS queues:

- APP_TO_MANAGER
- MANAGER_TO_WORKER
- WORKER_TO_MANAGER

Its responsibilities include:

- Receiving requests from Local Applications.
- Downloading input files from S3.
- Splitting reviews into batches.
- Dynamically launching Worker instances.
- Sending review batches to Workers.
- Collecting processed results.
- Uploading final outputs back to S3.
- Notifying the Local Application when processing is complete.

---

## Step 3 – Worker

Each Worker continuously listens for incoming tasks.

For every received review batch it performs:

- Sentiment Analysis
- Named Entity Recognition (NER)
- Sarcasm detection
- Result formatting

Finally, the Worker sends the processed results back to the Manager through Amazon SQS.

---

## Step 4 – HTML Generation

After receiving the completion notification, the Local Application:

- Downloads the processed results.
- Generates HTML reports.
- Colors review links according to sentiment.
- Displays detected named entities.
- Indicates whether each review is sarcastic.

---

# Message Flow

```
LocalApp
    |
    | Processing Request
    v
APP_TO_MANAGER
    |
    v
Manager
    |
    | Review Batches
    v
MANAGER_TO_WORKER
    |
    v
Workers
    |
    | Processed Reviews
    v
WORKER_TO_MANAGER
    |
    v
Manager
    |
    | DONE
    v
LocalApp
```

---

# Main Components

| Component | Responsibility |
|-----------|----------------|
| **LocalApp** | Uploads input files, communicates with the Manager, downloads results, and generates HTML reports. |
| **Manager** | Coordinates the distributed execution, manages Workers, schedules tasks, and aggregates results. |
| **Worker** | Processes review batches using NLP techniques and returns the analysis. |
| **Review** | Formats processed reviews, colors links according to sentiment, and detects sarcasm. |

---

# Dynamic Worker Allocation

Instead of creating a fixed number of Worker instances, the Manager dynamically allocates EC2 Workers according to the current workload.

This approach provides:

- Better resource utilization
- Reduced cloud costs
- Higher throughput
- Efficient parallel processing

Idle Workers are automatically terminated when no longer required.

---

# Persistence & Fault Tolerance

The system is designed to tolerate Worker failures.

- Messages are removed from Amazon SQS only after successful processing.
- Each Worker receives an extended visibility timeout (`5 × n` minutes) to allow long-running jobs to complete.
- If a Worker fails before acknowledging a task, the message automatically becomes visible again and can be processed by another Worker.
- The Manager continuously evaluates the workload and launches replacement Workers whenever necessary.
- A configurable maximum number of Workers prevents exceeding AWS resource limits.

---

# Thread Management

The Manager is implemented using multiple concurrent threads.

Two main listener threads run continuously:

- Applications Listener
- Workers Listener

Each listener delegates work to a ThreadPool, allowing multiple applications, files, and review batches to be processed concurrently without blocking the Manager.

---

# Scalability

The system supports multiple Local Applications running simultaneously.

Each application receives:

- A unique S3 bucket
- A dedicated response queue
- Independent processing pipeline

The computational work is fully delegated to Worker instances, allowing the Manager to focus solely on orchestration.

---

# Graceful Termination

When a termination request is received:

1. The Manager stops accepting new applications.
2. Existing tasks continue until completion.
3. All Worker instances are terminated.
4. Temporary SQS queues are deleted.
5. The Manager terminates itself.

This guarantees that no in-progress work is lost.

---

# Security

AWS credentials are stored locally and are never embedded in the source code.

The project uses the AWS SDK credential provider, ensuring that sensitive information is managed securely and remains outside the repository.

---

# Usage

Run the Local Application:

```bash
java -jar LocalApp.jar <inputFile1> ... <inputFileN> <outputFile1> ... <outputFileN> <n> [terminate]
```

Where:

| Parameter | Description |
|----------|-------------|
| **inputFileX** | Input review file. |
| **outputFileX** | Output report file. |
| **n** | Number of reviews assigned to each Worker task. |
| **terminate** | Optional flag that gracefully terminates the distributed system after completing all pending tasks. |

---

# Example Output

Each processed review contains:

- Original review link
- Color-coded sentiment
- Detected named entities
- Sarcasm indication

Example:

```
Link:
https://amazon.com/review/...

Sentiment:
Positive

Entities:
Amazon, Kindle, Battery

Sarcasm:
NOT_SARCASTIC
```

*(You can add screenshots of the generated HTML reports here.)*

---

# Performance

Example execution times measured during development:

| Configuration | Execution Time |
|--------------|----------------|
| 1 Application, 5 files, n = 10 | ~15 min |
| 1 Application, 5 files, n = 30 | ~8 min |
| 2 Applications, 5 files each | ~17–20 min |
| 3 Applications, 5 files each | ~25 min |

---

# Contributors

- **Shira Weiss Edri**
- **Ben Bandarkar**

