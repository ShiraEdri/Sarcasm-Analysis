DSP course

Assignment 1: Sarcasm Analysis

Authors:
Ben Bandarkar 318468758
Shira Edri 211722764

Instance type:
ami- 00e95a9222311e8ed //  standard amazon linux ami
type- M4_LARGE

Implementation:
usage
>java -jar LocalApp.jar <inputFileName1> ... <inputFileNameN> <outputFileName1> ... <outputFileNameN> <n> [terminate]

where:
        for x \in {1, ... , N}:
            inputFileNameX: input file X's name.
                ** the files are text files containing lists of reviews. 
            outputFileNameX: output file X's name.
        n: is number of jobs per worker for the task.
        terminate: put 'terminate' string to terminate the program (optional).


Explanation:

LOCAL APP:

Local App starts a manager instance if one is not found.
Local App creates a unique bucket and uploads the input files to this bucket.
The Local App waiting for the Manager to open SQS queue "APP_TO_MANAGER".
Then, it sends a message to the Manager including all the necessary information about the task: "UNIQUE_KEY|||n|||FirstFileIndex|||LastFileIndex".
if terminate flag in on, then the Local App sends a termination message to the Manager.
The Local App waiting for a response from the Manager idicating the process is done and then downloads the output files from s3. 
finally, the Local App delete the unique SQS queue and the unique bucket.

MANAGER:

Once the manager starts, he creates 3 queue: 
    1. APP_TO_MANAGER 
    2. MANAGER_TO_WORKER
    3. WORKER_TO_MANAGER

The manager's work is preformed using 2 main threads, with 2 different tasks that runs in parallel:
    1. apps listener
    2. workers listener

The first one listens on incoming messages from different local applications throgh APP_TO_MANAGER queue. 
Once it receives a new message it:

it extracts the file location and downloads it. 
submits the message as a task to a threadPool. 
each thread in the threadPool breaking the job to files and submit the job to another threadPool.
each thread in the other threadPool breaking the job of each file to n reviews per single job.
Then send it to the workers throgh MANAGER_TO_WORKER queue and creates calculated amount of workers according to the amount of tasks.
If a terminate message is received, the terminateFlag will turn on in order to announce termination.

The second thread listens on the completed job from workers through WORKER_TO_MANAGER queue.
Once it receives a new message it:

submits the message as a task to a threadPool.
each thread in the threadPool sums up the jobs to match files and check whether the whole task is complete.
If the task is done, the thread is uploads the file to the match location, and sends to the local application a message indicating the process is done. 



WORKER:

The workers waiting for a job from the manager through MANAGER_TO_WORKER queue.
Once a worker receives a message: "BucketName+fileIndex$$n|||link_1 $$$ text_1 $$$ rating_1||| ... |||link_n $$$ text_n $$$ rating_n", it extracts all the information.
Then, the worker apply sentiment analysis on the reviews it got and detect whether it is sarcasm or not.
Then he send the parsed result to the manager through WORKER_TO_MANAGER queue.
Finally, remove the processed message from the MANAGER_TO_WORKER queue.



Runtime:
1. 1 app with terminate, 5 file(1577 KB), n = 10 : 15 min
2. 1 app, 5 files(1577 KB), n = 30 : 8 min
3. 2 apps, 1 with terminate , 5 files for each(1557 KB), n = 10 : 20 min
4. 2 apps 5 files for each, n_1 = 10, n_2 = 20 (with terminate) : 17 min
5. 3 apps, 1 with terminate, 5 files for each, n = 10 : 25 min




Security:

We securely stored the credentials in a designated folder on our local computer, ensuring that they remain inaccessible to unauthorized parties. 
By keeping the sensitive information offline, within our premises, we maintain strict control over who can access and retrieve these details.
Our system is designed to ensure that this critical data remains isolated and protected, as it is not included or accessible within the codebase.


Scalability:

We've designed the project for scalability, meaning it dynamically adjusts the number of worker computers based on workload. 
We assume the users select a reasonable n according to workload, and the system opens an appropriate number of workers to handle tasks efficiently without overloading. 
However, due to AWS limitations for students, we're constrained to running a maximum of 9 computers simultaneously.
While theoretically, multiple managers could handle a large client base, our project using a single manager according to the requirements. 
To enhance manager efficiency within its limits, we employ a ThreadPool where each task runs concurrently in separate threads. 
This strategy ensures flexibility without burdening the manager with intensive tasks.


Persistence:

If a worker node dies it's job will eventually timeout.The next time the manager polls the incoming task queue, it will re-evaluate the necessary-workers amount and create new worker instances.
When a termination message arrives in the incoming tasks queue, the manager updates it state. When all former tasks are done it terminates the workers, purges the queues and closes itself.
A maximum-workers amount was hard-coded to ensure the system doesn't exceed the maximum instances limit.
The manager doesn't do any of the parsing work. Each worker node's work is completely agnostic to the any other worker node's work.

When a worker node fails, messages are not deleted immediately. 
Deletion occurs only after a worker completes its task. 
We set a visibility timeout of 5n minutes to avoid interrupting ongoing work due to large inputs. 
If a worker fails to complete within this time, the message becomes visible again for another worker to handle.


Threads:

As part of our scalability strategy, we integrated a custom ThreadPool-based implementation into the manager code. 
This optimizes the manager's performance in handling message exchanges with a large number of users.
Threads can be problematic if they complicate the code unnecessarily.


Termination:
As soon as the manager receives a terminate message, he closes the APPTOMANAGER queue,
Then he waits for the two main threads (a thread listening to LOCALAPPS and a thread listening to WORKERS) to finish working.
Both threads are programmed to check if a TERMINATE message has been received, and if Yes, they first finish all their tasks and only then finish. 
After the two threads are finished, the manager eliminates all the WORKERS, closes the two remaining queues (MANAGERTOWORKER and WORKERTOMANAGER) 
and at the end of the process the manager turns itself off.

------start here!!-----

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

