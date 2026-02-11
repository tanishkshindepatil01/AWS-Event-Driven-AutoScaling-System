# AWS-Event-Driven-AutoScaling-System
A decoupled, event-driven cloud architecture on AWS designed for asynchronous text processing. Features auto-scaling EC2 workers, SQS buffering, and architectural optimization analysis.

# AWS Event-Driven Auto-Scaling System (WordFreq & ArtAI)

**Course:** EMATM0051: Large-Scale Data Engineering   
**Institution:** University of Bristol 

## 📄 Overview
This repository documents the design and implementation of two cloud architectures on AWS:
1.  **WordFreq:** An implemented event-driven application that calculates word frequencies from unstructured text files using a decoupled, auto-scaling architecture.
2.  **ArtAI:** A conceptual high-availability serverless architecture designed for AI image processing.

---

## 🏗️ Part 1: WordFreq Implementation
The WordFreq system illustrates a decoupled architecture explicitly designed for the asynchronous processing of text data.It utilizes the fan-out pattern to separate storage from processing via message buffers.

### Architecture Workflow
1.  **Ingestion:** Text files are uploaded to an "Uploading" S3 bucket.
2.  **Trigger:** An S3 Event Notification triggers a message to the `wordfreq-jobs` SQS queue.
3.  **Processing:** EC2 worker nodes (Go application) poll the queue, retrieve the file, and calculate word counts.
4.  **Storage:** Results are stored in a DynamoDB table.
5.  **Completion:** Workers send an acknowledgement to a results SQS queue.


### Infrastructure Configuration
* **Compute:** EC2 instances (Ubuntu) running a Go worker application.
* **Storage:** Two S3 buckets (Uploading and Data Processing).
* **Queues:** Amazon SQS (Jobs Queue and Results Queue).
* **Database:** Amazon DynamoDB for storing frequency results.
* **Automation:** A custom AMI (`wordfreq-worker-ami`) and Launch Template were created to bootstrap instances automatically.

### ⚙️ Auto-Scaling Strategy
The system uses an Auto Scaling Group (ASG) based on SQS Queue Depth (`ApproximateNumberOfMessagesVisible`) rather than CPU utilization, as the workload is I/O bound.

* **Scale Out Policy:** Adds 1 capacity unit when visible messages $\ge$ 10.
* **Scale In Policy:** Removes 1 capacity unit when visible messages $\le$ 0.
* **Optimization:** Switched from "Simple Scaling" to "Step Scaling" and reduced warm-up/cooldown times from 120s to 30s, improving scaling speed from 5 minutes to under 90 seconds.

### 📊 Performance Analysis
Load testing was conducted by processing a dataset of 120 text files.

| Instance Type | vCPU / RAM | Processing Time | Verdict |
| :--- | :--- | :--- | :--- |
| **t2.micro** | 1 vCPU / 1GB | 6 min 15 sec | **Slowest.** CPU credits depleted quickly due to burstable performance limitations. |
| **c5.large** | 2 vCPU / 4GB | 1 min 50 sec | **Fastest.** Compute optimized; processed files 3x faster than baseline. |
| **r5.large** | 2 vCPU / 16GB | 2 min 05 sec | **Inefficient.** RAM was wasted; slower and more expensive than c5.large. |

**Conclusion:** The `c5.large` instance type offers the best balance of performance and cost for this specific workload.

---

## ☁️ Part 2: ArtAI Architecture (Design Only)
The "Art AI" webservice is designed as a fully serverless, three-tier architecture.

* **Global Edge:** Uses Amazon CloudFront for delivery and AWS WAF for perimeter security.
* **Public API:** Managed via Amazon API Gateway and authenticated using Amazon Cognito.
* **Private Backend:**
    * Business logic runs on **Amazon ECS (Fargate)**.
    * AI Inference is handled by **Amazon SageMaker**.
    * Data is stored in **Amazon S3** with encryption using **AWS KMS**.
* **Security:** strict unidirectional flow where backend resources are in private subnets, accessible only via VPC Endpoints.

---

## 🚀 Future Roadmap & Optimization
Based on the AWS Well-Architected Framework, the following improvements are proposed:

1.  **Serverless Migration:** Replace EC2 ASG with **AWS Lambda** triggered directly by S3 events to eliminate idle costs and reduce initialization time.
2. **Big Data Support:** Implement **Amazon EMR (Apache Spark)** to handle datasets scaling to 100TB, utilizing in-memory distributed computing.
3.  **Cost Optimization:** Implement **Spot Instances** for the worker nodes to reduce compute costs by up to 90%.
4.  **Data Lifecycle:** Use S3 Lifecycle Policies to move old data to **Glacier Deep Archive** for long-term retention.
