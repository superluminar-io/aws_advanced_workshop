# Lab 6: Message Queuing with SQS

## Overview

In this lab, you'll learn how to implement asynchronous message processing using Amazon Simple Queue Service (SQS). We'll build upon our CloudFront-enabled infrastructure from previous labs, adding message queuing capabilities to handle background tasks efficiently.

## Key Concepts

- **Message Queues**: Managed service for storing messages between application components
- **Dead Letter Queues**: Secondary queues for handling failed message processing
- **Message Visibility**: Controls how long messages are hidden during processing
- **Lambda Triggers**: Serverless compute that processes messages automatically
- **Batch Processing**: Efficient handling of multiple messages in a single invocation
- **Message Retention**: Configurable duration for storing messages in the queue
- **Access Control**: IAM-based permissions for message producers and consumers

## Architecture
We'll extend our Lab 5 architecture by adding:
1. SQS queue with dead-letter queue configuration
2. Lambda function for message processing
3. Update the ECS application to publish messages to the SQS queue
4. Updated ECS task role for message publishing

![Lab 6 Architecture](/images/lab_6_architecture.png)