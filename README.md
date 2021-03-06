# Push Full-Stack Setup

This project assembles the necessary CloudFormation and Python scripts
needed to deploy a complete Push stack with or without log processing for the
Push Messages API component (needed for the Push Developer Dashboard).

## Components

At it's core, Push is composed of two programs, one that holds connections open
to clients, and one that handles incoming messages. This is to handle separate
scaling concerns for holding millions of client connections. The core system is
called ``autopush``.

Other systems, such as the Push Developer Dashboard and Push Messages API run
independently of ``autopush`` and function by processing log output. Running
them requires the logging and log processing components to be deployed. Logs
can be accumulated into S3 using Firehose even if later processing isn't
desired.

## Prerequisites

- Python 2.7
- AWS credentials

For log processing:

- Properly configured private subnet per
  https://github.com/mozilla-services/push-processor/#lambda-vpc-access

Create a virtualenv, activate it, and install the requirements:

    $ virtualenv myenv
    $ source myenv/bin/activate
    $ pip install -r requirements.txt

You are now ready for one-command AWS deployments!

### Push Service Only

Run:

    $ python deploy.py push

Will output a CloudFormation template to stdout. If you'd like to use it, then
save it to a file, and upload it in the AWS Console to Create Stack in AWS
CloudFormation:

    $ python deploy.py push > push_stack.cf

Then specify this file in the AWS CloudFormation UI during Create Stack.

### Complete Push Stack

- EC2
    - Autopush                  -> Firehose - Push Messages
    - Autoendpoint              -> Firehose - Push Messages
    - Push Messages API
        - Same VPC subnet as Elasticache

- VPC
    - Private VPC
        - Default route to NAT Gateway
        - Lambda Push Processor Runs Here

- Security Groups
    - PushConnectionNode
        - Inbound from any to 80
        - Inbound from PushEndpointNode to internal routing ports
        - Outbound to any
    - PushEndpointNode
        - Inbound from any to 80
        - Outbound to any
    - PushMessagesNode
        - Inbound from any to 80
        - Outbound to any
    - Elasticache
        - Inbound from PushMessagesNode to 6379

- Lambda
    - Push Processor
        - Same VPC subnet as Elasticache Redis
        - S3 Trigger off Push Messages

- DynamoDB
    - Autopush Tables
    - Push Messages Table

- Firehose
    - PushMessages              -> S3 - Push Messages

- S3 Buckets
    - Push Messages
    - Push Processor Settings
    - Push Processor Lambda Zips

- Elasticache
    - Redis

- IAM roles
    - Autopush
        - DynamoDB
            - Read/Write - Autopush Tables
            - Create Tables
        - Firehose
            - PutRecord/PutRecordBatch
    - PushProcessor
        - S3
            - Read/Write - Push Processor Settings
        - DynamoDB
            - Read/Write - Push Messages Table
        - AWSLambdaVPCAccessExecutionRole policy
    - PushMessagesFirehoseDelivery
        - S3
            - Read/Write - Push Messages Bucket
    - PushMessagesAPI
        - DynamoDB
            - Read/Write - Push Messages Table
