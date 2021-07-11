<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/s3-antivirus/blob/master/docs/instructions-overview.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

## S3 AntiVirus Overview

The S3 AntiVirus setup provides antivirus protection to all objects uploaded to your S3 Buckets. It works by:

1. Setting up an S3 Bucket Notification to listen to S3 Object creation events.
2. Sends the S3 Event payload to an SQS Queue.
3. The SQS Queue is processed by an AutoScaling Fleet of EC2 Worker instances

S3 objects are scanned and tagged with `scan-status=clean`.  Infected objects are deleted or can be tagged with `scan-status=infected`. By default, they are deleted.

You can receive notifications of infected files and the action taken by subscribing to the Alerts SNS Topic that is created by the [boltopspro/s3-antivirus](https://github.com/boltopspro-docs/s3-antivirus) blueprint.

## Overall Flow

Here's a diagram to explain the overall flow:

![](https://img.boltops.com/boltopspro/blueprints/s3-antivirus/s3-antivirus-overview.png)

## Repos

The S3 AntiVirus setup consists of a few repos. Here is an overview of the S3 AntiVirus Repos:

Repo | Description
--- | ---
[boltopspro/s3-antivirus](https://github.com/boltopspro-docs/s3-antivirus) | Blueprint deploys EC2 worker instances that perform the scanning.
[boltopspro/s3-antivirus-autoenable](https://github.com/boltopspro-docs/s3-antivirus-autoenable) | Blueprint deploys serverless resources so all new bucket automatically get s3-antivirus protection enabled instantly.
[boltopspro/s3-antivirus-cli](https://github.com/boltopspro-docs/s3-antivirus-cli) | CLI Tool to enable protection for existing buckets quickly.

## Instructions

We'll cover the "Single Scan Stack Setup" here. The general steps are:

1. Deploy [boltopspro/s3-antivirus](https://github.com/boltopspro-docs/s3-antivirus) in the AWS Account and region with the most used S3 buckets. This creates the EC2 instances and also the SQS queue.  The instances and SQS Queue will be able to scan all S3 Buckets associated with AWS account regardless of which region the S3 bucket resides in.
2. Deploy [boltopspro/s3-antivirus-autoenable](https://github.com/boltopspro-docs/s3-antivirus-autoenable) to each region in the AWS account. This enables automatic protection for new buckets created in that region.  This blueprint must be deployed on a regional basis because CloudWatch Events are regional.
3. Use the [boltopspro/s3-antivirus-cli](https://github.com/boltopspro-docs/s3-antivirus-cli) tool to enable protection for existing buckets. The tool supports a batch command that is useful to enable several buckets quickly.

Enabling the s3-antivirus protection with the CLI tool sets up an S3 Event Bucket Notification so that any objects uploaded to s3 will trigger a scan.  Specifically, this part of event chain is set up:

![](https://img.boltops.com/boltopspro/blueprints/s3-antivirus/s3-antivirus-cli.png)

The EC2 worker instances then poll for messages off of the queue with the s3 object info and scan the s3 file for viruses.

![](https://img.boltops.com/boltopspro/blueprints/s3-antivirus/s3-antivirus-blueprint.png)

## Testing Instructions

Refer to [Testing Instructions](/docs/instructions-testing.md).
